---
name: discovery-scout
version: 0.7.1
description: |
  Automates Stage 0 (Idea Sourcing from government sandboxes) + Stage 1.1 (Desktop
  Research) of Suhail's Discovery Process. Scans Vision 2030 / ministry sandboxes
  for problem signals, finds analog non-Gulf startups, runs a KSA competitive +
  regulatory check, drafts a Desktop Research brief, and populates one Problem
  Signal Capture row + one Discovery Pipeline row in Notion. Draft-only by default;
  writes to Notion only on explicit user approval. Use when starting a new
  Discovery Pipeline idea from a government sandbox call or Vision 2030 program.
allowed-tools:
  - WebFetch
  - WebSearch
  - AskUserQuestion
  - Read
  - Write
  - Bash
---

# /discovery-scout: Stage 0 + Stage 1.1 of the Suhail Discovery Process

## Purpose

Produce **one fully-populated Discovery Pipeline row** (with its linked Problem Signal Capture row and a Stage 1.1 Desktop Research brief) from a single ministry input. Output of one successful run is one row in `Status = Research`, `Research Sub-stage = Desktop`, ready for a human to begin Stage 1.2 User Research.

This skill executes the "government sandbox → analog startup → KSA adaptation" play described in the Discovery Process doc. It does **not** score ideas (Q1–Q16), run user interviews, or make IC decisions — those are downstream human judgments.

## Iron Law

**Every factual claim must cite a source URL + access date.** No claim survives without a source line. If you cannot find a source, mark the claim as `[UNVERIFIED]` and surface it as an open question — do not fabricate.

## Optional: Perplexity Sonar Integration

This skill works fully on Claude Code's built-in `WebSearch` and `WebFetch` tools. **If a `PERPLEXITY_API_KEY` env var is set**, three phases (1, 1b, 3b) upgrade to Perplexity's Search API + Sonar models for richer cited sources, multi-query batching, and better Gulf-region geo-blocking resilience.

**Check for the env var at the start of every run:**

```bash
if [ -n "$PERPLEXITY_API_KEY" ]; then
  echo "PERPLEXITY_ENABLED"
else
  echo "PERPLEXITY_DISABLED — using WebSearch fallback"
fi
```

**When `PERPLEXITY_ENABLED`:**

- **Phase 1 (ministry sandbox scan):** use Perplexity Search API with multi-query (up to 5 in one call) instead of 4–5 separate WebSearches. Add `"country": "SA"` to bias toward KSA results and `"search_domain_filter": ["gov.sa", "vision2030.gov.sa", "wamda.com", "magnitt.com", ...]` to favor high-credibility sources.
- **Phase 1b (KSA competitive pre-scan):** same multi-query pattern, one Search API call per signal cluster instead of per signal.
- **Phase 3b (Customer Journey Today):** use **`sonar-deep-research`** (model name `sonar-deep-research`). This is the **recommended default for v0.6.1+** — Deep Research runs 50-70 internal search queries per call and synthesizes a cited multi-source answer (typically 40-50 citations), vastly better at finding *named customer-vendor pairings*, *specific charity adoptions*, and *operational evidence* than the cheaper `sonar` model. The 2026-06-10 A/B test on the Donor CRM venture confirmed Deep Research found a verified vendor adoption (Darah → Naseej MEDAD platform) that `sonar` missed entirely.
- **Fallback model:** `sonar` (~$0.006/call, ~10s) is the cheap iteration model — use it for quick "is there anything here?" probes before committing to a full Deep Research call.

**Call patterns:**

```bash
# Phase 1 / 1b — Search API (raw search, $5 per 1k requests)
curl -s -X POST https://api.perplexity.ai/search \
  -H "Authorization: Bearer $PERPLEXITY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": ["q1", "q2", "q3"],
    "max_results": 10,
    "country": "SA",
    "search_domain_filter": ["gov.sa", "vision2030.gov.sa", "wamda.com"],
    "search_context_size": "medium"
  }'
```

```bash
# Phase 3b — sonar-deep-research (RECOMMENDED DEFAULT, ~$0.55-0.65 per query, ~2-3 min)
# Multi-step research with 50-70 internal searches + cited synthesis.
# Set max_tokens to 8000 to give the model room for full synthesis after reasoning.
# Use a temp file for the JSON body to avoid bash quoting issues with long content.
cat > /tmp/dr_body.json <<'JSON'
{
  "model": "sonar-deep-research",
  "messages": [{"role": "user", "content": "<the Phase 3b customer-journey question, with specific named entities>"}],
  "max_tokens": 8000
}
JSON
curl -s -X POST https://api.perplexity.ai/chat/completions \
  -H "Authorization: Bearer $PERPLEXITY_API_KEY" \
  -H "Content-Type: application/json" \
  --max-time 290 \
  --data-binary @/tmp/dr_body.json
```

```bash
# Phase 3b fallback — sonar (cheap + fast, ~$0.006 per query, ~10s)
# Use when iterating or doing a quick "is there anything here?" probe.
curl -s -X POST https://api.perplexity.ai/chat/completions \
  -H "Authorization: Bearer $PERPLEXITY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sonar",
    "messages": [
      {"role": "user", "content": "Imagine a Saudi small charity director Googles \"donor management software\" today. Walk me through the top 5 results, what each charges, and what they would settle on. Cite sources."}
    ]
  }'
```

**When `PERPLEXITY_DISABLED`:** fall back to Claude Code's `WebSearch` and `WebFetch` tools — the skill still works fully. Document the fallback in the eventual `Review Notes` field so the human knows which research path was used.

**Cost estimate per autonomous run** (with Perplexity v0.6.1 defaults — `sonar-deep-research` for Phase 3b + `sonar` for Phase 1/1b): ~$0.60–$0.80 per run (1 Deep Research call dominates the cost). At 4 runs/month, ~$2.50–3.50/month total. The 2026-06-10 A/B test confirmed Deep Research delivers materially better evidence (50 citations vs 9, named customer-vendor pairings found) — the cost lift is worth it for IC-grade briefs.

**Security:** The API key MUST NEVER be hard-coded in this skill, committed to git, or echoed in run output. Only read from the `PERPLEXITY_API_KEY` env var. The README documents setup; the team distributes the key out-of-band (password manager, encrypted channel).

## Modes: Interactive vs. Autonomous

`/discovery-scout` runs in one of two modes, set by the first argument.

**Interactive mode (default)** — invoked as `/discovery-scout` or `/discovery-scout transport`. The skill asks for human input at three gates: Phase 2 selection (pick a signal from the shortlist), Phase 5 revisions (review the draft), Phase 6 approval (approve Notion write). Best for first-time ministry exploration or when the user wants editorial control over every artifact.

**Autonomous mode** — invoked as `/discovery-scout --autonomous transport` (or `--autonomous real-estate`, etc.). The skill runs end-to-end without gates: picks signals via the autonomous selection rules in Phase 2, fills in D/F/V/A + Q1–Q16 scores with rationale at Phase 5b, and writes to Notion at Phase 6 if quality gates pass. If gates fail, writes a `Status = Killed` diagnostic row instead of silently bailing. Best for scheduled / batch operation (e.g. weekly autonomous runs against rotating ministries).

In autonomous mode, the skill also runs **Phase 0 (Learn from past runs)** before Phase 1 — queries the Notion Discovery Pipeline for past entries, extracts advancer-vs-kill patterns, and applies them as soft heuristics in the current run.

**Mode-flag handling:** if the first argument is `--autonomous` or `--auto`, treat as autonomous. Any other first argument is interpreted as the ministry name and the mode stays interactive. To run autonomously on a specific ministry: `--autonomous <ministry>`.

## Inputs

Ask the user at start (use `AskUserQuestion`):

1. **Ministry / sector focus** — default `Transport (MoT / TGA)`. Other priorities: Health (MoH innovation), NGOs / non-profits, Hajj / religious tourism, plus expansion targets (SAMA, CST, NTDP, MISA, MoH).
2. **Recency window** — default `last 18 months`. Older claims must be flagged.
3. **Number of candidate signals to surface in Phase 1** — default `5–10`.

If invoked without arguments, ask all three. If invoked with arguments (e.g. `/discovery-scout transport`), accept defaults for the unspecified ones.

## Signal Classification (READ THIS BEFORE PHASE 1)

The Discovery Process doc classifies "Government Calls / Sandboxes" as a **Strong** signal channel because the government has already diagnosed the problem and a Suhail human has typically walked through the sandbox call with the regulator.

**Override for this skill:** when the source is a public URL the agent fetched (not a human-to-human conversation), classify the signal as **Weak**. Signal strength is about Suhail's proximity to the source, not the channel taxonomy. Internet-scraped signals have not been validated by a Suhail conversation with the regulator, so they sit at the user-validation distance of a Weak source.

This calibrates downstream Stage 1.2 User Research to **5–8 interviews** (Weak path), not 3–5 (Strong path).

## Strong-Signal Solution Trap (READ THIS BEFORE PHASES 1 AND 3)

Quoted verbatim from the Discovery Process doc:

> Strong signal sources typically come bundled with an implicit or explicit specification of the *solution* — what the source thinks should be built. A government call for projects defines technical requirements that may reflect procurement preferences rather than end-user workflow. Inheriting that framing without checking with the end user produces the **right problem and the wrong product**.

**How to apply:** when reading a sandbox or open call, **explicitly separate** the *problem* the regulator surfaces (the underlying need that exists in the world) from the *solution* the regulator proposes (the technical implementation in the call). The Problem Signal you record is the problem, not the proposed solution. The analog scan in Phase 3 then looks for any startup that solved the *problem* — not necessarily the way the regulator proposed.

---

## What Suhail Wants — autonomous-mode filter rules

These rules encode Suhail's preferences for what makes a venture worth pursuing. They drive the composite signal-selection score at Phase 2 (autonomous mode) and the quality gates at Phase 6. In interactive mode they show up as informational tags but the human still decides.

| Rule | Value | Type |
|---|---|---|
| **Sector edge (Q8)** | RE + NGOs primary (`SUHAIL-EDGE: STRONG`); FinTech + HealthTech + EdTech secondary; Mobility / Retail / Marketplace tertiary (`SUHAIL-EDGE: WEAK`) | filter + score |
| **Geography** | KSA-first required for launch. GCC / MENA expansion allowed in the `Geography` multi-select but not the launch market. | filter |
| **Business model preference** | B2B, B2G, SaaS, Subscription preferred (recurring revenue). B2C deprioritized but not blocked. | score |
| **Vision 2030 alignment** | Required — must name a pillar AND a specific program (NTP, NCNP, NTLS, NIDLP, GAA Strategy, etc.). Pillar-only alignment is insufficient. | filter |
| **Moat threshold** | Must pass the Phase 4 3-incumbent pressure test with ≥2 *real* moat answers (regulatory carveout, proprietary data, channel access, patient capital, sovereign-backed positioning — NOT sales pipeline, local team, first-mover). | filter |
| **Source recency** | ≥80% of cited sources <18 months old. Older sources allowed but must be flagged. | quality gate |
| **Source count** | ≥5 cited sources required for autonomous-write. | quality gate |
| **Source diversity** | At least one sector-press source (Wamda, MAGNiTT, Fast Company ME, Sifted, TechCrunch, Rest of World, regional industry press), not just gov/regulator. | quality gate |
| **Capital ceiling** | Initial venture capital commitment ≤ $30M USD (typical Series A range). CAPEX-heavy ventures (warehouse builds, hardware, regulated infrastructure) trigger asset-shape adjustment in Phase 2. Placeholder — Suhail to tune. | filter + score |
| **Time-to-revenue** | First recognized revenue ≤ 12 months from MVP launch. Ventures with longer paths must compensate with higher moat scores. Placeholder — Suhail to tune. | filter + score |
| **Sector saturation cap** | Max 3 active research/testing ventures per `Primary Sector` in the Discovery Pipeline at any time. New venture in a saturated sector takes a −2 composite penalty per excess venture. | filter |
| **Portfolio synergy/cannibalization** | Compare new venture against active portfolio (Sahl+, Waqf Ops Platform, Multi-Authority Approval, etc.). Cross-sell potential → +3 bonus. Customer/channel overlap that cannibalizes a portfolio company → −5 penalty. | filter + score |
| **Asset shape** | Classify venture as **asset-light SaaS** (+5 composite bonus), **mixed** (0), or **asset-heavy** (−3 composite penalty). CAPEX-heavy ventures need higher composite + moat to compensate. | filter + score |
| **Operator pipeline** | Does Suhail have spin-out CEO/operator talent for this venture? Manual rule for now; document as Stage 1.2 gate question rather than autonomous filter. | manual |
| **Investor signaling** | Have any Suhail LPs expressed interest in this sector? Manual rule for now; document as Stage 1.2 gate question rather than autonomous filter. | manual |
| **Sandbox-anchored (H2 2026 priority)** | Venture fits an active/announced KSA regulatory sandbox: TGA Regulatory Sandbox, SAMA Regulatory Sandbox, CITC Innovation Sandbox, MoH innovation program, or sector-specific sandbox. Sandbox enrollment is the cleanest regulatory path to launch within 6 months. Name the specific sandbox + which eligibility criteria the venture fits. | filter + score |
| **Proven international analog (H2 2026 priority)** | At least one international analog must meet: profitable AND >5 years operating, OR post-Series B AND >100K paying customers, OR >$50M ARR. De-risks the venture archetype. Listing 5 analogs in Phase 3 is not enough — at least one must be demonstrably scaled. | filter + score |
| **Launchability (time-to-MVP, H2 2026 priority)** | Estimated time-to-MVP ≤ 6 months from greenlight favored; >12 months penalized. Proxies the agent uses: asset-light SaaS, sandbox-cleared regulatory path, off-the-shelf tech stack, accessible launch partnerships. CAPEX-heavy and infrastructure ventures rarely meet this. | filter + score |

---

## Phase 0 — Learn from Past Runs (autonomous mode only)

Skip this phase in interactive mode.

In autonomous mode, before any web search, query the Notion Discovery Pipeline for past entries:

1. Fetch all rows in `collection://5e6f6355-ec64-4950-b1cb-66806ef24401` where `Status` is one of: `Advance to MVP`, `MVP`, `GTM`, `Killed`, `Iterate or Pivot`.
2. For each past row, extract: `Primary Sector`, `Source Type`, `Geography`, `Competitive moat`, `Status`, `Decision`, `Decision Reasoning`, `Conclusion`, `Idea`.
3. Classify outcomes:
   - **Advancers** — Status in {Advance to MVP, MVP, GTM} OR Decision = Advance to MVP / Conditional Advance
   - **Kills** — Status = Killed OR Decision = Kill
   - **Pivots** — Status = Iterate or Pivot
4. Extract patterns:
   - Which `Primary Sector` values have produced advancers vs kills?
   - What moat patterns appear in advancers vs kills?
   - What "Decision Reasoning" / "Conclusion" themes recur in kills?
5. Apply patterns as soft heuristics to the current run:
   - **Sector bonus**: +3 on Phase 2 composite for any signal whose `Primary Sector` matches a past advancer; −3 for any sector with ≥2 past kills.
   - **Idea-similarity penalty**: −5 on Phase 2 composite if the proposed `Core Problem` text overlaps semantically with a past Killed venture's `Core Problem`.
   - **Moat-stringency**: if kills frequently cited weak moats in `Decision Reasoning`, autonomous Phase 4 raises the moat threshold from ≥2 real answers to ≥3 real answers.

Calibrate by past-entry count:
- **<5 past entries:** apply heuristics weakly (halve all bonuses/penalties).
- **5–20 past entries:** apply at full weight.
- **>20 past entries:** weight by recency too — last 6 months count 2x.

If no past entries exist, skip pattern extraction silently and proceed to Phase 1.

**Always log which past-run patterns influenced the current run** — in the Review Notes field on the written Problem Signal row, list the named past ventures that influenced selection. Traceability is non-negotiable.

---

## Phase 1: Problem Discovery (the sandbox scan)

**Goal:** surface 5–10 candidate Problem Signals from the chosen ministry's public sandbox / innovation / open-call surface.

### Sources to scan (in priority order)

1. **Ministry sandbox or innovation page** for the chosen sector.
2. **Vision 2030 sector strategy / programs** at https://www.vision2030.gov.sa (e.g. National Transport and Logistics Strategy for Transport).
3. **Regulator press releases and recent statements** — last 18 months.
4. **GASTAT (General Authority for Statistics)** and **Saudi Open Data** for problem-magnitude signals.

Starting URLs by ministry (use these as seeds, then expand with WebSearch):
- **Transport:** `tga.gov.sa`, `mot.gov.sa`, `gaca.gov.sa`
- **Health:** `moh.gov.sa`, MoH Innovation Center pages
- **NGOs:** `hrsd.gov.sa` (non-profit licensing), `ndc.gov.sa` (National Center for Non-profits)
- **Hajj:** `haj.gov.sa`, `gph.gov.sa`
- **FinTech:** `sama.gov.sa`, `cma.org.sa`
- **Telecom/Tech:** `cst.gov.sa`, `ntdp.gov.sa`

### Source resilience (Gulf geo-blocking)

Direct `WebFetch` on `.gov.sa` domains and Gulf news sites (Daleel News, GCC Business News, some Wamda pages) frequently fails with `ECONNREFUSED` or `403 Forbidden` — these sites geo-restrict or block non-residential / non-Gulf IPs. Confirmed during the 2026-05-29 Transport smoke test: 3 of 4 first-pass fetches failed; WebSearch snippets saved the run.

**Fallback chain when WebFetch fails on a `.gov.sa` or Gulf source:**

1. **Don't retry the same URL.** Switch immediately to `WebSearch` with the page title or topic keywords.
2. **Use the snippet content** Google returned — it usually mirrors the relevant section of the blocked page.
3. **Cite the WebSearch result** with the original `.gov.sa` URL **plus** a note `(via search snippet; direct fetch blocked)` in `Source Detail`. This preserves provenance.
4. **Try English-language mirrors** if search snippets are thin: Saudi Press Agency (`spa.gov.sa`), Arab News, TradeArabia, Reuters, Bloomberg — these usually mirror regulator announcements and are not geo-blocked.
5. **If all three fail,** mark the claim `[UNVERIFIED — fetch + search + mirror all blocked]` and surface as an open question in the Phase 5 brief.

**Do not fabricate** to fill the gap. Empty source > invented source.

### Method

For each source:
1. `WebFetch` the page and extract problem-shaped statements: things that are broken, painful, delayed, expensive, risky, or underserved.
2. For each candidate signal, capture:
   - **Problem Statement** — what's broken (the problem, not the solution)
   - **Biggest Pain** — the outcome the user/buyer cares most about
   - **Roles / Segments** affected
   - **Context** — sector, location, situation
   - **Source Detail** — URL + access date (today's date)
   - **Frequency / Signal Strength** — how often this surfaced, across how many sources

### Output of Phase 1

A markdown table of 5–10 candidate signals, ranked by signal strength (frequency across sources + recency + magnitude implied by GASTAT/Open Data). Each row has a one-line "why this matters" and the source URL.

Apply the Strong-Signal Solution Trap guard: if the source specified a solution, your Problem Statement is the underlying problem only — not the proposed implementation.

**Regulator-Named-Solution flag.** If the ministry has already publicly named and started building a specific platform/initiative as the solution (e.g. MOTLS's "Unified Documents Platform" for logistics document workflows), tag the signal `[REGULATOR-NAMED-SOLUTION]` in the shortlist. This is a stronger condition than the general Solution Trap — the regulator has not just proposed an approach, they have committed budget and named a deliverable. Signals tagged this way are **partner / integrator plays**, not greenfield ventures. The Phase 2 human selector should see this tag before picking, so they know the venture shape is constrained going in.

**Single-source flag.** Any signal whose sources count to 1 gets `[SINGLE-SOURCE]` appended — the two-source rule has not yet been satisfied. The signal can still be selected at Phase 2, but the Phase 3 analog scan must produce the second source before the brief is assembled.

---

## Phase 1b: KSA Competitive Pre-Scan (lightweight)

**Why this exists:** the Phase 2 human selector needs to know which signals are saturated **before** picking — otherwise the deep Phase 3 analog scan and Phase 4 KSA check waste cycles on a signal the user would have dropped if they'd known. Added 2026-05-29 after the Transport smoke test surfaced FASAH/Tabadul as a sovereign incumbent for clearance + documents, and SWVL's exit as an opening for on-demand buses — both of which materially changed the pick.

**This is a pre-scan, not the deep Phase 4 check.** One WebSearch per signal, top-3 actors only. Output one column added to the Phase 1 shortlist table.

### Method

For each signal in the Phase 1 shortlist, issue one WebSearch covering Wamda + MAGNiTT + KSA app stores + sector-press (e.g. logisticsmiddleeast.com, fastcompanyme.com, argaam.com). Query template:

> `Saudi Arabia <problem keywords> startup OR operator OR platform 2025`

Capture top-3 active players (name + one-line positioning). If a sovereign platform exists (FASAH, Tabadul, Riyadh Bus, NEOM-operated, etc.), name it explicitly — sovereign incumbents change the venture shape from greenfield to partner/integrator.

### Tagging

For each signal, append a competition tag to the shortlist:

- **`OPEN`** — no active player found in KSA / GCC, or a previous player has exited (e.g. SWVL leaving KSA → on-demand buses became OPEN).
- **`PARTIAL`** — 1–3 active players, vertical / segment / geographic gaps exist.
- **`SATURATED`** — 3+ active well-funded players covering the obvious gaps.
- **`SATURATED + capex-heavy`** — saturated AND high entry cost (e.g. AV with Jahez + Lucid + PIF). Hard for a startup.
- **`SATURATED (sovereign)`** — a government / state-backed platform owns the layer (e.g. FASAH for customs). Almost always combined with `[REGULATOR-NAMED-SOLUTION]`. Venture shape is partner/integrator only.

### Venture-shape note

When tagging `SATURATED (sovereign)` or `[REGULATOR-NAMED-SOLUTION]`, add a one-line **Venture shape** field describing the constrained path: "Must integrate with FASAH" / "Add-on layer on top of carriers" / "Tier-2 city focus only" — so the human selector picks with the venture-shape constraint visible.

### Suhail Unfair-Advantage tag

After the competitive tag, append a **Suhail edge tag** per signal:

- **`SUHAIL-EDGE: STRONG`** — signal sits in **Real Estate** or **NGOs / non-profits**, where Suhail's investor + contact network is concentrated. These sectors have a real Q8 unfair advantage.
- **`SUHAIL-EDGE: WEAK`** — signal is in Mobility, FinTech, HealthTech, EdTech, or other sectors where Suhail does not have a meaningful Q8 edge. The venture would compete on generic capabilities (sales pipeline, local team, first-mover speed) against deeper-pocketed incumbents.

Add this tag to the Phase 1 shortlist table before the Phase 2 selection gate — so the human picks knowing whether Suhail has a credible structural edge. A signal can still be selected with `SUHAIL-EDGE: WEAK`, but the Phase 4 moat pressure test must produce a genuine compensating moat for that venture to advance.

### What this skips

Phase 1b is intentionally shallow. **It does not include** founder names, exact funding rounds, market-share splits, churn data, or regulatory deep-dive — those are Phase 4. Stop at the tag + top-3 + venture-shape line.

---

## Phase 2: Selection Gate

### Interactive mode (default)

Present the Phase 1 shortlist via `AskUserQuestion`. Format: lettered options A/B/C/... with the one-line "why this matters" per option. The user picks **one** signal to advance.

**Do not proceed without an explicit selection.** If the user dismisses the shortlist, end the run.

### Autonomous mode (replaces the human gate when `--autonomous`)

Do NOT ask the user. Score each candidate signal from Phase 1 + 1b with the composite formula below and pick the highest. Log the score breakdown per signal for traceability.

```
SCORE = 0
  +10  if SUHAIL-EDGE: STRONG
  + 5  if KSA-Competition: OPEN
  + 2  if KSA-Competition: PARTIAL
  - 5  if KSA-Competition: SATURATED
  -10  if [REGULATOR-NAMED-SOLUTION]
  + 3  if sources_count >= 3 independent
  + 2  if quantified magnitude in macro evidence (a hard number from GASTAT, World Bank, official ministry data)
  - 5  if any cited source > 18 months old
  + 5  if Vision 2030 alignment includes a NAMED program (not just a pillar)
  + (Phase 0 sector bonus, weighted by past-entry count per the Phase 0 calibration)
  - (Phase 0 sector penalty)
  - 5  if Phase 0 idea-similarity penalty triggers (Core Problem overlaps with a past Killed venture)

  # v0.3.0 additions (Theme 2 + Theme 5):
  + 5  if asset shape = asset-light SaaS (low CAPEX, high gross margin)
  + 0  if asset shape = mixed
  - 3  if asset shape = asset-heavy (warehouse builds, hardware, regulated infrastructure, fleet)
  + 3  if portfolio-synergy bonus (cross-sell potential with active Suhail portfolio venture)
  - 5  if portfolio-cannibalization penalty (customer/channel overlap that competes with a portfolio company)
  - 2 per excess if sector saturation cap exceeded (>3 active research/testing ventures in same Primary Sector)
  - 3  if estimated time-to-revenue > 12 months without compensating moat strength
  - 5  if estimated initial capital commitment > $30M USD (above ceiling)

  # v0.4.0 additions (H2 2026 launch-priority rules):
  + 5  if sandbox-anchored (venture fits a named KSA government regulatory sandbox — TGA / SAMA / CITC / MoH / sector)
  - 5  if no qualifying sandbox exists for the venture's sector (unclear regulatory path → can't launch in 6 months)
  +10  if proven international analog (≥1 analog that is profitable + >5yr, OR post-Series B + >100K customers, OR >$50M ARR)
  - 5  if no qualifying scaled analog (venture archetype not yet de-risked at scale internationally)
  + 5  if estimated time-to-MVP ≤ 6 months from greenlight
  - 5  if estimated time-to-MVP > 12 months from greenlight

  # v0.5.0 adjustments (1-MVP-per-month cadence simplification):
  # Sandbox rule weakens: sandbox enrollment alone takes weeks, so it's not required for a 30-day MVP.
  # Replace v0.4.0 sandbox lines with:
  + 3  if sandbox-anchored (informational bonus — good optionality for later, not required for MVP)
  + 0  if no sandbox exists for the venture's sector (no penalty in v0.5.0)
  # The proven international analog rule stays at +10 / -5 (analog de-risking is still important).
  # The time-to-MVP rule is SUPERSEDED by the Phase 5c 30-Day MVP Test (binary PASS/FAIL).
  # Remove the v0.4.0 time-to-MVP +5/-5 entries from the composite; they live in Phase 5c now.
```

**Tie-breakers**, in order:
1. Higher SUHAIL-EDGE rank wins (STRONG > WEAK).
2. Larger quantified market-size magnitude in macro evidence wins.
3. Most-recent primary source wins.

**Abort condition.** If the top candidate's composite score is < 5, abort before Phase 3 and proceed straight to Phase 6 in **Failed-Quality-Gate** mode (write a Killed diagnostic row instead of a healthy venture row). The `Conclusion` field gets: "Autonomous mode: top signal composite score below threshold (<5). Re-scope ministry or run interactive mode."

**Log the composite score breakdown** for every signal in the eventual `Review Notes` field. The selection logic must be reconstructible from the Notion record alone.

### Phase 2b — Portfolio overlap + sector saturation (autonomous mode, runs after Phase 2 selection)

After the autonomous selection picks a winner but before proceeding to Phase 3, run two checks against the active Discovery Pipeline state.

**1. Sector saturation check.** Query the Notion Discovery Pipeline for all rows where `Status` is in `{Sourcing, Research, Testing, Decision Pending}`. Count rows per `Primary Sector`. If the winning signal's intended `Primary Sector` already has ≥3 active entries, apply a `−2 composite penalty per excess venture` (so 4 active = −2, 5 active = −4, etc.). If the adjusted composite drops below 5, abort and proceed to Killed-row fallback at Phase 6 with `Conclusion = "Sector saturation: <N> active ventures already in <Primary Sector>; new venture below adjusted composite threshold."`

**2. Portfolio overlap detection.** For each active portfolio venture (Sahl+, Waqf Operations Platform, Multi-Authority Approval Workflow, Warehouse-as-a-Service, plus any new ones), compare the new candidate's `Core Problem`, `Target Segment`, and `Value Proposition` for semantic overlap:
- **Cross-sell synergy** (e.g., new B2B SaaS targets the same customer the portfolio company already serves through a different product): **+3 composite bonus**. Note the synergy explicitly in `Review Notes`.
- **Channel cannibalization** (e.g., new venture competes for the same customer-segment dollars as a portfolio company): **−5 composite penalty**. If the penalty drops composite below 5, abort and write a Killed diagnostic row with `Conclusion = "Portfolio cannibalization risk with <named portfolio venture>: new venture targets <overlap dimension>. Re-scope or pivot."`

Log both checks' findings in the eventual `Review Notes` regardless of outcome.

---

## Phase 3: Analog Scan (non-Gulf startups solving the analogous problem)

**Goal:** for the selected problem, find startups in non-Gulf markets that solved the analogous problem. Output: 5–8 analogs with traction data.

### Geographic priority order

1. **LatAm** — Brazil, Mexico, Colombia, Argentina. Emerging-market parallels, mobile-first, regulatory-driven digitization.
2. **Türkiye** — most structurally Saudi-comparable: youth bulge, mobile-first, regulatory-driven digitization, recent fintech / mobility activity.
3. **United States** — best for category-leading benchmarks even when not directly transferable.
4. **Adjacent emerging analogs** — Indonesia, Egypt, India. Especially useful for Hajj / religious tourism / mass-density-event tech.

### Hajj exception

Hajj and Umrah have no direct geographic analog. For Hajj-sector problems, search for **functional analogs**:
- Mass-density event tech (Kumbh Mela in India, World Youth Day, Vatican / Lourdes pilgrimage tech)
- Religious tourism platforms (Israel, Vatican, India)
- Large-scale crowd-flow + safety systems (Olympics, FIFA World Cup, Hajj-adjacent academic research)

When using a functional analog, flag the comparability stretch explicitly in the output.

### Sources

- **Crunchbase, Dealroom, Tracxn** — search via WebFetch on their public pages (free tier). For paid tiers, the user has no API key as of 2026-05-29 — rely on public pages.
- **Regional press:** TechCrunch (global), Sifted (Europe), Rest of World (emerging markets), LatamList, Contxto (LatAm), Wamda + MAGNiTT (MENA/GCC — used in Phase 4, not here).
- **Founder blogs and Substacks** for primary-source traction claims.

### Per analog, capture

- Name, geography, founding year
- Business model (one line)
- Traction signals (funding rounds, users, revenue if public) — **two-source rule** for any quoted figure
- Key differentiator
- Why analogous (one sentence linking back to the selected problem)
- URL + access date

### Recency filter

Prefer sources <18 months old. Older sources get an explicit "(N years old)" flag in the output.

---

## Phase 3b — Customer Journey Today (mandatory, applies to both modes)

Before Phase 4 KSA competitive analysis, **put yourself in the target user's shoes** and trace what they actually do today to solve this problem. Listing competitors is not enough — the brief must walk through concrete steps a real user would take.

**Why this exists:** Without this, the agent overstates moats and misses obvious alternatives — especially generalist AI tools (ChatGPT, Claude, Perplexity) that may already produce "good enough" answers for free. Identified during the 2026-05-31 review of three autonomous outputs where the moats were thinner than written because the customer-journey research was missing.

### The five paths (explore at least 3)

For the specific target user defined in Phase 5.3 (`Target Segment`), trace:

1. **Google search path.** What query they would actually type (think like the user, not the founder). What the top results show. What they would likely settle on, and at what price/effort.
2. **AI chatbot path.** What they would ask ChatGPT / Claude / Perplexity / Gemini if they tried generalist AI first. Generate a representative prompt and reason about what response a frontier model would give. **Is the response "good enough" already?** If yes, the venture has thin AI-commoditization moat — flag it.
3. **Peer ask path.** What a peer in the same role (another charity director, another landlord, another developer) would tell them — what's the social-proof default in this community.
4. **Existing-platform path.** What happens when they try the closest existing tool (Donorbox already in Arabic, Salesforce NPSP free tier, Marafiqy at a low pricing tier, etc.). Does it work for them or not, and why?
5. **Workaround path.** What patchwork tools they cobble together today (Excel + WhatsApp + email + manual processes).

### What to document per path

For each path explored:

| Field | Content |
|---|---|
| Query / action | What they search or do |
| What they find | Top result(s) summarized |
| What's missing | Why they'd be unsatisfied |
| What they settle on | What they actually end up using |
| Cost / friction | Time + money cost of the current solution |

### The "is anything already good enough?" check

After tracing the paths, answer explicitly: **on any path, is the current solution "good enough" that a paying customer wouldn't switch?** If yes, the venture's moat must specifically explain why the new product is meaningfully better — not just "Saudi-localized" or "easier UX" but a concrete reason a user would switch from a free / acceptable status quo. Otherwise, downgrade the venture's verdict by one band (e.g., STRONG ADVANCE → CONDITIONAL ADVANCE).

### Output

Write a structured subsection in the Phase 5 brief titled "What [target user] does today" with ≥3 paths fully traced. Required.

---

## Phase 3c — Social + Competitive Recency Layer (X / Saudi Arabia)

**Critical framing — read this first.** Phase 3c does **NOT validate customer pain.** Saudi charity directors, real-estate developer ops leads, and waqf institution administrators rarely complain publicly on X — the sector is professional, NCNP-regulated entities have political cost to public criticism, and real complaints live in private channels (WhatsApp, board meetings). **Problem validation is the job of Stage 1.2 customer interviews, not Phase 3c.**

What Phase 3c DOES deliver:
- **Regulatory recency** — what ministries / authorities just announced (NCNP enforcement actions, HRSD platform migrations, REGA scope changes, etc.)
- **Competitive recency** — what named competitors are announcing in real-time (Naseej-Zoho partnership announced Jun 8, 2026; Inaya Medical College adopting MEDAD Jun 10, 2026; Salesforce Saudi careers ramp 2025)
- **Sector activity tracking** — who's posting from official accounts (@EhsanSA, @HRSD_SA, @ncnp_sa, @Naseej, @Medad_Cloud, @SaudiVision2030)
- **Historical / cached signal** — older but relevant posts (NCNP sanctions July 2025, official penalty schedules)

This phase runs in two complementary layers — both optional based on env vars:

### Layer 1: Perplexity Search API with x.com domain filter (PERPLEXITY_API_KEY)

**Strengths:** Cheap (~$0.015 for 3 queries). Good cached / historical signal — surfaces NCNP enforcement actions from mid-2025, government-account posts from late 2024, penalty schedules from 2024. Strong on `regulatory + historical` axis.

**Weakness:** X's 2023 auth-wall limits recent indexing. Posts <48 hours old typically missed. Cannot retrieve engagement metrics.

**Why this exists:** Saudi Arabia has one of the highest per-capita X penetrations in MENA. Government accounts (e.g., @EhsanSA, @HRSD_SA), sector aggregators (e.g., @saudifnpos), individual users, and Saudi influencers all post Arabic + English signal about specific sectors. The 2026-06-11 test on Donor CRM surfaced live evidence of NCNP enforcement actions (18 sanctions in 2024-2025), HRSD's "تبرع" platform migration to national rails, official penalty schedules, and sector-aggregator activity — none of which appeared in standard WebSearch or Sonar/Deep Research results.

### The three queries (all use Perplexity Search API with `search_domain_filter: ["x.com", "twitter.com"]` + `country: "SA"`)

| Query | Goal | Example topic shape |
|---|---|---|
| **1. Pain validation** | Real voice-of-customer about the problem. Mix Arabic + English keywords. | `["Saudi <user-role> <problem-word> complaints", "<Arabic problem phrase>", "<sovereign-platform> user feedback"]` |
| **2. Regulator / government voice** | Official announcements, enforcement signals, partnership news from the relevant ministry/authority. | `["@<ministry-handle> <sector> 2026", "<authority> <sector> regulation announcement", "<Vision 2030 program> tweets"]` |
| **3. Competitive sentiment** | What's said about named competitors / alternatives. Sentiment + adoption signals + complaints. | `["<competitor> Saudi adoption", "<competitor> customer feedback", "<alternative-platform> review"]` |

Each query returns 10-15 X posts. Synthesize into a `Social Signal Scan (X / Saudi Arabia)` subsection of the Phase 5 brief with:
- 2-3 most relevant posts per category (URL + post date + 1-line summary)
- 1-line takeaway per category about what the signal tells us
- An "Implications for the venture" closing bullet that ties social signal to the existing competitive moat / customer journey analysis

### Bash pattern (Perplexity-X)

```bash
# Three Search API calls per venture (pain validation, regulator voice, competitive sentiment), $0.005 each
curl -s -X POST https://api.perplexity.ai/search \
  -H "Authorization: Bearer $PERPLEXITY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": ["<sector keywords + complaint terms>", "<arabic equivalent>", "<sovereign-platform mentions>"],
    "max_results": 15,
    "country": "SA",
    "search_domain_filter": ["x.com", "twitter.com"],
    "search_context_size": "medium"
  }'
```

### Layer 2: Grok-Top via xAI x_search tool (XAI_API_KEY) — v0.7.0+

**Strengths:** Native X access (no auth-wall — xAI owns X). Real-time / very recent (posts from yesterday surface). Constructs sophisticated bilingual Boolean queries with `lang:ar` + `since:` filters automatically. Found Naseej-Zoho partnership announcement (Jun 8, 2026) + Inaya Medical College MEDAD adoption (Jun 10, 2026) in 2026-06-11 test — competitive intel **neither Perplexity nor Deep Research found**.

**Weakness:** Costlier (~$1.20 per query with 5-6 internal x_search calls). Default "Latest" mode prioritizes chronology over relevance — must explicitly instruct "Top" mode in the prompt to get relevance ranking. Verbose output requires synthesis.

**Critical prompt instruction (always include):**

> "When using x_keyword_search, ALWAYS set mode to 'Top' (relevance ranking), NEVER 'Latest' (chronological). The most relevant Saudi charity / RE / venture-sector posts are often older posts with high engagement, not the newest posts."

### Bash pattern (Grok-Top)

```bash
# One Grok x_search call per venture, ~$1.20 each
cat > /tmp/grok_body.json <<'JSON'
{
  "model": "grok-4.3",
  "stream": false,
  "input": [
    {
      "role": "user",
      "content": "IMPORTANT: When using x_keyword_search, ALWAYS set mode to \"Top\" (relevance ranking), NEVER \"Latest\".\n\n<the Phase 3c competitive-recency question, with named competitors, regulators, ministry handles, and date floor `since:2024-01-01`>"
    }
  ],
  "tools": [{"type": "x_search"}]
}
JSON
curl -s -X POST https://api.x.ai/v1/responses \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  --max-time 180 \
  --data-binary @/tmp/grok_body.json
```

The xAI `/v1/responses` endpoint returns an `output` array with reasoning steps + `custom_tool_call` items + a final `message` item. Parse for `output[].type == "message"` to get the synthesized text. Cost is in `usage.cost_in_usd_ticks` (divide by 1e9 to get USD).

### When to use which layer

| Situation | Run Perplexity-X | Run Grok-Top |
|---|---|---|
| Default autonomous run (cost-conscious) | ✓ | ✗ |
| IC-bound brief (max information) | ✓ | ✓ |
| Venture has identified incumbent worth monitoring | ✓ | ✓ |
| Competitive landscape stable, no recent moves | ✓ | ✗ |
| Recent competitor announcement suspected | optional | ✓ |

If both env vars are set, **run both by default in IC-bound autonomous runs** (set via `--ic-bound` flag in argument). Otherwise, Perplexity-X only.

### Limitations (document in every brief that uses Phase 3c)

- **Perplexity-X cache lag:** posts <48 hours old typically missed. Mostly cached snapshots.
- **Grok cost:** $1.20 per call dominates Phase 3c cost — only run when justified.
- **No engagement metrics in either layer:** likes / retweets / replies counts only available via direct X API ($100/mo Basic), not via Perplexity or Grok x_search.
- **Not problem validation:** the X signal in this sector is regulatory + competitive, NOT customer-complaint. Stage 1.2 interviews remain the problem validator.

**Future upgrade path (v0.8.x candidate):** Phase 3d — LinkedIn voice-of-customer scan. Saudi charity directors, RE developer ops leads, waqf admins are more active on LinkedIn (professional context allows discussing operational challenges) than X. Requires LinkedIn API or scraper subscription. Higher probability of actual problem validation than X.

### Output to Notion

Append a `## Social + Competitive Recency Scan (X / Saudi Arabia)` section to the Discovery Pipeline row's page body, after "What [target user] does today". Structure:

1. **Brief framing reminder** — "This section surfaces competitive + regulatory recency from X. It does NOT validate customer pain — that requires Stage 1.2 interviews."
2. **Perplexity-X findings** (if PERPLEXITY_API_KEY set) — pain proxies + regulator voice + competitive sentiment (3 sub-sections, ~3-5 posts each)
3. **Grok-Top findings** (if XAI_API_KEY set + IC-bound run) — competitive recency layer with named announcements, partnerships, customer adoptions found in last 30 days
4. **Implications for the venture** — 1-paragraph synthesis of both layers' implications.

---

## Phase 4: KSA Competitive + Regulatory Check

**Goal:** for the top 3 analogs from Phase 3, check the KSA / GCC competitive landscape and the regulatory picture.

### Per top-3 analog

**Competitive check** — is anyone in KSA / GCC already doing this?
- Search Wamda, MAGNiTT, app stores (Google Play KSA, App Store KSA), LinkedIn, regional press.
- Tag: `saturated` (3+ active players) / `partial` (1–2 with gaps) / `open` (no active player found).

**Regulatory check** — what does KSA regulation say?
- **Relevant ministry / authority / regulator** (named).
- **Licenses, permits, sandboxes** that apply.
- **Saudization** requirements (likely percentage by sector).
- **Foreign-ownership** rules (MISA).
- **Data residency** obligations.
- Tag the regulatory picture: `enabling` / `constraining` / `fatal blocker`.

**Vision 2030 alignment** — which pillar (Vibrant Society / Thriving Economy / Ambitious Nation) and which named program (NTP, FSDP, NTLS, etc.).

### Moat pressure test (required — applies to the venture, not the analogs)

After the KSA competitive + regulatory + Vision 2030 reads are done, run an explicit moat pressure test for the proposed venture shape. **Ask:**

> **"Why wouldn't [obvious incumbent] build this themselves?"**

Name 2–3 candidate incumbents (e.g. Uber / Careem for ride-hail, FASAH / Tabadul for customs software, Aramex / SMSA for delivery, the chosen ministry itself for regulator-named-solution plays, or the closest international analog if they've stated KSA expansion intent — e.g. Volt Lines for corporate commute). For each, write a one-sentence honest answer.

**Weak moat answers** (do NOT count as real moats):
- "We have a better B2B sales pipeline"
- "We have local relationships / better local team"
- "We are first-mover"
- "We move faster than incumbents"

**Real moat answers:**
- Regulatory carveout the incumbent can't access (sandbox slot, license restriction, foreign-ownership cap)
- Proprietary data or IP the incumbent doesn't have
- **Channel access via Suhail's RE / NGO network** (per Phase 1b `SUHAIL-EDGE` tag) — building access, NGO endorsement, board-level intro
- Patient-capital structure / mission alignment that outlasts incumbent willingness to subsidize
- Sovereign-backed positioning the commercial incumbent can't credibly claim

If all moat answers are weak, **tag the venture `[MOAT: WEAK]`** in the Phase 5 brief. The brief should explicitly recommend the human reconsider before committing to Stage 1.2 user research — either pivot the venture shape to leverage a real moat, or drop the signal and pick another from the shortlist.

### Sovereign Incumbency Risk (required — applies to the venture)

Saudi venture-building under Vision 2030 carries a systematic risk: **sovereign entities may extend their scope and absorb private opportunities**. Examples observed across earlier runs:
- **Naseej / REGA** could extend the GAA Awqaf portal scope to include institution-side operations, shrinking the Waqf Operations Platform's space.
- **NIDLP / MoMRAH** could expand the 59 planned logistics centers to include operator-level services, shrinking the Warehouse-as-a-Service Marketplace's space.
- **FASAH / Tabadul** owns the customs trade-flow layer; vertical SaaS on top remains possible but bounded.

For every venture, assess:

| Dimension | Score | Note |
|---|---|---|
| **Likelihood that a sovereign entity extends scope to absorb this venture (24 months)** | LOW / MED / HIGH | Cite the named sovereign actor + their current roadmap signals |
| **Time-to-impact estimate if sovereign extends** | <6 mo / 6-18 mo / >18 mo | Based on procurement / build timelines + announced strategy |
| **Defensive moves available to Suhail** | List | E.g., partner with the sovereign actor early; deeper customer lock-in via long-term contracts; depth in a niche the sovereign won't address; M&A path to sovereign-incumbent |

Add this as a structured subsection in the Phase 5 brief titled "Sovereign Incumbency Risk." If **likelihood = HIGH AND time-to-impact ≤ 18 mo AND no credible defensive moves**, downgrade the venture's autonomous composite score by **−5** (in addition to all other adjustments). This may push it below the abort threshold.

---

## Phase 5: Draft Assembly

Build three artifacts:

### 5.1 The Desktop Research brief (markdown, ~1–2 pages)

This matches the Stage 1.1 structure from the Discovery Process doc verbatim, **plus a plain-English opener and a glossary** so the brief is readable cold by a non-domain reader.

**Readable-cold rule (mandatory, v0.7.1+ strengthened):**

The IC reader will most likely have *no specific Saudi-sector context* beyond "venture builder." Every brief — including every revision section added later (v0.6.x, v0.7.x, etc.) — must stand alone without forcing the reader to do external research on referenced entities.

1. **Open with "What this venture is — in plain terms"** (1-2 paragraphs). Explain the core concept as if to a smart reader with zero domain context. Define every specialized term on first use.

2. **MANDATORY "Cast of Characters" table immediately after the opener.** A structured table listing every named entity referenced anywhere in the brief: regulators, ministries, sovereign platforms, vendors, named competitors, Saudi customer-target organizations, X / handles. Columns: `Entity / Type / One-line description / Why it matters to THIS venture`. The reader uses this as a roadmap. **When ANY revision section is added later, the new entities referenced go into this table FIRST, then are used in the section.** No new entity may be referenced in the body of the brief before it exists in the Cast of Characters.

3. **Expand every acronym on first use** — `GAA (General Authority for Awqaf)`, `NCNP (National Center for Non-Profit Sector)`, `MISA (Ministry of Investment Saudi Arabia)`. After expansion the acronym can stand alone *within the same section* — but when a new major subsection introduces it again, **expand again on first mention of the subsection**. This is intentional repetition: each subsection should be readable standalone.

4. **First-mention parenthetical per major subsection (NEW v0.7.1).** "Naseej (Saudi technology vendor; won the GAA Awqaf portal contract April 2025; now also Zoho Authorized Partner)" on its first mention in `Industry & Why Now`, AGAIN on its first mention in `Competitive Landscape`, AGAIN on its first mention in `Phase 3c Social Signal Scan`. Yes, repetitive — but makes each section standalone. Glance-readers don't get lost.

5. **Explain every specialized concept** on first use — what a waqf is, what a regulatory sandbox is, what Vision 2030 is, what Sharia compliance means for software, what Saudization requires.

6. **Research-trace per major finding (NEW v0.7.1).** Don't just state findings — anchor each one to *how it was sourced*. Format: `<claim> — <tool that found it: WebSearch, Perplexity Sonar, Perplexity Deep Research, Perplexity X-filter, Grok x_search, manual cite>; <confidence: confirmed / likely / hypothesis / single-source>; <link or citation if applicable>`. Example: "Naseej-Zoho partnership announced Jun 8, 2026 — Grok x_search (Top mode), confirmed single-source from @Naseej announcement, https://x.com/Naseej/status/2063863599416648100".

7. **End with a Glossary section** listing every defined term in alphabetical order. Required. The Cast of Characters table (#2 above) covers WHO; the Glossary covers WHAT (concepts, acronyms, methodology terms).

The brief structure:

```markdown
# Desktop Research Brief — <Idea name>

*Generated by /discovery-scout on YYYY-MM-DD*

## What this venture is — in plain terms

[1-2 short paragraphs explaining the venture in language a non-domain reader can follow. Define every specialized term as it first appears.]

## Cast of Characters

Every named entity referenced in this brief, with its role and why it matters to this specific venture. Reader uses this as a roadmap. **When a revision section is added later, new entities go here FIRST.**

| Entity | Type | One-line description | Why it matters to THIS venture |
|---|---|---|---|
| [Entity 1, e.g., NCNP] | [e.g., Saudi sector regulator] | [e.g., National Center for Non-Profit Sector — Saudi government body that licenses charities and sets sector regulations] | [e.g., target customers are NCNP-licensed; compliance with NCNP rules is mandatory] |
| [Entity 2] | [e.g., Saudi vendor] | [...] | [...] |
| [Entity 3] | [e.g., Sovereign platform] | [...] | [...] |
| ... | ... | ... | ... |

Cover at minimum: every Saudi regulator/authority mentioned, every Saudi sovereign platform, every vendor (Saudi and global) mentioned, every named Saudi customer-target organization, every X handle referenced. Aim for 15-30 rows in a typical brief.

## Industry & "Why Now"

- **Market structure:** category definition, customer segments, where value is captured.
- **Key value-chain players:** customers, suppliers, distributors, platforms, regulators, incumbents.
- **Secular shifts driving demand:** demographic, technology, behavioral, capital-market, or policy changes.
- **Catalysts in the next 24–36 months:** regulatory deadlines, government programs, infrastructure changes, funding cycles.
- **Early implications for unit economics:** likely revenue model, ARPU range, sales cycle.

## KSA Regulatory Environment

- **Relevant regulator(s):** ministry, authority, municipality, sector regulator, or licensing body.
- **Required licenses, permits, approvals, data/privacy obligations, Saudization requirements.**
- **Applicable sandboxes, pilot programs, government calls for projects.**
- **Restrictions:** foreign ownership, pricing controls, data residency, sector-specific compliance.
- **Direction of travel:** tightening / loosening / unclear / active reform — with evidence.
- **Regulatory implication:** proceed / proceed with expert validation / sandbox-first / partner-required / potential fatal blocker.

## What [target user] does today (Customer Journey)

Output of Phase 3b. Mandatory section — without this, the brief overstates moats.

Trace at least 3 paths the target user would take to solve this problem today. Per path: query/action, what they find, what's missing, what they settle on, cost/friction.

1. **Google search path** — query + top results + what they settle on
2. **AI chatbot path** — what they ask ChatGPT/Claude/Perplexity + whether the response is "good enough"
3. **Peer ask path** — what a peer in the same role tells them
4. **Existing-platform path** — what happens when they try the closest existing tool (Donorbox, Marafiqy, Ehsan, etc.)
5. **Workaround path** — Excel + WhatsApp + email + manual processes

**The "is anything already good enough?" check** — explicitly answer: on any path, is the current solution good enough that a paying customer wouldn't switch? If yes, the moat section must specifically explain why our product is meaningfully better; otherwise downgrade the final verdict by one band.

## Competitive Landscape

- **3–5 closest competitors:** name, geography, customer segment, positioning, pricing if visible, funding/traction, apparent strengths.
- **Adjacent global benchmarks** (the top analogs from Phase 3).
- **White-space hypothesis:** what is underserved, underpriced, overcomplicated, inaccessible, or poorly localized for KSA.
- **Differentiation implication:** what Suhail could do differently through network, distribution, capital, operating assets, or local credibility.

## Why wouldn't [incumbent] build this?

For each of the top-3 candidate incumbents (named), write one honest sentence. If all three answers are weak (generic sales pipeline, local team, first-mover speed), prefix the brief with `[MOAT: WEAK]` and recommend the human reconsider before advancing to Stage 1.2.

| Incumbent | Why wouldn't they build this? |
|---|---|
| [Incumbent A — named] | [honest one-line answer] |
| [Incumbent B — named] | [honest one-line answer] |
| [Incumbent C — named] | [honest one-line answer] |

Weak answers (sales pipeline / local team / first-mover) do not count. Real answers: regulatory carveout, proprietary data, Suhail RE/NGO channel access, patient capital, sovereign-backed positioning.

## Stage 1.2 User Research Agenda (from /office-hours)

Run the six YC office-hours forcing questions (Q1 Demand Reality, Q2 Status Quo, Q3 Desperate Specificity, Q4 Narrowest Wedge, Q5 Observation & Surprise, Q6 Future-Fit) plus the Phase 3 premises against the brief. Classify each item into:

- **Answered from desk** — claims defensible from macro / regulatory / competitive evidence already in the brief, or from Suhail-network facts available without user research. State the answer here.
- **Flag for Stage 1.2** — must be tested with real users. For each: the research task, the success criterion at 1.2, and red flags that would disqualify a positive read.

Rule (per the office-hours-at-Stage-1.1 methodology): at desktop completion, office-hours questions become Stage 1.2 interview-guide items, not demand-validation verdicts. Most items will fall in the "flag for 1.2" bucket — that is the correct outcome at desktop completion. Do not force customer evidence that desk work cannot produce.

Structure:

### Answered from desk
- Q6 Future-Fit: [desk-derivable macro-trend answer with counter-trend to watch]
- Q3 (access half): [Suhail-network access path inventory — named contacts at named institutions, must be produced before 1.2 scheduling starts]
- Q4 (hypothesis half): [wedge candidate with rationale from regulatory timing or sector data]
- Premise A (sizing half): [conservative TAM math from desk; defensible at order of magnitude]
- [+ any others derivable from the brief]

### Flag for Stage 1.2
- Q1 Demand Reality — task: [interview design] · success: [specific behavioral signal] · red flags: [what to dismiss]
- Q2 Status Quo — task: [observation design] · success: [documented baseline workflows] · red flags: [vague answers]
- Q3 (KPIs half) — task: [buyer psychology mapping per named contact]
- Q4 (validation half) — task: [wedge willingness-to-pay test] · success: [LOI or verbal commit at price]
- Q5 Observation & Surprise — task: [embedded observation] · success: [≥2 surprises that change brief assumptions]
- Premises B, C, D, E — task + success criterion per premise

### How to use this agenda at Stage 1.2
1. Complete the desk-answerable items first — especially the Q3 access list. Without named contacts, there is no interview pipeline.
2. Use the Stage 1.2 task descriptions as the interview guide. Each forcing question maps to a conversation section.
3. Check each success criterion at the end of the 5–8 interview block. Any criterion not met → extend the research block or flag the gap in the IC Due Diligence Report.
4. If a desk-answered item is contradicted by user research, pivot the venture shape before Stage 2 testing.

## Sources Consulted

- Per claim, list: source URL, date accessed (YYYY-MM-DD), claim it supports.
- Flag any source >18 months old.
- Flag any claim that could not be sourced as `[UNVERIFIED]`.

## Glossary

List every acronym, named institution, and specialized concept that appeared anywhere in the brief, in alphabetical order, with a one-line plain-English definition. Required unless the brief is <300 words.
```

### 5.2 Problem Signal Capture row (draft as markdown table)

Fields to populate (these are the actual Notion property names — do not invent new ones):

| Property | Value |
|---|---|
| `Problem Statement` (title) | the problem, NOT the regulator's proposed solution |
| `Sourcing Channel` | `Government Calls / Sandboxes` |
| `Frequency / Signal Strength` | text — describe how often this surfaced + that it's Weak per the classification rule |
| `Signal Count` | number — independent mentions across sources |
| `Roles / Segments` | text — affected user/buyer roles |
| `Context` | text — sector, location, situation |
| `Biggest Pain` | text — outcome the user/buyer cares most about |
| `Desired Solution` | text — only if the source volunteered one (e.g. the regulator's proposed solution goes here, NOT in Problem Statement) |
| `Source Detail` | text — URL + access date + any quoted excerpt |
| `Date Captured` | date — today's date |
| `Owner` | leave blank for the user to assign |
| `Review Notes` | text — "Auto-captured by /discovery-scout. Signal = Weak per signal-classification rule (internet-sourced)." |

### 5.3 Discovery Pipeline row (draft as markdown table)

Fields to populate at this stage. **Q1–Q16 scores MUST be left blank** — they are IC-stage judgments, not Stage 1.1 outputs.

| Property | Value |
|---|---|
| `Idea` (title) | concise name for the venture, NOT the problem statement |
| `Core Problem / Pain Point` | text — restates the underlying problem |
| `Target Segment` | text — hypothesis only |
| `Value Proposition` | text — hypothesis only, format: "Our product helps [segment] who want to [job] by relieving [pain] and creating [gain]." |
| `Strategic Fit` | text — which Vision 2030 pillar / program; why it fits Suhail's thesis |
| `Primary Sector` | select — must be one of: FinTech, PropTech, HealthTech, EdTech, Mobility, Retail, Consumer App, B2B SaaS, Marketplace, Other |
| `Business Model` | multi-select — any of: B2B, B2C, B2B2C, B2G, Marketplace, SaaS, Transactional, Subscription, Hybrid |
| `Geography` | multi-select — default `KSA`; add `GCC (the Gulf)` / `MENA` / `Global` if regional expansion is plausible |
| `Source Type` | select — `Trend scan` |
| `Source Detail` | text — concise source string with URLs |
| `Problem Signal` | relation — set AFTER the Problem Signal Capture row is written (Phase 6) |
| `Market size` | text — TAM / SAM / SOM ranges for KSA with source/method per estimate |
| `Competitive moat` | text — what could make this defensible (network effects, proprietary data, regulatory access, distribution, switching costs, Suhail assets) |
| `Status` | select — `Research` |
| `Research Sub-stage` | select — `Desktop` |
| `Q1 Score` through `Q16 Score` | **LEAVE BLANK** — IC judgment, not agent output |
| `Deal Lead` | leave blank for the user to assign |

### Sector-mapping cheat sheet

When picking `Primary Sector`, map the ministry → sector:
- Transport (MoT/TGA/GACA) → `Mobility`
- Health (MoH) → `HealthTech`
- NGOs / non-profits → `Other` (note the gap in the schema in the run report)
- Hajj → `Other` (note the gap; "Religious tourism / mass-event tech" isn't in the enum)
- FinTech (SAMA/CMA) → `FinTech`
- Telecom (CST/NTDP) → `B2B SaaS` or `Consumer App` depending on the venture

### Business model defaults by ministry play

- Sandbox / regulator-driven plays → `B2G` likely, possibly + `SaaS` or `Marketplace`
- Mobility consumer plays → `B2C` + `Transactional`
- Health provider tools → `B2B` + `SaaS`
- Hajj operator tools → `B2B2C` + `Transactional`

---

## Phase 5b — Scoring

After the brief and row drafts are assembled, fill both scoring frameworks with explicit rationale per score. These are **desktop-only preliminary estimates**, marked `[STAGE-1.1 ESTIMATE]` on every score. The Notion DB will auto-compute Evaluation Score, Evaluation Band, Category Averages, Gate Pass formulas from the Q1–Q16 inputs.

### D/F/V/A evidence levels (1–5 each)

Use the Discovery Process doc's evidence ladder:

- Level 1 — What people say (interviews, stated interest)
- Level 2–3 — Tangible reactions (responses to concepts, mockups, storyboards)
- Level 4 — Call to action (sign-ups, demo requests, waitlist joins, referrals)
- Level 5 — Real commitment (LOIs, pre-orders, deposits, simulated purchase)

At Stage 1.1 (desktop) we have neither customer interviews nor reactions. **Evidence levels almost always cap at Level 1.** Any score ≥2 must cite specific desktop evidence that approximates that level (e.g., a published industry survey acting as proxy for "what people say").

| Field | Score | Rationale (one line, with [STAGE-1.1 ESTIMATE] suffix) |
|---|---|---|
| `Desirability Evidence` | 1–5 | Cite the strongest macro/regulatory proxy for user pull |
| `Feasibility Evidence` | 1–5 | Cite regulatory + technical buildability evidence |
| `Viability Evidence` | 1–5 | Cite economic / revenue model evidence |
| `Adaptability Evidence` | 1–5 | Cite market-environment + model-flexibility evidence |

### Q1–Q16 IC scorecard (1–5 each)

Score each question against the Discovery Process doc's anchored criteria, using ONLY desktop evidence. Most Stage 1.1 scores will be 2–3 (mixed signals, promising but unproven). Q1 (Vision 2030 alignment) and Q8 (Suhail unfair advantage) are the two most confidently scorable from desk. Cite the specific desktop evidence per score.

| Q# | Category | Score | Rationale |
|---|---|---|---|
| Q1 | Strategic Alignment — Vision 2030 fit | 1–5 | Named Vision 2030 program(s) + alignment to Suhail thesis |
| Q2 | Strategic Alignment — Impact potential | 1–5 | Measurable impact dimensions (financial, social, sectoral) |
| Q3 | Market Attractiveness — Economics | 1–5 | SOM range, margin hypothesis, LTV:CAC hypothesis |
| Q4 | Market Attractiveness — Timing | 1–5 | Catalysts, regulatory window, headwinds vs tailwinds |
| Q5 | Differentiation — Differentiated value | 1–5 | White-space hypothesis + defensibility argument |
| Q6 | Differentiation — vs. alternatives | 1–5 | Named incumbent comparison |
| Q7 | Differentiation — ERRC logic | 1–5 | Eliminate / Reduce / Raise / Create deltas vs incumbents |
| Q8 | Differentiation — Suhail unfair advantage | 1–5 | SUHAIL-EDGE tag + specific Suhail network/capital/relationships |
| Q9 | Scalability — Beyond KSA | 1–5 | Regional/global replication path |
| Q10 | Scalability — Margin trajectory | 1–5 | SaaS-like vs ops-heavy long-run margins |
| Q11 | Partner Dependency — Critical partners? | 1–5 | Named dependencies (concentration & criticality) |
| Q12 | Partner Dependency — Strengthen or risk? | 1–5 | Net leverage vs concentration risk |
| Q13 | Partner Dependency — Accessible? | 1–5 | Suhail network access path |
| Q14 | Feasibility Flags — Constraints | 1–5 | Tech / ops / regulatory / GTM constraints |
| Q15 | Feasibility Flags — Team capability | 1–5 | Suhail capability vs venture needs |
| Q16 | Feasibility Flags — Structural blockers | 1–5 | Known blockers or "none identified" |

**Anti-inflation rule.** If the rationale for any score reads as a restatement of the score itself ("Score 4 because the venture has strong differentiation") with no cited evidence, drop the score by 1 and re-write the rationale to cite specific desktop evidence. If no evidence can be cited, the score is 1 or 2 by default.

### Output

Write all 20 scores + rationales into the Discovery Pipeline row. The Notion DB formulas (`Evaluation Score`, `Evaluation Band`, `Gate Category Floor Pass`, `Gate Desirability Pass`, `Gate All Pass`, `Stage Gate Recommendation`) will compute automatically from the Q1–Q16 inputs.

---

## Phase 5c — 30-Day MVP Test (1-MVP-per-month priority)

Beyond the IC scorecard, run the **30-Day MVP Test** — three binary questions that estimate whether Suhail could plausibly ship an MVP within 30 days. Replaces the v0.4.0 5-component Launchability Sub-Score (simpler, sharper bar matching the 1-MVP-per-month launch cadence per [[project-suhail-six-month-launch-priority]]).

### What "concierge MVP" means (read this first)

A **concierge MVP** is a startup pattern from *The Lean Startup* (Eric Ries). **Instead of building software for your first 10 customers, the team manually delivers the value by hand — like a hotel concierge does.** The MVP proves demand exists *before* you build software to scale it. The MVP is human-powered, not code-powered.

Famous examples:
- **Food on the Table** (canonical): the founder personally called each customer every week, asked what was in their pantry, wrote a meal plan by hand, emailed it. Customers paid. Only after ~100 paying customers did he build the software.
- **Airbnb 2008:** founders manually photographed every listing themselves with a borrowed camera. No "host photography platform."
- **Zappos (very early):** the founder photographed shoes at local stores, posted them online; when an order came in he walked to the store, bought the shoes, shipped them. No inventory, no warehouse — just him + a spreadsheet.

**Concierge MVPs for Suhail-archetype ventures (what to picture when scoring Q1):**

| Venture archetype | Concierge version for first 10 customers |
|---|---|
| Saudi rental management SaaS for individual landlords | Manually collect rent via bank transfer + WhatsApp reminders; generate invoices in Excel; deliver monthly statements by email. Charge $50/month per landlord. |
| Donation portal SaaS for individual charities | Set up each charity's donation page manually on Notion + payment link (Tap/HyperPay); process donations by hand into their accounting. |
| Zakat calculator + receipts SaaS for SMEs | Spreadsheet + email. Take SME's books, calculate zakat manually, send PDF receipt. Charge per receipt. |
| "Notion for Awqaf" | Build first 3 waqf institutions' Notion workspaces yourself + 2-hour training; charge for setup + monthly support. |
| Grant-application copilot for charities | Manually fill grant applications using LLM + the org's data. Charge per application. |

**The opposite of concierge** — venture shapes that **can't** be concierge'd because the software IS the value, so Q1 = NO:
- Multi-authority workflow automation (the integration *is* the product; you can't manually integrate 5 government APIs for 10 customers)
- Custom Sharia-compliant reporting engines (the engine *is* the value; manual reports for 10 institutions don't validate it)
- Physical infrastructure plays (warehouses, hardware, fleets — you can't manually run physical assets you don't own)

These are real businesses but they don't fit the 30-Day MVP Test — they need real engineering before customer #1.

### The three questions

| # | Question | What "YES" means |
|---|---|---|
| **1. Concierge-able?** | Can the core value be demonstrated as a **concierge service** (humans doing the work manually) for the first 10 customers? | No software build is required to validate the MVP. The team can ship value in days by doing the work by hand. Filters out venture shapes whose value is inseparable from the software (e.g., multi-regulator workflow automation). |
| **2. Off-the-shelf tech?** | Are the tools needed to ship the MVP **off-the-shelf** — no custom regulatory work, no hardware, no deep legal/Sharia review required as a blocker? | No multi-week dependencies. Available APIs (Tap, HyperPay, ZATCA-compliant invoicing libraries, Notion, Calendly, Stripe Atlas equivalents, etc.) cover the stack. Filters out CAPEX-heavy and sovereign-incumbent-integration-required ventures. |
| **3. Customers in 14 days?** | Can ≥3 prospect customers be reached **within 14 days via Suhail's existing network** (RE / NGO contacts, portfolio companies, investor relationships)? | No cold-sales cycle. The first MVP customers are warm intros, not outreach. Filters out ventures that require months of sales pipeline building before MVP traction. |

### Scoring

- **3/3 YES → PASS** → eligible for the H2 2026 sprint as a top-priority pick.
- **2/3 YES → PASS WITH CONDITION** → eligible IF the failing question has an explicit, credible workaround documented in the brief (e.g., "Q2 fails on Sharia review, but we can use existing pre-approved Sharia-compliant template X to bypass review for v1"). Mark the conditional clearly in `Review Notes`.
- **≤1 YES → FAIL** → BACKLOG. Real venture but not this month. Document in the brief why it fails the 30-day bar so future sessions can decide whether to revisit.

### Write to Notion

Add the 3 question answers + scoring outcome + the workaround note (if PASS WITH CONDITION) as a structured subsection in the Discovery Pipeline row's page body, titled "30-Day MVP Test." Also append the outcome (`PASS / PASS WITH CONDITION / FAIL`) to the `Review Notes` field for at-a-glance visibility.

### Why this is simpler than v0.4.0's 5-component sub-score

Three binary questions force the agent to take a clear position rather than averaging 1-5 component scores. A failure on any of the three is concrete (and recoverable, if the workaround is real). This matches the 1-MVP-per-month launch cadence — venture viability for the sprint is a yes/no question, not a sliding scale.

---

## Phase 6: Write Gate

### Interactive mode (default)

Post the three artifacts (brief + Problem Signal row + Discovery Pipeline row + the Phase 5b scoring tables) in chat. Then use `AskUserQuestion`:

> Approve writing these to Notion?
>
> A) Approve — write both rows and the brief to Notion
> B) Revise — point out what to change, I'll re-draft and re-ask
> C) Cancel — drop the run, write nothing

**On A (approve):**
1. Call `mcp__notion__notion-create-pages` with the Problem Signal Capture data source URL (`collection://fc466b72-eaec-4ac9-b692-5e745966f1d4`). Set all properties from 5.2. **Capture the returned page URL.**
2. Call `mcp__notion__notion-create-pages` with the Discovery Pipeline data source URL (`collection://5e6f6355-ec64-4950-b1cb-66806ef24401`). Set all properties from 5.3 + the Phase 5b D/F/V/A and Q1–Q16 scores. Set the `Problem Signal` relation to the URL captured in step 1. Set the page body to the markdown brief from 5.1.
3. Report both new page URLs to the user.

**On B (revise):** ask the user what to change. Re-draft the affected artifacts. Re-ask Phase 6.

### Autonomous mode (replaces the approval gate when `--autonomous`)

Do NOT ask the user. Run the quality gates below; if all pass, write to Notion automatically. If any fail, write a Killed diagnostic row instead.

**Quality gates** (ALL must pass for an autonomous write):

1. **Sources count.** ≥5 unique source URLs cited across the brief.
2. **Source diversity.** ≥1 source from a sector-press outlet (Wamda, MAGNiTT, Fast Company ME, Sifted, TechCrunch, Rest of World, regional industry press). NOT counting gov / regulator / Vision 2030 sources for this gate.
3. **Source recency.** ≥80% of cited sources <18 months old. (Compute against today's date.)
4. **Moat threshold.** Phase 4 moat pressure test produced ≥2 *real* moat answers. Generic answers (sales pipeline, local team, first-mover) do not count. If Phase 0 raised the threshold to ≥3 (because past kills frequently cited weak moats), use ≥3.
5. **Scoring rationale.** Every D/F/V/A score AND every Q1–Q16 score has a one-line rationale citing specific desktop evidence. Restating the score is not a rationale (see Phase 5b anti-inflation rule).
6. **Critical fields populated.** `Core Problem / Pain Point`, `Value Proposition`, `Strategic Fit`, `Market size`, `Competitive moat` all non-empty. `Geography` includes `"KSA"`. `Status` = "Research". `Research Sub-stage` = "Desktop".
7. **Composite selection score.** Phase 2 composite for the selected signal ≥5. (This is also enforced at Phase 2, but verify here as a final defense.)
8. **Customer Journey Today.** Phase 3b produced ≥3 fully-traced paths in the brief (Google search, AI chatbot, peer ask, existing-platform, workaround). The "is anything already good enough?" check has been answered explicitly. If a path produced a "good enough" status quo, the moat section addresses why the venture meaningfully beats it — and the verdict has been downgraded by one band if the rebuttal is thin.

**On all gates pass — autonomous write:**

1. Call `mcp__notion__notion-create-pages` against the Problem Signal Capture data source. Capture the returned URL.
2. Set the Problem Signal `Review Notes` to:
   ```
   Auto-generated by /discovery-scout v0.2.0 --autonomous on YYYY-MM-DD.
   AWAITING HUMAN REVIEW.
   Composite selection score: <N> (breakdown: <one-line per scoring factor>).
   Phase 0 past-run patterns applied: <named past ventures, or "none">.
   Quality gates: ALL PASSED.
   ```
3. Call `mcp__notion__notion-create-pages` against the Discovery Pipeline data source. Include all properties from 5.3, all 20 Phase 5b scores with rationale, and set the `Problem Signal` relation to the URL from step 1. Set the page body to the brief markdown from 5.1 + the Stage 1.2 Research Agenda + the scoring tables.
4. Report both new page URLs in the run output.

**On any gate fail — autonomous Killed-row fallback:**

1. Do NOT write a Problem Signal Capture row (avoid pollution of the signal log).
2. Call `mcp__notion__notion-create-pages` against the Discovery Pipeline data source with these fields ONLY:
   - `Idea`: `AUTONOMOUS RUN FAILED — <Ministry> on YYYY-MM-DD`
   - `Status`: `Killed`
   - `Decision`: `Kill`
   - `date:Decision Date:start`: today
   - `Decision Reasoning`: `Autonomous quality gates not met. See Conclusion for diagnosis. Manual interactive run recommended for this ministry.`
   - `Conclusion`: One-paragraph diagnosis listing exactly which gates failed and what was lacking. E.g., "Gate 2 (source diversity) failed — all 6 cited sources were .gov.sa or vision2030.gov.sa; no sector-press source surfaced. Gate 4 (moat) failed — 1 real moat answer, 2 generic. Composite selection score: 4. Recommended action: re-scope ministry to <X> or run interactive mode on <ministry>."
   - `Source Type`: `Trend scan`
   - `Source Detail`: short brief of what was attempted
3. Report the failure + Killed row URL in the run output.

The Killed row is intentional. It preserves the diagnostic in Notion so the team learns what kinds of ministries / signals don't survive autonomous quality gates — which is itself signal worth keeping. Subsequent Phase 0 runs read these Killed rows.

**On C (cancel):** print "Run cancelled — no Notion writes." Exit.

---

## Phase 7 — Final Recommendation + Cross-Venture Ranking (autonomous mode, runs after Phase 6 write)

After a successful Notion write (or the Killed-row fallback), produce a structured final recommendation that closes the "so what?" gap. The recommendation goes both in the run output and in the `Review Notes` field of the written row.

### 7a. Final Recommendation Verdict

Compute from the autonomous composite + estimated weighted Evaluation Score:

| Verdict | Trigger conditions | What it means |
|---|---|---|
| **STRONG ADVANCE — SPRINT** | Composite ≥ 20 AND Eval Score ≥ 4.0 AND 30-Day MVP Test = PASS | High confidence + MVP-shippable in 30 days. **Top H2 2026 sprint candidate this month.** Commit to MVP build immediately; Stage 1.2 customer validation runs in parallel with the build. |
| **STRONG ADVANCE — SPRINT (conditional)** | Composite ≥ 20 AND Eval Score ≥ 4.0 AND 30-Day MVP Test = PASS WITH CONDITION | Same as above but the credible workaround for the failing MVP-test question must be confirmed before kickoff. |
| **STRONG ADVANCE — BACKLOG** | Composite ≥ 20 AND Eval Score ≥ 4.0 AND 30-Day MVP Test = FAIL | High-confidence venture but not MVP-shippable in 30 days. Real opportunity for a slower-cadence pipeline, but not picked autonomously during the launch-priority window. |
| **CONDITIONAL ADVANCE — SPRINT** | Composite 15–19 OR Eval Score 3.6–3.99 AND 30-Day MVP Test = PASS | Medium confidence but MVP-shippable in 30 days. Sprint anyway — small ventures can absorb medium confidence; we learn from MVP traction. |
| **CONDITIONAL ADVANCE — BACKLOG** | Composite 15–19 OR Eval Score 3.6–3.99 AND 30-Day MVP Test = FAIL | Medium confidence + not MVP-shippable in 30 days. Backlog. Stage 1.2 user research must improve a specific weak score before this is sprint-ready. |
| **ITERATE** | Composite 10–14 OR Eval Score 3.0–3.59 | Low confidence regardless of MVP test. Re-scope the venture before Stage 1.2 — narrow segment, change business model, or pivot the wedge. |
| **SKIP / KILL** | Composite < 10 OR Eval Score < 3.0 OR any quality gate failed | Don't pursue. Write the diagnostic row so the Phase 0 learning loop records this as a failed pattern. |

**Sprint gate (H2 2026 launch-priority window):** During the active launch-priority window (per [[project-suhail-six-month-launch-priority]]), the autonomous agent **only writes a healthy venture row if the verdict is one of the SPRINT variants**. BACKLOG verdicts get a Killed-row fallback with `Conclusion = "30-Day MVP Test failed — real venture but not shippable in this month's sprint. See brief for what would need to change. Re-pick next sprint after refining."`

Output the verdict in the run output and append to `Review Notes`. Format:
```
FINAL RECOMMENDATION: <VERDICT>
30-Day MVP Test: PASS / PASS WITH CONDITION / FAIL
  Q1 Concierge: YES / NO — <one-line evidence>
  Q2 Off-the-shelf: YES / NO — <one-line evidence>
  Q3 Customers in 14 days: YES / NO — <one-line evidence>
Reasoning: <one paragraph citing composite + Eval Score + 30-Day Test outcome + specific weak dimension>
Workaround required (if PASS WITH CONDITION): <explicit description>
Sprint kickoff plan: <if SPRINT verdict: 1-week sprint plan for MVP build + first-10-customer onboarding>
```

### 7b. Cross-Venture Ranking

Query Notion Discovery Pipeline for all rows where `Status` is in `{Sourcing, Research, Testing, Decision Pending}`. For each, extract `Idea`, `Evaluation Score` (the auto-computed formula value, if available), and `Primary Sector`. Rank the **new venture** against this active set on:

1. **Composite selection score** (the autonomous score from Phase 2 — only comparable across autonomous runs)
2. **Estimated Eval Score** (the Phase 5b weighted score)
3. **Sector-relative rank** (where does the new venture rank within its `Primary Sector`?)

Append to `Review Notes`:
```
CROSS-VENTURE RANK:
- Composite: #<N> of <total> active (rank vs full pipeline)
- Eval Score: #<N> of <total> active (rank vs full pipeline)
- Sector rank: #<N> of <total> in <Primary Sector> active
- Pipeline median Eval Score: <X.XX>
- Pipeline top quartile Eval Score: <X.XX>
- This venture's position vs median: <above / at / below>
```

If the new venture ranks **below the pipeline median Eval Score**, append a soft flag to the FINAL RECOMMENDATION:
```
PIPELINE PRIORITY FLAG: This venture ranks below median in the active pipeline. Suhail may want to deprioritize Stage 1.2 work on this venture until higher-ranked active ventures are advanced or killed.
```

The ranking forces every new autonomous output to compete against existing active work rather than be considered in isolation. Over time this prevents pipeline bloat and forces focus.

---

## Quality Controls (enforce throughout)

1. **Every claim has a source line.** URL + date accessed. No exceptions.
2. **Two-source rule** for market size and traction figures. If only one source, mark as `[SINGLE-SOURCE]`.
3. **Recency filter:** prefer sources <18 months old. Flag older sources with their age.
4. **Strong-Signal Solution Trap guard** at Phases 1 and 3 — separate the problem from the regulator's proposed solution.
5. **Hajj functional-analog exception** at Phase 3 — flag the comparability stretch when applied.
6. **Q1–Q16 are forbidden** at Stage 1.1. The skill must not score ideas.

If any of these are violated during the run, surface the violation in the Phase 6 draft so the user can correct it before the Notion write.

---

## Completion Report (after Phase 6)

After Phase 6 completes, output this structured report:

```
DISCOVERY-SCOUT REPORT
═══════════════════════════════════════════════
Ministry:               <Transport | Health | …>
Selected signal:        <one-line problem statement>
Analogs surfaced:       <count> in <geographies>
KSA competitive tag:    <saturated | partial | open>
KSA regulatory tag:     <enabling | constraining | fatal blocker>
Vision 2030 pillar:     <Vibrant Society | Thriving Economy | Ambitious Nation>
Vision 2030 program:    <NTP | FSDP | NTLS | …>
Sources cited:          <count> total, <count> < 18 months
Unverified claims:      <count>
Notion writes:          <Problem Signal URL>
                        <Discovery Pipeline URL>
Status:                 DONE | DONE_WITH_CONCERNS | BLOCKED | CANCELLED
═══════════════════════════════════════════════
```

`DONE_WITH_CONCERNS` cases include: <3 analogs found, >2 unverified claims, recency filter triggered on >50% of sources, sector enum had no clean match (e.g. NGOs → Other).

`BLOCKED` cases: Notion write failed, no signals surfaced in Phase 1 (re-scope ministry), user cancelled at Phase 6 = `CANCELLED` (not BLOCKED).

---

## What this skill does NOT do

- It does not run user interviews (Stage 1.2 — that's a human-led activity).
- It does not score Q1–Q16 (IC stage).
- It does not write to the `Owner` or `Deal Lead` fields (human assignment).
- It does not fill `Decision`, `Stage Gate Recommendation`, `Evaluation Score`, or any scoring formula fields (they're auto-computed or downstream).
- It does not push to GitHub or trigger any deploy.

## What to do when stuck

- **No signals surfaced in Phase 1:** widen the source list (add sector conferences via 10times.com, MENAbytes, Wamda for the chosen sector). If still nothing after the second pass, BLOCK and ask the user to re-scope the ministry.
- **No analogs in Phase 3:** drop to the "adjacent emerging analogs" tier (Indonesia, Egypt, India) before giving up. For Hajj, explicitly invoke the functional-analog exception.
- **Regulatory picture unclear:** flag `[UNVERIFIED]` and note that an expert call is the natural next step. Do not fabricate a regulatory read.
- **Multiple plausible Primary Sector picks:** pick the closest, note the alternative in the run report.
