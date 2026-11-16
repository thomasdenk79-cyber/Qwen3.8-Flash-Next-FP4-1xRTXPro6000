# Risk register and stop conditions

| ID | Hazard | Likelihood / impact | Detector | Required response |
|---|---|---|---|---|
| R01 | Stock NVFP4 writer BLNHC data/scale alias | known / critical | #61225 physical address test | reject layout or use jointly validated backend writer+reader |
| R02 | FlashInfer NVFP4 QSA absent from wheel | known / high | import + JIT warmup on pinned wheel | pin reviewed source/nightly, never silently fallback |
| R03 | QSA global scales mismatched to checkpoint | medium / critical | writer/reader + per-layer quality test | fail startup, no inferred 1.0 |
| R04 | Draft FP8/NVFP4 MTP KV wrong | medium / high | acceptance, forced continuation, rollback | isolate MTP layer and test dtype skip |
| R05 | Recurrent GDN/Mamba state not restored | high / critical | cold/warm recurrence and step-wise state checksum | count offload miss; no partial hit |
| R06 | PLE conv short-state omitted | medium / critical | restore and MTP acceptance/NaN | restore entire state, cf #59997 |
| R07 | QSA indexer state or RoPE positions stale | medium / critical | retrieval at 2K..262K, multi-turn | disable offload feature |
| R08 | 64GB pinned RAM OOM | high / high | cgroup memory, locked bytes, OS headroom | budget 4GiB before 8GiB, never overcommit |
| R09 | TB4 bottleneck makes offload slower than recompute | high / medium | H2D and SSD timing | classify as performance failure, keep opt-in |
| R10 | MTP4 + long context + graph wrong dispatch | known issue class / critical | 1..8 prompt, 262K boundary, graph replay | block long-context release |
| R11 | Full-model rebase breaks V1/V3 SHA manifests | high / high | installer --check/roundtrip | stop, compatibility patch only |
| R12 | Old offload files reused across incompatible dtype | medium / critical | format/key namespace tests | isolate FS root/digest, reject old metadata |
| R13 | Partial-tail store/read with chunk>1 | known issue / high | 64/65/127/128 aligned tests | start blocks_per_chunk=1 |
| R14 | Stale profiling KV views pin VRAM | known / medium | memory snapshot after graph | audit #58961 |
| R15 | Fused #57097 FP8 writer accidentally used for NVFP4 | high / critical | store-path trace, alias roundtrip | separate staging + FlashInfer writer |
| R16 | 1.5M aggregate pool misreported as per-request context | high / medium | startup logs, max_model_len | independent admission and YaRN verification |
| R17 | Artifact release without runtime test | high / critical | CI vs GPU signoff matrix | mark BLOCKED; never claim released |
| R18 | Workstream collision across agents | high / medium | overlapping git edits, concurrent branches | max four agents, one file owner per task |

All risks require reproducible evidence in the task PR. The orchestrator's correctness priority overrides throughput targets.