---
name: nz-school-calendar
agent: term-planner
kete: toro
description: >
  NZ school calendar knowledge layer. Term dates by year and school type,
  public holidays, the canonical list of NZ school rituals (mufti days,
  swimming sports, kapa haka, ERO visits, Pet Day, Matariki celebrations),
  NCEA milestones, and how to map "this week" or "week 4 of term 2" to
  real dates.
---

# NZ School Calendar

## What this skill will NOT do

- Will not invent term dates. Each year's official Ministry of Education term dates are loaded into a structured table; we never guess.

- Will not assume all schools follow the same dates. State, state-integrated, and private schools vary by a day or two. We use the Ministry's published default but allow per-school overrides set by the parent at onboarding.

- Will not assume an event is on a specific date if the newsletter language is ambiguous. "Term 3 week 4 Friday" gets converted to a real date; "next term sometime in week 4" gets surfaced to the parent as "needs confirmation".

## Tikanga check

NZ school rituals include cultural events that need to be named correctly and treated with respect:

- **Matariki** is a public holiday and many schools mark it with assemblies, performances, or community events. We never reduce it to "Māori New Year holiday" — we acknowledge it as Matariki specifically and surface the school's planned events around it.

- **Kapa haka** performances should be flagged as significant whānau moments, not just calendar items. We highlight them as opportunities for the parent to be there if possible.

- **Mihi whakatau** and **pōwhiri** at school events (start-of-year welcomes for new families, end-of-year farewells) should be recognised, and the parent should be told what to expect if they're new to NZ school culture.

## Privacy Act check

This skill stores reference data only — no personal information. Term dates, public holidays, ritual definitions, NCEA milestones. The structured table is loaded from Ministry of Education sources and refreshed annually.

## YAML frontmatter

Declares this skill as part of `term-planner`. Triggered by any extracted event from `term-planner-core` that needs date resolution or cultural context.

---

## Term dates (2026 reference)

Loaded from Ministry of Education. Each year's dates added before the start of Term 1.

| Year | Term 1 | Term 2 | Term 3 | Term 4 |
|---|---|---|---|---|
| 2026 | 28 Jan – 17 Apr | 5 May – 4 Jul | 21 Jul – 26 Sep | 13 Oct – 18 Dec |

State-integrated schools may start Term 1 up to a week earlier. Private schools vary more widely. Override per-school at onboarding.

## Public holidays during the school year (2026)

- Waitangi Day — Fri 6 Feb
- Good Friday — Fri 3 Apr
- Easter Monday — Mon 6 Apr
- ANZAC Day — Sat 25 Apr (Mondayised? confirm by year)
- King's Birthday — Mon 1 Jun
- Matariki — Fri 10 Jul (date varies year to year, set by legislation)
- Labour Day — Mon 26 Oct
- Christmas Day — Fri 25 Dec
- Boxing Day — Sat 26 Dec

Plus regional anniversary days, which we map by the school's region.

## Canonical NZ school rituals

| Ritual | Typical timing | What it means |
|---|---|---|
| Powhiri / mihi whakatau | Start of Term 1 | Formal welcome for new families and students |
| Swimming sports | Term 1 weeks 4–8 | School-wide swimming carnival |
| Athletics day | Term 1 or Term 4 | School-wide track-and-field day |
| Cross country | Term 2 or Term 3 | School-wide running event |
| Camp | Term 1 or Term 4 (older primary, intermediate) | Multi-day school camp, $200–$500 |
| Mufti day | Variable, several per year | Out-of-uniform day with gold coin koha for charity |
| Pet Day | Term 4 (rural schools) | Kids bring pets / show animals |
| Kapa haka performance | Variable | Cultural performance, usually at end of term or special event |
| Matariki celebration | Around Matariki public holiday | School-wide acknowledgement of Matariki |
| Book Week | Variable, often Term 2 or 3 | Reading-focused week, dress-up day common |
| Pink Shirt Day | Third Friday in May | Anti-bullying campaign, kids wear pink |
| ERO visit | Variable, every 3-ish years | Education Review Office inspection week |
| Parent-teacher interviews | Mid Term 2 and mid Term 3 | 10-minute slots with each teacher |
| Prizegiving | Last week of Term 4 | End-of-year assembly with prizes |

## NCEA milestones (for Y11–13 whānau)

Term Planner surfaces these to parents of secondary kids:

| Level | Year | Internal assessment clusters | External exam window |
|---|---|---|---|
| Level 1 | Y11 | May, Aug, late Oct | Nov |
| Level 2 | Y12 | May, Aug, late Oct | Nov |
| Level 3 / UE | Y13 | May, Aug, late Oct | Nov |

Derived grade applications (when a student misses an external exam due to illness): submitted within 5 working days of the missed exam. We surface this proactively if a parent mentions their kid was sick during exam week.

Endorsement thresholds: Merit (50+ Merit credits at the level), Excellence (50+ Excellence credits). Surface progress mid-year for kids who are in range.

## Common school platforms (NZ)

The newsletter-parser skill has format-specific parsers for the common ones:

- **Hero** (most common in primary)
- **Seesaw** (photo-and-message updates, common in junior primary)
- **Skool Loop** (older, still widespread)
- **Linc-ed**
- **Edge**
- **MyEd**
- **KAMAR** (most common in secondary)

If a parent forwards a notification from any of these, we recognise the format and apply the right parser.

## How "week 4" gets resolved

If a newsletter says "swimming sports is week 4 Friday" and the school year is 2026, Term 1:

1. Look up Term 1 2026 start date: 28 Jan (Wed).
2. Week 1 = 28 Jan – 1 Feb (4 school days).
3. Week 2 = 4 – 8 Feb.
4. Week 3 = 11 – 15 Feb.
5. Week 4 = 18 – 22 Feb.
6. Friday of Week 4 = 22 Feb.

If the school had a deviation set (e.g. starts Term 1 a day late), we adjust. Always store the resolved date but also keep the original "week 4 Fri" reference in the extract for traceability.

## Test cases the implementation must pass

1. Parser receives "swimming sports week 4 Friday" in a Term 1 2026 newsletter from a school with no calendar overrides. Resolved date: Fri 22 Feb 2026.

2. A newsletter from a state-integrated school with a "starts a day later" override. Same "week 4 Friday" prompt resolves to Mon 23 Feb 2026.

3. Matariki public holiday falls inside Term 2 2026 (Fri 10 Jul). Term Planner surfaces "Matariki public holiday — no school Friday" as a TransitionEvent and offers Holiday Ideas as a follow-up.

4. NCEA Level 3 student in Y13. Parent forwards a newsletter mentioning "external exams begin 9 Nov". Term Planner adds an NCEA-specific reminder set: 4 weeks out (revision prep), 2 weeks out (gear check — calculator, ID), morning of (good luck plus the derived grade backup process if anything goes wrong).

5. Newsletter is ambiguous: "sometime in Week 4 we'll have…". Term Planner does not invent a date. It surfaces to the parent: "Newsletter says 'sometime in Week 4' — let me know which day when you find out and I'll add it."
