# Source integration and patch policy

## Source baseline
\`main\` at \`e833fd69\` is identical to \`hyudryu/main\` at planning time. Existing V1 image pinned to \`ghcr.io/local-inference-lab/vllm@sha256:745feb916ee816f5409fd58386114be4f22fbede7f9c0e0a7b8c4fe8dac0fb29\` and V3 overlays installed by \`V3/scripts/install.py\`; V1's four manifest targets and V3's two targets are authoritative. All patches require exact base digest verification.

## Decision procedure for every candidate
1. Inspect merged status on GitHub **as of integration time**, compare diff to the precise pinned source, find the same function/bug already fixed or redesigned, list changed paths and tests.
2. Classify \`present\`, \`absent\`, \`superseded\`, \`incompatible\`, \`experimental\` or \`not_relevant\`. An upstream merged PR must NOT be blindly applied over a customized source file.
3. For \`absent\`, create minimal clean compatibility patch on pinned source or evaluate a newer pinned base. Preserve semantics, original licenses, comments, signed-off data; add regression tests and manifest pre/post hashes.
4. Run \`V1/scripts/install.py --check\`, \`V3/scripts/install.py --check\`, restore/install roundtrip and baseline tests against the new image. Reject hash mismatch without bypass.
5. Build distinct image tags per feature, no in-place modification of running V3. Test A/B with same hardware, power, context, dataset and sampling. Keep original image runnable.
6. All test results and true outcome go in task's result section and status. Never silently mark complete.

## Mandatory conflict clusters
- QSA: V3 base \`vllm/models/qwen4_exp/nvidia/qsa.py\`, fused QSA preparation #57097, projection fusion #59533, NVFP4 FlashInfer #64199, stale Triton #54846. Rebase on *one* chosen reference sequence.
- Storage: \`vllm/models/qwen4_exp/common/qsa_cache.py\`, #54890 indexer FP8, #58961 profiling cache release, MTP layer skip #60023. Pre-/post-page shape and lifetime tests.
- Spec: V3 modified \`vllm/models/qwen4_exp/nvidia/mtp.py\` and \`vllm/v1/worker/gpu/spec_decode/eagle/utils.py\`, modern upstream MTP rewrites. Protect exact V3 draft head behavior, acceptance, graph sizes and hash manifest.
- PLE: current pinned CUDA-mapped shared NVIDIA PLE (\`VLLM_PLE_SHARED_TABLE_DIR\`), newer #58489, #58114 and #58720. Do not replace by upstream disk PLE mode or a different checkpoint without independent proof.
- Offload: \`vllm/v1/kv_offload\`, hybrid manager and scheduler, #55760/#55861/#66938, open #58413/#66881. Never cherry-pick a single function from a state-machine series without its dependencies.
- NVIDIA megakernel and SGLang patches: **out of scope** for V3. They do not share an interchangeable runtime contract.

## Quality bars
- Correctness patch: numerical/protocol tests before performance.
- Performance patch: no visible correctness regression, >=3% repeated end-to-end advantage on matching local workload OR defensible memory/TTFT gain, accompanied by a cost/complexity review. No single microbench is sufficient.
- Changes to checkpoint output distribution, lossy format, MTP acceptance or restore representation require separate decision and explicit release note.