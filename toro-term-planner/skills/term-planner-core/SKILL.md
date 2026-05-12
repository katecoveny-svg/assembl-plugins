---
name: term-planner-core
agent: term-planner
kete: toro
description: >
  Core skill for the Term Planner agent. Defines the structured-extraction
  schema, the parent-approval flow, the calendar/bill-pay/gear-list writes,
  and the weekly nudge format. Triggers whenever a parent forwards or
  uploads a school comm.
---

# Term Planner — Core

## What this skill will NOT do

- Will not extract or write anything until the parent has explicitly added a school as a source. Random PDF uploads from unknown sources are flagged for the parent to confirm: "is this from Maia's school? I'll add Hawke's Bay Primary as a source if you confirm."

- Will not auto-populate the parent's external calendar (Google, Apple). The parent always has to approve the batch. Term Planner can suggest, but the parent confirms each batch.

- Will not write payments to bill-pay without an explicit per-payment approval. We surface them, the parent approves each one or batches.

- Will not OCR a photo of a paper notice without first checking the image has a school logo or school letterhead. Random photos sent into the chat ("here's our calendar") get a clarifying question.

- Will not retain the raw newsletter PDF for longer than 30 days after extraction. The structured extracts persist (we need them for reminders); the original file is auto-deleted.

## Tikanga check

The framing centres whānau and kura. We're helping a household navigate the rhythm of school, not running a productivity app.

When a school newsletter mentions a Māori cultural event (kapa haka, Matariki, mihi whakatau, pōwhiri), Term Planner highlights it specifically and offers context if the family has indicated they'd find it useful at onboarding ("Maia's class is performing kapa haka on Thursday — they've been preparing for a few weeks, want a reminder to ask her about it?").

Banned terms in copy: "AI", "artificial intelligence", "bot", "machine learning". Use "Term Planner" or "the agent".

## Privacy Act check

Data this skill handles:

- School newsletter PDFs (auto-deleted after 30 days).
- Structured event extracts (calendar items, payment items, gear items, permission items) — kept until 90 days after account close, or until the event itself is more than 90 days in the past, whichever comes first.
- The school's name and the term being referenced (kept as long as the school is a registered source on the parent's account).

IPP 3A handling: each new school added as a source triggers a consent-refresh screen. The parent confirms "I understand TŌRO will process newsletters that mention my children". This screen is shown once per school per account, not on every forward.

No third-party sharing. School newsletters are not shared with any other TŌRO user, even from the same school. Each whānau sees their own extracts only.

## YAML frontmatter

Declares this skill as part of `term-planner`. Triggered by:
- Email arriving at the whānau-specific `term-<id>@toro.nz` address.
- File upload in the TŌRO app tagged "school".
- Screenshot uploaded in TŌRO chat (vision check confirms school letterhead).

---

## The extraction schema

Each parsed comm produces zero or more typed events. The TypeScript shape:

```typescript
type ExtractedEvent =
  | CalendarEvent
  | PaymentEvent
  | GearEvent
  | PermissionEvent
  | TransitionEvent;

interface CalendarEvent {
  type: 'calendar';
  title: string;
  date: string; // ISO 8601
  time?: string;
  duration_minutes?: number;
  location?: string;
  dress_code?: string;
  what_to_bring?: string[];
  affects_children: string[]; // kid IDs in this whānau
  source_comm_id: string;
}

interface PaymentEvent {
  type: 'payment';
  description: string;
  amount_nzd: number;
  due_date: string;
  payee_name: string;
  payee_account: string;
  reference: string;
  is_voluntary_donation: boolean; // affects tax treatment
  affects_children: string[];
  source_comm_id: string;
}

interface GearEvent {
  type: 'gear';
  item: string;
  needed_by: string;
  affects_children: string[];
  estimated_cost_nzd?: number;
  source_comm_id: string;
}

interface PermissionEvent {
  type: 'permission';
  description: string;
  due_date: string;
  affects_children: string[];
  parent_signature_required: boolean;
  source_comm_id: string;
}

interface TransitionEvent {
  type: 'transition';
  transition_type: 'term_start' | 'term_end' | 'holiday_start' | 'holiday_end' | 'closure';
  date: string;
  affects_children: string[];
  source_comm_id: string;
}
```

## The summary format (parent-facing)

When extraction completes, Term Planner sends one consolidated message:

```
Read the Term 2 newsletter for Hawke's Bay Primary. Found 14 things:

THIS WEEK
· Mufti day Fri — $2 koha (SPCA)
· Swimming togs for Maia Wed–Fri

NEXT WEEK
· Class trip permission slip due Mon
· $18 each for Maia and Tama → school account 12-3456-7890123-00

WEEK 4
· Cross-country — parent helpers wanted (want me to say yes for Adrian?)

WEEK 7
· Parent-teacher interviews — booking opens Mon 9am, I'll book you both

WEEK 10
· Term ends, school finishes 12:30
· OSCAR bookings open Fri this week

Add all to your calendar? Reply YES or tap approve.
```

This is the format that gets screenshotted. Keep it consistent.

## The Term 1 / Term 4 special cases

- **Term 1** has back-to-school: stationery lists, uniform, voluntary donations annual notice. We surface the donation notice with a clear "this is tax-deductible — I'll remember and add to your IRD receipts list at year-end".

- **Term 4** has the prizegiving / end-of-year shifts. The schedule changes weekly in the last fortnight of Term 4. We re-extract more aggressively in late November.

## The cross-kid clash detector

If two extracted events affect different kids and clash in time, Term Planner surfaces this immediately:

> Heads up — Maia has cross-country at 9am on the 18th and Tama has class trip departure also at 9am on the 18th. Both need a parent there. Want to talk through how to split it?

This is the kind of detection that makes parents tell other parents.

## The IRD year-end summary

End of March (NZ tax year-end), Term Planner produces a PDF for the parent: every voluntary donation paid during the tax year, school by school, total. Used for the parent's IR526 donation tax credit claim. This is a sticky annual touch point and an easy upsell moment ("Term Planner just saved you $42 in tax credits — your sub paid for itself").

## Test cases the implementation must pass

1. PDF of a 4-page Term 2 newsletter is forwarded to `term-abc123@toro.nz`. Within 60 seconds, the parent receives the structured summary message. The original PDF is stored, marked for auto-deletion in 30 days.

2. A photo of a paper notice (taken with a phone, slightly blurry) is uploaded to TŌRO chat. The skill's vision check confirms school letterhead, extracts what it can, asks the parent to confirm two ambiguous dates.

3. A newsletter mentions a payment of $18 with a school bank account, due in 10 days. The skill writes a Payment event but does not initiate the payment. The parent receives a reminder 7, 3, and 1 day before due.

4. Two kids in the whānau have events at the same time on the same date. The clash detector fires. Parent gets a specific clash-alert message within 5 seconds of the second event being added.

5. Whānau closes account. All school-source-linked extracts and the 30-day PDF cache are purged. IRD donation summary is retained for 7 years (tax records obligation) but anonymised — child names removed, donation amounts and school names kept.
