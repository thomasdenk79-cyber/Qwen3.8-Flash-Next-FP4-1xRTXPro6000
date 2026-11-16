# Tiered cache: GPU -> bounded CPU -> filesystem

## Correct upstream mechanism
Current vLLM already provides \`OffloadingConnector\`, \`CPUOffloadingSpec\` and \`TieringOffloadingSpec\` (https://docs.vllm.ai/en/latest/features/kv_offloading_usage/). Modern source: \`vllm/v1/kv_offload/{cpu,tiering}/\`, \`vllm/v1/kv_offload/tiering/spec.py\`. This is **prefix-cache offload**, not magically live active-request KV migration.

~~~mermaid
sequenceDiagram
  participant S as Scheduler
  participant G as GPU KV + hybrid groups
  participant H as CPU pinned primary
  participant F as Filesystem tier
  S->>G: Complete block / checkpoint
  G->>H: async store + proof/metadata
  H->>F: asynchronous secondary write
  S->>F: lookup by full compatible prefix hash
  F->>H: stage selected chunks (limited RAM)
  H->>G: async load complete required groups
  G-->>S: hit only after all required data is valid
  S->>G: resume continuation (MTP proof / GDN)
~~~

## Example configuration (modern upstream only; VERIFY CLI support and exact schema on pinned build)
~~~json
{
  "kv_connector": "OffloadingConnector",
  "kv_role": "kv_both",
  "kv_connector_extra_config": {
    "spec_name": "TieringOffloadingSpec",
    "cpu_bytes_to_use": 8589934592,
    "blocks_per_chunk": 1,
    "eviction_policy": "lru",
    "secondary_tiers": [{
      "type": "fs",
      "root_dir": "/srv/llm/kv-cache/vllm-v3-nvfp4-exp",
      "n_read_threads": 8,
      "n_write_threads": 8
    }]
  }
}
~~~
The JSON is a **design example**, not a V3-ready deployment command. Under the vLLM guide, CPU primary has direct GPU access; filesystem secondary is staged via CPU and may contain blocks from earlier runs. \`canonical_layout\` does not support packed/cross-layer KV layouts, so **default it off** for experimental NVFP4/hybrid until proven. Do not conflate filesystem tier with NIXL connector: this local FS backend is not "NIXL POSIX FILE" by default. NIXL is a separate P2P/OBJ transport in upstream; naming must be accurate.

## Sequencing and tests
A. CPU only with FP8 + MTP4, then NVFP4 + MTP4. Determine if V3 pinned core has OffloadingConnector; if not, build an isolated pinned-source backport or test the latest compatible upstream with V3 patches revalidated.
B. Tiered FS with FP8, then NVFP4. Reuse a different directory and cache format identity per dtype and model revision; no sharing across incompatible engines. Set explicit 4 GiB and 8 GiB CPU test arms and record whether promotion finishes before eviction.
C. A session 200K+, paused, displaced by independent requests, then resumed: require **a real secondary hit**, logged bytes read, no full re-prefill, correct continuation against an FP8-no-offload semantic oracle. Use both full-block and partial-tail boundaries. Repeat across engine restart separately.
D. Cancellation, disk-full, truncated file, bad metadata, permission denied, checksum mismatch, unsupported quant format and tiny CPU buffer must degrade to a logged cache miss/recompute or clear failure, never corrupted output.
E. Host 64GB budget: NVIDIA PLE shared table published as ~26.82GiB CUDA-mapped, plus 8GiB pinned cache, OS, python, filesystem cache, worker RSS, tool agents. Measure cgroup memory.current, MemAvailable, pinned bytes, fragmentation and swap pressure. Do not set cache to an assumed safe size.
F. TB4: report SSD read rate, staged CPU copy throughput, host-to-GPU copy throughput and TTFT. If 200K restore is slower than recompute, the offload may still be correct but not economically useful.

## Important upstream warnings
- Upstream documentation: CPU-only tier smaller than aggregate GPU cache can just mirror hot blocks, giving little extra hit rate. A small CPU tier makes sense **only if the secondary tier actually persists and serves** blocks.
- Hybrid Mamba align and MTP require partial-tail checkpoints and proof tokens; upstream recommends \`blocks_per_chunk:1\`. PR #58413 for >1 remains OPEN.
- Cache policy/retention #55760 + #55861, PR #66938 mid-block split, and offload bug fixes must be audited before measuring hits.
- On-disk FS backend creates \`<model>_<digest>\` and per-rank block directories; compatible restart reuse is **conditional** on hashes, cache format and key. Test on the actual branch.
- Do not use large filesystem cleanup commands; use per-experiment path only. Disable or reduce PLE page-cache pressure before blaming the SSD tier.

## Definitions
Cold prefill: none of required prefix resident in any tier.
GPU warm: direct GPU prefix cache hit.
CPU warm: GPU miss, complete compatible CPU block+state hit.
SSD warm: GPU/CPU miss, required valid groups promoted from FS.
Restart warm: new process reuses prior on-disk blocks with validated identity and successful continuation.
Report all five separately.