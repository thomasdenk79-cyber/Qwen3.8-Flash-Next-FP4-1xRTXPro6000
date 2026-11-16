# Qwen3.8 Flash-Next V3: NVFP4-KV and tiered-prefix project

**Research snapshot:** 2026-11-16. **Status:** architecture / task pack only; no runtime modifications or GPU tests have been performed as part of this package.

## Scope and ground truth
- Target repo: \`thomasdenk79-cyber/Qwen3.8-Flash-Next-FP4-1xRTXPro6000\`, base commit \`e833fd69\`, identical to \`hyudryu/main\` at creation.
- Runtime: vLLM V2 model runner (not "vLLM engine V3"), custom V1 b12x/hyperconnection patches + V3 draft-vocabulary/MTP4 patches, NVIDIA FP4-Local pinned revision \`b9ce18a6887e18c089fdea8509a33e63429a3300\`.
- Baseline launch: \`V3/scripts/serve-args.json\` and \`serve.env\`, \`--kv-cache-dtype fp8\`, block 64, \`max-model-len=262144\`, \`max-num-seqs=16\`, TP1; V1 image digest in \`V1/Dockerfile\`.
- Existing published on-host memory snapshot (\`V2/results/startup-memory.json\`): **792,224 FP8 KV tokens with MTP**, about **14.59 GiB KV**, ~74.6 GiB weights+non-torch, ~1.98 GiB peak activation and ~0.25 GiB CUDA graphs, 0.96 utilization; target-only control **1,102,524 tokens**. This is an Oct 29 reference, NOT a present T15/P16 measurement and NOT NVFP4 capacity.
- Published V3 short-prompt coding output tps (original vs V3): C1 257.4 -> 287.0, C4 669.5 -> 662.2, C8 943.4 -> 965.6. Prompts ~235-257 tokens; not 200K agent throughput. Source: \`README.md\`.
- Hardware target RTX PRO 6000 Blackwell SM120 96GB TP1, **Thunderbolt 4 eGPU**, Linux, optional host 64/192GB RAM. Do not extrapolate direct PCIe x16 or GB300/Rubin results.

## Execution map
1. **Baseline and provenance** T00-T01: pin exact source, log sizes, upstream audit, restore unit baseline.
2. **NVFP4 GPU KV** T02-T06: layout contract, port, writer/reader tests, MTP/long-context gates, measure true capacity.
3. **Host and disk offload** T07-T10: CPUOffloadingSpec first; TieringOffloadingSpec second; hybrid state + restart + no-cache controls.
4. **Optimization and release** T11-T14: merged fixes, open PR probes, performance and startup gates, staged deployment.

Read \`omo/tasks/INDEX.md\` and one task file at a time. Architecture is in \`omo/architecture/\`; latest evidence in \`omo/research/\`.

## Architecture decisions (preliminary)
- Keep NVIDIA FP4-Local checkpoint and V3 draft head. **No ThinkingCap mixing and no SGLang conversion in this project.**
- Prefer upstream vLLM #64199 NVFP4-QSA design (FlashInfer writer + reader) over stale #54846 custom Triton path, conditional on SM120 and pinned-build validation.
- Do not blindly force \`VLLM_KV_CACHE_LAYOUT=BLHNC\` as a universal fix. Layout depends on the selected writer/reader. The C++ stock NVFP4 path is known unsafe with BLNHC (#61225); the #64199 FlashInfer path is a distinct implementation.
- Model group state (QSA main K/V, indexer, GDN/Mamba, PLE conv, MTP proof) must be fully restorable. GPU KV compression does not imply offload support.
- CPU-only offload does not enlarge GPU-resident KV, and a tiny CPU cache may merely mirror GPU prefix blocks; the SSD tier must demonstrate actual net reuse.
- vLLM current upstream is moving quickly; port **semantic fixes with tests**, not dozens of files wholesale.

## Evidence / references
See \`omo/research/PATCH_REGISTER.md\` and \`omo/research/SOURCES.md\`. Every claimed benchmark or cache capability must identify the exact runtime build, checkpoint and workload.