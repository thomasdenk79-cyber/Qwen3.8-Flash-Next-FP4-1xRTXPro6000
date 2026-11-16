# NVFP4 QSA KV: physical contract and correctness gates

## Reference and candidate
- Reference: current V3 \`--kv-cache-dtype fp8\`, validated with NVIDIA FP4-Local model and MTP4.
- Primary NVFP4 candidate: https://github.com/vllm-project/vllm/pull/64199 (OPEN 2026-11-16), \`vllm/models/qwen4_exp/nvidia/{qsa.py,qsa_indexer.py}\` and \`vllm/model_executor/warmup/qwen4_exp_qsa_warmup.py\`. Uses FlashInfer \`nvfp4_quantize_append_paged_kv_cache_with_slot_mapping\` and \`flashinfer.sparse.qsa_sparse_paged_attention\`. FlashInfer PR https://github.com/flashinfer-ai/flashinfer/pull/6804 required, present in FlashInfer 0.7.3+, **experimental wheel missing JIT kernel sources** (source/nightly required per #64199).
- Older alternative: https://github.com/vllm-project/vllm/pull/54846, bespoke Triton QSA NVFP4 read path, stale/unmerged, conflicts with fused prepare #57097. Do not cherry-pick without a rebase design.
- Incident: https://github.com/vllm-project/vllm/issues/61225 (OPEN). vLLM 0.30 stock C++ NVFP4 writer and view reader corrupt **BLNHC** token-major layout. Head-major BLHNC worked in that test. Independent reproduction shows scale views alias data in BLNHC; writer-only stride patch is insufficient. This issue's implementation and #64199's FlashInfer writer are different: investigate independently.

## Page schema (abstract, not a hardcoded layout)
- Main QSA K/V quantized data: packed E2M1 nibbles, 2 values/byte.
- Scale blocks: E4M3 FP8 factor per 16 values plus any global K/V per-layer scales. Confirm K-linear/V-swizzled or FlashInfer-specific layout **by inspecting exact pinned writer source**.
- Page geometry derived by the selected backend; never infer allocation solely from theoretical 0.5 bytes/element.
- Indexer raw-key ring, compressed keys, positions and Mamba/PLE states are separate, generally BF16/FP32 or indexer-FP8. They are **not automatically halved** by main KV NVFP4.
- Validate main and MTP draft layer independently. MTP draft may use FP8 or BF16 even if target is NVFP4; explicit per-layer skip logic #60023 must be checked.
- CUDA graph pointer lifetime, staging scratch, kernel warmup and graph replay depend on layout identity. Do not reuse FP8 fused prepare writer for packed NVFP4 pages.

## Mandatory executable tests
1. Allocate **via the real vLLM allocator**, using Qwen3.8 effective layout, block 64 and actual K/V heads and head dimension. Do not rely on contiguous tensors.
2. Write BF16 test inputs to NVFP4 pages; extract exactly the physical data+scales via reader; compare against independent E2M1/E4M3 reference quantizer, with tolerances derived from standalone quantization error.
3. Address-overlap test: no data/scale/K/V alias at all for the allocated page, including in BLNHC and BLHNC. Unsupported layout MUST raise at startup. Include multi-page, nonzero block tables, offsetted slots and last-token edge.
4. Test QSA output against FP64 attention over **dequantized stored KV** (isolates read kernel); then FP8/BF16 model quality vs NVFP4 end-to-end (isolates quantization cost).
5. Test prefill and decode, mixed batches, chunk boundary 64/65, 4095/4096/4097, long 128K/200K/262K, CUDA eager/captured/replay and repeated graph reuse.
6. Validate K scale/V scale identity and calibrated/non-calibrated branches; no arbitrary fallback to 1.0 if checkpoint contains scales.
7. Run deterministic layout regression from #61225, long-context 3-depth needle tests, 10k+ independent metadata operations, FP8-to-NVFP4 alternating restarts (separate cache namespace).
8. MTP4: draft acceptance non-collapse, proof/rollback/cancel, EOS, temperature 0 and >0. A near-1.0 acceptance length after restore is a major red flag even if outputs seem plausible.

## Fail-closed
- Abort on invalid packed byte shape, mismatched page strides, inconsistent scale metadata, unsupported SM120 kernel or absent FlashInfer JIT sources.
- No "disable assert", ignore unknown dtype, silent BF16 fallback or copy partially packed pages.
- No NVFP4 availability claim until actual RTX PRO 6000 TP1, V3 MTP4 end-to-end long-context tests pass.

## Rebase integration order
Frozen V3 -> isolated upstream compatible reference with #64199 -> port qsa/read/write/prepare + tests -> patch manifest with before/after SHA -> FP8 reference unchanged -> internal gate -> capacity benchmark -> only then offload.
Report exact FlashInfer API/version and whether its Cake/reference backend supports SM120; do not assume from sm80 results.