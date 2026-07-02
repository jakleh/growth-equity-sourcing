# CRITIQUE — SPEC.md (Rev. 3) merged critique

Run: 2026-07-02 · Nodes: 6 (L1-C1…C5 + SEAM) · Findings before dedup: 39 (7 high / 21 med / 11 low) + 16 check-globally items · After dedup + adjudication: **35 (6 high / 16 med / 13 low)**; 2 finding-pairs merged, 2 findings + 8 checks resolved by full-spec text (Appendix A).

## High

1. **No one is assigned to materialize `matrix/portfolio_venue_matrix.csv`.** §3(Q4)+§11 ↔ §4/§10/§12 ↔ §1(DoD-6) · contradiction/infeasible. Matrix rows live in `evidence/*.json data.matrix_rows`; §10's synthesizer writes only `sections/<qid>.md`; §4 forbids the orchestrator reading evidence JSON — so DoD-6 is uncheckable, or parallel Q4 batches race on one CSV. Fix: per-batch `matrix/q4<batch>.csv` + one assembly step; amend §4 whitelist. — found-by: SEAM(F1)
2. **Budget counts Read-tool content but logging fires only on WebFetch/WebSearch.** §5 ↔ §6/§9r1 · contradiction. §5: snapshot text "later Read into context does" count; the only mandated `read`-event trigger is "after EVERY WebFetch/WebSearch" (§9r1, verified). Telemetry, §13 dashboard, and DoD-7 systematically undercount the heaviest reads (quote re-derivation, synthesizer work). Fix: log after every Read too; allow file paths in `url`. — found-by: L1-C3(1), SEAM(F2)
3. **Researchers can't see the §11 contract they must match.** §9 OUTPUT ("matching the contract in the spec's §11", verified at spec l.533) · broken-cross-ref. researcher.md is the verbatim system prompt; briefs don't carry the schema, so each agent improvises the JSON shape that synthesizers and tooling key on. Fix: inline the §11 skeleton in researcher.md or every brief. — found-by: L1-C4(1)
4. **snap.sh dies silently on curl failure.** §8 (`set -euo pipefail` + `code=$(curl …)`, verified l.428–431) ↔ §9r6 · missing-failure-mode. DNS/timeout/TLS failures abort before meta.json or the snapshot event is written, leaving stale partials — the exact "silent abandonment" §9r6 calls unrecoverable, on the blocked-venue paths §9r4 anticipates. Fix: `|| code=000`, delete partials, always emit an event. — found-by: L1-C4(2)
5. **DoD boxes 7–8 assert historical facts with no remediation path.** §1 (verified: "do not stop before all boxes check") + §5 post-hoc enforcement · missing-failure-mode. A mid-wave overrun or one unlogged dead end makes the box permanently false; spec defines no downgrade/annotate procedure, so the completion signal breaks or gets silently reinterpreted. Fix: reword as "any breach documented in findings.md per <procedure>" and define it. — found-by: L1-C1(1)
6. **Synthesizers have no overrun remedy, and splits guarantee pressure.** §5 split-3 (single base-qid synthesizer reads ALL `evidence/q4*.json`) + §10.3 · missing-failure-mode. §10.3's "verify on sampled excerpts" softens but doesn't bound it: the more researchers split, the bigger one synthesizer's mandatory read set; no synthesizer split/partial path exists. Fix: per-batch synthesizers + merge step, or a synthesizer partial protocol. — found-by: L1-C3(2)

## Medium

7. **Same-day Wayback captures fall between the predates rules, and `n` is unreachable.** §3(Q4)/§7 · ambiguity. `to={YYYYMMDD}` is inclusive; a capture on the investment day is neither "strictly earlier" nor empty — unassigned. No rule ever yields `predates_investment: n` though the schema declares y/n/unknown. Fix: `to = date−1` + truncate-to-8-digits rule; define non-strictly-earlier → unknown. — found-by: L1-C2(2), L1-C3(4)
8. **Predates verdicts have no snapshot behind them.** §3(Q4)/§7 ↔ §1(DoD-1)/§2p2 · missing-failure-mode. The CDX response is bare-curl'd to stdout — no artifact; `wayback_url` evidences content, not the timestamp fact. (Minor overreach in original: §5's "curl doesn't count" carve-out is about disk snapshots, not stdout.) Fix: save CDX responses as `.cdx.txt` + meta.json and cite them. — found-by: SEAM(F5)
9. **Model-mediated fallback produces no `.txt` for the synthesizer to verify against.** §9r4 ↔ §10.2 ↔ §11 · contradiction. The Write-digest path names no files; §10 verifies "its snapshot .txt", §11 points at `.html`. Fix: mandate `<slug>.txt` + `<slug>.meta.json` under `snapshots/<qid>/`. — found-by: L1-C4(3)
10. **Dead-end `reason` vocabulary is uncontrolled and already self-inconsistent.** §14.4 filters on `bot-blocked`; §9r6's own example is `"bot-blocked-403"` (verified l.519 vs l.725) · ambiguity. Blocked venues become invisible to the fail-fast filter; budget burns. Fix: one reason enum referenced from §6/§9/§14/§15. — found-by: L1-C5(4)
11. **"Evidence has fully landed" is undefined, and Q4 spans Waves 2–3.** §14.4–6 · undefined-term. Q4 may be synthesized on partial batches or stall forever. Fix: landed = all spawned researchers for the qid logged `done` and all batches dispatched. — found-by: L1-C5(2)
12. **Cap enforcement is ambiguous between two divergent numbers, with no breach action.** §14.4 ↔ §13 ↔ §5 · ambiguity. Summed `read` est_tokens vs `done` cum_tokens diverge by design (§13 warns >2k); §5 says enforcement is post-hoc but never says what happens on breach. Fix: enforce on summed reads; on breach, annotate section + prefer splits. — found-by: L1-C5(6)
13. **No branch for Q1 failing outright.** §14.5 · missing-failure-mode. Wave 2 gates on census.csv existing; total Q1 failure (all venues blocked, splits fail) has no timeout/degraded mode — Q4 stalls or vanishes. Fix: Q1 UNRESOLVED → mark Q4 blocked-on-input and proceed. — found-by: L1-C5(5)
14. **Census reconciliation after batches are cut is unspecified.** §3(Q1/Q4) · missing-failure-mode. §14.5 handles the partial-census→Wave-3 case (demoted from high for that reason), but companies added after Wave-3 batches are cut still silently never get matrix rows. Fix: version census rows (`added_wave_N`) + pre-wave diff. — found-by: L1-C2(1)
15. **Fallback ladder's "last resort" sits after the terminal rung.** §15 and §9r4 both order live → Wayback → `unknown`+dead_end, then append WebFetch-digest as "last resort" (check-globally upheld: §9 replicates the same ambiguity). Agents will diverge on whether to try the digest before recording unknown. Fix: state ladder as live → Wayback → digest → unknown. — found-by: L1-C5(3)
16. **Qid case drift breaks split globs and snapshot paths.** §5 (`evidence/q4*.json` vs children `Q4a`) ↔ §11 (`"qid":"Q4a"`, path `snapshots/q4a/`) ↔ §8 (`$qid` verbatim) · interface mismatch. Case-sensitive globs silently drop split evidence. Fix: "qids are lowercased in all paths." — found-by: SEAM(F3)
17. **Split-evidence glob collides with child qids; synthesis ownership of children unstated.** §10.1/§11 · ambiguity. `Q4a` children's files match the parent glob; double-synthesis or orphaned children. Related to but distinct from #16. Fix: `<parent>.part-<n>.json` naming or explicit parent-only synthesis. — found-by: L1-C4(5)
18. **DoD-2 demands a meta.json check the orchestrator's read whitelist forbids.** §1(DoD-2) ↔ §4/§12 ("never raw snapshots", verified l.300) · contradiction. Also flagged as C3's check on DoD executability — upheld. Fix: reword DoD-2 to "as reported in section footnotes (§10.5)" or whitelist meta.json spot-checks. — found-by: SEAM(F4), L1-C3(cg7)
19. **"The 32k cap is enforced" (§2p6) vs attestation-grade self-report (§1-7, §5).** goodhart-risk. Self-reported, self-logged metric gating the DoD invites silent under-logging. Fix: soften P6 + one independent cross-check (bytes of snapshots read). — found-by: L1-C1(2)
20. **"Matrix is populated" has no acceptance criterion.** §1(DoD-6) · ambiguity. §7 gives columns but no row scope; headers + one unknown row passes. Fix: bind to "one row per census company × checked venue, all columns filled or unknown+reason." — found-by: L1-C1(4)
21. **Q3's "match fraction" has no denominator or negative-threshold definition.** §3(Q3) · goodhart-risk. Same data yields 40–65% depending on how `ambiguous` counts; the P0 go/no-go is decided by researcher convention. Fix: define fraction and threshold. — found-by: L1-C2(3)
22. **Early-run dashboard crash on missing columns.** §13 `board()` ↔ §6 (confirmed: `spawn` events carry no `est_tokens`/`question_id`) · missing-failure-mode. After the orchestrator's first `spawn`, rows are non-empty but `df[df.event=="read"].est_tokens` raises; the zero-rows guard doesn't cover it. Fix: reindex columns with fill values. — found-by: L1-C5(1)

## Low

23. **Zero-dead-end spot-check method** (§1-8): demoted from medium — §14.4 does define the method ("audit dead-end counts vs replies; spot-check any zero-dead-end agent with >5 fetches"), contra the leaf's "no method named"; residual gap: unlogged dead ends remain undetectable in principle. — found-by: L1-C1(3)
24. **Multi-line grep quote failure** (§7): demoted — §8's extractor collapses all whitespace to one line (`re.sub(r"\s+"," ")`, l.437), so the premise mostly fails; residual: quotes recorded with non-normalized internal whitespace still grep-fail. — found-by: L1-C3(3)
25. **Wayback timestamp acquisition absent from researcher.md** (§9r4b): demoted — Q4 briefs embed the CDX curl (§3 l.141–142); non-Q4 researchers still lack it. — found-by: L1-C4(4)
26. **`streamlit run … works` untestable** (§1-9): launch-vs-renders ambiguity. Fix: "launches AND shows nonzero read tokens per researcher." — found-by: L1-C1(5)
27. **Two referents for "telemetry"** (§2 Instrumentation vs §1/§6): Q9 researcher may conflate run instrumentation with the project's open schema question. — found-by: L1-C1(6)
28. **backlog.yaml header enumerates 2 classes; file uses `A-input` and `P0-flag`** (§3): out-of-schema values undefined. — found-by: L1-C2(6)
29. **Q1's "structured JSON list" destination unspecified** (§3 Q1): column half resolved (census columns match §7 exactly); list location still unpinned. — found-by: L1-C2(7)
30. **WebSearch `read` events have no representable source** (§6): schema keys on `url`; only dead_end gets `url_or_query`. — found-by: L1-C3(5)
31. **printf-built JSON in snap.sh does no escaping** (§8, verified l.441–444): `"` or `\` in URLs corrupts meta.json and the JSONL line. — found-by: L1-C4(6)
32. **Crashed synthesizer leaves zero telemetry and no failure signal** (§10.4; §14.4 audit doesn't check for missing `synth-<qid>.jsonl` — check upheld). — found-by: L1-C4(7), L1-C3(cg8)
33. **Status enum drift**: `not_researchable` (§11 l.594) vs `NOT-RESEARCHABLE` (§1/§10/§12) vs bare `partial` (§5/§9); no canonical owner (C4's check upheld). — found-by: SEAM(F6), L1-C4(cg2)
34. **§16 Phase-0 prompt points at "§8 Phase 0" but phases live in §14**: demoted further — the kickoff prompt itself already contains "Verify the scripts and dashboard parse. Then stop… restart" (l.749), so only the section label is wrong. — found-by: L1-C5(7)
35. **"Confirm agents are registered" has no mechanism** (§14.2): §8's gotcha says "verify both agents are invocable" but names no check. Fix: trivial spawn attempt. — found-by: L1-C5(8)

## Appendix A — resolved

- **C1.7 + C5.9** (configured-model logging/template): `resolved-by-§6` — spawn events for subagents carry `model_configured` (l.333); §12 header is fillable.
- **C1.8** (synthesizer cardinality): `resolved-by-§14.4/§10` — one synthesizer per qid, consistent with §2's "one synthesizer per question".
- **C1.9** ("multiple T's" undefined): `resolved-by-§2` — defined in the same chunk (l.47: breadth gates → deep exploration → widen N); leaf missed it.
- **C2.4** (fail-fast flag semantics): `resolved-by-§9r4/§14.4/§15` — trigger (first hard 403), setter (orchestrator marks briefs), behavior (skip live, straight to Wayback) all specified; only the literal brief format is open (and the reason-string mismatch survives as item 10).
- **C2.5** (class-B section ownership): `resolved-by-§14.8` — "B-class (Q11–Q13): no researchers. Write their sections yourself."
- **C2.7 (schema half)**: `resolved-by-§7` — census columns `company, announce_date, source_url` match Q1's exactly.
- **C3.6** ("§2.6" sub-ref): `resolved-by-§2` — principle 6 exists and is the pruning principle.
- **C4.8** (24k/32k band semantics): `resolved-by-§5` — soft-stop-fetching at 24k / hard 32k matches §9r2 exactly.
- **C5.10** ("frozen corpus"): `resolved-by-§10` — corpus defined as `evidence/<qid>*.json` + exactly the referenced snapshots; freeze point implicit in §14.4's landed-gate.

Upheld checks (folded into items above): C3.7→#18, C3.8→#32, C4.9→#33, C5.1/C5.3/C5.4/C5.6 confirmed against §6/§9/§14, C2.1's §14-gate check → partial (demotion, #14), C4.4's brief check → partial (#25).

## Appendix B — workflow observations (anecdotal, single run)

- **No padding**: all six nodes stayed on-scope; severity inflation was mild (3 items demoted on verification, none fabricated).
- **Leaf self-chunk miss**: L1-C1(9) flagged "multiple T's" as undefined when its own chunk defines it — the only clear leaf reading error.
- **Anchor drift, not misquotes**: C5.7 and C1.3 attributed gaps to the spec that adjacent text (same or other section) partially fills; no fabricated quotes found.
- **Cross-chunk blindness worked as designed**: C3.3's premise is refuted by §8 code in another chunk — exactly what check-globally/merge is for, though the leaf didn't tag it check-globally.
- **check-globally calibration was good**: 9 of ~16 checks resolved cleanly by full-spec text; leaves correctly deferred rather than asserting.
- **SEAM earned its keep**: 2 of 6 high items (F1, F2) are cross-section and invisible to any single leaf; F2 independently duplicated a leaf finding (good convergence signal).
- **Map accuracy**: good overall; it omits §14 step 8 (B-class self-write), which caused C2.5 to be raised as a medium that the spec answers.
- **Chunk scoping**: reasonable; the §7↔§9↔§10 snapshot/verification pipeline spans C3/C4 and generated the most merge work — a future split might keep §7–§11 together.
