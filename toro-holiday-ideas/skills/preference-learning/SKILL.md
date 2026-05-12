---
name: preference-learning
agent: holiday-ideas
kete: toro
description: >
  Preference-learning layer for Holiday Ideas. Refines recommendations
  over time based on parent feedback (accepted / rejected / modified
  plans). By the third school holiday, recommendations are noticeably
  better than the first.
---

# Preference Learning

## What this skill will NOT do

- Will not train a global model on whānau data. Preferences are per-whānau, stored against that whānau's TŌRO account. We do not aggregate across whānau to improve global recommendations.

- Will not infer kid preferences from kid behaviour (we don't have kid behaviour — we only have what the parent tells us). Preferences come from the parent's stated answers at onboarding, refined by which suggestions the parent accepts vs rejects in subsequent plans.

- Will not surface "we've learned X about your kid" insights in a way that's surveillance-feeling. No "Tama really likes museums" dashboards. The learning is invisible — recommendations just get better.

## Tikanga check

The preference model respects the whānau's agency. We don't lock the model — every plan asks for input, every plan can be rejected wholesale, and the parent can reset preferences at any time.

When we detect a strong preference for cultural activities (kapa haka workshops, marae visits, kohanga reo holiday programmes), we surface those types of events more, not as quota-fill but as genuine fit.

## Privacy Act check

Data this skill maintains per whānau:

- Per-kid interest tags (e.g. "swimming", "dinosaurs", "art", "rugby"). Initially set by parent, refined over time.
- Per-whānau budget tolerance (we learn how aggressively a whānau picks free options vs paid).
- Per-whānau radius (we learn how far they're willing to drive).
- Per-whānau outdoor-vs-indoor preference (some whānau are aggressively outdoor regardless of weather; some pick indoor whenever possible).

Retention: 90 days after account close.

Right of access: parent can request a download of their preference data at any time (Privacy Act IPP 6). We provide it as a JSON file with a brief explanation.

Right of correction: parent can edit any preference at any time. Edits are immediate.

## YAML frontmatter

Declares this skill as part of `holiday-ideas`. Triggered after every plan generation (to record what was recommended) and after every parent response to a plan (to record acceptance / rejection / modification).

---

## The preference model (per whānau)

```typescript
interface WhanauPreferences {
  whanau_id: string;
  kids: {
    [kid_id: string]: {
      interests: { tag: string; weight: number }[];
      // Higher weight = more preferred; weights between 0 and 1.
      avoided: string[]; // tags the parent has explicitly said no to
    };
  };
  budget_tolerance: 'frugal' | 'moderate' | 'open'; // learned, refined per holiday
  outdoor_preference: number; // 0 (strongly indoor) to 1 (strongly outdoor)
  travel_radius_km: number; // learned from accepted vs rejected based on distance
  preferred_times: ('morning' | 'afternoon' | 'all-day')[];
  last_updated: string;
}
```

## The learning signal

After a plan is delivered, every parent action produces a signal:

- **Accepted whole plan as-is** → strong positive for all included activities, neutral for excluded.
- **Modified a day** (swapped one activity for another) → negative for the swapped-out, positive for the swap-in. Important: the parent may have swapped for reasons unrelated to fit (e.g. realised they had a dentist appointment) — we discount these signals.
- **Rejected outright** → negative for all included, ask the parent why if they're willing.
- **Booked one of the "alt" options instead of the "plan"** → mild positive for the alt's tags, neutral for the plan's.

Signals weight more recent feedback higher. A preference shift in the most recent holiday outweighs a year-old one.

## The cold-start (first holiday)

First holiday after signup: we lean on what the parent told us at onboarding (a 6-question form). After that, we lean on the model.

Onboarding questions:
1. Each kid's age and a free-text "what do they like" field. We tag-extract from the text.
2. Approximate household budget for a typical school holiday.
3. How far you're willing to drive for an activity (in km).
4. Strong preferences (indoor / outdoor / no preference).
5. Any cultural activities you'd want surfaced (kapa haka, te reo immersion, kohanga reo, marae visits, none).
6. Any constraints we should know about (kid in a cast, only one car for the household, etc.).

## The refresh moments

The model auto-refreshes at three points per year:

- **Before Term 2 holiday** — first refresh of the year, leans on onboarding + any plan generated so far.
- **Before Term 3 holiday** — second refresh, model has more data.
- **Before Term 4 / summer holidays** — third refresh, model is most accurate.

Each refresh, we show the parent a one-screen summary: "Here's what I've learned. Tama seems to be loving outdoor stuff this year. Maia's into anything art-related. You skew toward free options on weekdays and one big paid activity on the weekend. Anything off?" Parent can correct.

## The reset

Parent can hit "reset preferences" at any time. This wipes the learned model back to onboarding-only. Useful when kids change phases (a kid who loved dinosaurs at 6 is over them at 9).

## Test cases the implementation must pass

1. New whānau signs up. First holiday plan uses onboarding-only data. Parent accepts 60% of the plan. After acceptance, preference model is updated with positive signals for accepted activities, neutral for rejected.

2. Second holiday plan. Recommendations should noticeably reflect the learning (e.g. if Tama's parent rejected all museum options last time, museum count goes down for Tama this time).

3. Parent requests a preferences download. They receive a JSON file containing the full preference model for their whānau within 60 seconds.

4. Parent hits reset. Preference model returns to onboarding state. Next plan generation behaves like a first-holiday whānau.

5. Whānau closes account. All preference data deleted within 90 days. No traces left in any aggregate or analytics dataset.
