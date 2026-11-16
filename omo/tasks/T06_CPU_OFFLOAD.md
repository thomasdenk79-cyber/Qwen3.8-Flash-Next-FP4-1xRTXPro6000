# T06 — CPUOffloadingSpec and hybrid prefix checkpoint correctness
**Status:** TODO. **Dependencies:** T04 passed; T01 upstream catalog. **Owner:** KV cache implementer, reviewer isolated. **Gate:** G4.

## Objective
Make modern vLLM \`OffloadingConnector\` CPU-tier prefix cache work with actual V3 TP1 V2 runner, MTP4 and Qwen hybrid groups. **Do FP8 first**, then NVFP4. Do not write a parallel custom HiCache.

## Investigation
- Inspect whether pinned base contains \`vllm/v1/kv_offload/cpu/spec.py\`, \`vllm/v1/kv_connector\`, \`OffloadingConnector\`. If absent, build a branch with required dependency chain from upstream; don't import a single helper by itself.
- Reference https://docs.vllm.ai/en/latest/features/kv_offloading_usage/: \`CPUOffloadingSpec\` with \`cpu_bytes_to_use\`, hybrid Mamba align + partial-tail checkpoint, \`blocks_per_chunk=1\`, MTP lookahead proof tokens.
- Audit #55760/#55861 dense retention fixes and #66938 dependency chain for mid-block reentry. Inventory source presence before port.
- Check that serialized group blocks contain main QSA packed bytes and scales, indexer state/positions, GDN recurrence, PLE conv, MTP draft/proof state. Which states are recomputed must be explicit and validated, not presumed.

## Small commits
A. CPU-only connector and config (default off).
B. FP8 full prefix store/restore, same-process.
C. FP8 MTP4 partial-tail/Mamba checks, repeated agent turn.
D. NVFP4 physical packed pages and scale storage without canonical_layout, GPU->CPU->GPU exact bytes.
E. Controlled CPU tier sweep 4GiB,8GiB while 64GB cgroup and PLE memory accounting remain valid.

## Example modern upstream setting (verify runtime)
~~~json
{"kv_connector":"OffloadingConnector","kv_role":"kv_both",
 "kv_connector_extra_config":{"cpu_bytes_to_use":4294967296,"blocks_per_chunk":1}}
~~~

## Tests
- Full FP8 no-offload baseline vs CPU tier, first compute then GPU eviction then CPU hit; exact GDN/PLE/indexer state or equivalent continuation proof.
- Prompt boundaries 64/65, 2048/2049, 262143/262144; repeated MTP accept/reject; two unrelated conversations and prefix hash invalidation.
- Track CPU hits, saved prefill tokens, H2D volume, p95 TTFT, memory.current, mlock bytes. Stop on stale checkpoint, acceptance collapse or OOM.

## PASS
Demonstrated CPU external hit with correct continuation and validated 4/8GiB host budget. If small CPU-only cache provides no useful retention, document this and proceed to FS stage rather than faking a performance gain.