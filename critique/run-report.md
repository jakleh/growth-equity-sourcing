# Run report — recursive spec-critique, single clean-spec execution

Run: 2026-07-02 · Target: `SPEC.md` (Rev. 3, 47,620 chars ≈ 11,905 est tokens) · Tree depth: 1 (stopping criterion met at first partition) · Nodes: 5 leaves + 1 seam + 1 merge · Config: CHUNK_MAX=4000 / CHUNK_MIN=1500 / MAX_PARALLEL=4 / LEAF_CAP=1500.

## Per-node table

| node_id | level | §s spanned | chunk_tokens_est | output_tokens_est | findings (h/m/l + cg) | wave | wave wall_secs |
|---|---|---|---|---|---|---|---|
| L1-C1 | 1 | header, §0–§2 | 2538 | 1453 | 1/3/2 + 3cg | 1 | 233 |
| L1-C2 | 1 | §3 | 2896 | 1401 | 1/4/2 + 3cg | 1 | 233 |
| L1-C3 | 1 | §4–§7 | 2027 | 1302 | 2/2/1 + 3cg | 1 | 233 |
| L1-C4 | 1 | §8–§11 | 2436 | 1536 | 2/3/2 + 2cg | 1 | 233 |
| L1-C5 | 1 | §12–§16 | 2010 | 1677 | 0/5/2 + 5cg | 2 | 131 |
| SEAM  | 1 | whole spec | 11905 | 1403 | 2/3/1 | 3 | 159 |
| MERGE | 1 | map + all nodes (+ SPEC.md greps) | n/a | 3562 | 35 merged (6/16/13) | 4 | 366 |

**Totals:** 39 raw findings + 16 check-globally items → after merge/adjudication **35 items (6 high / 16 medium / 13 low)**, 9 checks + 2 findings resolved-by-§X (Appendix A of CRITIQUE.md). Chunk input total ≈ 11,907 est tokens (= full spec, no overlap, verbatim). Node output total ≈ 12,334 est tokens. Total wall (all waves) ≈ 889 s.

## Orchestrator spot-check (Phase 5)

7 anchors re-verified directly against SPEC.md line numbers: items #1, #3, #4, #5, #7, #10, #22 — all hold as stated. One nuance the merge kept implicit: for #3 (researchers can't see §11), researchers have the Read tool and cwd=repo root, so an agent *could* discover ./SPEC.md unaided — the defect is reliance on undirected discovery rather than strict impossibility; severity high is still defensible because nothing in the agent file or brief names the spec's path.

## Protocol deviations

1. `critique/map.md` finished at ~661 est tokens vs the ≤600 cap (first draft 1116, trimmed twice; stopped there rather than cut orientation content).
2. Leaf output caps: L1-C4 at 1536 (+2%) and L1-C5 at 1677 (+12%) vs the 1500 cap by `wc -c`; both leaves self-estimated ~1100. Not re-run — content was on-scope, and padding (not length) is the named failure mode.
3. Config `SPEC_PATH` filename did not exist; adjusted to `./SPEC.md` per the run doc's own comment (logged in telemetry spawn note) without a user round-trip.
4. Wave wall time was recorded per wave as designed; SEAM and MERGE ran as single-agent "waves" and their wall_secs are per-agent.

## Telemetry caveats (honest limits)

- All token figures are `ceil(chars/4)` estimates from `wc -c`; true tokenizer counts differ (the Fable-5 tokenizer runs ~30% heavier than pre-4.7 baselines per the design notes).
- Per-subagent thinking/usage tokens are not observable in-session; the design notes' §5 `thinking_tokens` column is **null by construction** for this run.
- Wall time resolution is per-wave for parallel leaves — per-node latency inside wave 1 is not recoverable.
- Leaf self-reported output estimates ran ~20–35% below `wc -c`-derived figures; the telemetry uses the `wc -c` numbers.

## Workflow observations

See CRITIQUE.md Appendix B (anecdotal, n=1). Headlines: no padding observed; one leaf reading error (L1-C1 flagged a term its own chunk defines); SEAM contributed 2 of 6 high-severity items that no leaf could see, plus one independent duplication of a leaf finding (convergence signal); chunk-scope note for future runs — the §7↔§9↔§10 snapshot/verification pipeline spans two chunks and generated most of the merge work; keeping §7–§11 together is worth trying.
