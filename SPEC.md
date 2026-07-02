# SPEC — Fable 5 Research Fleet: Open-Question Resolution Run

Feed this file to Claude Code. You (the main Claude Code session) are the orchestrator. You will scaffold a repo, deploy a fleet of parallel `claude-fable-5` researcher subagents, enforce per-agent token budgets, and assemble a single `findings.md` in which every claim cites an exact primary-source URL — and every unresolved question is explicitly marked as such.

Prerequisite file: `ge-sourcing-handoff.md` must be in the repo root. Read it once, fully, before anything else. Researchers never receive it — they get distilled question briefs only.

## 0. Assumptions this spec makes (surface these to the user if wrong)

1. 32k is a hard per-agent ceiling, not a floor (the request said ">32k"; interpreted as a typo for "<32k" given the reasoning-degradation rationale).
2. "Fable 5 agents" → frontmatter `model: claude-fable-5`. Per Claude Code model resolution, if that string is unavailable or excluded by an org allowlist, the subagent silently falls back to the inherited session model — log which model actually ran.
3. No paid API keys are assumed. The run uses Claude Code's native `WebSearch`/`WebFetch` only. Brave/Exa/Tavily etc. are subjects of research (Q7), not dependencies of it.
4. Subagents cannot spawn subagents. All splitting routes through you, the orchestrator.

## 1. Objective & Definition of Done

Objective. Resolve as many open questions as possible from the GE sourcing project — the six `[OPEN]` items in the handoff doc plus the questions raised in the follow-on working session (encoded in §3) — producing `findings.md` where every claim is grounded in a snapshotted primary source with its exact URL, and every gap is honestly registered.

Definition of Done — do not stop before all boxes check:

* [ ] `findings.md` exists; every factual claim carries a footnote with exact URL + access timestamp + local snapshot path.
* [ ] Every backlog question (§3) has a status: `RESOLVED` / `PARTIAL` / `UNRESOLVED` / `NOT-RESEARCHABLE`.
* [ ] The six original `[OPEN]` items each appear in a delta table mapping old status → new status.
* [ ] `NOT-RESEARCHABLE` questions state exactly what input would resolve them (firm data, design decision, etc.) — with zero speculation offered as resolution.
* [ ] `matrix/portfolio_venue_matrix.csv` is populated, with gaps marked `unknown` + reason, never guessed.
* [ ] Telemetry proves no agent's cumulative read estimate exceeded 32,000 tokens. Splits were used instead.
* [ ] The dead-end log is non-empty. (An empty dead-end log means researchers weren't logging failures — that itself is a failure. Same principle as the pipeline's kill-logging.)
* [ ] `streamlit run dashboard/app.py` works.

## 2. Inherited principles (non-negotiable, from the handoff doc)

1. Log every dead end. The false-negative principle applied to research: a query or URL silently abandoned is invisible and unrecoverable. Every abandoned path gets a logged reason.
2. Snapshot everything cited. Sources mutate and vanish. A claim without a stored snapshot is not a claim.
3. Cited ≠ true. Synthesis agents verify each claim against the snapshot before it enters `findings.md`. Unverifiable claims are downgraded to `[UNVERIFIED]` and moved to the uncertainty register, never silently dropped or silently kept.
4. Discovery and interpretation are decoupled. Researchers collect; synthesizers reason over the frozen corpus and never touch the live web.
5. Never speculatively resolve a question that requires firm data or a design decision. Mark it, state the required input, move on.
6. Budget is a proxy. The handoff doc's own words: context degradation is signal-to-noise, not token count. The 32k ceiling is enforced, but a clean 15k read beats a noisy 30k — researchers prune aggressively and stop reading pages that aren't paying.
7. Aggregators are leads, not evidence. Tracxn/blog/roundup pages may point somewhere; only primaries (the firm's own site, G2 product pages, EDGAR filings, archive.org captures, official docs/pricing pages) are citable.

## 3. Question backlog → write this to `backlog.yaml`

```yaml
# class A = web-researchable | class B = requires firm input or is a design decision
# priority: P0 highest
questions:
  - id: Q1
    class: A
    priority: P0
    title: Volition Capital portfolio census + thesis parameters
    brief: >
      Enumerate Volition Capital's portfolio (current + realized) with company
      names and investment announcement dates. Capture the firm's stated thesis
      parameters verbatim-adjacent: founder-ownership language (verify whether a
      specific % threshold is actually stated anywhere — do NOT assume one),
      capital-efficiency language, sector focus. Output structured JSON list.
    seed: [volitioncapital.com/portfolio, volitioncapital.com/news, press releases per company]
    note: Q4 depends on this. Deliver even a partial census early.

  - id: Q2
    class: A
    priority: P1
    title: Canonical definition of "hardware-enabled software"
    brief: >
      Locate the Battery Ventures definition (vertical software anchoring
      intelligence in first-party data gathered by company-deployed hardware) at
      its exact source URL. Capture any competing/adjacent definitions from other
      credible investors. Short question — budget accordingly.

  - id: Q3
    class: A
    priority: P0
    title: G2 "IoT Platforms" category composition vs the Q2 definition
    brief: >
      Sample >=15 products from G2's IoT Platforms category (multiple pages).
      For each: does the vendor deploy its OWN hardware and anchor the software
      value in first-party data from it? Classify yes/no/ambiguous with product-
      page evidence per company. Quick comparison pass on adjacent categories
      (Industrial IoT, IoT Device Management). Output: match fraction + lists.
      This tests whether "IoT Platforms" is a usable enumeration proxy.

  - id: Q4
    class: A
    priority: P0
    title: Portfolio x venue presence matrix + predates-investment check
    depends_on: Q1
    brief: >
      For each portfolio company from Q1, check presence on: G2 (+ assigned
      category), Crunchbase public profile, Inc. 5000 (inc.com profile pages show
      years honored), Deloitte Fast 500, obvious trade-show exhibitor lists.
      For each hit, attempt the PREDATES check via the Wayback availability API:
      GET http://archive.org/wayback/available?url={URL}&timestamp={YYYYMMDD of investment}
      predates=yes only if a capture EARLIER than the investment date exists.
      CRITICAL EPISTEMICS: absence of a Wayback capture is NOT evidence the page
      didn't exist — record 'unknown', never 'no'. Output rows for
      matrix/portfolio_venue_matrix.csv (schema in §7).
    note: Large. Orchestrator pre-splits into batches of <=5 companies.

  - id: Q5
    class: A
    priority: P1
    title: Venue retrieval mechanics
    brief: >
      For G2, Crunchbase, Inc. 5000, archive.org, SEC EDGAR full-text search:
      what programmatic access exists? APIs (tiers, pricing, auth), bulk exports,
      or scrape-only? Note that EDGAR full-text search covers Form D filings
      (free primary for funding events) — document the endpoint. Output: per-venue
      access-method table with documentation links.

  - id: Q6
    class: A
    priority: P1
    title: Entity-resolution tooling verification
    brief: >
      Verify against official docs: Splink (input schema expectations, blocking
      rules, backends), RapidFuzz, company-name normalization libraries (e.g.
      cleanco), dedupe alternatives. Deliver: what each actually requires as
      input, with doc links — enough to confirm/refute the "founder-anchored
      candidate set -> Splink on the narrow set" plan.

  - id: Q7
    class: A
    priority: P1
    title: Agentic search-layer landscape verification
    brief: >
      Verify with current pricing/docs pages: Brave Search API, Exa, Tavily,
      Perplexity Sonar, Firecrawl — index independence, semantic search
      capability, pricing, rate limits. The prior session's claims (e.g. Brave
      ~$5/1k queries) are HYPOTHESES to check, not facts to repeat.

  - id: Q8
    class: A-input   # research provides inputs; resolution is a design decision
    priority: P2
    title: Optimal stopping between breadth and depth passes — decision inputs
    brief: >
      Short literature scan: secretary problem, multi-armed bandits for pipeline
      stopping, IR "when to stop searching" results. Deliver 3-5 candidate
      stopping rules WITH citations. Do NOT pick one — this maps to a design
      [OPEN] and stays open.

  - id: Q9
    class: A-input
    priority: P3
    title: Flat vs split telemetry tables — practice inputs
    brief: >
      Brief scan of event-logging schema practice (wide events vs normalized,
      OpenTelemetry event modeling). Inputs only; the [OPEN] stays open.

  - id: Q10
    class: A
    priority: P2
    title: Contrast class feasibility for the inflection-strain lens
    brief: >
      Can a comparable "not-invested" set be constructed from public data (e.g.
      same G2 category + similar founding era + no growth round per EDGAR Form D
      absence)? Feasibility memo with concrete evidence of data availability.
      Output feasibility verdict + method sketch, not the contrast class itself.

  - id: Q11
    class: B
    priority: P0-flag
    title: Historical lead data from the firm
    resolution: NOT-RESEARCHABLE
    brief: >
      Requires Volition sharing historical lead/decision data. findings.md must
      state this plainly: gates all backtest claims in the pitch until obtained.

  - id: Q12
    class: B
    priority: P2
    title: Feedback mechanics from reasoning back to discovery
    resolution: NOT-RESEARCHABLE (design decision; may cite Q8 inputs)

  - id: Q13
    class: B
    priority: P2
    title: Tier 3 -> Tier 2 promotion thresholds
    resolution: NOT-RESEARCHABLE (design decision)
```

## 4. Architecture

```
orchestrator (you, main session)
 ├─ reads handoff doc + backlog; owns waves, budgets, splits, assembly
 ├─ researcher subagents  (parallel, <=4 per wave, model claude-fable-5)
 │    web -> snapshots/ + evidence/<qid>.json + telemetry events
 ├─ synthesizer subagents (parallel, offline — corpus only)
 │    evidence + snapshots -> verified sections/<qid>.md + uncertainty entries
 └─ assembly (you): sections/*.md -> findings.md ; DoD check
```

Context hygiene for the orchestrator: subagent replies to you are ≤10 lines. All detail lives in files. You read `sections/*.md` and telemetry aggregates — never raw snapshots, never full evidence JSON.

## 5. Token budgets & the split protocol

* Hard ceiling: 32,000 est. tokens of cumulative read content per agent (fetched pages + files read). Rationale: reasoning degradation threshold; also per §2.6, prune before you hit it.
* Soft budget: 24,000. At 24k a researcher stops fetching and finalizes.
* Self-warning at 20,000.
* Output contract ≤ ~1,500 tokens; task brief ≤ ~1,500 tokens — leaves headroom under the ceiling.
* Accounting: `est_tokens = ceil(chars / 4)` on every fetched/read body, self-logged (§6). This is a proxy — true per-subagent token counts aren't exposed in-session; the estimate is the tracked metric and the dashboard says so.
* Split protocol (orchestrator-executed — researchers cannot spawn agents):
   1. Researcher nearing budget with the question unfinished returns `status: partial` + `split_proposal`: 2–4 disjoint child briefs.
   2. You spawn children as `Q4a`, `Q4b`, … each with a fresh budget and only the narrowed brief (never the parent's raw reads).
   3. Parent evidence is preserved; the synthesizer for the base qid reads all `evidence/q4*.json`.
   4. Splitting is always preferred over budget overrun. Pre-split anything obviously large (Q4 → company batches of ≤5) before first spawn.

## 6. Telemetry

Append-only JSONL at `telemetry/agents.jsonl`. One JSON object per line:

```json
{"ts":"2026-07-01T14:03:22Z","agent_id":"r-q4a","role":"researcher","question_id":"Q4a","event":"read","url":"https://...","est_tokens":1830,"cum_tokens":9410}
```

* `event` ∈ `spawn | read | snapshot | dead_end | split_proposed | done` (orchestrator logs its own `spawn`/`done` with `role:"orchestrator"`).
* `dead_end` events carry `{"url_or_query":..., "reason":...}`.
* `done` carries `{"status":..., "cum_tokens":..., "model_used":...}`.

Helper — `scripts/log.sh`:

```bash
#!/usr/bin/env bash
# usage: scripts/log.sh '<single-line JSON>'
mkdir -p telemetry
printf '%s\n' "$1" >> telemetry/agents.jsonl
```

(Single-line appends via O_APPEND are atomic at these sizes; safe for parallel agents.)

Researcher logging rule: after EVERY WebFetch/WebSearch, before doing anything else, log a `read` event with the estimate and new cumulative. No exceptions — an unlogged read is a budget leak.

## 7. Snapshots, citations, and the matrix

Snapshots. Any source that will be cited: write the retrieved content as received to `snapshots/<qid>/<slug>.md` plus `snapshots/<qid>/<slug>.meta.json`:

```json
{"url":"https://...","accessed":"2026-07-01T14:03:22Z","agent_id":"r-q4a","est_tokens":1830}
```

Log a `snapshot` event. Reads that end up uncited may skip the snapshot but never skip the `read` log.

Citation format in findings.md (footnote per claim):

```
[7] https://exact.url/path — accessed 2026-07-01 — snapshot: snapshots/q4/g2-halos.md
    wayback (date-sensitive claims only): https://web.archive.org/web/20240312.../...
```

Never a bare domain, never a URL that wasn't actually fetched, never an invented archive link.

`matrix/portfolio_venue_matrix.csv` columns: `company, investment_date, investment_date_source_url, venue, present(y/n/unknown), venue_label_or_category, earliest_evidence_date, evidence_url, wayback_url, predates_investment(y/n/unknown), notes`

## 8. Scaffold (Phase 0 — create all of this before any research)

```
research-run/
  backlog.yaml            # from §3
  telemetry/agents.jsonl  # empty
  snapshots/  evidence/  sections/  matrix/  scripts/
  scripts/log.sh          # from §6, chmod +x
  dashboard/app.py        # from §13
  .claude/agents/researcher.md    # §9
  .claude/agents/synthesizer.md   # §10
  findings.md             # stub header only
```

Registration gotcha: file-based subagents load at session start; files created mid-session may not register. Hence the two-phase kickoff in §15 — scaffold, restart, execute. After restart, verify both agents are invocable before Wave 1; if not, halt and tell the user.

## 9. `.claude/agents/researcher.md` — write verbatim

```markdown
---
name: researcher
description: Single-question web researcher for the GE sourcing research run. Fetches, snapshots, logs telemetry, returns structured evidence JSON. Use for backlog questions only.
tools: WebSearch, WebFetch, Read, Write, Bash
model: claude-fable-5
---
You are one researcher in a parallel fleet. You receive exactly ONE question
brief: {qid, brief, seed venues/queries, agent_id}. Resolve it with cited
primary evidence — or return an honest partial.

HARD RULES
1. TELEMETRY: after EVERY WebFetch/WebSearch, immediately run
   scripts/log.sh with a `read` event (est_tokens = ceil(chars/4), plus your
   running cum_tokens). An unlogged read is a protocol violation.
2. BUDGET: warn yourself in your notes at cum 20k; STOP all fetching at 24k;
   never exceed 32k total read. If unfinished at the stop line, return
   status "partial" with a split_proposal of 2-4 disjoint child briefs.
3. EVIDENCE: cite only URLs you actually fetched this session. Never
   reconstruct a URL from memory. Snapshot every cited source per the
   snapshot rules (write content + meta.json, log a `snapshot` event).
4. PRIMARIES ONLY as evidence: the firm's own site, G2 product/category
   pages, EDGAR filings, archive.org captures, official docs/pricing pages,
   Inc.com profile pages. Aggregators and blogs are leads; corroborate
   before citing, or mark the claim low-confidence.
5. DEAD ENDS: every abandoned query/URL gets a `dead_end` telemetry event
   with a reason. Silent abandonment is the one unrecoverable mistake.
6. EPISTEMICS: paraphrase; record a short exact quote (<=25 words) + locator
   per claim so the verifier can check entailment. Absence of evidence
   (e.g., no Wayback capture, paywalled page) is recorded as "unknown" with
   the reason — never inferred as a negative, never guessed around.
7. PAYWALLS/AUTH: do not circumvent. Log as dead_end reason "paywalled",
   try a free primary substitute (e.g., EDGAR Form D for funding), else
   mark the gap.
8. SIGNAL: stop reading any page that isn't paying for its tokens. A clean
   15k read beats a noisy 30k.

OUTPUT
Write evidence/<qid>.json matching the contract in the spec's §11, log a
`done` event (status, cum_tokens, model_used), then reply to the
orchestrator with <=5 lines: status, #claims, #dead_ends, cum_tokens,
split_proposal? (yes/no).
```

## 10. `.claude/agents/synthesizer.md` — write verbatim

```markdown
---
name: synthesizer
description: Offline verifier-writer for the GE sourcing research run. Reads one question's evidence + snapshots, verifies claims, writes the findings section. Never fetches the live web.
tools: Read, Write, Bash
model: claude-fable-5
---
You are a synthesis agent. Input: one qid. You reason over the FROZEN corpus
only — evidence/<qid>*.json and exactly the snapshot files they reference.
You have no web tools; do not ask for them.

PROCESS
1. Read evidence/<qid>*.json (a split question has several files).
2. For each claim: open its snapshot, locate the recorded quote/locator,
   and check the claim is actually entailed by the snapshot text.
   - Entailed -> keep, with footnote.
   - Not found / not entailed / overstated -> mark [UNVERIFIED], move to the
     Unverified & Open list with one line on what failed. Never silently
     drop, never silently keep.
3. Budget: same 32k read ceiling; log `read` telemetry events for files via
   scripts/log.sh. If snapshots are too large, verify on sampled excerpts
   and say so explicitly in the section.
4. Write sections/<qid>.md:
   ## <qid> — <title>
   **Status:** RESOLVED | PARTIAL | UNRESOLVED | NOT-RESEARCHABLE
   **Answer:** prose with [n] footnotes for every claim.
   **Confidence:** High/Medium/Low — one line why (source quality,
   corroboration count, recency).
   **What would raise confidence:** concrete next inputs.
   **Unverified & open:** the [UNVERIFIED] items + honest gaps.
   **Dead ends:** count + notable examples.
   **Footnotes:** exact URL — accessed date — snapshot path (— wayback URL
   where date-sensitive).
5. Log `done`. Reply to orchestrator with <=5 lines.
```

## 11. Researcher output contract — `evidence/<qid>.json`

```json
{
  "qid": "Q4a",
  "status": "resolved | partial | unresolved | not_researchable",
  "summary": "<=120 words",
  "claims": [
    {
      "id": "c1",
      "text": "paraphrased claim",
      "urls": ["https://exact.url"],
      "snapshot": "snapshots/q4a/slug.md",
      "quote": "<=25-word exact quote",
      "quote_locator": "heading/para hint",
      "confidence": "high | med | low",
      "why": "one line"
    }
  ],
  "data": { "matrix_rows": [] },
  "dead_ends": [ {"url_or_query": "...", "reason": "..."} ],
  "est_tokens_read": 0,
  "split_proposal": [ {"child_qid": "Q4a1", "brief": "...", "seed": []} ]
}
```

## 12. Assembly (orchestrator) — `findings.md`

Read only `sections/*.md` + telemetry aggregates. Structure:

```markdown
# Findings — GE Sourcing Open Questions
Run: <date> · model(s): <from telemetry> · agents spawned: N · total est. read tokens: T · max single-agent: M

## Open-question delta (the six original [OPEN] items)
| Original [OPEN] item | Mapped Q | New status | One-line outcome |

## Executive summary
<=1 page, from section statuses only.

## Q-by-Q sections
(concatenate sections/*.md in backlog order)

## Uncertainty register
Every PARTIAL/UNRESOLVED/[UNVERIFIED] item, aggregated: what's unknown, why,
what input resolves it.

## Dead-end log summary
Counts by question + pointer to telemetry/agents.jsonl.

## Method note
Read-token accounting is an estimate (chars/4) self-logged per fetch; true
per-subagent token usage is not exposed in-session. Synthesis ran offline
over snapshotted primaries only.
```

Then run the §1 DoD checklist and fix anything failing before declaring done.

## 13. `dashboard/app.py` — reference implementation (adapt, keep small)

```python
import json, pathlib, pandas as pd, streamlit as st

st.set_page_config(page_title="Fable 5 Fleet Telemetry", layout="wide")
st.title("Research fleet — est. read-token consumption")
st.caption("Estimates = ceil(chars/4), self-logged per fetch. Ceiling 32k / soft 24k per agent.")

@st.fragment(run_every="5s")   # needs streamlit>=1.37; else replace with a Refresh button
def board():
    p = pathlib.Path("telemetry/agents.jsonl")
    if not p.exists():
        st.info("no telemetry yet"); return
    rows = [json.loads(l) for l in p.read_text().splitlines() if l.strip()]
    if not rows:
        st.info("no telemetry yet"); return
    df = pd.DataFrame(rows)
    reads = df[df.event.isin(["read","snapshot"])].groupby("agent_id").est_tokens.sum().rename("read_tokens")
    meta  = df.groupby("agent_id").agg(qid=("question_id","first"), last_seen=("ts","max"))
    done  = set(df[df.event=="done"].agent_id)
    b = meta.join(reads).fillna(0).reset_index()
    b["status"] = b.agent_id.map(lambda a: "done" if a in done else "running")
    c1, c2, c3 = st.columns(3)
    c1.metric("Total est. read tokens", int(b.read_tokens.sum()))
    c2.metric("Agents", len(b))
    c3.metric("Over 24k soft budget", int((b.read_tokens > 24000).sum()))
    for _, r in b.sort_values("read_tokens", ascending=False).iterrows():
        st.progress(min(r.read_tokens/32000, 1.0),
                    text=f"{r.agent_id} · {r.qid} · {int(r.read_tokens):,} tok · {r.status}")
    st.dataframe(b, use_container_width=True)
    de = df[df.event=="dead_end"]
    st.subheader(f"Dead ends logged: {len(de)}")
    if len(de): st.dataframe(de[["ts","agent_id","question_id","url_or_query","reason"]]
                             if "url_or_query" in de.columns else de, use_container_width=True)

board()
```

## 14. Runbook (orchestrator)

1. Phase 0 (first session): read handoff doc → build full scaffold (§8) exactly → verify files → tell the user to restart Claude Code and run the execute kickoff. Stop.
2. Phase 1 (after restart): confirm `researcher` and `synthesizer` are registered (halt + tell user if not). Log orchestrator `spawn`.
3. Wave 1 (≤4 parallel): Q1, Q2, Q6, Q7.
4. Between every wave: read telemetry; confirm no agent >32k; execute any `split_proposal`s; spawn a synthesizer for each qid whose evidence has fully landed (synthesizers can run alongside later research waves).
5. Wave 2: Q3, Q5 + first Q4 batches (you pre-split Q4 by ≤5-company batches from Q1's census).
6. Wave 3: remaining Q4 batches, Q8, Q10.
7. Wave 4: Q9 + any split children/stragglers.
8. B-class (Q11–Q13): no researchers. Write their sections yourself: status NOT-RESEARCHABLE + required input, per §2.5.
9. Assemble `findings.md` (§12) → run DoD → fix → log orchestrator `done` → point the user at `findings.md` and `streamlit run dashboard/app.py`.

## 15. Guardrails

* Public pages only; no login, no paywall circumvention, no scraping tricks. Paywall → logged dead end + free-primary substitute (EDGAR Form D for funding) or an honest gap.
* Space out fetches to the same host; be a polite client.
* Prefer archive.org captures for anything historical or date-sensitive.
* Wayback non-coverage ≠ non-existence. `unknown` is a first-class answer everywhere in this run.
* If any private-company financial figure surfaces, it is estimate-grade by standing constraint — label it so.

## 16. Kickoff prompts

Phase 0 (scaffold):

> Read SPEC (this file) and ge-sourcing-handoff.md. Execute §8 Phase 0 only: build the full scaffold including both agent files, backlog.yaml, scripts, and dashboard. Then stop and tell me to restart the session.

Phase 1 (execute):

> Read SPEC and ge-sourcing-handoff.md. You are the orchestrator. Execute the §14 runbook from step 2. Do not fetch web content yourself — delegate all research to researcher subagents in parallel waves of ≤4, enforce the §5 budgets via telemetry between waves, run synthesizers offline as evidence lands, and finish only when the §1 DoD checklist fully passes.
