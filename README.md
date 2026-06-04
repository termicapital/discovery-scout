# discovery-scout

Claude Code skill for [Suhail Innovation Company](https://suhailsa.com)'s Discovery Process. Automates **Stage 0 (Idea Sourcing)** and **Stage 1.1 (Desktop Research)** of venture-building: scans KSA ministry sandboxes + Vision 2030 programs, finds proven international analogs, runs competitive + regulatory + moat pressure tests, drafts a plain-English Desktop Research brief, scores against the 16-Q IC scorecard + D/F/V/A evidence levels, and writes to Notion.

## What it does

**Two modes:**

- **Interactive** (`/discovery-scout` or `/discovery-scout transport`) — human-in-the-loop at three gates: signal selection (Phase 2), draft review (Phase 5), Notion write (Phase 6). Best for first-time ministry exploration or when you want editorial control over every artifact.
- **Autonomous** (`/discovery-scout --autonomous <ministry>`) — runs end-to-end without gates. Reads past Notion runs as Phase 0 learning signal. Composite scoring filters for Suhail-edge sectors, sandbox-anchored ventures, proven international analogs, and 30-day-MVP-shippable shapes. Killed-row fallback when quality gates fail — diagnostic itself feeds future learning.

**Output of one successful run:**

- One row in `Problem Signal Capture` (Sourcing Channel = "Government Calls / Sandboxes", Weak signal per the internet-sourced rule)
- One row in `Discovery Pipeline` (Status = Research, Sub-stage = Desktop, with full brief as page body + all 20 scores)
- A Stage 1.2 Research Agenda appended to the brief

## Key rules (v0.5.1)

- **Suhail-edge sectors** — Real Estate + NGOs are the primary filter. Other sectors must compensate with stronger moats.
- **Proven international analog required** — at least one analog must be profitable + >5yr operating, OR post-Series B + >100K paying customers, OR >$50M ARR.
- **30-Day MVP Test** (gates the H2 2026 1-MVP-per-month launch sprint) — concierge-able + off-the-shelf + customers reachable in 14 days via Suhail's network.
- **Moat pressure test** — every brief answers "why wouldn't [obvious incumbent] build this?" Generic moats (sales pipeline, local team, first-mover) don't count.
- **Briefs written readable cold** — plain-English opener, acronyms expanded on first use, glossary required.
- **Stage 1.2 Research Agenda** derived from /office-hours forcing questions, classifying each into desk-answerable vs. flag-for-user-research.

## Install

```bash
git clone https://github.com/termicapital/discovery-scout.git ~/.claude/skills/discovery-scout
```

Then `/discovery-scout` is available in any Claude Code session.

## Requires

- **Claude Code** with the **Notion MCP** connected
- **Access to Suhail's Notion databases**:
  - Problem Signal Capture (`collection://fc466b72-eaec-4ac9-b692-5e745966f1d4`)
  - Discovery Pipeline (`collection://5e6f6355-ec64-4950-b1cb-66806ef24401`)
- **WebSearch + WebFetch** tools (standard in Claude Code)

## Versions

| Version | Highlights |
|---|---|
| 0.1.0 | Initial implementation: 6-phase workflow, interactive only |
| 0.1.1 | WebFetch → WebSearch fallback for Gulf geo-blocking; `[REGULATOR-NAMED-SOLUTION]` + `[SINGLE-SOURCE]` tags |
| 0.1.2 | Phase 1b KSA Competitive Pre-Scan added |
| 0.1.3 | `SUHAIL-EDGE: STRONG/WEAK` tag + Moat Pressure Test + "Why wouldn't [incumbent] build this?" in brief template |
| 0.1.4 | "What this venture is — in plain terms" opener + acronym-on-first-use + Glossary |
| 0.1.5 | Stage 1.2 Research Agenda section in brief template (office-hours integration) |
| 0.2.0 | Autonomous mode + Phase 0 (learn from past runs) + composite scoring formula + quality gates + Killed-row fallback |
| 0.3.0 | Phase 2b portfolio overlap + sector saturation cap + sovereign incumbency risk + Phase 7 Final Recommendation + cross-venture ranking |
| 0.4.0 | H2 2026 launch-priority rules: sandbox-anchored, proven international analog, launchability scoring |
| 0.5.0 | Simplification: replace 5-component Launchability with 30-Day MVP Test (3 binary questions); SPRINT vs BACKLOG verdict split |
| 0.5.1 | Plain-English "What concierge MVP means" section added to Phase 5c (readable-cold rule applied to skill text itself) |

## Test history

Iterated against real Suhail Discovery Pipeline runs:

- **Waqf Operations Platform** — interactive end-to-end run; surfaced rules around briefs-readable-cold, moat-pressure-test, office-hours-at-Stage-1.1
- **Multi-Authority Approval Workflow for KSA RE Developers** — autonomous run
- **Warehouse-as-a-Service Marketplace for KSA SME Importers** — autonomous run; revealed CAPEX-shape filter need
- **Saudi Donor CRM for Small Charities** — autonomous (v0.5.1) — STRONG ADVANCE SPRINT
- **Individual Landlord Rental Management SaaS** — autonomous (v0.5.1) — CONDITIONAL ADVANCE SPRINT
- **AI Grant-Application Copilot for Saudi Charities** — autonomous (v0.5.1) — CONDITIONAL ADVANCE SPRINT

## Methodology references

- Suhail Discovery Process (Notion internal)
- Strategyzer 4 risk dimensions (Desirability / Feasibility / Viability / Adaptability)
- Eric Ries, *The Lean Startup* (concierge MVP pattern)
- YC Office Hours forcing questions (six pre-product diagnostic questions)
- Built alongside Garry Tan's [gstack](https://github.com/garrytan/gstack) — adopts the gstack convention for skill structure but lives standalone.

## License

Private — Suhail Innovation Company internal use.
