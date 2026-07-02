# Map of SPEC.md (Rev. 3) — orientation only, zero evaluation

A Claude Code "orchestrator" session runs parallel researcher subagents + offline synthesizers resolving 13 questions, producing findings.md with snapshot-backed citations.

## Sections

- **Header** — spec is self-contained (supersedes handoff doc); all sessions cwd = repo root.
- **§0** Assumptions 1–5: 32k cap; model fallback + self-report unreliability; no paid keys; no nested subagents; WebFetch returns a lossy AI digest, not the page.
- **§1** Objective + 9-item Definition of Done checklist.
- **§2** Distilled project context (tiers, set construction, lens, instrumentation, survivorship bias) + 7 principles.
- **§3** Mapping table (six [OPEN] → Q8–Q13) + backlog.yaml, Q1–Q13 (classes A/A-input/B, priorities, briefs).
- **§4** Roles; ≤4 researchers/wave; orchestrator reads only sections/*.md, telemetry aggregates, census.csv; replies ≤10 lines.
- **§5** Caps 32k hard/24k soft/20k warn; est_tokens=ceil(chars/4); evidence files uncapped; post-hoc enforcement; split protocol.
- **§6** Telemetry JSONL events; scripts/log.sh (stdin); synthesizers Write telemetry/synth-<qid>.jsonl; log after every fetch.
- **§7** Raw curl snapshots (scripts/snap.sh: .html/.txt/meta.json); grep quote check; model-mediated fallback; footnotes; CDX predates check; 2 CSV schemas.
- **§8** Repo-root scaffold tree; snap.sh code; .claude/settings.json allowlist; restart-to-register gotcha.
- **§9** researcher.md verbatim: tools/model, 9 hard rules, output contract.
- **§10** synthesizer.md verbatim: Read/Write only; entailment check vs .txt; section template; one-shot telemetry.
- **§11** evidence/<qid>.json schema (claims, quote_verified, snapshot_fidelity, dead_ends, split_proposal).
- **§12** findings.md template (delta table, exec summary, uncertainty register, method note) + DoD check.
- **§13** Streamlit dashboard code: reads telemetry/*.jsonl, sums read events only, cross-checks cum_tokens.
- **§14** Runbook: scaffold → restart → execute; waves 1–4; between-wave audits; Wave 2 gated on census.csv.
- **§15** Guardrails: public pages only; fallback ladder; polite fetching; CDX only; unknown is first-class.
- **§16** Kickoff prompts (Phase 0, Phase 1).

## Load-bearing cross-references

§5+§6 schema → §9/§10 rules, §13 dashboard, §1 DoD · §7 snapshot/quote rules → §9 r3–4, §10 step 2, §11 fields · §8 scripts+settings → §9 r1/r3, §7, §14 · §3 mapping → §12 delta table · Q1 census.csv → §4 + §14 Wave-2 gate · §0.5 WebFetch → §7, §9 · §0.2 model self-report → §6 done, §12.
