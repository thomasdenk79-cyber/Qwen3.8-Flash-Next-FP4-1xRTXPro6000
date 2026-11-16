# T09 — Merge/backport only useful upstream fixes with exact compatibility proof
**Status:** TODO. **Dependencies:** T01; can run after T04 in isolated branches. **Owner:** single compatibility implementer, reviewer. **Gate:** per-change G1/G2/G6.

## Rule
"Upstream merged" is not "missing from pinned V3." Preserve NVIDIA FP4-Local/PLE/b12x/MTP4 source hashes and never update the entire vLLM tree just to include a one-line correctness fix. This task has several **independent mini-PRs**; do not combine them in one mega commit.

## Mini-PR sequence, sorted by benefit and risk
- T09a **QSA profiling memory**: #58961, compare \`vllm/models/qwen4_exp/common/qsa_cache.py\` to V3; fix stale view references only if absent; prove graph/profile GPU memory released.
- T09b **PLE prefetch graph lifetime**: #58489 in \`nvidia/ngram_embedding.py\`; prefetch non-owning tensor lifetime across PIECEWISE graphs, repeat mixed decode/prefill; full PLE state sanity.
- T09c **MTP hybrid prefix retention**: #55760/#55861. Distinguish default dense checkpoints vs explicitly configured retention; test repeated multi-turn prefix hits and MTP proof.
- T09d **QSA scratch and MTP dtype**: #57105 reserve stable QSA indexer logits scratch; #60023 honor \`--kv-cache-dtype-skip-layers\` for MTP draft (optionally BF16); each with independent memory and draft acceptance A/B.
- T09e **QSA FP8 indexer**: #54890, benchmark indexer cache separately from main KV; validate RoPE position encoding and long retrieval. No "scale 1.0" if branch expects calibrated scales.
- T09f **Startup & lean metadata**: #58720 indexed expert loader, #59853 smaller M-RoPE table, #58114 PLE metadata; measure startup and end-to-end performance on SM120/TB4. Keep only proven improvement.
- T09g **Mid-block prefix**: #66938 merged Nov16 depends on #67717; import *full compatible dependency* only if needed for T06/T07 and tests pass. Otherwise defer.
- T09h **Fused QSA prep/projections**: #57097 and #59533 semantic conflict with #64199. Audit ordering, preserve separate NVFP4 writer staging, compare real C1/C4/C8; optional and independent.
- T09i **Additional recent merged fixes**: Refresh GitHub search for 2026-11-16 onward and inspect relevant correctness/SM120 QSA/PLE/MTP/FS changes. Record rejected candidates with reasons.
- T09j **Not for local FS baseline**: #59997 NIXL P/D PLE-page fix, unless later implementing disaggregated transfers.

## Task-local template
1. Write \`why.patch\`: bug in base, exact source SHA, failing test, current upstream merged commit, 3-way semantic diff, conflicts, expected benefit.
2. Add one minimal compatibility patch or upgrade isolated pinned image, guarded off by default if optional.
3. Write targeted unit and integration tests, plus FP8/MTP4 regression. Prove no source hash/install break.
4. Measure one A/B at same config; retain/revert and document decision.
5. Reviewer signoff, PR/commit and changelog. Only then begin next mini-PR.

## STOP
No effective tests, upstream function already changed fundamentally, wrong backend/hardware, negative quality effect, or no credible benefit: mark candidate REJECTED/DEFERRED in patch register. Do not claim all merged fixes were "merged into the fork" merely because register was written.