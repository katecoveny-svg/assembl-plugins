---
name: newsletter-parser
agent: term-planner
kete: toro
description: >
  Newsletter parsing layer for Term Planner. Handles inputs in many shapes
  (PDF, JPG/PNG screenshot, plain-text email body, EML attachment, Hero/
  Seesaw notification payload) and produces a normalised structured extract
  for downstream skills to use. Builds on the existing TŌRO parser work.
---

# Newsletter Parser

## What this skill will NOT do

- Will not infer information not present in the source comm. If a date isn't in the newsletter, we don't guess one based on "usually swimming sports is in February" — we surface the gap to the parent.

- Will not parse anything that doesn't look like a school comm. The pre-extraction check looks for school letterhead, school name on the page, or a recognised platform's notification format. Random documents get returned to the parent with "is this from a school? Add it as a source first."

- Will not run OCR on a photo without first checking the photo is sharp enough. If the image is too blurry, we ask the parent to retake it rather than producing garbage extracts.

- Will not extract beyond the first 20 pages of any single newsletter. Long-form documents get truncated and the parent is warned.

## Tikanga check

The parser is a backend skill that doesn't directly produce customer-facing copy. The tikanga check happens downstream when `term-planner-core` builds the parent-facing summary. The parser's job is faithful extraction; cultural framing happens at the summary layer.

When parsing detects Māori-language content in a newsletter (kapa haka programmes, te reo announcements, tikanga events), we tag those extracts with `cultural_significance: high` so the summary skill knows to surface them prominently rather than burying them in a list.

## Privacy Act check

- The original source comm is held for 30 days, then auto-deleted. Structured extracts persist.
- We never train on parent-uploaded school comms. Per-whānau extractions stay per-whānau.
- We do not aggregate across schools or share any single school's content with another whānau.

## YAML frontmatter

Declares this skill as part of `term-planner`. Triggered as the first step in any new-comm-arrives flow.

---

## Building on what's already started

This skill assumes you have an existing newsletter parser that handles text extraction. This SKILL.md formalises:

1. The **input router** — what kind of comm have we received, which parser do we call.
2. The **normalised extraction schema** — what shape we produce regardless of input format.
3. The **fallback flow** — what we do when extraction is partial or uncertain.

If the existing parser is text-only (PDF text-layer extraction), the gap to close is the vision layer for screenshots and photos of paper notices.

## Input router

```typescript
function routeIncomingComm(comm: IncomingComm): ParserKey {
  // Email arrived at term-<id>@toro.nz
  if (comm.source === 'email') {
    if (comm.has_pdf_attachment) return 'pdf';
    if (comm.body_html && looks_like_hero_notification(comm.body_html)) return 'hero';
    if (comm.body_html && looks_like_seesaw_notification(comm.body_html)) return 'seesaw';
    if (comm.body_html && looks_like_skoolloop(comm.body_html)) return 'skoolloop';
    return 'plain_text';
  }
  // Uploaded via TŌRO app
  if (comm.source === 'upload') {
    if (comm.mime_type === 'application/pdf') return 'pdf';
    if (comm.mime_type.startsWith('image/')) return 'vision';
    return 'unsupported';
  }
  // Screenshot pasted into chat
  if (comm.source === 'chat_image') return 'vision';
  return 'unsupported';
}
```

## The PDF parser

Uses `pdfplumber` for text extraction with layout preservation. School newsletters often have tables (event schedules, fees) that need structural extraction not just flat text.

Steps:
1. Extract all text with layout.
2. Extract all tables separately.
3. Run the LLM pass over the combined text + table data with a prompt that asks for structured ExtractedEvent[] output.
4. Validate each event against the schema (date parseable, amount numeric, etc).
5. Flag any extracts marked low-confidence by the LLM for parent review.

## The vision parser

For screenshots and photos:
1. Pre-check: image must be at least 800x600, contrast must be reasonable, must contain text. Reject blurry / dark images with a polite ask-again.
2. Look for school identifier: logo match against the schools registered to this whānau, or text containing the school name.
3. Vision LLM extracts the same ExtractedEvent[] structure.
4. Confidence threshold higher than PDF — we ask for confirmation on more borderline extracts.

## The Hero notification parser

Hero notifications have a relatively consistent HTML structure. Parse with selectors:
- `.notification-title` → event title or category
- `.notification-body` → main text, run through the LLM event extractor
- `.notification-meta` → date, time, school, class
- `.notification-attachments` → if a PDF is attached, route to PDF parser

If Hero changes its notification format, the LLM extractor still works on the body text — selectors are an optimisation, not a hard dependency.

## The Seesaw notification parser

Seesaw is photo-and-message-heavy. We extract:
- The message text (usually short)
- Any attached date or time
- Affected class / kid

Seesaw is rarely the source of formal school events; it's more "your kid did this today". We tag Seesaw extracts as `informational` and don't push them into the calendar unless they explicitly contain a future date.

## The plain-text email parser

When a parent forwards a school's email newsletter that's pure text or simple HTML body, the LLM is the parser. Prompt:

```
You are extracting events from a New Zealand school newsletter. Output
strictly an array of ExtractedEvent JSON objects per the schema in
schema.ts. If a date is ambiguous, set the date field to null and put
the ambiguous reference into the metadata field. If no events are
present, output an empty array. Never invent information not in the
source.
```

## The unsupported flow

If we can't parse the input at all (corrupt PDF, unrecognised format, image that fails the quality check), we return to the parent with:

> Couldn't read this one — can you try forwarding it as a PDF, or take a clearer photo and try again?

We do not invent extracts. We do not partially extract. All-or-flag.

## Test cases the implementation must pass

1. PDF of a 4-page newsletter with mixed text and a table of fees. Parser returns ExtractedEvent[] with calendar events from the text and payment events from the table.

2. Screenshot of a Hero notification (PNG, taken on iPhone). Vision parser extracts the same events that would come from the underlying Hero HTML.

3. Blurry photo of a paper notice. Quality pre-check fails. Parent gets the retake request within 5 seconds.

4. A newsletter with a date that says "Term 4 Week 3" but no explicit calendar date. Parser stores it as `{date: null, date_reference: "Term 4 Week 3"}`. The `nz-school-calendar` skill resolves to a real date downstream.

5. A non-school PDF (e.g. an electricity bill) uploaded to the school inbox. Pre-check rejects it. Parent gets: "I don't think this is from a school — want me to add it as a different kind of source?"
