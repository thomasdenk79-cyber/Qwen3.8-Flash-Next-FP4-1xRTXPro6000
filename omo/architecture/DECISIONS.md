# Architecture decision records

## ADR-001: Retain V3 rather than switch to Pennyroyal
**Accepted as initial base.** vLLM V3 with NVIDIA FP4-Local, V1 CSF/b12x and V3 MTP4 was published near 966 C8 output tok/s on short coding prompts and boots in the operator's environment. SGLang/Lesilva code cannot be cherry-picked into vLLM runtime. A separate SGLang experiment is outside this project.

## ADR-002: One quantized KV backend owns all bytes
**Accepted.** For NVFP4 QSA prefer vLLM open #64199 FlashInfer writer+reader. Reject stock C++ writer with token-major layout per #61225 until validated. Run backend's own exact physical-layout probes; no blind \`VLLM_KV_CACHE_LAYOUT\` workaround for unrelated writer.

## ADR-003: Tiered offload must be upstream-aligned
**Accepted, conditional on source parity.** Use OffloadingConnector/CPUOffloadingSpec/TieringOffloadingSpec FS backend; do not write a parallel HiCache. Start \`blocks_per_chunk:1\`, canonical_layout false for packed hybrid caches; separate FP8/NVFP4 FS roots.

## ADR-004: Model must remain NVIDIA FP4-Local
**Accepted.** Preserve pinned revision and current b12x; no ThinkingCap overlay, alternate PLE, draft swap or expert approximation within this program. Quantized KV is the controlled variable.

## ADR-005: 1.5M GPU tokens is a stretch goal
**Accepted.** Published V1/V2 has ~792K FP8 with MTP, ~1.1M target-only. Actual NVFP4 result depends on all hybrid KV groups and MTP memory. 1.2M minimum stretch and 1.5M aspirational; publish measured effective capacity only.

## ADR-006: Open research PRs are isolated
**Accepted.** #58863 RecoverSSM, #60967 Mamba group packing, #66881 frontier checkpoints/GDS and #64199 NVFP4 require branch-specific proof. Their open status precludes treating them as stable upstream; being open is not a reason to reject them if self-contained and fully tested.

## ADR-007: Limit orchestration fanout
**Accepted.** Maximum four active OMO subagents, strict file ownership and atomic phases. Keep entrypoints, status and handovers concise for Qwen's context budget.