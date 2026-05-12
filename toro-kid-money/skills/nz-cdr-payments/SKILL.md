---
name: nz-cdr-payments
agent: kid-money
kete: toro
description: >
  Payment initiation under New Zealand's Consumer Data Right (CDR). Handles
  the parent CDR consent flow, bank-side authorisation, Confirmation of
  Payee checks, payment initiation via the major banks' open banking APIs,
  and the audit log of every payment with bank reference numbers. This is
  the skill that actually moves money from the parent's account into the
  kid's account.
---

# NZ CDR Payments

## What this skill will NOT do

- Will not initiate a payment without a valid, current, parent-side CDR consent token from the relevant bank. Consent expires every 365 days under the regulations; we refresh at day 330 so a payment never blocks because of an expired token.

- Will not initiate a payment to an account the parent has not explicitly registered as belonging to one of their kids. We do not let a parent type in an arbitrary account number on the fly — the kid's destination account is registered once at onboarding with a one-cent verification deposit.

- Will not initiate a payment without a Confirmation of Payee (CoP) check passing first. If the account name does not match the kid's name on the registered account, the payment is held and the parent is notified.

- Will not charge any fee for payment initiation in v1. The CDR regulations prohibit data holders charging customers or accredited requestors for the data transfer itself; we extend that posture to our own product and offer payment initiation at no extra cost to parents.

- Will not store the parent's bank login credentials, ever. Authentication happens on the bank's side via the OAuth-style flow defined in the CDR standards. We only ever receive consent tokens, never passwords.

- Will not log the payment until the bank has returned a successful execution confirmation. We do not optimistically display "paid" — we wait for the bank response, typically a few seconds.

## Tikanga check

Money is taonga. The framing on parent-facing messages should treat each payment as deliberate, not automated. Use "I'll send Tama's $2.50 now" not "the system is processing the transfer". On the kid side, the language is around acknowledgement and koha (see `kid-money-core`).

When a Give jar pays out to a charity, the message names the charity in full and frames the action as koha given by the kid, not a transaction processed by TŌRO.

## Privacy Act check

Data this skill collects:

- Parent's CDR consent token from each registered bank (encrypted at rest, rotated as part of consent refresh).
- Bank account numbers (parent's source account, kid's destination account) — collected once at onboarding, stored as account references not raw numbers.
- Every payment is logged with: parent ID, kid ID, amount, timestamp, bank reference, CoP result, success/failure.
- Bank reference numbers are kept indefinitely as financial records; the Privacy Act 2020's retention rules under IPP 9 require we hold these only as long as necessary, but financial reporting obligations under tax law override (we hold for seven years).

The parent is the primary subject for CDR purposes. The kid is the recipient. We notify the kid at signup that money sent to their account will appear with a "TŌRO chore payment" reference visible to anyone they share their bank statement with — kids might not realise this.

## YAML frontmatter

The frontmatter declares this skill as part of `kid-money`. It is the only skill in the Kid Money plugin that touches the regulated banking layer. Other skills (`kid-money-core`, `three-jar-split`) must call this skill rather than reaching into bank APIs directly.

---

## The accredited requestor wrapper

TŌRO is the accredited requestor under the CDR. Application details are in `application/cdr_accredited_requestor_checklist.md`. Once accredited, the bank APIs become available; before then, this skill operates in a fallback mode where it surfaces "transfer $X to Tama" prompts in the parent's app, the parent does the transfer themselves in their bank's app, and TŌRO records the payment as confirmed via a screenshot upload.

## Bank coverage at launch (Phase 2)

Designated data holders, in order of go-live:

| Bank | Payments live | Customer data live |
|---|---|---|
| ANZ | 1 Dec 2025 | 1 Dec 2025 (bank statements excluded until 1 Apr 2026) |
| ASB | 1 Dec 2025 | 1 Dec 2025 (some loan transaction history excluded until 1 Jun 2026) |
| BNZ | 1 Dec 2025 | 1 Dec 2025 |
| Westpac | 1 Dec 2025 | 1 Dec 2025 (some historic statements excluded until 1 Jan 2027) |
| Kiwibank | 1 Jun 2026 | 1 Dec 2026 |
| Other deposit takers | Opt-in from 1 Dec 2025 | Opt-in |

We target Phase 2 launch for week 10 of the build. Kiwibank's payments designation on 1 Jun 2026 lines up almost exactly with that, so Kiwibank households can pay from day one of the public Kid Money launch.

## The consent flow (parent side)

1. Parent picks their bank from a list in the TŌRO onboarding screen.

2. They're redirected to the bank's CDR consent page (this is a bank-hosted screen, not a TŌRO screen — required by the standards).

3. Bank asks: "Do you authorise TŌRO to initiate payments from your everyday transaction account, up to $200 per payment, until [date 365 days from now]?"

4. Parent confirms with their bank's normal authentication (usually fingerprint or two-factor).

5. Bank returns a consent token to TŌRO. We store it encrypted, linked to the parent's TŌRO account.

6. Parent is brought back into TŌRO, where they confirm the kid's destination account (account number + name) and complete a one-cent verification deposit to prove the account exists and they have the right details.

7. Done. Parent can now approve payments and they will go through.

## The payment flow

```
parent_approves_chore_payment(kid_id, amount_nzd, chore_id)
  → load parent's CDR consent token for the destination kid's bank
  → run Confirmation of Payee check (account name vs kid's registered name)
  → if CoP fails: hold payment, notify parent
  → if CoP passes: call bank's payment initiation API
  → wait for bank execution confirmation (typically 2-5 seconds)
  → on success: log to audit table, fire three-jar-split event
  → on failure: notify parent with bank's reason code, allow retry
```

## Audit log schema (Supabase)

```sql
create table cdr_payments (
  id uuid primary key default gen_random_uuid(),
  parent_id uuid not null references whanau_members(id),
  kid_id uuid not null references whanau_members(id),
  amount_cents int not null,
  source_account_ref text not null,
  destination_account_ref text not null,
  source_bank text not null,
  destination_bank text not null,
  cop_result text not null check (cop_result in ('match', 'partial', 'no_match', 'unsupported')),
  bank_reference text,
  status text not null check (status in ('pending', 'success', 'failed', 'held')),
  failure_reason text,
  initiated_at timestamptz not null default now(),
  confirmed_at timestamptz,
  chore_id uuid references chores(id),
  metadata jsonb
);

create index idx_cdr_payments_parent on cdr_payments(parent_id, initiated_at desc);
create index idx_cdr_payments_kid on cdr_payments(kid_id, initiated_at desc);
```

RLS: parent can read their own rows; kid never queries this table directly; the audit log is append-only (no updates, no deletes, except via the privacy-act retention sweep at account close).

## Confirmation of Payee — handling mismatches

If CoP returns "partial" (the name on the destination account is close but not exact — e.g. "Tama Hudson" registered, "T Hudson" on bank), we surface a one-tap "yes this is the right account" to the parent and proceed if they confirm. We do not auto-proceed on partials.

If CoP returns "no_match" (e.g. "Adrian Smith" on the destination), we hard-block the payment and surface a warning: "the destination account name doesn't match Tama's registered name. Check the account details with your bank before continuing." This is the kind of message that protects users from accidental misdirected payments and also from social engineering.

## Test cases the implementation must pass

1. Parent's CDR consent token is 359 days old. Next payment attempt triggers a silent refresh, parent reauthorises in 30 seconds, payment then proceeds.

2. CoP returns "no_match". Payment is held. Parent gets a notification within 5 seconds. No money has moved. No bank reference is logged.

3. Bank's payment initiation API returns a 5xx error. Payment status goes to "failed" with the bank's reason code captured. Parent is notified. Retry button works.

4. Whānau closes account. Privacy retention sweep runs 90 days later. Bank reference numbers and amounts are retained (tax law requirement, 7 years), but parent_id and kid_id are nulled to pseudonymise the records.

5. Parent attempts to send a payment over the $200 single-payment ceiling set in the consent. Payment is rejected with a clear message: "your CDR consent caps payments at $200 each — want me to raise the limit (you'll need to re-authorise at your bank)?"
