# T10 — High-risk open PR research (optional): Mamba capacity / RecoverSSM / agent cache retention
**Status:** TODO. **Dependencies:** T05 + T08. **Owner:** architecture researcher then one implementer per *isolated* experiment. **Gate:** stand-alone G1-G6.

## Candidate A: #60967 — packed Mamba group allocation
Open PR \`vllm-project/vllm#60967\` changes \`vllm/v1/core/kv_cache_planning.py\` and \`vllm/platforms/interface.py\`. It reduces Mamba padding for block-outer layouts and changes state group packing. Benchmark on Qwen3.8 TP1/MTP4 with actual cache-group specs, not GLM/SM90 numbers. Check dependency #55449 and physical cache ABI. Protect offload serialization/namespace and V3 allocator.
Decision: try only if it saves measurable VRAM/raises GPU token pool and preserves all state; otherwise defer.

## Candidate B: #58863 (+ #59366) — RecoverSSM GDN+PLE
OPEN https://github.com/vllm-project/vllm/pull/58863; changes ~34 files, introduces \`--use-replayssm\`, verify replay records, compact MTP commit, eliminates per-draft Mamba blocks. Requires V2 runner, GDN Triton and particular CUDA Graph metadata; #59366 adds fused CUDA MTP but is stacked on #58863.
Proof: per-token GDN/PLE recurrent states, MTP accept/reject, partial-cache restore, C4/C8, SM120, real 262K prompt. Compare cost of Triton fallback vs existing V3 b12x. No automatic migration; never mix half of protocol.
Decision: highest speculative capacity opportunity if measured memory delta is large enough, but risk of slower C1 and broken MTP is major.

## Candidate C: #66881 — exact-end Mamba frontier checkpoints / GDS
Open, changes scheduler/block manager/state graph (~34 files), potential high value for long multi-turn OMO. Explore *after* FS offload baseline with 200K–500K conversation traces and churn. Test latency, external hits, eviction fairness and state proof. Must preserve exact-end and full-group correctness.

## Candidate D: open offload fixes and risks
- #58413 partial-tail \`blocks_per_chunk>1\`; keep 1 until proven.
- #66407 closed unmerged multi-boundary engine crash; check replacement patch before enabling partial-tail optimizations.
- #59102 NIXL host-buffer P/D chunk fix: distinct from local FS path; import only on that architecture.
- #64199 main NVFP4 QSA already handled in T02/T03, NOT repeated.

## Stop conditions
Tests cannot execute SM120 TP1; source ABI incompatible; requires unreviewed TensorRT/FlashInfer version; C1/C4 slowdown without substantial memory gain; cache serialization incompatible with saved prefixes; any silent quality regression. Publish a rejection or defer note rather than merging speculative code.