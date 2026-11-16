# T14 — Maintain upstream patch discipline
**Status:** TODO. **Dependencies:** T13. **Owner:** orchestrator/maintainer.

Monthly or at each relevant upstream release:
1. Refresh the vLLM PR register (merged, open, superseded) and FlashInfer experimental QSA API status, NVIDIA checkpoint revision, Torch/CUDA/drivers and known corruption issues. Preserve exact SHA diff snapshots.
2. Run static manifest checks and CPU tests in CI. GPU correctness/performance only in a dedicated scheduled GPU window; don't run a second full model concurrently.
3. Compare modern upstream source to custom V1 b12x, V3 MTP, V4 NVFP4 and offload patches. Favor upstream feature retirement only after matching quality and throughput prove parity.
4. Re-run numerical/QSA layout/no-alias, MTP4, FS restart, partial-tail, long-context 262K/YaRN and C8 throughput gates before switching base images.
5. Maintain a searchable ADR and rejection log. Preserve previous known-good image, model revision and FS namespace for rollback.
6. Update \`AGENTS.md\` only when architecture changes; minimize prompt inflation and keep each OMO task self-contained.

Completion: repeatable upgrade workflow with a tested PR and a versioned release; not a background monitoring promise.