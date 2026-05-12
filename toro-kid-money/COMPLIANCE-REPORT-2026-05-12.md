# Canon compliance report — toro-kid-money

**Date:** 2026-05-12 NZST
**Author:** Kaihanga (per `KAIHANGABRIEFTOROLAUNCH20260512.md`)
**Canon reference:** `CANON-plugin-architecture-2026-05-08.md`
**Verdict:** ⚠️ **REGISTERED with 2 drift items flagged — Code to action before status flips to `active`**

---

## Summary

| Surface | Pass / Fail | Notes |
|---|---|---|
| `.claude-plugin/plugin.json` | ✅ Present (708 bytes) | Parses as valid JSON |
| `agents/kid-money.yaml` (top-level keys) | ✅ All 7 required keys present | `slug`, `display_name`, `kete`, `status`, `model`, `tier`, `description` |
| `agents/kid-money.yaml` (skills list) | ✅ Non-empty list | 3 skills referenced |
| `agents/kid-money.yaml` (never_build) | ✅ Non-empty list with id + reason | Each entry validates |
| `agents/kid-money.yaml` (privacy_act_2020 block) | ✅ Has `ipp_3a_notice`, `retention`, `third_party_sharing` | Block well-formed |
| `agents/kid-money.yaml` (tikanga_check block) | ❌ **FAIL** | `framing` missing 2 of 4 pou — details below |
| `skills/kid-money-core/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |
| `skills/nz-cdr-payments/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |
| `skills/three-jar-split/SKILL.md` | ✅ Frontmatter + all 3 mandatory sections | |

**Overall:** 9 of 9 skill checks pass · 6 of 7 agent YAML checks pass · 1 drift in `tikanga_check.framing` · 1 informational drift in `status` field

---

## Drift 1 — `tikanga_check.framing` missing two of the four pou (canon §6)

**Canon requires:** `tikanga_check.framing` must reference all four pou explicitly: **rangatiratanga, kaitiakitanga, manaakitanga, whanaungatanga**.

**Current value in `agents/kid-money.yaml`:**

```yaml
tikanga_check:
  framing:
    - mahi
    - koha
    - kaitiakitanga
    - manaakitanga
```

**Coverage:**

| Pou | Present? | Notes |
|---|---|---|
| rangatiratanga | ❌ Missing | Self-determination over data — relevant for parents authorising CDR consent for tamariki |
| kaitiakitanga | ✅ Present | |
| manaakitanga | ✅ Present | |
| whanaungatanga | ❌ Missing | Cross-generational relational care — particularly relevant for koha jar (giving to whānau, marae, kura) |

`mahi` (work/effort) and `koha` (reciprocal gift) are valid framing concepts and **should be retained** — but the four pou must sit alongside them, not in place of them. Suggested fix (Code to apply):

```yaml
tikanga_check:
  framing:
    - rangatiratanga    # parents retain authority over tamariki's CDR consent
    - kaitiakitanga     # tamariki's financial data treated as taonga (existing)
    - manaakitanga      # reciprocal care between parent + child (existing)
    - whanaungatanga    # koha jar flows toward whānau / marae / kura
    - mahi              # effort earned (existing — retain)
    - koha              # third jar's reciprocal-giving framing (existing — retain)
```

## Drift 2 — `status: active` contradicts brief

**Brief specifies:** "Status for all three: `scaffolded` (not `active` yet — Code hasn't built the runtime)."
**Current value in YAML:** `status: active`

This is informational drift only — Kaihanga will register the row with `is_active = false` in `agent_prompts` to honour the scaffolded posture, regardless of what the YAML says. Code to align the YAML to `status: scaffolded` so file and DB match.

## Drift 3 (informational) — `model` field prefix convention

YAML says `model: claude-opus-4-7`. Existing assembl `agent_prompts.model_preference` rows use the `anthropic/<model>` prefix convention (e.g. `anthropic/claude-haiku-4-5-20251001`). Kaihanga will normalise to `anthropic/claude-opus-4-7` on insert. Code may want to standardise YAML to match.

---

## What Kaihanga did anyway (not blocked by drift)

Per brief: drift is FLAGGED, not FIXED. Kaihanga registered the row in `agent_prompts` with:

- `agent_name = 'kid-money'`
- `pack = 'toro'`
- `display_name = 'Kid Money'` (from YAML)
- `system_prompt = <agents/kid-money.md contents with source-path header>`
- `is_active = false` (= scaffolded posture per brief)
- `model_preference = 'anthropic/claude-opus-4-7'`
- `version = 1`

⚠️ **Naming-collision warning held for Kate's call:** `agent_prompts` already contains `toro-money` (display_name "Toro Pūtea", is_active=true) — different agent_name, different model (sonnet vs opus), different scope (general financial-literacy vs CDR-payments wedge). Both rows now coexist. Kate to decide whether `kid-money` REPLACES `toro-money` (deactivate toro-money) or supplements it.

---

## Source paths (canon §3 traceability)

| Surface | Path |
|---|---|
| Agent YAML | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-kid-money/agents/kid-money.yaml` |
| Agent system prompt | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-kid-money/agents/kid-money.md` |
| Plugin manifest | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-kid-money/.claude-plugin/plugin.json` |
| Skill: kid-money-core | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-kid-money/skills/kid-money-core/SKILL.md` |
| Skill: nz-cdr-payments | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-kid-money/skills/nz-cdr-payments/SKILL.md` |
| Skill: three-jar-split | `github.com/katecoveny-svg/assembl-plugins/blob/main/toro-kid-money/skills/three-jar-split/SKILL.md` |

Note: `agent_prompts` schema currently lacks a `source_path` column. Source URL is embedded as an HTML-comment header in `system_prompt` until Code adds the column.

---

## Action items for Code (before status flips to active)

1. Add `rangatiratanga` and `whanaungatanga` to `agents/kid-money.yaml` `tikanga_check.framing` (keep `mahi`, `koha` alongside)
2. Change `status: active` → `status: scaffolded` in `agents/kid-money.yaml`
3. Decide with Kate: does `kid-money` replace `toro-money`? If yes, set `is_active=false` on `agent_prompts.agent_name='toro-money'`
4. (Schema, separate ticket) Add `source_path text` column to `agent_prompts`; backfill from HTML-comment headers
5. (Optional) Normalise `model:` to `anthropic/<model>` convention in agent YAMLs
