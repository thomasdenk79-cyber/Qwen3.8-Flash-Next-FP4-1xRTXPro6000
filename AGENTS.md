# Repository agent contract (vLLM V3)

Read [OMO_START.md](OMO_START.md), [omo/README.md](omo/README.md), and the assigned atomic task **before editing code**.

## Non-negotiable rules
1. This is the V3 vLLM/b12x **NVIDIA Qwen3.8-Flash-Next-FP4-Local** fork, not a SGLang port. Preserve V1 + V3 source patch manifests, MTP4, dynamic draft vocabulary, exact checkpoint revision, TP1, OpenAI API, and the existing launch baseline. V2 is experimental and not promoted.
2. Baseline commit: \`e833fd69a7ee8b0946008603717d605325a77285\`. Pin exact base image digest, Python packages, FlashInfer, CUDA and checkpoint revision in the recorded machine inventory **before any backport**. Do not assume current upstream source equals the pinned image.
3. **Maximum four concurrently active OMO subagents.** One orchestrator owns planning, one implementer owns a file group, one reviewer validates, and optional researcher handles sources. No multiple agents editing the same files/branch.
4. Work in one isolated feature branch per task. Each task has one narrowly bounded patch, tests, rollback, measurements and English documentation. Use frequent coherent commits and push after locally successful tests. Do not automatically merge the production branch.
5. **Never silently corrupt main or draft KV, Mamba/GDN state, QSA indexer, PLE short-conv state, or MTP acceptance.** Fail closed on unsupported layouts, incompatible cache metadata, incomplete restore and quantized-scale mismatches.
6. Upstream merged means "candidate", not "already in our base". First inventory presence by source/diff/function and existing tests. Reject duplicate fixes and avoid whole-file overlays on modified files.
7. Preserve existing checkpoint files, Docker/Podman images, port allocations and production deployment. Separate experimental container, isolated ports, unique cache directory. No force pushes, destructive cleanup or automatic model downloads.
8. No statements of GPU pass, correctness parity, C8 throughput, 1.5M KV, SSD restart persistence or warmstart until logs prove them.
9. If base source hashes differ from \`V1/patches/manifest.json\` or \`V3/patches/manifest.json\`, **STOP**. Do not bypass the manifest or install partially.
10. Read \`omo/architecture/TEST_MATRIX.md\` for gates. A PR is not complete if a gate is absent; mark it BLOCKED, do not quietly remove the test.

## Work loop
- Read task -> preflight -> create feature branch -> implement one slice -> static/unit/golden tests -> paired run -> reviewer checks -> commit -> update task status and \`omo/STATUS.md\` -> push -> PR.
- Write \`omo/runs/<task-id>/<timestamp>/\` reports only when produced on the actual host; keep large logs, secrets and prompts out of Git. Commit a sanitized summary and exact command.
- Before compaction, update \`omo/HANDOVER.md\` with exact SHA, current file set, pass/fail/blocked, next command and unresolved risks.
- Never start dependent tasks before their gate passes. See \`omo/tasks/INDEX.md\`.

## Priorities
P0: baseline/measurement, hardware-specific layout/quant correctness, NVFP4-QSA support, full hybrid restore.
P1: CPU tier, SSD tier, mixed C4/C8 concurrency and long-context proof.
P2: useful merged upstream fixes, experimental RecoverSSM/advanced eviction and perf optimizations only after their own A/B gates.

When instructions conflict, prefer correctness and reproducibility over optimization or speed.