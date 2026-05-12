---
name: three-jar-split
agent: kid-money
kete: toro
description: >
  Splits every Kid Money payment into three jars — Spend, Save, Give — on
  the kid's side ledger. Defaults are set by the parent at signup. Older
  kids can request split changes (parent approves). When the Give jar
  reaches $5, it pays out to the kid's chosen NZ charity.
---

# Three-Jar Split

## What this skill will NOT do

- Will not move actual money between accounts. The kid's bank account holds the full amount; the three-jar split is a TŌRO-side ledger only, displayed to the kid as their Spend / Save / Give balances. This is intentional: NZ kids' accounts at the major banks do not support sub-accounts at the level we'd need, and creating four separate bank accounts per kid is friction that kills onboarding.

- Will not let a kid spend from the Save or Give jars directly. Spend is the only jar that maps to "this is your money to use right now". Save accumulates. Give pays out automatically at the $5 threshold to the chosen charity.

- Will not allow a kid under 12 to change their own split percentages. Kids aged 12 and over can request changes; the parent has to approve before any change takes effect.

- Will not pay out a Give jar to anything other than a registered NZ charity from a curated list at signup. We do not let kids redirect their Give jar to YouTube creators, in-app purchases, or any commercial entity.

## Tikanga check

The three jars are framed culturally, not just functionally:

- **Spend** = "yours now" — your everyday koha for being part of the whānau.
- **Save** = "future you" — kaitiakitanga, looking after the kid you'll be in a year or two.
- **Give** = "for others" — koha given to the wider community.

When the Give jar pays out, the kid gets a SMS that names the charity and frames the donation specifically as koha:

> "Your $5 koha just went to SPCA. Ka pai Tama. They'll send a thank you when it lands."

The Save jar quietly nudges older kids toward thinking about what they're saving for. "What's the Save jar for, Tama?" — kid can answer "new bike" and TŌRO stores it as a goal, showing progress over time.

## Privacy Act check

This skill stores:

- Per-kid jar balances (just three integer values in cents).
- The kid's chosen charity for Give payouts.
- Optional: the kid's stated Save goal as free text (max 80 chars, parent-visible).
- Payout history to the charity (date and amount, no kid name sent to the charity).

The kid's Save goal is the only piece of free text we collect from a kid in v1. We sanitise it (no email addresses, no phone numbers, no URLs) and store it. Parent can see it; nobody outside the whānau ever sees it.

Retention: 90 days after account close. Charity payout records are kept indefinitely as financial records.

## YAML frontmatter

Declares this skill as part of `kid-money`. Triggered when a payment from `nz-cdr-payments` succeeds, and when the kid views their balance.

---

## Default split percentages

| Age band | Spend | Save | Give |
|---|---|---|---|
| 5–7 | 70% | 20% | 10% |
| 8–11 | 60% | 30% | 10% |
| 12–14 | 50% | 40% | 10% |
| 15–17 | 50% | 40% | 10% |

The 10% Give floor is intentional across all ages. It's not user-configurable below 10% in v1. (If a parent really pushes back on this, we may make it configurable later, but the default of "always give something" is part of the brand.)

## The split flow

When `nz-cdr-payments` confirms a successful payment of $X for a kid, this skill:

1. Looks up the kid's current split percentages.
2. Calculates the three amounts, rounding to the cent. Any rounding remainder goes to Spend.
3. Writes three rows to `kid_ledger` (one per jar).
4. Sends the kid a SMS: "Got it Tama. $2.50 in your account — $1.50 Spend, $0.75 Save, $0.25 to SPCA. Ka pai."
5. Checks if the Give jar has crossed $5. If yes, triggers the Give payout flow.

## The Give payout flow

When a kid's Give jar reaches $5:

1. TŌRO notifies the parent: "Tama's Give jar just hit $5 — paying out to SPCA today."
2. TŌRO waits 24 hours for parent objection (a "hold this" button in the app).
3. If no objection, TŌRO initiates the payout. This is a real payment via the same `nz-cdr-payments` skill, parent's account → SPCA bank account. The kid's bank balance does not change (the kid's bank account is treated as the consolidated pot; the Give jar lives in our ledger).
4. Actually, second thought — this needs a separate flow. The parent has to fund the Give payout out of their account, OR the kid's bank account if the kid is old enough to have CDR-eligible direct debit set up. We need to pick one. Default in v1: the parent's account funds it, and we deduct $5 from the kid's "real bank balance" by sending a follow-up "you've given $5 koha to SPCA" message so the kid sees it as their money even though the parent's account technically moved it.

This is fiddly. Phase 3 we may revisit and use the kid's own account, once we have a clearer picture of how 12+ year olds can give CDR consent on their own banks.

## The Save goal nudge

For kids aged 10+, when the Save jar reaches certain milestones ($20, $50, $100, $250, $500), TŌRO sends:

> "Tama, your Save jar's at $50. Anything you're saving for?"

If the kid replies with text, we store it (sanitised) and show progress against it. "$50 of $200 saved for a new bike — 25% there 🚲". Visual progress is shown in the kid's web view (no app required).

## Charity list (NZ-curated)

At signup, the parent picks 1–3 charities from a curated list. The kid then picks one as their Give recipient (kids 8+) or the parent picks for younger kids. The curated list starts with:

- SPCA New Zealand
- KidsCan
- Forest & Bird
- Starship Foundation
- The Salvation Army NZ
- Ronald McDonald House Charities NZ
- Plunket
- Tearfund NZ

Expansion criteria: charity must be registered with Charities Services NZ, must have a bank account that accepts direct credits, must have a written agreement with TŌRO acknowledging that incoming koha will appear in their statement with a "TŌRO / [Whanau ID]" reference.

We do not take a cut. We do not take a referral fee. If a charity offers one, we decline. This is part of the brand and it is non-negotiable.

## Test cases the implementation must pass

1. $2.50 payment to a 10-year-old with default split. $1.50 lands in Spend ledger, $0.75 in Save, $0.25 in Give. The 1-cent rounding remainder goes to Spend (so it'd actually be $1.50 + $0.75 + $0.25 = $2.50 exactly here, but try $1.45 to test rounding: $0.87 + $0.435→$0.44 + $0.145→$0.14 = $1.45 with rounding handled).

2. Give jar accumulates to $5.10 over several payments. Payout flow triggers, $5 goes to SPCA, $0.10 stays in Give for next time.

3. Parent objects within the 24-hour window. Payout is held. $5 returns to the Give jar (still pending). Parent can either reject (money stays in Give for later) or approve (payout proceeds).

4. Kid age 13 requests a split change from default 50/40/10 to 70/20/10. Parent gets a request in the app, parent can approve / reject / counter-propose (e.g. 60/30/10). The change applies to future payments, not retroactively.

5. Whānau closes account. Save and Spend jar balances are paid out to the kid's bank account in one final transfer. Give jar is paid out to the chosen charity (rounded down to the nearest dollar) or held back to the parent if under $5. Records are retained per privacy retention policy.
