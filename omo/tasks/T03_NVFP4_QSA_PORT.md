# T03 — Port upstream #64199 NVFP4 QSA into pinned V1+V3 runtime
**Status:** TODO. **Dependencies:** T02 passed. **Owner:** single model/runtime implementer. **Gate:** G1.

## Architectural instruction
Prefer vLLM https://github.com/vllm-project/vllm/pull/64199 as **design source**. Its three touched paths are:
- \`vllm/models/qwen4_exp/nvidia/qsa.py\`
- \`vllm/models/qwen4_exp/nvidia/qsa_indexer.py\`
- \`vllm/model_executor/warmup/qwen4_exp_qsa_warmup.py\`
It uses FlashInfer sparse QSA NVFP4 writer/reader, workspace staging for fused QSA prepare, and skips Triton main-attention warmup for NVFP4. Read exact PR diff and tested versions before editing.

## Substeps (each separate coherent commit)
1. **Base adapter**: inspect pinned \`qsa.py\`, fused preparation, \`get_kv_cache_spec\`, physical page group allocation; design a compatibility adapter preserving FP8/BF16 untouched. Do not copy whole modern QSA module.
2. **Writer**: implement BF16 intermediate staging from fused prepare and FlashInfer NVFP4 paged append; verify exactly one owner for data/scales; ensure source/head/global scale application correct. Port tests from #64199/FlashInfer, adjust to pinned CUDA/Torch APIs.
3. **Reader**: integrate \`flashinfer.sparse.qsa_sparse_paged_attention\` with auto/reference kernel verified on SM120 and right workspace. Ensure qsa indexer metadata, selected page indices, sparse block ordering and K/V scales unchanged from FP8.
4. **Warmup**: update qsa warmup and JIT source gating; source install must be reproducible. No silent fallback to unsupported KV dtype. CUDA Graph replay and dynamic/captured shape smoke.
5. **Manifest packaging**: implement new \`V4/\` overlay image or independent \`experimental/nvfp4/\` source patch set with exact pre/post SHA256, test install --check --restore and V1+V3 manifest compatibility. Keep existing V3 image and scripts unchanged. The new image tag must never reuse \`qwen38-fp4-tp1:v3\`.
6. **Fallback**: default FP8 unless explicit supported NVFP4 option. Preserve V3 b12x CSF, NVIDIA PLE, draft-vocab and MTP4. No dependency on SGLang.

## Tests / acceptance
- Exact T02 allocator/no-alias tests on real vLLM cache, BF16/FP8 control, NVFP4 QSA reference, 1..16 requests, 1..4096+ token batches, eager+CUDA Graph, indexer correctness, no CUDAGraph capture allocation leak.
- Model errors must name unsupported \`NVFP4_KV/QSA\`, physical layout and absent FlashInfer source version.
- Full \`pytest\` targeted names as available from matched upstream \`tests/models/qwen4_exp/test_qsa_prepare.py\` and \`test_qsa_reference.py\`; run own tests if pinned suite differs. Exact commands recorded.
- Review source diff against PR #64199; no accidental copy of unrelated modern vLLM changes.

## STOP
Kernel only works SM80 or has missing source JIT on SM120, layout overlap, scale mismatch, MTP draft corrupt, or FP8 performance/quality breaks: revert experimental feature to off, keep original V3.