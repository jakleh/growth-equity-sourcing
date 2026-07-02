# SEAM critique — cross-section defects only

## F1 — Nobody is assigned to materialize `matrix/portfolio_venue_matrix.csv`
* Anchor: §3(Q4) + §11 ↔ §4/§12 ↔ §1(DoD-6) ↔ §10
* Class: contradiction / infeasible-as-written
* Defect: §3 Q4 says "Output rows for matrix/portfolio_venue_matrix.csv" and §11 puts those rows in `data.matrix_rows` inside evidence JSON — but §10's synthesizer process (steps 1–6) never writes the CSV, and §4/§12 forbid the orchestrator from reading evidence JSON ("never full evidence JSON"; census.csv is "the one structured file"). If researchers instead Write the CSV directly, parallel Q4 batches race on one file — the exact hazard §6 engineered log.sh to avoid for telemetry, with no analogous mechanism here.
* Consequence: DoD box 6 ("matrix is populated") cannot be checked off under the stated role constraints, or agents improvise concurrent CSV appends that corrupt/duplicate rows.
* Severity: high
* Fix sketch: have Q4 researchers write per-batch `matrix/q4<batch>.csv` files and make one assembly step (orchestrator or a dedicated synthesizer) concatenate them; amend §4's read whitelist accordingly.

## F2 — The cap counts Read-tool content, but the logging rule only fires on WebFetch/WebSearch
* Anchor: §5 ↔ §6 (logging rule) + §9(r1)
* Class: contradiction
* Defect: §5 defines the budget over "fetched digests + files read into context" and explicitly says snapshot content "later Read into context does" count — but the sole enforcement mechanism (§6's "after EVERY WebFetch/WebSearch... log a read event", repeated as §9 rule 1) never mandates logging Read-tool calls. Researchers Read large snapshot .txt files for quote re-derivation (§9 r3); synthesizers do almost nothing but Read.
* Consequence: telemetry systematically undercounts exactly the heaviest reads; DoD box 7's "no agent exceeded 32k" attestation and §13's divergence check (self-reported cum_tokens vs summed reads) both silently lie or spuriously warn.
* Severity: high
* Fix sketch: extend §9 r1 and §10 step 4 to "after every WebFetch/WebSearch **or Read**, log a read event"; §13's caption should say what the sums include.

## F3 — Qid case drift breaks the split-evidence glob and snapshot paths
* Anchor: §5(split-3) ↔ §11/§7 examples ↔ §8(snap.sh)
* Class: interface mismatch / ambiguity
* Defect: §5 names split children `Q4a`, `Q4b` and says the synthesizer "reads all `evidence/q4*.json`" (lowercase), while §11's example file uses `"qid": "Q4a"` but path `snapshots/q4a/slug.html`, and §7's footnote example uses `snapshots/q4/...`; snap.sh uses `$qid` verbatim as the directory. No section states a casing rule, and Linux globs/paths are case-sensitive.
* Consequence: a synthesizer globbing `evidence/q4*.json` silently misses `evidence/Q4a.json` — split evidence dropped without any error, violating §2 principle 3's "never silently dropped".
* Severity: medium
* Fix sketch: one sentence in §5 or §11: "qids are lowercased in all file and directory names; agent_ids/telemetry may use display case."

## F4 — DoD demands checks the orchestrator's read whitelist forbids
* Anchor: §1(DoD-2) ↔ §4/§12
* Class: contradiction
* Defect: DoD box 2 requires verifying every cited snapshot has `fidelity: raw` "in its meta.json", but §4 says the orchestrator reads only sections/*.md, telemetry aggregates, and census.csv — "never raw snapshots" — and §12 repeats the restriction. Fidelity does surface in §10's footnote format, but the DoD wording points at meta.json, which the checker may not open.
* Consequence: at assembly time the orchestrator must either break the §4 hygiene rule or tick the box on second-hand data while the DoD text claims a meta.json check.
* Severity: medium
* Fix sketch: reword DoD-2 to "as reported in each section's footnotes (§10 step 5)", or explicitly whitelist meta.json spot-checks.

## F5 — Predates=yes claims have no snapshot behind them
* Anchor: §3(Q4)/§7(predates) ↔ §1(DoD-1) + §2(principle 2)
* Class: missing-failure-mode
* Defect: the predates verdict rests on a CDX API response fetched by bare `curl` to stdout — no snap.sh invocation, no artifact — yet DoD-1 and principle 2 require every factual claim to carry a local snapshot path, and the matrix schema records only `wayback_url` (the captured page, which shows content, not the capture-timestamp fact). The CDX output also lands in context but, per §5's "curl doesn't count" carve-out, is never budget-logged.
* Consequence: the delta table's most decision-relevant cells are either uncited (DoD-1 fails) or cite a wayback page that doesn't actually evidence the timestamp claim.
* Severity: medium
* Fix sketch: require Q4 agents to save each CDX response (`curl ... > snapshots/<qid>/<slug>.cdx.txt` + meta.json) and cite that file in `evidence_url`/notes.

## F6 — Status vocabulary drifts across the researcher→synthesizer→findings pipeline
* Anchor: §11 ↔ §1/§10/§12 + §5
* Class: redundancy (drift)
* Defect: §11's contract enumerates `resolved | partial | unresolved | not_researchable` (lowercase, underscore); §1, §10 and §12 use `RESOLVED / PARTIAL / UNRESOLVED / NOT-RESEARCHABLE` (upper, hyphen); §5/§9 use bare `status: partial`. No section owns the canonical enum or the mapping.
* Consequence: string-matching anywhere (delta-table assembly, DoD status audit) misses variants; a mixed-case findings.md needs manual rework mid-assembly.
* Severity: low
* Fix sketch: declare the §11 spelling canonical for files and state that §12 upcases for display.
