# T07 — TieringOffloadingSpec local filesystem storage
**Status:** TODO. **Dependencies:** T06. **Owner:** cache implementer, one writer. **Gate:** initial G5.

## Goal
Implement or backport vLLM's existing multi-tier \`TieringOffloadingSpec\` with local FS secondary. Do NOT claim local FS is automatically NIXL POSIX; it is distinct. Keep CPU staging bounded to 4/8GiB and let SSD keep many inactive agent prefixes.

## Source landmarks
- https://docs.vllm.ai/en/latest/features/kv_offloading_usage/
- \`vllm/v1/kv_offload/tiering/spec.py\`, \`manager.py\`, \`factory.py\` and FS secondary implementation in installed upstream tree.
- \`OffloadingConnector\` KV keys, cross-run format digests, hash seed, complete-hybrid-group specifications.
- Upstream #57145 hybrid FS reload tests; check actual dependencies and runner support.
- Open #58413 blocks_per_chunk>1 partial-tail issue: **start 1 only**.

## Atomic steps
A. Port or enable FS backend (existing code wherever possible), add command-line/config validation, separate isolated path \`/srv/llm/kv-cache/v3-fp8-test\`, stage via bounded CPU tier, active logs and disk quotas.
B. FP8+MTP4 same-process store, evict, FS hit. Require nonzero disk *read* counter and no full recompute.
C. Hybrid state/proof restored at full and partial-tail boundaries. Confirm no stale QSA indexer positions, GDN conv, PLE state and draft KV; compare decoded output against recomputation.
D. NVFP4+MTP4 with a different FS namespace \`/srv/llm/kv-cache/v4-nvfp4-test\`; verify scales and data exact after GPU->CPU->FS->CPU->GPU.
E. Compare 4 vs 8GiB host stage, 4/8/16 FS read threads and SSD speed at 200K/250K; retain only demonstrably useful configuration.

## Safety
- No shared directory with production or mixed formats. Stage records must be versioned, key namespace tied to model hash, quant layout and engine spec.
- Disable \`canonical_layout\` while using packed/cross-layer KV as upstream explicitly rejects it; if future implementation supports it, qualify separately.
- No dangerous recursive delete. On corrupted/truncated chunks or missing scale sidecars prefer fail-closed miss with clear diagnosis.
- Unique mount, quota and permission; avoid silently consuming entire SSD.

## PASS
At least one true SSD external prefix hit under memory pressure on same host with full hybrid state and MTP4, validated TTFT/recompute delta, 64GB RAM budget. Restart persistence is **T08**, not inferred.