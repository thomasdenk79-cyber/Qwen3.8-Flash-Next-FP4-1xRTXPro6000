# Test matrix, exact gates and benchmark definitions

## Environments
- CPU-only CI: static analysis, manifest checks, config validation, tensor-schema fakes, filesystem format/key tests and property-based state machine tests. Must not claim GPU parity.
- GPU-A: RTX PRO 6000 Blackwell SM120 96GB, TP1, Thunderbolt 4, Linux; record host T15/P16, 64/192GB RAM, driver, CUDA, Python, Torch, vLLM image digest, FlashInfer SHA, QAD model revision, clock/power limits.
- GPU-B optional modern-upstream reference, distinct image and no shared cache namespace. Compare in paired ABBA starts, initial JIT excluded from warm start but separately measured.

## Gates
G0 Baseline reproducible: original V1+V3 patch roundtrip, same model revision, V3 MTP4 semantic/throughput benchmark, startup and full memory-group logs.
G1 KV numerical: real allocator BLHNC/BLNHC tests, writer-reader no-alias, FP64 over dequantized cache, scales checked, OOB/NaN/sanitizer, eager vs graph.
G2 Model quality: simple deterministic truths, code editing with tests, JSON/tool parser, 8K, 32K, 128K, 200K, 262K retrieval at 10/50/90%; optional 350K/500K only after YaRN proof; medium-long natural code documents, multilingual. FP8 vs NVFP4 controls.
G3 Spec state: V3 MTP4 acceptance, mixed prefill/decode, rejection/rollback, cancellation, EOS, temperatures 0/0.7, concurrent 1/2/4/8, full/draft cache groups, short prompts length 1–8 (cf #53051).
G4 Host tier: FP8 and then NVFP4 CPU-only offload, valid prefix hit, no stale GDN/PLE/indexer state, 4/8GiB host pressure controls.
G5 FS tier: actual SSD read/write counters, 200K+ eviction restore, partial-tail boundaries, two distinct requests sharing prefixes, process restart, corrupt/truncated files, wrong namespace/dtype, OOM/no-space, no silent reuse.
G6 Performance: C1/2/4/6/8, cold and five cache states, prompt 1K/8K/32K/128K/200K/250K, output 256/1024/1536+, long-context 350K/500K conditional. Report throughput, TTFT p50/p95, TPOT, rate of completed coding tasks per hour, preemptions, MTP acceptance, PCIe/SSD bytes and startup latency.
G7 Deployment: separate image and port, rollback, no production modification, observability and reproducible release artifact.

## Non-negotiable references
- Historical V3 benchmark uses short prompts (~235 tokens) and 1536 outputs; never report it as 250K-agent throughput.
- Run FP8 original, FP8 with only upstream correctness fixes, NVFP4 GPU only, NVFP4 + CPU tier, NVFP4 + FS tier. Only one experimental dimension changes per comparison.
- Absolute bitwise equality is not required across altered FP8/NVFP4 numerics or known nondeterministic MoE routing; compare NLL/logprobs against within-FP8 noise, task results, needle recall and GDN-state accuracy. Where state/load/store should be byte-identical, **require byte identity**.
- For equal workload C8 long-session run, no claim of capacity or throughput unless all intended requests complete; note queuing and cache stalls.
- Require 0/0 silent corruption, 0 unexpected CUDA illegal memory access, 0 unauthorized stale offload hits, 0 untracked format conversions. Any failure immediately disables affected feature.
- A credible SSD hit must show nonzero FS read bytes, reported external cached tokens, lower recomputed tokens and matching continuation after eviction/restart. \`cached_tokens\` alone is insufficient.

## Bench records
Each run writes sanitized JSON: timestamp, repo SHA, image digest, model SHA, GPU and host, flashinfer/toolchain hashes, effective flags and env allowlist, memory budget, prompt tokens/output tokens, request counts, all finish reasons, throughput, TTFT/TPOT distributions, MTP acceptance, offload counters, error/log references and exact command. Do not commit prompt content, tokens or private credentials.