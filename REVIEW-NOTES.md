# REVIEW-NOTES — adversarial review & revision of SPEC.md

**Process.** Three inputs, produced independently and then adjudicated:

1. **Orchestrator review** (this session): independent findings written down *before* reading the agent review, plus direct fidelity checks against the handoff doc once provided.
2. **Adversarial review agent** (`claude-fable-5`): 26 severity-ranked findings. It verified dashboard failure modes by executing the code against pandas, checked external claims against official docs/pages where the proxy allowed, and explicitly labeled which findings were judgment calls.
3. **Documentation fact-checker** (claude-code-guide agent): doc-cited answers on subagent discovery, frontmatter `model:` semantics, WebFetch/WebSearch behavior, model observability, permissions, and nesting.

Per the request, the agent's review was treated with skepticism: every finding was cross-checked against my own list, the docs agent's citations, or the handoff text before adoption. The diff between the baseline commit ("as provided") and the revision commit is the full set of changes.

## Documentation facts the revision rests on

- `.claude/agents/` is discovered by walking **up** from the session cwd; file-edited agents require a session restart (mid-session file writes don't register). → agents moved to repo root; two-phase restart kept; preflight diagnoses cwd-vs-restart.
- WebFetch returns a **small-model digest** of the page (truncated for long pages), not the page. → dual-artifact snapshots (extract + raw curl capture), grounding grades, method-note disclosure.
- WebSearch returns **titles + URLs only** (no snippets). → noted in §0.3; budget math unaffected.
- Frontmatter `model:` accepts aliases/full IDs/`inherit`; invalid or org-excluded models fall back to the inherited model. **No documented way** for a subagent to introspect its model or the parent to observe it. → `model_used` telemetry renamed `model_configured`; §0.2 rewritten; "run the session on the target model" mitigation added.
- Subagents **can** nest (depth ≤5) in current Claude Code — the baseline spec's §0.4 ("subagents cannot spawn subagents") was factually stale. → reframed as a design rule, enforced by omitting `Agent`/`Task` from both agent files.
- Unattended parallel runs stall on permission prompts unless `permissions.allow` rules exist. → `.claude/settings.json` scaffolded (§8), preflight check added (§14.2).

## Adjudication (merged findings → disposition)

Severity is the final adjudicated severity, not necessarily the agent's. "Source": O = my review, A = review agent, D = docs agent, U = user requirement.

| # | Finding | Source | Verdict | Disposition in revised spec |
|---|---|---|---|---|
| 1 | `.claude/agents/` nested under `research-run/` never registers from a repo-root session; all relative paths (log.sh's `mkdir -p telemetry`, dashboard path) silently depend on an unpinned cwd | O+A+D | Accepted | Repo root = run root; agents at root; cwd pinned everywhere; all scripts self-locate (§4, §8, §9, §10, §14) |
| 2 | WebFetch digest ≠ page: snapshots were derivative artifacts and entailment checks verified model output; "content as received" was misleading | O+A+D | Accepted | §7 rewritten: extract + best-effort raw capture via `scripts/snap.sh`; `capture_kind`; synthesizer greps raw without reading it; grounding grades (raw > digest > UNVERIFIED); raw-capture coverage % in findings header; method-note disclosure |
| 3 | Wayback availability API returns *closest-either-direction* capture → systematically converts determinable `predates=y` into `unknown` (a false-negative generator, against the project's first principle); also `http://` and JSON-through-WebFetch lossiness | O+A | Accepted | Replaced with CDX `to=<date>&limit=1` via `scripts/cdx.sh` (curl, https, 200-filter, built-in politeness delay); rationale documented in §7 |
| 4 | DoD deadlock: "telemetry **proves** no agent exceeded 32k" is unfixable post-breach and overclaims (self-reported estimates; crashed agents show compliant zeros), while "do not stop before all boxes check" forbids stopping | O+A | Accepted | DoD boxes now check-or-waive with documented reason+impact; box reworded to "no *logged* overrun; breaches documented + evidence flagged" (§1) |
| 5 | Orchestrator hygiene rule ("never full evidence JSON") makes two runbook duties impossible: pre-splitting Q4 from Q1's census, and executing split proposals whose briefs live only in evidence JSON | O+A (A extended to splits) | Accepted | Control files: `matrix/census.csv` (written incrementally — also fixes "deliver partial census early" having had no mechanism) and `evidence/<qid>.splits.yaml`; hygiene rule amended with the explicit carve-out (§4, §5, §11) |
| 6 | Nobody writes `matrix/portfolio_venue_matrix.csv` — rows died inside evidence JSON no role could touch; DoD box unsatisfiable | O+A | Accepted (A's parallel-append variant rejected) | Batch synthesizers verify rows → `matrix/parts/<qid>.csv`; orchestrator mechanically concatenates. Parts avoid parallel-append interleaving entirely (§7, §10, §14.11) |
| 7 | Dashboard double-counts (sums `read`+`snapshot`), crashes at Phase-1 start (missing `est_tokens` column), crashes on dead-end column mix and on one malformed JSONL line; no `requirements.txt`; no >32k metric | O+A (A verified by execution) | Accepted | §13 rewritten: reads-only sum, column guards, per-line try/except with skipped-line warning, 32k metric, `requirements.txt` + install and render check in Phase 0; §6 now specifies `snapshot` event fields (root cause: they were never defined) |
| 8 | Split protocol unbounded: children get fresh budgets with no depth cap → no termination guarantee; "evidence fully landed" undefined for families; one synthesizer can't verify a whole Q4 family in 32k | O+A | Accepted (A's depth-1 adopted over my depth-2) | Depth 1, children never split; landed = parent + all children done; per-batch synthesizers + one MERGE synthesizer (§5, §10, §14) |
| 9 | Q10 brief bakes in negative-evidence inference ("no growth round per Form D absence") the run bans elsewhere; Q5 asserts EDGAR-FTS-covers-Form-D as fact; Q2 presumes Battery provenance — all inconsistent with Q7's hypothesis discipline | O+A | Accepted, tempered | All three briefs rewritten verify-or-refute. Agent's "Form D electronic mandate ~2009" kept **out** of the spec (it flagged it unverified); the brief instructs researchers to enumerate and verify false-negative modes instead |
| 10 | Handoff's survivorship-bias standing constraint dropped exactly where it bites (Q4 matrix presented as venue-quality evidence) | O+A | Accepted | Constraint verbatim in Q4 brief, §12 method note, §15, Appendix A.5 |
| 11 | No prompt-injection guardrail for web-fed researchers holding Bash; synthesizer corpus same exposure | O+A | Accepted | §9 rule 8 (data-never-instructions, injection-suspected dead-end logging, no private hosts), §10 preamble, §15 |
| 12 | Q4 batch economics: 5 companies × 5 venues may not fit 24k; wave-of-4 collides with 8–18 batches; runbook never authorizes repeating waves; fleet size/cost never surfaced | O+A | Accepted, tempered | Default batch 3 (agent argued 2–3 from pessimistic arithmetic; my per-company estimate said 5 was borderline-optimistic — 3 splits the difference, adjustable on telemetry); "wave = spawn round, repeat until drained" (§14.3); fleet-scale assumption §0.6 |
| 13 | G2/Crunchbase bot-blocking will starve Q3/Q4 with no fallback route | O+A (severity unverified) | Accepted as caution | Fallback order live → archive capture → unknown in briefs and §15; search-snippet-only presence = low confidence; capture-dated claims. Encoded as expectation + procedure, not asserted fact |
| 14 | Six-[OPEN]-items delta table depends on wording that exists nowhere in the spec; handoff absent from repo → run unexecutable as written | O+A+U | Accepted | Appendix A.1 inlines all six verbatim with mapping; all runtime handoff references deleted (preamble, §1, §14, §16); handoff demoted to optional background |
| 15 | §2 principles misattribute: §2.7 (aggregators-are-leads) is not in the handoff; "the handoff doc's own words" framing | O+A | Accepted | §2 header reworded: 1–6 carried from upstream (excerpts in Appendix A), 7 is run-added |
| 16 | Handoff's feedback-mechanics [OPEN] contains its own "Q2" (a per-company diligence example) which collides with spec Q2 when inlined | A | Accepted — genuinely new catch | Disambiguating bracket in Appendix A.1 row 3 |
| 17 | Status vocabulary can't express Q8/Q9's outcome (inputs delivered, decision stays open) — RESOLVED and UNRESOLVED both misreport the delta | A | Accepted — genuinely new catch | `INPUTS-DELIVERED` added to §1/§3/§10/§11/§12 |
| 18 | "Cite only URLs you fetched" conflicts with machine-derived wayback URLs and census-inherited provenance URLs | A | Accepted — genuinely new catch | Two labeled exceptions, narrowly scoped (§7, §9 rule 3) |
| 19 | `log.sh '<single-line JSON>'` invites quoting corruption (apostrophes in reasons) feeding the dashboard crash path; inconsistent `ts` across agents | O+A | Accepted, different fix | Agent proposed `jq -nc`; rejected jq (availability unverified on target machines) in favor of python3 key=value builder — python3 is already required by the dashboard. `ts` now injected by log.sh |
| 20 | `model_used` telemetry is self-attestation presented as verification | O+A+D | Accepted | Renamed `model_configured`; §0.2 + §12 header + method note state the limit |
| 21 | Footnote numbering collides across concatenated sections | O+A | Accepted | Per-section namespacing `[Q4-3]` (§7, §10, §12) |
| 22 | Fleet-level host politeness: 4 same-wave agents can hammer one host despite per-agent rules | O+A | Accepted | Orchestrator staggers venue-heavy briefs per round (§14.3); scripts embed delays |
| 23 | ≤5-line vs ≤10-line reply inconsistency | O+A | Accepted (cosmetic) | ≤10 cap, aim 5, both agent files |
| 24 | backlog.yaml enum drift (`A-input`, `P0-flag` undocumented in header comment) | O | Accepted | Header comment documents all values + expected A-input terminal status |
| 25 | Q3 match fraction lacks an honest denominator; no sampling-spread instruction | O | Accepted | Brief records category total count + page-spread sampling |
| 26 | No run-state persistence for a long multi-wave orchestration (compaction risk) | O | Accepted | `run-state.md` in scaffold; updated every round; recovery procedure §14.5 |
| 27 | Q3 depends on Q2's definition with no hand-off mechanism if Q2 fails | O | Accepted | §14.9: pass Q2's definition (or the Appendix A.4 working definition) into Q3's brief |
| 28 | Baseline §0.4 "subagents cannot spawn subagents" factually stale | D | Accepted | Reframed as enforced design rule (§0.4) |

### Agent claims rejected, tempered, or held at arm's length

- **"Use archive.org captures as the primary route for Q3"** — tempered. For a *current-composition* question, a capture can be stale; the revision keeps live-first with archive fallback and requires capture-dated claims. Adopting archive-first would have quietly changed what Q3 measures.
- **Form D electronic-mandate date (~March 2009)** — the agent itself flagged this as recalled/unverified; excluded from the spec text. The Q10 brief has researchers verify the false-negative modes rather than inherit the date.
- **G2/Crunchbase blocking severity, archive.org 429 behavior, Volition portfolio size** — all flagged unverified by the agent; encoded as expectations/procedures (fallbacks, pacing, batch counts driven by the actual census) rather than asserted facts.
- **`jq` for telemetry JSON** — rejected for python3 (see #19).
- **Coordinated parallel CSV appends** for the matrix — rejected for parts + mechanical concatenation (see #6).
- **Batch size 2–3 from its token arithmetic** — its per-company estimate (25–75k per 5-company batch) looked pessimistic against WebSearch-returns-titles-only and digest sizes; settled on 3 with telemetry-driven adjustment rather than adopting the arithmetic wholesale.
- **Docs-agent conflict caught**: the docs agent's summary claimed nested `.claude/agents/` subdirectory layouts are fine, contradicting the doc line it quoted ("discovered by walking up from cwd"). The review agent caught this independently; the revision follows the quoted doc text (root placement), which is also the robust choice under either reading.
- The agent produced **zero findings I assessed as outright false**; its severity ranking was adjusted in places (e.g., its MINOR #23/#25 fold into larger accepted items).

## Deliberately preserved (both reviews agreed these must not be lost)

Two-phase scaffold→restart→execute flow (matches documented agent-loading behavior); split-through-orchestrator control structure; chars/4 honesty (proxy, disclosed); `unknown` as first-class answer and the absence-≠-negative rule; downgrade-never-drop `[UNVERIFIED]` handling; offline synthesis over a frozen corpus; dead-end logging with the non-empty DoD check (now with spot-check, and explicitly *not* backfillable); append-only single-line JSONL telemetry; wave-based ≤4 parallelism; B-class questions never speculatively resolved.
