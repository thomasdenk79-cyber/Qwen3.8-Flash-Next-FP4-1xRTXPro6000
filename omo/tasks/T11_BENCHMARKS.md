# T11 — Long-context OMO C1/C4/C8 benchmarks and throughput economics
**Status:** TODO. **Dependencies:** T05 + T08 + T09 relevant passed changes. **Owner:** perf researcher, independent reviewer. **Gate:** G6.

## Correct baseline
The original V3 publication measured coding 287 C1, 662 C4 and 966 C8 tok/s with short ~235-257 input tokens and 1536 outputs (not long-context). Repeat it, then compare with NVFP4 and offload in same host/clock/driver environment. Report original short-prompt numbers only as historical source benchmarks, not current T15 claim.

## Workloads (deterministic datasets, real token IDs)
- C1, C2, C4, C6, C8; 1K/8K/32K/128K/200K/250K contexts, 256/1024/1536 output tokens, both \`enable_thinking\` on/off when supported.
- Mixed C8: 4×230K implementer prefixes + 4×24K exploration prefixes (before output ~1.016M). Verify each request actually admitted and completes; no hard-coded 1.5M needed if pool smaller.
- One 250K agent, leave idle, other agents evict GPU KV, return via CPU/FS and continue coding.
- Real coding: Python/Rust/Go/TypeScript fix and test, Oracle/PostgreSQL SQL migration, bug location, multi-file edits, use a local fixed repository fixture and evaluate tests; avoid exposing proprietary prompts.
- Cache states: cold, GPU warm, CPU hit, SSD hit, post-restart hit.
- Power normalized; stable thermal state; no simultaneous alternative engine on 96GB GPU.

## Metrics and method
- Per-request TTFT, TPOT p50/p95/p99, total completion tokens / wall second, per-agent decode rate, queue wait, MTP accepted/draft/verify, prefix hits and recompute, CPU/FS bytes, GPU used/free, PCIe bandwidth, actual coding tasks passed per wall hour.
- Aggregate throughput comparisons must include completed requests; incomplete/stalled runs invalid, list counts.
- Paired ABBA alternating runs, 3+ clean rounds minimum and more for close deltas. Same sampling, checkpoint, input IDs, output cap, model revision and CPU cache state; warmup excluded but startup reported separately.
- Measure exact 250K context throughput; do not extrapolate 1K short prompt rates. Distinguish prefill+decode E2E vs pure decode-only.

## Release decision
No meaningful throughput regression in long-agent C1/C4 vs FP8 V3 without explicit ADR; C8 improves tasks/hour or capacity/reuse without stale states. 1.2M+ KV and 900+ tok/s C8 remain desired thresholds, not automatic pass. Cost/quality decides final choice.