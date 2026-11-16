# Capacity model and baseline evidence

## Hard baseline from repository
File \`V2/results/startup-memory.json\` includes 2026-10-29 RTX PRO 6000 measurements under the published V1/V2 experimental image:
- MTP enabled: FP8 pool **792,224 tokens**, **14.59 GiB current KV cache memory**.
- Model + non-torch approx **74.6 GiB**, activation peak **1.98 GiB**, CUDA Graph pool **0.25 GiB**, desired util 0.96.
- MTP disabled target-only: pool **1,102,524 tokens**, KV allocated **17.69 GiB**, model + non-torch **71.45 GiB**, CUDA Graph pool **0.09 GiB**.
- These are historical **V1/V2** measurements on one reference machine, not measured V3 pool, not guaranteed for TB4/host 64GB. The graph pool was ~0.25GiB in this reference: do not claim it alone consumes many GiB.

## Why 1.5M cannot simply be inferred by multiplying 792K by 1.7
- Main QSA K/V can compress approx 1 byte/element to 0.5+scales bytes, but a hybrid vLLM pool includes group alignment, indexer caches, GDN/Mamba state, MTP draft/verify groups, staging and page allocator geometry. Only some bytes shrink.
- Lesilva SGLang's published main-QSA storage ~12.16GiB/1M FP8 to ~7.15GiB/1M NVFP4 is useful reference, **not a vLLM complete-group budget**.
- 792,224 × 1.7 ≈ 1.35M is a mathematical *all-bytes-compress* upper heuristic, not a projected measured pool. Actual increase could be significantly lower; need per-group allocations.
- \`max_model_len=262144\` remains per-request admission ceiling independently of total pool. A 500K request needs separate RoPE/YaRN correctness and allocator planning.

## Inventory template
| Component | FP8 bytes | NVFP4 bytes | How observed |
|---|---:|---:|---|
| Model GPU weights and QAD CSF metadata | TBD | unchanged | loader logs / torch memory |
| Main QSA K/V physical pages | TBD | TBD | cache spec per layer × pages |
| QSA indexer raw key and compressed key pages | TBD | same until indexed-FP8 validated | cache group spec |
| GDN/linear attention recurrent state | TBD | same | Mamba cache spec |
| PLE short-conv state | TBD | same | PLE group spec |
| MTP draft attention/group state | TBD | may require dtype override | group spec |
| CUDA graphs/activation/staging | TBD | TBD | profiler snapshots |
| Fragmentation/reserve / display | TBD | TBD | nvidia-smi + allocator stats |

## Required computations
- Each cache group: \`physical_bytes_per_block = ceil(sum(layer/group bytes, padding, scale, alignment))\`; \`blocks = floor(assigned_bytes / physical_bytes_per_block)\`; effective max resident tokens is constrained by **all** group allocations, not only main KV.
- Compare same weights, MTP4, graph config, block size, max_num_seqs and max_model_len.
- Record total GPU free, available reserve, peak pinned host memory, NVMe capacity before using 0.96 utilization. If any production stability issue, use lower utilization rather than failing open.
- Goals: 1.2M stretch 1.5M total pool; mixed C8: 4×230K + 4×24K = 1,016K input tokens before output/proof/reserve. Do not allocate to 100% pool, and allow for outputs, MTP and partial blocks.
- Target-only run is an **isolated accounting diagnostic**, not a replacement for production MTP4. Benchmarking no-MTP as MTP4 would be invalid.

## Optimization levers, in order
1. Correct NVFP4 QSA physical storage.
2. Verify FP8 indexer (#54890), packed-Mamba group accounting (#60967 OPEN), cached profiling KV release (#58961), and MTP draft dtype override (#60023).
3. Explicit per-layer memory accounting, avoid unnecessary workspace reservation; check #57105's stable workspace allocation before changing it.
4. Evaluate RecoverSSM (#58863 OPEN) for speculative recurrent-state capacity if it passes TP1 SM120 + MTP4 correctness. This is a **large state-machine change**, not a free tweak.
5. Any changed max-num-seqs, cudagraph capture sizes, max-num-batched-tokens or max_model_len requires independent A/B and may not save substantial memory.

## Go/no-go
Success is measured stable effective GPU pool, workload completion, hit correctness and throughput. If the pool remains under 1.2M, produce a byte-accounted gap and new evidence-backed ADR. Do not fabricate 1.5M by conflating SSD/CPU capacity with GPU-resident pages.