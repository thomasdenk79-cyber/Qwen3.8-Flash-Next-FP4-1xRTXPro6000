# Atomic OMO work queue and dependency DAG

Each task resides in \`omo/tasks/Txx_*.md\`. Follow exact order. At most four OMO subagents, but **one writer per owned file subtree**.

| Task | Purpose | Dependency | Gate |
|---|---|---|---|
| T00 | Machine/image/model baseline and memory accounting | none | G0 |
| T01 | Upstream and patch conflict inventory | T00 | architecture signoff |
| T02 | NVFP4 layout and standalone allocator proof | T01 | G1 |
| T03 | FlashInfer #64199 NVFP4 QSA port | T02 | G1 |
| T04 | MTP4, QSA, CUDA Graph, long-context correctness | T03 | G2+G3 |
| T05 | GPU KV capacity instrumentation and optimization | T04 | measurable capacity |
| T06 | CPUOffloadingSpec and hybrid checkpoint proof | T04 | G4 |
| T07 | TieringOffloadingSpec filesystem tier | T06 | G5 |
| T08 | SSD restart, failure injection, 64GB memory budget | T07 | G5+G7 |
| T09 | Merged upstream correctness/perf backports | T01; before release | individual regression gates |
| T10 | Qwen Mamba/RecoverSSM capacity experiments | T05+T08 | optional evidence, no default |
| T11 | C1/C4/C8 performance, OMO task throughput | T05+T08+T09 | G6 |
| T12 | Startup, deployment, telemetry, rollback | T11 | G7 |
| T13 | Final audit, docs, release candidate | T12 | ALL |
| T14 | New-upstream follow-up & maintenance plan | T13 | future |

~~~mermaid
flowchart LR
  T00-->T01-->T02-->T03-->T04
  T04-->T05-->T11
  T04-->T06-->T07-->T08-->T11
  T01-->T09-->T11
  T05-->T10
  T08-->T10
  T11-->T12-->T13-->T14
~~~

## Definition of "done"
- Exact source SHAs and runtime effective flags recorded.
- All relevant unit/GPU/serving tests **executed**, with pass/fail evidence. CPU-only checks may unblock preparation but not GPU release.
- Reviewer inspects quantitative evidence and source diff; no silent fallback.
- Independent commit(s), pushed feature branch, PR with rollback, updated \`omo/STATUS.md\` and \`omo/HANDOVER.md\`.
- Failed or untestable branch remains **BLOCKED** and default-off, even when code compiles.

## Working style
T00 and T01 may share one prep session. One task per implementing agent; use researcher on upstream PR diffs and reviewer on G1/G2 protocol while implementer owns runtime files. Always finish a task before starting another that modifies the same modules. Do not make one enormous implementation prompt.