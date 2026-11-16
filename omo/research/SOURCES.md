# Primary-source research ledger

Snapshot: **2026-11-16**. This is a dated review, not a permanent assertion about GitHub. OMO T01 must refresh commit/merge statuses and file diffs before porting.

## Exact local provenance
- Owner fork https://github.com/thomasdenk79-cyber/Qwen3.8-Flash-Next-FP4-1xRTXPro6000 at \`e833fd69a7ee8b0946008603717d605325a77285\`, same as https://github.com/hyudryu/Qwen3.8-Flash-Next-FP4-1xRTXPro6000 at review.
- \`V1/Dockerfile\` pins \`ghcr.io/local-inference-lab/vllm@sha256:745feb916ee816f5409fd58386114be4f22fbede7f9c0e0a7b8c4fe8dac0fb29\`; the exact vLLM source commit in this image **must be obtained from container**, not guessed from tag.
- NVIDIA checkpoint \`nvidia/Qwen3.8-Flash-Next-FP4-Local@b9ce18a6887e18c089fdea8509a33e63429a3300\`.
- \`V1/patches/manifest.json\` hashes hyperconnection, FlashInfer b12x CSF decoder/tuning and MoE preparation.
- \`V3/patches/manifest.json\` hashes \`vllm/models/qwen4_exp/nvidia/mtp.py\`, \`vllm/v1/worker/gpu/spec_decode/eagle/utils.py\`.
- \`V2/results/startup-memory.json\`: historical FP8 792,224 tokens with MTP, 1,102,524 target-only; use as starting accounting baseline, **not current result**.

## Key upstream sources
| Evidence | Direct URL | Specific finding |
|---|---|---|
| vLLM FP8 QSA | https://github.com/vllm-project/vllm/pull/55557 | Merged; sample KV pool 434,886 BF16 to 771,704 FP8 at 0.91 GPU util, distinct build |
| vLLM experimental NVFP4 QSA | https://github.com/vllm-project/vllm/pull/64199 | OPEN; FlashInfer-owned QSA writer+reader, 593,398 NVFP4 vs 390,645 FP8 tokens on test system; SM80 benchmark, **not SM120 V3** |
| FlashInfer QSA | https://github.com/flashinfer-ai/flashinfer/pull/6804 | Experimental sparse QSA NVFP4 API; built into 0.7.3+, PyPI releases 0.7.3/0.7.4 omit experimental JIT sources according to #64199; source/nightly pin required |
| Older NVFP4 Triton QSA | https://github.com/vllm-project/vllm/pull/54846 | OPEN/stale; older alternative implementation, conflicts with newer fused prepare |
| NVFP4 layout corruption | https://github.com/vllm-project/vllm/issues/61225 | OPEN, BLNHC stock C++ writer/scales overlap; BLHNC workaround independently reproduced; not automatically identical to #64199 |
| Indexer FP8 | https://github.com/vllm-project/vllm/pull/54890 | Merged 2026-09-07; indexer separate from main QSA KV |
| vLLM tiering guide | https://docs.vllm.ai/en/latest/features/kv_offloading_usage/ | CPU and FS tiering, hybrid align, optional restart reuse with compatible config; small CPU-only tier may add no hit rate |
| Hybrid FS offload qualification | https://github.com/vllm-project/vllm/pull/57145 | Merged; Qwen3.8 FP8 TP4/eager tests, *not* V3 NVFP4 MTP4 |
| MTP hybrid retention | https://github.com/vllm-project/vllm/pull/55760 and https://github.com/vllm-project/vllm/pull/55861 | Merged; dense prefix retention defaults for hybrid+MTP |
| PLE graph-lifetime | https://github.com/vllm-project/vllm/pull/58489 | Merged; pinned PLE prefetch IDs out of graph pool |
| QSA profiling leak | https://github.com/vllm-project/vllm/pull/58961 | Merged; releases profiling references to stale KV |
| QSA workspace fragmentation | https://github.com/vllm-project/vllm/pull/57105 | Merged; reserve stable worst-case workspace |
| MTP per-layer KV dtype | https://github.com/vllm-project/vllm/pull/60023 | Merged; MTP draft layer may benefit from BF16 while target KV is FP8 |
| Mid-block Mamba resume | https://github.com/vllm-project/vllm/pull/66938 | Merged 2026-11-16; verify its dependency #67717 for port |
| Fused QSA prepare | https://github.com/vllm-project/vllm/pull/57097 | Merged; fuses BF16/FP8 QSA stores, NVFP4 path needs staging/writer |
| QSA QKVG projection merge | https://github.com/vllm-project/vllm/pull/59533 | Merged, perf candidate, broad loader/quant compatibility risk |
| Model PLE metadata | https://github.com/vllm-project/vllm/pull/58114 | Merged, MTP speculative metadata perf candidate |
| Expert load mapping | https://github.com/vllm-project/vllm/pull/58720 | Merged; ~25s loading time on DGX Spark, not promised on TB4 |
| Smaller M-RoPE tables | https://github.com/vllm-project/vllm/pull/59853 | Merged; Qwen3.8 M-RoPE cache shrink |
| PLE short-conv NIXL | https://github.com/vllm-project/vllm/pull/59997 | Merged; disaggregated P/D bug, out of initial local FS scope |
| Mamba packed groups | https://github.com/vllm-project/vllm/pull/60967 | OPEN; potential capacity improvement, need Qwen3.8 group evidence |
| Qwen GDN RecoverSSM | https://github.com/vllm-project/vllm/pull/58863 | OPEN; can reduce speculative Mamba blocks, high-risk V2 runner/metadata |
| Fused RecoverSSM MTP | https://github.com/vllm-project/vllm/pull/59366 | OPEN; stacked on #58863; don't split from dependency |
| Exact-end Mamba checkpoints / GDS | https://github.com/vllm-project/vllm/pull/66881 | OPEN, opt-in hybrid cache, test only after stable offload |
| Partial-tail >1 offload chunks | https://github.com/vllm-project/vllm/pull/58413 | OPEN; keep blocks_per_chunk=1 first |
| Partial-tail engine crash | https://github.com/vllm-project/vllm/pull/66407 | Closed unmerged; inspect whether fixed elsewhere before treating as solved |
| Prompt >262K crash | https://github.com/vllm-project/vllm/issues/62383 | OPEN v0.32-related YaRN/chunked prefill regression on another environment |
| Long prompt indexer invisibility | https://github.com/vllm-project/vllm/issues/62776 | OPEN GB10/v0.32 incident; test on TP1 SM120, don't generalize |
| Spec prefill graph misclassification | https://github.com/vllm-project/vllm/issues/53051 | OPEN reported silent GDN state issue for 1+MTP-k short prompt |

## Reference other runtime only
- https://github.com/Lesilva/qwen38-flashnext-nvfp4-kv shows ~12.16->7.15GiB per 1M tokens in its QSA representation. SGLang patch **not directly reusable in vLLM**. Use as independent numeric/capacity comparator.
- https://github.com/jpezzulli/sglang-rtxpro6000 is not the intended base. Do not import SGLang caching into vLLM.
- NVIDIA Checkpoint https://huggingface.co/nvidia/Qwen3.8-Flash-Next-FP4-Local.

## Freshness protocol
T01: GitHub API retrieve current status, merged_at, sha, changed filenames, diff, dependencies and current upstream equivalent. Save citations + exact fetched SHAs in \`omo/runs/T01/\` summary. All source claims in PRs need target hardware and actual benchmark proof.