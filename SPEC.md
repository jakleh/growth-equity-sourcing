# SPEC — Fable 5 Research Fleet: Open-Question Resolution Run

*Rev. 3 (2026-07-02) — Rev. 2 revised the draft after an adversarial review pass; Rev. 3 folds in the project handoff so the spec is self-contained. See commit messages for change logs.*

**Feed this file to Claude Code.** You (the main Claude Code session) are the **orchestrator**. You will scaffold the repo, deploy a fleet of parallel `claude-fable-5` researcher subagents, enforce per-agent fetch-volume caps, and assemble a single `findings.md` in which every claim cites an exact primary-source URL backed by a raw snapshot — and every unresolved question is explicitly marked as such.

**Self-contained:** this spec embeds all the project context the run needs (§2) and supersedes `ge-sourcing-handoff.md` for this run — no other file is required. Researchers never receive project background wholesale; they get the distilled question briefs in §3 only, which is why those briefs carry their own context.

**Session placement:** all sessions (Phase 0 and Phase 1) run with the **repo root as cwd**. Claude Code discovers project subagents by walking *up* from the cwd, never down — which is why the scaffold in §8 is flat at the repo root and `.claude/agents/` sits directly under it.

---

## 0. Assumptions this spec makes (surface these to the user if wrong)

1. **32k is a hard per-agent cap**, not a floor (the request said ">32k"; interpreted as a typo for "<32k" given the reasoning-degradation rationale).
2. "Fable 5 agents" → frontmatter `model: claude-fable-5`. Per the subagent docs, a model excluded by an org allowlist is rejected and the subagent falls back to the inherited session model; behavior for an outright-invalid model ID is **not documented**. A subagent has no reliable way to introspect which model it actually ran on — so the orchestrator logs the *configured* model at spawn, and any model name a subagent reports is recorded as `model_self_reported`, labeled as such in `findings.md`.
3. **No paid API keys are assumed.** The run uses Claude Code's native `WebSearch`/`WebFetch` for discovery and `curl` (via Bash) for raw snapshots. Brave/Exa/Tavily etc. are *subjects of research* (Q7), not dependencies of it.
4. Subagents **cannot spawn subagents**. All splitting routes through you, the orchestrator.
5. **WebFetch is lossy by design.** It converts the page to markdown and answers your prompt with a *small, fast model* — you receive an AI digest, not the page. Digests may paraphrase, truncate (large pages are cut at a fixed limit), or omit. Responses are cached ~15 minutes; cross-host redirects are returned to the caller rather than followed. Everything in §7 exists because of this: **cited evidence is grounded in raw `curl` snapshots, never in WebFetch digests.**

---

## 1. Objective & Definition of Done

**Objective.** Resolve as many open questions as possible from the GE sourcing project — its six standing `[OPEN]` items (mapped to Q8–Q13; see the mapping table in §3) plus the questions raised in the follow-on working session (Q1–Q7), all encoded in §3 — producing `findings.md` where every claim is grounded in a snapshotted primary source with its exact URL, and every gap is honestly registered.

**Definition of Done — do not stop before all boxes check:**

- [ ] `findings.md` exists; **every factual claim** carries a footnote with exact URL + access timestamp + local snapshot path.
- [ ] Every cited snapshot is **raw** (`fidelity: raw` in its meta.json) unless explicitly labeled `model-mediated`, in which case the claim's confidence is capped at `low` and it appears in the uncertainty register.
- [ ] Every backlog question (§3) has a status: `RESOLVED` / `PARTIAL` / `UNRESOLVED` / `NOT-RESEARCHABLE`.
- [ ] The six standing `[OPEN]` items each appear in a delta table mapping old status → new status, per the §3 mapping table.
- [ ] `NOT-RESEARCHABLE` questions state *exactly what input would resolve them* (firm data, design decision, etc.) — with **zero speculation** offered as resolution.
- [ ] `matrix/portfolio_venue_matrix.csv` is populated, with gaps marked `unknown` + reason, never guessed.
- [ ] Telemetry shows no agent's **self-reported** cumulative read estimate exceeded 32,000 tokens. Splits were used instead. (Estimates are self-logged proxies — see §5 — so this is attestation-grade, and `findings.md` says so.)
- [ ] Dead-end accounting is **auditable**: each researcher's reported `#dead_ends` matches its telemetry events, and any agent reporting zero dead ends across >5 fetches was spot-checked by the orchestrator (note the check in `findings.md`). Fabricating a dead end to satisfy tooling is itself a protocol violation — an honest zero beats an invented entry.
- [ ] `streamlit run dashboard/app.py` works.

---

## 2. Project context & inherited principles (self-contained)

**The project.** An agentic workflow that narrows a large company list to those with the highest expected value of a first meeting, for a real growth-equity firm engagement (Volition Capital). Its governing reframe: **this is a data collection problem** — Phase 1 finds and stores primary sources; reasoning over them happens later, separately. That decoupling is what makes the system tunable, keeps agent hallucinations out of the stored corpus, and turns backtesting into "re-reason over a frozen corpus" instead of an impossible re-crawl. This research run inherits that architecture wholesale — and one more of the project's standing constraints justifies the fleet design itself: *multi-agent gains apply to breadth-first retrieval, not reasoning* — hence parallel researchers, one synthesizer per question.

**Background needed to interpret the backlog (distilled from the project; §3 briefs embed what each researcher needs):**

- **Three-tier source architecture.** Tier 1: known endpoints, binary gates (evaluation, not search — e.g. funding-history checks). Tier 2: known endpoints, qualitative depth (claims and scores, possibly multiple primaries jointly). Tier 3: unknown endpoints, exploratory — the only true search; the agent traverses from the investment philosophy. Promotion path: a Tier 3 endpoint that repeatedly proves signal-bearing (human-verified) is promoted to Tier 2; to Tier 1 only if the judgment distills to a crisp binary. Gates that stop predicting get retired. The funnel is **multiple T's**: cheap breadth gates kill most of the set → survivors get deep exploration → empty depth results feed back to widen N.
- **Set construction (Module 1).** You cannot query "cash-efficient" — no endpoint returns it, and the trait is adversely selected against visibility (capital efficiency minimizes public footprint; TechCrunch is the photographic negative of the target universe). Therefore: **enumerate a vertical, don't scan the economy** — a bounded set is the only thing whose completeness you can reason about. Ranked channels: (1) Inc. 5000 / Deloitte Fast 500 / regional fast-growth lists; (2) trade-show exhibitor lists and association directories; (3) G2/Capterra category grids (review accumulation ≈ paying customers; the category ≈ the population); (4) job-posting velocity; (5) niche trade press (Tier 3 material only). The locked next step: one vertical B2B SaaS niche → bounded set from one G2 category ∩ one trade-show exhibitor list, flagging Inc. 5000 / Fast 500 appearances → funding-history-vs-scale as the first Tier 1 gate. **Q3 and Q4 exist to test the load-bearing venues of exactly this step.**
- **The inflection-strain lens (Tier 2 signal, never a gate).** Detect companies at the moment their current org can't take them to the next level — exactly when GE capital plus operating help is worth most. Observables: hiring titles/JDs misaligned with what the stage requires, revenue outrunning org sophistication, founder still personally closing at scale, first-ever VP req, abrupt product/geo expansion. It is directionally ambiguous by construction, and validating the firm-side "what they *should* be hiring for" model needs a contrast class or it's self-confirmation — that contrast class is Q10.
- **Instrumentation.** The project's schema is locked as three normalized tables (Primary_Sources, Gate_Evaluations, Agent_Executions); whether telemetry is flat or split is the standing [OPEN] behind Q9, current lean: split.
- **Survivorship bias.** Portfolio-derived signals are not clean gate sources — anything learned from the Q4 matrix is descriptive input for lens design and backtesting, never a validated gate.

**Principles (non-negotiable):**

1. **Log every dead end.** The false-negative principle applied to research: a query or URL silently abandoned is invisible and unrecoverable. Every abandoned path gets a logged reason.
2. **Snapshot everything cited — raw.** Sources mutate and vanish, and WebFetch digests are not sources (§0.5). A claim without a stored raw snapshot is not a claim (model-mediated snapshots are a labeled, low-confidence exception, §7).
3. **Cited ≠ true.** Synthesis agents verify each claim against the snapshot before it enters `findings.md`. Unverifiable claims are downgraded to `[UNVERIFIED]` and moved to the uncertainty register, never silently dropped or silently kept.
4. **Discovery and interpretation are decoupled.** Researchers collect; synthesizers reason over the frozen corpus and **never touch the live web** — enforced structurally: synthesizers get no web tools and no Bash (§10).
5. **Never speculatively resolve** a question that requires firm data or a design decision. Mark it, state the required input, move on.
6. **Budget is a proxy.** The project's own framing: context degradation is signal-to-noise, not token count. The 32k cap is enforced, but a clean 15k read beats a noisy 30k — researchers prune aggressively and stop reading pages that aren't paying.
7. **Aggregators are leads, not evidence.** Tracxn/blog/roundup pages may point somewhere; only primaries (the firm's own site, G2 product pages, EDGAR filings, archive.org captures, official docs/pricing pages) are citable.

---

## 3. Question backlog → write this to `backlog.yaml`

**Mapping: the project's six standing `[OPEN]` items → backlog questions.** This table is the skeleton of the `findings.md` delta table (§12); Q1–Q7 are additional questions from the follow-on working session.

| Standing `[OPEN]` item | Mapped Q |
|---|---|
| Historical lead data from the firm (gates all backtest claims in the pitch) | Q11 |
| Optimal stopping between breadth and depth passes | Q8 |
| Feedback mechanics from reasoning back to discovery | Q12 |
| Promotion thresholds (Tier 3 → Tier 2) | Q13 |
| Contrast class for the inflection-strain lens | Q10 |
| Flat vs. split telemetry tables | Q9 |

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
      capital-efficiency language, sector focus. Output structured JSON list
      AND write matrix/census.csv (company, announce_date, source_url) — the
      orchestrator cuts Q4 batches from that CSV, so it must be machine-clean.
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
      WHY THIS MATTERS: G2/Capterra category grids are enumeration channel #3
      in the project's set-construction strategy (review accumulation
      approximates paying customers; the category approximates the vertical's
      population), and the locked next step builds its bounded set from one G2
      category intersected with one trade-show exhibitor list. This question
      tests whether "IoT Platforms" is a usable enumeration proxy for the
      hardware-enabled-software thesis — a negative answer is itself a P0
      finding.
      ACCESS WARNING: G2 sits behind aggressive bot protection; live fetches
      will often 403. Follow the §15 fallback ladder (live -> Wayback capture
      of the same page -> unknown + dead_end). If G2 is hard-blocked at the
      venue level, say so early — that is itself a P0 finding about the proxy.

  - id: Q4
    class: A
    priority: P0
    title: Portfolio x venue presence matrix + predates-investment check
    depends_on: Q1
    brief: >
      For each portfolio company in your batch (from matrix/census.csv), check
      presence on: G2 (+ assigned category), Crunchbase public profile, Inc.
      5000 (inc.com profile pages show years honored), Deloitte Fast 500,
      obvious trade-show exhibitor lists. These venues bot-block; use the §15
      fallback ladder and respect venue-level fail-fast flags in your brief.
      For each hit, run the PREDATES check via the Wayback CDX API:
        curl 'http://web.archive.org/cdx/search/cdx?url={URL}&to={YYYYMMDD of investment}&limit=1'
      A returned capture with timestamp STRICTLY EARLIER than the investment
      date => predates=yes (record the capture timestamp + wayback URL).
      Empty result => predates=unknown. (Do NOT use the availability API — it
      returns the capture CLOSEST to a timestamp, which can postdate the
      investment even when earlier captures exist.)
      CRITICAL EPISTEMICS: absence of a Wayback capture is NOT evidence the
      page didn't exist — record 'unknown', never 'no'. And the matrix itself
      is portfolio-derived, so it carries survivorship bias by construction:
      it is descriptive input for lens design and backtesting, never evidence
      for a gate. Output rows for matrix/portfolio_venue_matrix.csv (schema
      in §7).
    note: Large. Orchestrator pre-splits into batches of <=5 companies from census.csv.

  - id: Q5
    class: A
    priority: P1
    title: Venue retrieval mechanics
    brief: >
      For G2, Crunchbase, Inc. 5000, archive.org, SEC EDGAR full-text search:
      what programmatic access exists? APIs (tiers, pricing, auth), bulk exports,
      or scrape-only? HYPOTHESIS TO VERIFY (not fact to repeat): that EDGAR
      full-text search indexes Form D filings and would serve as a free primary
      for funding events. Establish from official SEC documentation: FTS
      coverage start date (believed 2001+), whether Form D primary documents
      are indexed, and the Form D electronic-filing history (mandatory only
      since ~March 2009 — earlier rounds are invisible to it). Output: per-venue
      access-method table with documentation links.

  - id: Q6
    class: A
    priority: P1
    title: Entity-resolution tooling verification
    brief: >
      Verify against official docs: Splink — including the project's specific
      claims about it, which are HYPOTHESES to check (UK Ministry of Justice
      origin, Fellegi-Sunter model, DuckDB backend), plus input schema
      expectations, blocking rules, other backends. Also RapidFuzz,
      company-name normalization libraries (e.g. cleanco), dedupe
      alternatives. Deliver: what each actually requires as input, with doc
      links — enough to confirm/refute the "founder-anchored candidate set ->
      Splink on the narrow set" plan. Context: the project splits entity
      resolution into two problems — canonicalization at scale (commoditized;
      don't build; Splink stays in pocket iff the multi-channel merge in set
      construction actually hurts) and attribution of a found source to a
      company_id (a confidence field set during triage, not an infrastructure
      layer). Tier 1/2 evaluation has no resolution step at all.

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
      Context: the project's funnel is multiple T's — cheap breadth gates kill
      most of the bounded set, survivors get deep exploration, and empty depth
      results feed back to widen N or re-run breadth; when to stop alternating
      is the standing [OPEN]. Short literature scan: secretary problem,
      multi-armed bandits for pipeline stopping, IR "when to stop searching"
      results. Deliver 3-5 candidate stopping rules WITH citations. Do NOT
      pick one — this maps to a design [OPEN] and stays open.

  - id: Q9
    class: A-input
    priority: P3
    title: Flat vs split telemetry tables — practice inputs
    brief: >
      Context: the project's instrumentation schema is locked as three
      normalized tables (Primary_Sources, Gate_Evaluations, Agent_Executions);
      the standing [OPEN] is whether telemetry is flat or split, with a stated
      lean toward split. Brief scan of event-logging schema practice (wide
      events vs normalized, OpenTelemetry event modeling). Inputs only; the
      [OPEN] stays open — report whether practice supports or undercuts the
      "leaning split" prior, without resolving it.

  - id: Q10
    class: A
    priority: P2
    title: Contrast class feasibility for the inflection-strain lens
    brief: >
      Context — the inflection-strain lens (a Tier 2 signal, never a gate):
      detect companies at the moment their current org can't take them to the
      next level, exactly when GE capital plus operating help is worth most.
      Observables: hiring titles/JDs misaligned with what the stage requires,
      revenue outrunning org sophistication, founder still personally closing
      at scale, first-ever VP req, abrupt product/geo expansion. Validating
      the firm-side "what they SHOULD be hiring for" model needs a contrast
      class, or the lens is self-confirmation — that contrast class is this
      question. Can a comparable "not-invested" set be constructed from public data (e.g.
      same G2 category + similar founding era)? EDGAR Form D absence may be
      used only as ONE WEAK NEGATIVE SIGNAL requiring corroboration — many
      rounds never produce a Form D (4(a)(2) placements, late/never filers,
      foreign issuers) and electronic filing is only mandatory since ~2009, so
      "no Form D" does NOT entail "no round". The feasibility memo must
      explicitly address the false-negative rate of any such proxy. Output
      feasibility verdict + method sketch, not the contrast class itself.

  - id: Q11
    class: B
    priority: P0-flag
    title: Historical lead data from the firm
    resolution: NOT-RESEARCHABLE
    brief: >
      Requires Volition sharing historical lead/decision data. findings.md must
      state this plainly: gates all backtest claims in the pitch until
      obtained. The same data also gates the project's two cheap pre-build
      tests: the null hypothesis (do three reliable endpoints plus a rubric
      reproduce the firm's decisions?) and the veto-dependency check (for
      companies the firm already decided on, did the decision hinge on
      demo/founder-intuition material beyond known endpoints?). Neither can
      run without it.

  - id: Q12
    class: B
    priority: P2
    title: Feedback mechanics from reasoning back to discovery
    resolution: NOT-RESEARCHABLE (design decision; may cite Q8 inputs)
    brief: >
      The standing [OPEN]: what "we didn't answer Q2" concretely triggers —
      how a failed or empty reasoning pass feeds back to widen N or re-run
      breadth discovery. Design decision; the findings section states the
      required decision and may cite Q8's stopping-rule inputs.

  - id: Q13
    class: B
    priority: P2
    title: Tier 3 -> Tier 2 promotion thresholds
    resolution: NOT-RESEARCHABLE (design decision)
    brief: >
      The standing [OPEN]: how many human-verified signal-bearing hits promote
      a Tier 3 exploratory endpoint to a Tier 2 known endpoint — and the
      companion retirement rule for gates that stop predicting. Design
      decision; the findings section states the required input.
```

---

## 4. Architecture

```
orchestrator (you, main session)
 ├─ reads this spec + backlog; owns waves, budgets, splits, assembly
 ├─ researcher subagents  (parallel, <=4 per wave, model claude-fable-5)
 │    web -> raw snapshots/ + evidence/<qid>.json + telemetry events
 ├─ synthesizer subagents (parallel, offline — corpus only; no web, no Bash)
 │    evidence + snapshots -> verified sections/<qid>.md + uncertainty entries
 └─ assembly (you): sections/*.md -> findings.md ; DoD check
```

**Context hygiene for the orchestrator:** subagent replies to you are **≤10 lines**. All detail lives in files. You read `sections/*.md`, telemetry aggregates, and `matrix/census.csv` (the one structured file you need to cut Q4 batches) — never raw snapshots, never full evidence JSON.

---

## 5. Fetch-volume caps & the split protocol

- **Hard cap: 32,000 est. tokens** of cumulative *read* content per agent (fetched digests + files read into context). This is a **fetch-volume cap serving as a proxy for context pressure** — it does not measure the agent's true context, which also holds the system prompt, the brief, tool-call round-trips (including the telemetry logging this spec mandates), and reasoning. Treat it as the tracked ceiling, and per §2.6, prune long before you hit it.
- **Soft budget: 24,000.** At 24k a researcher stops fetching and finalizes.
- **Self-warning at 20,000.**
- Task briefs are ≤ ~1,500 tokens. Replies to the orchestrator are ≤10 lines. **Evidence/CSV files are uncapped and do not count against the writer's read cap** — writing is generation, not reading. (They do count against the *synthesizer's* cap when it reads them.)
- **Accounting:** `est_tokens = ceil(chars / 4)` on every fetched/read body, self-logged (§6). For WebFetch/WebSearch this is an honest eyeball estimate of the tool result the agent saw — true per-subagent token counts aren't exposed in-session. The estimate is the tracked metric; the dashboard and `findings.md` both say so. Raw snapshot files fetched via `curl` do **not** count (they land on disk, not in context) — but any part of a snapshot later Read into context does.
- **Enforcement is post-hoc:** the orchestrator audits telemetry between waves (§14). An agent can overrun mid-wave with nothing stopping it except its own instructions — a stated limitation of this design, mitigated by the 24k soft stop.
- **Split protocol (orchestrator-executed — researchers cannot spawn agents):**
  1. Researcher nearing budget with the question unfinished returns `status: partial` + `split_proposal`: 2–4 *disjoint* child briefs.
  2. You spawn children as `Q4a`, `Q4b`, … each with a fresh budget and only the narrowed brief (never the parent's raw reads).
  3. Parent evidence is preserved; the synthesizer for the base qid reads all `evidence/q4*.json`.
  4. Splitting is always preferred over budget overrun. Pre-split anything obviously large (Q4 → company batches of ≤5) before first spawn.

---

## 6. Telemetry

Two channels, one schema:

- **Researchers + orchestrator** append to `telemetry/agents.jsonl` via `scripts/log.sh` (Bash).
- **Synthesizers have no Bash** (§10): each writes its own `telemetry/synth-<qid>.jsonl` via the Write tool — one writer per file, no append races; written once, complete, before the agent finishes. (Live progress for synthesizers is coarser; that's the accepted trade for structurally enforcing "offline".)

One JSON object per line:

```json
{"ts":"2026-07-01T14:03:22Z","agent_id":"r-q4a","role":"researcher","question_id":"Q4a","event":"read","url":"https://...","est_tokens":1830,"cum_tokens":9410}
```

- `event` ∈ `spawn | read | snapshot | dead_end | split_proposed | done` (orchestrator logs its own `spawn`/`done` with `role:"orchestrator"`; its `spawn` events for subagents carry `model_configured`).
- `dead_end` events carry `{"url_or_query":..., "reason":...}`.
- `done` carries `{"status":..., "cum_tokens":..., "model_self_reported":...}` — self-reported, unverifiable in-session (§0.2).
- `snapshot` events carry `{"url":..., "http_status":..., "bytes":...}` — **no `est_tokens`**; snapshot bytes are disk artifacts, not budget (§5), and only `read` events feed budget math.

Helper — `scripts/log.sh` (reads the JSON line from **stdin**, so quotes in URLs/reasons can't corrupt it):

```bash
#!/usr/bin/env bash
# usage: scripts/log.sh <<'EOF'
# {"ts":"...","agent_id":"...","event":"...",...}
# EOF
mkdir -p telemetry
IFS= read -r line
printf '%s\n' "$line" >> telemetry/agents.jsonl
```

(Single-line appends via O_APPEND are atomic at these sizes; safe for parallel agents.)

The scaffold ships `.claude/settings.json` (§8) pre-allowing `scripts/log.sh`, `scripts/snap.sh`, and the handful of read-only commands researchers need — without it, four parallel researchers logging after every fetch would drown the session in permission prompts.

**Researcher logging rule: after EVERY WebFetch/WebSearch, before doing anything else, log a `read` event with the estimate and new cumulative.** No exceptions — an unlogged read is a budget leak.

---

## 7. Snapshots, citations, and the matrix

**Why raw snapshots.** WebFetch returns a small model's digest of the page (§0.5). A digest can paraphrase or hallucinate; a quote pulled from it may not exist on the real page; and verifying a claim against the digest it came from is circular. So: **WebFetch is for reading and deciding; `curl` is for evidence.**

**Snapshot mechanics.** Any source that will be cited: run `scripts/snap.sh <url> <qid> <slug> <agent_id>` (reference implementation in §8). It writes:

- `snapshots/<qid>/<slug>.html` — the raw body as received;
- `snapshots/<qid>/<slug>.txt` — a tag-stripped, whitespace-normalized text extraction (what quotes are checked against);
- `snapshots/<qid>/<slug>.meta.json`:

```json
{"url":"https://...","accessed":"2026-07-01T14:03:22Z","agent_id":"r-q4a","http_status":200,"bytes":184203,"fidelity":"raw"}
```

and logs a `snapshot` event.

**Quote verification (researcher-side, before recording a claim):** `grep -qiF "<quote>" snapshots/<qid>/<slug>.txt` must succeed. If it doesn't, re-derive the quote *from the .txt* — never from the WebFetch digest. Record the result in the claim's `quote_verified` field (§11).

**Model-mediated fallback.** If the raw fetch fails (non-200, bot challenge), you may — as a last resort after the §15 fallback ladder — snapshot the WebFetch digest via Write with `"fidelity":"model-mediated"` in its meta.json. Such claims are capped at `confidence: low`, flagged in the section, and routed to the uncertainty register. Log the failed raw fetch as a `dead_end`.

Reads that end up uncited may skip the snapshot but never skip the `read` log.

**Citation format in findings.md** (footnote per claim):

```
[7] https://exact.url/path — accessed 2026-07-01 — snapshot: snapshots/q4/g2-halos.html (raw)
    wayback (date-sensitive claims only): https://web.archive.org/web/20240312.../...
```

Never a bare domain, never a URL that wasn't actually fetched, never an invented archive link.

**Predates check (Q4):** use the CDX API, not the availability API:

```
curl 'http://web.archive.org/cdx/search/cdx?url={URL}&to={YYYYMMDD of investment}&limit=1'
```

Non-empty result with capture timestamp **strictly earlier** than the investment date → `predates=yes` (record timestamp + wayback URL). Empty → `unknown`. (The availability API returns the capture *closest* to a timestamp — possibly later — and cannot distinguish "no earlier capture" from "a later one is closer"; it systematically manufactures false negatives on exactly the field this matrix exists to compute.)

**`matrix/portfolio_venue_matrix.csv` columns:**
`company, investment_date, investment_date_source_url, venue, present(y/n/unknown), venue_label_or_category, earliest_evidence_date, evidence_url, wayback_url, predates_investment(y/n/unknown), notes`

**`matrix/census.csv` columns (written by Q1, read by the orchestrator):**
`company, announce_date, source_url`

---

## 8. Scaffold (Phase 0 — create all of this before any research)

Flat at the **repo root** — subagent discovery walks up from cwd, so `.claude/agents/` must sit at or above where sessions run; and every relative path below (scripts/, telemetry/, dashboard/) assumes root-cwd sessions.

```
<repo root>/
  backlog.yaml            # from §3
  telemetry/agents.jsonl  # empty
  snapshots/  evidence/  sections/  matrix/  scripts/
  scripts/log.sh          # from §6, chmod +x
  scripts/snap.sh         # below, chmod +x
  dashboard/app.py        # from §13
  .claude/settings.json   # below
  .claude/agents/researcher.md    # §9
  .claude/agents/synthesizer.md   # §10
  findings.md             # stub header only
```

**`scripts/snap.sh`** — reference implementation (adapt if needed; keep the three artifacts + the event):

```bash
#!/usr/bin/env bash
# usage: scripts/snap.sh <url> <qid> <slug> <agent_id>
set -euo pipefail
url=$1; qid=$2; slug=$3; agent=$4
dir="snapshots/$qid"; mkdir -p "$dir"
code=$(curl -sL --max-time 60 -o "$dir/$slug.html" -w '%{http_code}' "$url")
python3 - "$dir/$slug.html" > "$dir/$slug.txt" <<'PY'
import html, re, sys
t = open(sys.argv[1], encoding="utf-8", errors="replace").read()
t = re.sub(r"<(script|style)[^>]*>.*?</\1>", " ", t, flags=re.S | re.I)
t = re.sub(r"<[^>]+>", " ", t)
print(html.unescape(re.sub(r"\s+", " ", t)).strip())
PY
ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
bytes=$(wc -c < "$dir/$slug.html" | tr -d ' ')
printf '{"url":"%s","accessed":"%s","agent_id":"%s","http_status":%s,"bytes":%s,"fidelity":"raw"}\n' \
  "$url" "$ts" "$agent" "$code" "$bytes" > "$dir/$slug.meta.json"
scripts/log.sh <<EOF
{"ts":"$ts","agent_id":"$agent","role":"researcher","question_id":"$qid","event":"snapshot","url":"$url","http_status":$code,"bytes":$bytes}
EOF
echo "snapshot: $dir/$slug.html ($bytes bytes, http $code)"
```

**`.claude/settings.json`** — pre-allow the telemetry/snapshot loop so parallel researchers don't flood the session with permission prompts:

```json
{
  "permissions": {
    "allow": [
      "Bash(scripts/log.sh:*)",
      "Bash(scripts/snap.sh:*)",
      "Bash(curl:*)",
      "Bash(grep:*)",
      "Bash(mkdir:*)",
      "Bash(date:*)",
      "Bash(wc:*)"
    ]
  }
}
```

> **Registration gotcha:** file-based subagents load at session start; files created mid-session are not picked up. Hence the **two-phase kickoff** in §16 — scaffold, restart, execute. After restart, verify both agents are invocable before Wave 1; if not, halt and tell the user. (Agents created through the `/agents` UI register immediately — an acceptable manual alternative if the restart is inconvenient.)

---

## 9. `.claude/agents/researcher.md` — write verbatim

```markdown
---
name: researcher
description: Single-question web researcher for the GE sourcing research run. Fetches, snapshots raw sources, logs telemetry, returns structured evidence JSON. Use for backlog questions only.
tools: WebSearch, WebFetch, Read, Write, Bash
model: claude-fable-5
---
You are one researcher in a parallel fleet. You receive exactly ONE question
brief: {qid, brief, seed venues/queries, agent_id}. Resolve it with cited
primary evidence — or return an honest partial.

KNOW YOUR TOOLS
WebFetch does NOT return the page. It returns a small model's answer to your
prompt about the page — possibly paraphrased, truncated, or wrong. Responses
are cached ~15 min; cross-host redirects come back to you unfollowed (re-fetch
the target explicitly). Use WebFetch to read and decide. Evidence comes only
from raw snapshots (scripts/snap.sh -> curl).

HARD RULES
1. TELEMETRY: after EVERY WebFetch/WebSearch, immediately log a `read` event
   via scripts/log.sh (heredoc — JSON on stdin, never as a quoted argument).
   est_tokens = ceil(chars/4), an honest eyeball estimate of the tool result
   you saw, plus your running cum_tokens. An unlogged read is a protocol
   violation.
2. BUDGET: warn yourself in your notes at cum 20k; STOP all fetching at 24k;
   never exceed 32k total read. If unfinished at the stop line, return
   status "partial" with a split_proposal of 2-4 disjoint child briefs.
3. EVIDENCE: cite only URLs you actually fetched this session. Never
   reconstruct a URL from memory. Before citing, snapshot raw via
   scripts/snap.sh <url> <qid> <slug> <agent_id>, then verify your quote:
   grep -qiF "<quote>" snapshots/<qid>/<slug>.txt must succeed. If it fails,
   re-derive the quote FROM THE .txt, never from the WebFetch digest. Record
   quote_verified accordingly.
4. FALLBACK LADDER for blocked pages (G2/Crunchbase/Inc.com etc. bot-block):
   (a) live raw fetch via snap.sh; (b) Wayback capture of the same URL
   (fetch https://web.archive.org/web/<timestamp>/<url>, snapshot THAT —
   it is a citable primary); (c) record `unknown` + dead_end. Last resort
   only: snapshot the WebFetch digest via Write with "fidelity":
   "model-mediated" in meta.json — such claims are capped at confidence low.
   If your brief flags a venue as hard-blocked (fail-fast), skip (a) and go
   straight to Wayback.
5. PRIMARIES ONLY as evidence: the firm's own site, G2 product/category
   pages, EDGAR filings, archive.org captures, official docs/pricing pages,
   Inc.com profile pages. Aggregators and blogs are leads; corroborate
   before citing, or mark the claim low-confidence.
6. DEAD ENDS: every abandoned query/URL gets a `dead_end` telemetry event
   with a reason (e.g. "bot-blocked-403", "paywalled", "irrelevant").
   Silent abandonment is the one unrecoverable mistake. Never fabricate a
   dead end — an honest zero is a valid answer.
7. EPISTEMICS: paraphrase; record a short exact quote (<=25 words) + locator
   per claim so the verifier can check entailment against the raw .txt.
   Absence of evidence (e.g., no Wayback capture, paywalled page) is
   recorded as "unknown" with the reason — never inferred as a negative,
   never guessed around.
8. PAYWALLS/AUTH: do not circumvent. Log as dead_end reason "paywalled",
   try a free primary substitute, else mark the gap.
9. SIGNAL: stop reading any page that isn't paying for its tokens. A clean
   15k read beats a noisy 30k.

OUTPUT
Write evidence/<qid>.json matching the contract in the spec's §11 (uncapped —
file output does not count against your read budget), log a `done` event
(status, cum_tokens, model_self_reported — say what you believe you are, it
will be labeled self-reported), then reply to the orchestrator with <=10
lines: status, #claims, #dead_ends, cum_tokens, any venue-level blocks hit,
split_proposal? (yes/no).
```

---

## 10. `.claude/agents/synthesizer.md` — write verbatim

```markdown
---
name: synthesizer
description: Offline verifier-writer for the GE sourcing research run. Reads one question's evidence + snapshots, verifies claims, writes the findings section. Never fetches the live web.
tools: Read, Write
model: claude-fable-5
---
You are a synthesis agent. Input: one qid. You reason over the FROZEN corpus
only — evidence/<qid>*.json and exactly the snapshot files they reference.
You have no web tools and no Bash — by design, so "offline" is structural,
not honor-system. Do not ask for more tools.

PROCESS
1. Read evidence/<qid>*.json (a split question has several files).
2. For each claim: Read its snapshot .txt, locate the recorded quote/locator,
   and check the claim is actually entailed by the snapshot text.
   - Entailed -> keep, with footnote.
   - Not found / not entailed / overstated -> mark [UNVERIFIED], move to the
     Unverified & Open list with one line on what failed. Never silently
     drop, never silently keep.
   - Snapshot meta says "fidelity":"model-mediated" -> confidence is low by
     rule; flag it and route to the uncertainty register regardless of
     entailment (the snapshot itself is an AI digest, not the source).
3. Budget: same 32k read cap. If snapshots are too large, verify on sampled
   excerpts and say so explicitly in the section.
4. Telemetry: you have no Bash. Accumulate your events (read/done, same §6
   schema) and Write them ONCE as telemetry/synth-<qid>.jsonl before
   finishing. One file, one writer, complete.
5. Write sections/<qid>.md:
   ## <qid> — <title>
   **Status:** RESOLVED | PARTIAL | UNRESOLVED | NOT-RESEARCHABLE
   **Answer:** prose with [n] footnotes for every claim.
   **Confidence:** High/Medium/Low — one line why (source quality,
   corroboration count, recency, snapshot fidelity).
   **What would raise confidence:** concrete next inputs.
   **Unverified & open:** the [UNVERIFIED] items + honest gaps.
   **Dead ends:** count + notable examples.
   **Footnotes:** exact URL — accessed date — snapshot path + fidelity
   (— wayback URL where date-sensitive).
6. Reply to orchestrator with <=10 lines.
```

---

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
      "snapshot": "snapshots/q4a/slug.html",
      "snapshot_fidelity": "raw | model-mediated",
      "quote": "<=25-word exact quote",
      "quote_locator": "heading/para hint",
      "quote_verified": "grep-pass | grep-fail-rederived | unverified",
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

This file is uncapped (§5) — completeness beats brevity here; the synthesizer, not the orchestrator, reads it.

---

## 12. Assembly (orchestrator) — `findings.md`

Read only `sections/*.md`, telemetry aggregates, and `matrix/census.csv`. Structure:

```markdown
# Findings — GE Sourcing Open Questions
Run: <date> · model configured: <from orchestrator spawn logs> (subagent self-reports labeled as such) · agents spawned: N · total est. read tokens: T · max single-agent: M

## Open-question delta (the six standing [OPEN] items, rows per the §3 mapping table)
| Standing [OPEN] item | Mapped Q | New status | One-line outcome |

## Executive summary
<=1 page, from section statuses only.

## Q-by-Q sections
(concatenate sections/*.md in backlog order)

## Uncertainty register
Every PARTIAL/UNRESOLVED/[UNVERIFIED] item plus every model-mediated-snapshot
claim, aggregated: what's unknown, why, what input resolves it.

## Dead-end log summary
Counts by question + pointer to telemetry/. Note the spot-check result for
any agent that reported zero dead ends across >5 fetches (§1 DoD).

## Method note
Read-token accounting is an estimate (chars/4) self-logged per fetch; true
per-subagent token usage is not exposed in-session, and enforcement was
post-hoc between waves. Cited snapshots are raw curl captures except where
labeled model-mediated (an AI digest — low confidence by rule). Model
identity per agent is as-configured at spawn; subagent self-reports are
attestations. Synthesis ran offline (no web tools, no Bash) over the frozen
corpus only.
```

Then run the §1 DoD checklist and fix anything failing before declaring done.

---

## 13. `dashboard/app.py` — reference implementation (adapt, keep small)

```python
import glob, json, pathlib, pandas as pd, streamlit as st

st.set_page_config(page_title="Fable 5 Fleet Telemetry", layout="wide")
st.title("Research fleet — est. read-token consumption")
st.caption("Estimates = ceil(chars/4), self-reported per fetch. Cap 32k / soft 24k per agent. "
           "Budget math counts `read` events only.")

@st.fragment(run_every="5s")   # needs streamlit>=1.37; else replace with a Refresh button
def board():
    rows, bad = [], 0
    for f in glob.glob("telemetry/*.jsonl"):
        for l in pathlib.Path(f).read_text().splitlines():
            if not l.strip():
                continue
            try:
                rows.append(json.loads(l))
            except json.JSONDecodeError:
                bad += 1
    if not rows:
        st.info("no telemetry yet"); return
    df = pd.DataFrame(rows)
    # budget = `read` events only; `snapshot` events are disk artifacts, not context
    reads = df[df.event == "read"].groupby("agent_id").est_tokens.sum().rename("read_tokens")
    meta  = df.groupby("agent_id").agg(qid=("question_id", "first"), last_seen=("ts", "max"))
    dones = df[df.event == "done"].drop_duplicates("agent_id", keep="last").set_index("agent_id")
    b = meta.join(reads).reset_index()
    b["read_tokens"] = b.read_tokens.fillna(0)
    b["status"] = b.agent_id.map(lambda a: "done" if a in dones.index else "running")
    if "cum_tokens" in dones.columns:  # cross-check self-reported totals vs summed reads
        b = b.merge(dones.cum_tokens.rename("self_reported"), left_on="agent_id",
                    right_index=True, how="left")
        b["divergence"] = (b.self_reported - b.read_tokens).abs()
    c1, c2, c3 = st.columns(3)
    c1.metric("Total est. read tokens", int(b.read_tokens.sum()))
    c2.metric("Agents", len(b))
    c3.metric("Over 24k soft budget", int((b.read_tokens > 24000).sum()))
    if bad:
        st.warning(f"{bad} malformed telemetry line(s) skipped")
    if "divergence" in b.columns and (b.divergence > 2000).any():
        st.warning("self-reported cum_tokens diverges >2k from summed reads for: "
                   + ", ".join(b[b.divergence > 2000].agent_id))
    for _, r in b.sort_values("read_tokens", ascending=False).iterrows():
        qid = r.qid if pd.notna(r.qid) else "—"
        st.progress(min(r.read_tokens / 32000, 1.0),
                    text=f"{r.agent_id} · {qid} · {int(r.read_tokens):,} tok · {r.status}")
    st.dataframe(b, use_container_width=True)
    de = df[df.event == "dead_end"]
    st.subheader(f"Dead ends logged: {len(de)}")
    if len(de):
        cols = [c for c in ["ts", "agent_id", "question_id", "url_or_query", "reason"] if c in de.columns]
        st.dataframe(de[cols], use_container_width=True)

board()
```

---

## 14. Runbook (orchestrator)

1. **Phase 0 (first session, cwd = repo root):** read this spec fully → build full scaffold (§8) exactly → verify files (`bash -n` the scripts, `python3 -m py_compile dashboard/app.py`) → tell the user to **restart Claude Code** (same cwd) and run the execute kickoff. Stop.
2. **Phase 1 (after restart, cwd = repo root):** confirm `researcher` and `synthesizer` are registered (halt + tell user if not). Log orchestrator `spawn`.
3. **Wave 1 (≤4 parallel):** Q1, Q2, Q6, Q7.
4. Between every wave: read telemetry; confirm no agent's self-reported reads >32k; audit dead-end counts vs replies (spot-check any zero-dead-end agent with >5 fetches); note any venue-level `bot-blocked` dead ends and mark those venues **fail-fast** in subsequent briefs (later batches go straight to Wayback); execute any `split_proposal`s; spawn a synthesizer for each qid whose evidence has fully landed (synthesizers can run alongside later research waves).
5. **Wave 2:** Q3, Q5 + first Q4 batches — **gated on `matrix/census.csv` existing** (Q1's deliverable; if the census is partial, batch only the companies that landed and schedule the rest for Wave 3). Cut batches of ≤5 companies from the CSV; put each batch's companies + investment dates directly in the child brief.
6. **Wave 3:** remaining Q4 batches, Q8, Q10.
7. **Wave 4:** Q9 + any split children/stragglers.
8. B-class (Q11–Q13): no researchers. Write their sections yourself: status NOT-RESEARCHABLE + required input, per §2.5.
9. Assemble `findings.md` (§12) → run DoD → fix → log orchestrator `done` → point the user at `findings.md` and `streamlit run dashboard/app.py`.

---

## 15. Guardrails

- Public pages only; no login, no paywall circumvention, no scraping tricks (a plain `curl` of a public URL, at polite rates, is fine; defeating bot challenges is not). Paywall → logged dead end + free-primary substitute or an honest gap.
- **Blocked-venue fallback ladder** (G2, Crunchbase, Inc.com and friends bot-block aggressively): live raw fetch → Wayback capture of the same page → `unknown` + dead_end. First hard 403 on a venue gets logged as a venue-level dead end; the orchestrator flags that venue fail-fast for later batches. WebFetch-digest snapshots are a last resort, labeled `model-mediated`, confidence-capped at low.
- Space out fetches to the same host; be a polite client.
- Prefer archive.org captures for anything historical or date-sensitive — they are stable and citable primaries, and they double as the predates evidence.
- Predates checks use the **CDX API** (§7), never the availability API.
- Wayback non-coverage ≠ non-existence. `unknown` is a first-class answer everywhere in this run.
- If any private-company financial figure surfaces, it is estimate-grade by standing constraint — label it so.

---

## 16. Kickoff prompts

**Phase 0 (scaffold):**
> Read SPEC.md (this file) fully — it is self-contained. Execute §8 Phase 0 only: build the full scaffold at the repo root, including both agent files under .claude/agents/, .claude/settings.json, backlog.yaml, scripts, and dashboard. Verify the scripts and dashboard parse. Then stop and tell me to restart the session from the repo root.

**Phase 1 (execute):**
> Read SPEC.md fully — it is self-contained. You are the orchestrator. Execute the §14 runbook from step 2. Do not fetch web content yourself — delegate all research to researcher subagents in parallel waves of ≤4, enforce the §5 caps via telemetry between waves, run synthesizers offline as evidence lands, and finish only when the §1 DoD checklist fully passes.
