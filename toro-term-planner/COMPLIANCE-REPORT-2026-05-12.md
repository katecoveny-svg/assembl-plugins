# Canon compliance report — toro-term-planner

**Date:** 2026-05-12 NZST
**Author:** Kaihanga (per `KAIHANGABRIEFTOROLAUNCH20260512.md`)
**Canon reference:** `CANON-plugin-architecture-2026-05-08.md`
**Verdict:** ⚠️ **REGISTERED with 2 drift items flagged — Code to action before status flips to `active`**

---

## Summary

| Surface | Pass / Fail | Notes |
|---|---|---|
| `.claude-plugin/plugin.json` | ✅ Present (789 bytes) | Parses as valid JSON |
| `agents/term-planner.yaml` (top-level keys) | ✅ All 7 required keys present | |
| `agents/term-planner.yaml` (skills list) | ✅ Non-empty list | 3 skills referenced |
| `agents/term-planner.yaml` (never_build) | ✅ Non-empty list with id + reason | |
| `agents/term-planner.yaml` (privacy_act_2020 block) | ✅ Has `ipp_3a_notice`, `retention`, `third_party_sharing` | |
| `agents/term-planner.yaml` (tikanga_check block) | ❌ **FAIL** | `framing` references **none** of the four pou — details below |
| `skills/newsletter-parser/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |
| `skills/nz-school-calendar/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |
| `skills/term-planner-core/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |

**Overall:** 9 of 9 skill checks pass · 6 of 7 agent YAML checks pass · 1 drift in `tikanga_check.framing` (worst of the three plugins) · 1 informational drift in `status` field

---

## Drift 1 — `tikanga_check.framing` references zero of the four pou (canon §6)

**Canon requires:** `tikanga_check.framing` must reference all four pou explicitly: **rangatiratanga, kaitiakitanga, manaakitanga, whanaungatanga**.

**Current value in `agents/term-planner.yaml`:**

```yaml
tikanga_check:
  framing:
    - whanau
    - hapori
```

**Coverage:**

| Pou | Present? |
|---|---|
| rangatiratanga | ❌ Missing |
| kaitiakitanga | ❌ Missing |
| manaakitanga | ❌ Missing |
| whanaungatanga | ❌ Missing |

`whanau` (family) and `hapori` (community) are correct framing concepts for a term-planner — but they are descriptive contexts, not the four pou the canon requires. Suggested fix (Code to apply):

```yaml
tikanga_check:
  framing:
    - rangatiratanga    # parents retain authority over their term plan
    - kaitiakitanga     # term plan + newsletter intake treated as taonga
    - manaakitanga      # the plan exists to care for tamariki schedules
    - whanaungatanga    # cross-school + cross-family coordination
    - whanau            # context (existing — retain)
    - hapori            # context (existing — retain)
```

The four pou must appear by name. The Tā stage validator (Mahara stage) will reject content if framing-derived language is absent from outputs, so this drift will surface at runtime once the agent goes live.

## Drift 2 — `status: active` contradicts brief

**Brief specifies:** "Status for all three: `scaffolded` (not `active` yet — Code hasn't built the runtime)."
**Current value in YAML:** `status: active`

Kaihanga registered with `is_active = false` regardless. Code to align YAML to `status: scaffolded`.

## Drift 3 (informational) — `model` field prefix convention

YAML says `model: claude-opus-4-7`. Existing convention is `anthropic/<model>`. Kaihanga normalised on insert to `anthropic/claude-opus-4-7`.

---

## What Kaihanga did anyway (not blocked by drift)

Per brief: drift is FLAGGED, not FIXED. Row registered in `agent_prompts`:

- `agent_name = 'term-planner'`
- `pack = 'toro'`
- `display_name = 'Term Planner'`
- `system_prompt = <agents/term-planner.md contents with source-path header>`
- `is_active = false` (= scaffolded)
- `model_preference = 'anthropic/claude-opus-4-7'`
- `version = 1`

ℹ️ **Naming overlap (informational, no collision):** This sits alongside the existing `toro-education` (Toro Learn) and `toro-homework` (Toro Mahi Kāinga) rows. Different scope: term-planner consumes school newsletters and produces term-level term plans; toro-education handles in-the-moment learning support; toro-homework helps with individual assignments. No deactivation needed — they cover different stages of the school journey.

The handover doc mentions `agentmail-inbound` is a related patch on disk that builds the school-email intake. Term Planner depends on that landing first to be runtime-useful.

---

## Source paths (canon §3 traceability)

| Surface | Path |
|---|---|
| Agent YAML | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-term-planner/agents/term-planner.yaml` |
| Agent system prompt | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-term-planner/agents/term-planner.md` |
| Plugin manifest | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-term-planner/.claude-plugin/plugin.json` |
| Skill: newsletter-parser | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-term-planner/skills/newsletter-parser/SKILL.md` |
| Skill: nz-school-calendar | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-term-planner/skills/nz-school-calendar/SKILL.md` |
| Skill: term-planner-core | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-term-planner/skills/term-planner-core/SKILL.md` |

---

## Action items for Code (before status flips to active)

1. Add all four pou to `agents/term-planner.yaml` `tikanga_check.framing` (keep `whanau`, `hapori` alongside)
2. Change `status: active` → `status: scaffolded`
3. Confirm `agentmail-inbound` patch landed before flipping term-planner to active (runtime dependency)
4. (Schema ticket) Add `source_path` column to `agent_prompts`
5. (Optional) Normalise `model:` to `anthropic/<model>` convention
