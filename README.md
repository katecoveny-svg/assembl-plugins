# assembl-plugins

> Nothing in this repository constitutes legal, tax, accounting, financial, immigration, customs, biosecurity, or health and safety advice. These agents draft work product — entries, memos, checklists, calculations, draft correspondence — for review by a qualified professional or licensed adviser. They do not file with Inland Revenue, lodge customs entries, submit WorkSafe notifications, make ACC claims, register entities with the Companies Office, send Privacy Commissioner breach notifications, or submit any document to a NZ government agency on the user's behalf; every output is staged for human sign-off. The user is responsible for verifying outputs and for compliance with all applicable New Zealand laws, including but not limited to the Privacy Act 2020, the Consumer Guarantees Act 1993, the Fair Trading Act 1986, the AML/CFT Act 2009, the Customs and Excise Act 2018, the Building Act 2004, the Construction Contracts Act 2002, the Health and Safety at Work Act 2015, and any tikanga Māori obligations relevant to their work.

---

assembl's canonical plugin marketplace — file-based plugins for Claude Code and Claude Managed Agents, adapted to the NZ SME market.

This repo is the source of truth. The Supabase `agent_prompts` table is a runtime cache that loads from this repo on each deploy. Every change is a git diff. Compliance review reads markdown and JSON directly.

## The seven industry ketes

| Slug | Industry pack | Pilot customer |
|---|---|---|
| `waihanga` | Construction | TOA Architecture |
| `manaaki` | Hospitality | TBD |
| `pikau` | Freight & Customs | Aironaut Customs |
| `arataki` | Tourism & Visitor Experience | — |
| `auaha` | Creative Industries | — |
| `ako` | Education | — |
| `hoko` | Retail & E-commerce | — |

Plus `assembl-core`: mandatory baseline plugin (Privacy Act 2020, tikanga compliance, NZBN lookup, CGA quick reference). Install first.

## TŌRO — whānau navigator ($29/mo family tier)

TŌRO is the family-facing kete, separate from the industry packs. It is delivered as three composable sub-plugins, each shipping on its own schedule:

| Slug | Role | Build phase |
|---|---|---|
| `toro-term-planner` | Free wedge. Parses NZ school newsletters → calendar, payments, gear, permission slips. | Ships first (builds on existing `agentmail-inbound`). |
| `toro-kid-money` | Viral wedge. Chores → photo proof → CDR-initiated payment to kid's bank → three-jar split (Spend / Save / Give, 10% to charity). | Phase 1 manual-transfer in weeks 1–6; Phase 2 CDR-enabled at ~week 14. |
| `toro-holiday-ideas` | Four-times-a-year sticky feature. NZ-shaped school-holiday plans (OSCAR + free council + day trips + rainy-day backups). | Ships in time for July school holidays. |

> The top-level `toro/` folder is a legacy stub from an earlier architecture (Professional Services). It is retained for git history but is no longer canonical. New TŌRO work lands in `toro-kid-money/`, `toro-term-planner/`, `toro-holiday-ideas/`.

## Install (Claude Code marketplace)

Once the GitHub remote `assembl-co/assembl-plugins` exists:

```
/plugin marketplace add assembl-co/assembl-plugins
/plugin install assembl-core@assembl-plugins
/plugin install pikau@assembl-plugins             # freight & customs (pilot)
/plugin install waihanga@assembl-plugins          # construction (pilot)
/plugin install toro-term-planner@assembl-plugins # free wedge, ships first
/plugin install toro-kid-money@assembl-plugins    # viral wedge, soft-launch wk6
```

The other ketes are stubs at v0.0.1 — install only as their build days complete (canon §9, Day 9-12+).

## MCP servers

Live in `mcp-servers/`. assembl-core's `.mcp.json` registers them:

| Server | Status | Source |
|---|---|---|
| `mcp-nzbn` | live | NZBN public register, https://www.nzbn.govt.nz/ |
| `mcp-companies-office` | live | Companies Office public register, https://www.business.govt.nz/services/business-data |
| `mcp-ird` | stub | IRD myIR — requires OAuth, build when licensed |
| `mcp-customs-tariff` | stub | NZ Customs Working Tariff (read-only), public, build for PIKAU |
| `mcp-mfat-sanctions` | stub | MFAT NZ Sanctions Register, public, build for PIKAU |
| `mcp-lbp-register` | stub | Licensed Building Practitioners register, public, build for WAIHANGA |

The "never build" MCP servers (canon §8.1) are permanently excluded: NZ Customs TSW lodgement, IRD myIR filing, WorkSafe notifiable event submission, Companies Office entity registration, Privacy Commissioner breach notification, MPI biosecurity declarations.

## Source

Locked source of truth: `CANON-plugin-architecture-2026-05-08.md` (Kate Hudson, assembl Ltd).
Reference architecture: github.com/anthropics/financial-services-plugins.

## License

Apache 2.0. See [LICENSE](LICENSE).
