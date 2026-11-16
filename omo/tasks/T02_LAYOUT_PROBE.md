# T02 — Physical NVFP4 writer/reader and SM120 kernel feasibility probe
**Status:** TODO. **Dependencies:** T01. **Owner:** implementer, isolated experiments only. **Gate:** G1 precondition.

## Specific target
Evaluate vLLM #64199 with FlashInfer QSA NVFP4 writer+reader on RTX PRO 6000 SM120. Do not modify model weights and do not switch production to NVFP4.

## Small commits / substeps
A. Build a separate reviewed FlashInfer experimental source/nightly installation pinned to exact SHA; document license and package ABI vs Torch/CUDA. Confirm \`flashinfer.sparse.qsa_sparse_paged_attention\` and \`nvfp4_quantize_append_paged_kv_cache_with_slot_mapping\` actually exist and JIT compile in SM120. Negative test stock PyPI 0.7.4 missing JIT sources if reproducible.
B. Port standalone allocator geometry from the *installed* vLLM V3 base, create a scratch-only test harness that writes BF16 K/V + scales to pages and reads them; compare to independent reference. Cover tokens 1,63,64,65,128,4095,4096,8193 and multi-page block tables, K/V heads and global scales.
C. Reproduce #61225 BLNHC alias and BLHNC valid case with **stock C++ writer path** in isolated probe; separately test #64199's **FlashInfer-owned** physical layout. Never infer FlashInfer's packing from C++ behavior. Verify address-level nonaliasing before any decode.
D. If SM120 unsupported or JIT incompatible, STOP and deliver upstream-rebased alternative study (#54846), with performance/accuracy risk. Do not emulate NVFP4 with an FP8 cache while claiming success.

## Files likely touched
Scratch tests under \`V3/tests/nvfp4/\` or \`tests/nvfp4/\`; prefer out-of-tree to avoid rewriting pinned source. If tests need model backend imports, record exact paths from #64199 \`vllm/models/qwen4_exp/nvidia/qsa.py\`, \`qsa_indexer.py\` and current installed spec.

## Evidence
- Full exact GPU test commands + compiler command, output hashes, K/V and scale physical offset geometry, relerror on dequantized stored data; writer/reader agreement, no overlapping data/scale K/V, no OOB/NaN, supported SM120 path.
- Latency microbench allowed as diagnostic only. Validate graph warmup separately.
- CPU-only tests do not satisfy this gate.

## PASS
The intended FlashInfer NVFP4-QSA backend can read exactly the bytes its writer creates, across real-layout pages and graph replay. If not, stop; no subsequent NVFP4 implementation permitted.