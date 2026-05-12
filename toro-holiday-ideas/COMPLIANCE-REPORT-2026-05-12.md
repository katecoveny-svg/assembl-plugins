# Canon compliance report — toro-holiday-ideas

**Date:** 2026-05-12 NZST
**Author:** Kaihanga (per `KAIHANGABRIEFTOROLAUNCH20260512.md`)
**Canon reference:** `CANON-plugin-architecture-2026-05-08.md`
**Verdict:** ⚠️ **REGISTERED with 2 drift items flagged — Code to action before status flips to `active`**

---

## Summary

| Surface | Pass / Fail | Notes |
|---|---|---|
| `.claude-plugin/plugin.json` | ✅ Present (695 bytes) | Parses as valid JSON |
| `agents/holiday-ideas.yaml` (top-level keys) | ✅ All 7 required keys present | |
| `agents/holiday-ideas.yaml` (skills list) | ✅ Non-empty list | 3 skills referenced |
| `agents/holiday-ideas.yaml` (never_build) | ✅ Non-empty list with id + reason | |
| `agents/holiday-ideas.yaml` (privacy_act_2020 block) | ✅ Has `ipp_3a_notice`, `retention`, `third_party_sharing` | |
| `agents/holiday-ideas.yaml` (tikanga_check block) | ❌ **FAIL** | `framing` references **none** of the four pou — details below |
| `skills/holiday-ideas-core/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |
| `skills/nz-holiday-sources/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |
| `skills/preference-learning/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |

**Overall:** 9 of 9 skill checks pass · 6 of 7 agent YAML checks pass · 1 drift in `tikanga_check.framing` · 1 informational drift in `status` field

---

## Drift 1 — `tikanga_check.framing` references zero of the four pou (canon §6)

**Canon requires:** `tikanga_check.framing` must reference all four pou explicitly: **rangatiratanga, kaitiakitanga, manaakitanga, whanaungatanga**.

**Current value in `agents/holiday-ideas.yaml`:**

```yaml
tikanga_check:
  framing:
    - whanau
    - hapori
    - whenua
```

**Coverage:**

| Pou | Present? |
|---|---|
| rangatiratanga | ❌ Missing |
| kaitiakitanga | ❌ Missing |
| manaakitanga | ❌ Missing |
| whanaungatanga | ❌ Missing |

`whanau`, `hapori`, `whenua` (family, community, land) are excellent framing for a holiday-ideas agent — especially `whenua` for NZ-anchored options — but they are descriptive contexts, not the four pou the canon requires. Suggested fix (Code to apply):

```yaml
tikanga_check:
  framing:
    - rangatiratanga    # whānau retains authority over preference data
    - kaitiakitanga     # preference history treated as taonga
    - manaakitanga      # holiday choices honour visitor + host
    - whanaungatanga    # holidays are relational — extended whānau, marae stays
    - whanau            # context (existing — retain)
    - hapori            # context (existing — retain)
    - whenua            # context (existing — retain, especially for OSCAR + sport NZ club camps)
```

## Drift 2 — `status: active` contradicts brief

**Brief specifies:** "Status for all three: `scaffolded`."
**Current value in YAML:** `status: active`

Kaihanga registered with `is_active = false`. Code to align YAML.

## Drift 3 (informational) — `model` field prefix convention

`claude-opus-4-7` normalised to `anthropic/claude-opus-4-7` on insert.

---

## What Kaihanga did anyway (not blocked by drift)

Row registered in `agent_prompts`:

- `agent_name = 'holiday-ideas'`
- `pack = 'toro'`
- `display_name = 'Holiday Ideas'`
- `system_prompt = <agents/holiday-ideas.md contents with source-path header>`
- `is_active = false` (= scaffolded)
- `model_preference = 'anthropic/claude-opus-4-7'`
- `version = 1`

ℹ️ **No naming collision.** `toro` namespace has no existing `holiday-ideas` row. The closest neighbour is `toro-logistics` (Toro Go) which handles in-the-moment family transport (school runs, etc.). Holiday Ideas is the once-a-quarter planning surface (OSCAR-approved programmes, MoE-subsidised camps, council activities, sport NZ club camps per the handover). Different surfaces, no overlap.

---

## Source paths (canon §3 traceability)

| Surface | Path |
|---|---|
| Agent YAML | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-holiday-ideas/agents/holiday-ideas.yaml` |
| Agent system prompt | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-holiday-ideas/agents/holiday-ideas.md` |
| Plugin manifest | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-holiday-ideas/.claude-plugin/plugin.json` |
| Skill: holiday-ideas-core | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-holiday-ideas/skills/holiday-ideas-core/SKILL.md` |
| Skill: nz-holiday-sources | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-holiday-ideas/skills/nz-holiday-sources/SKILL.md` |
| Skill: preference-learning | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-holiday-ideas/skills/preference-learning/SKILL.md` |

---

## Action items for Code (before status flips to active)

1. Add all four pou to `agents/holiday-ideas.yaml` `tikanga_check.framing` (keep `whanau`, `hapori`, `whenua` alongside)
2. Change `status: active` → `status: scaffolded`
3. (Schema ticket) Add `source_path` column to `agent_prompts`
4. (Optional) Normalise `model:` to `anthropic/<model>` convention
