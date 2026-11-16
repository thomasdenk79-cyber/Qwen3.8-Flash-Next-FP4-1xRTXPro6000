# T12 — Deployment safety, startup latency, metrics, rollback and host profiles
**Status:** TODO. **Dependencies:** T11. **Owner:** ops implementer, reviewer. **Gate:** G7.

## Deliverables in code
1. Isolated Docker/Podman image with pinned exact reviewed git commits, hashes, CUDA/FlashInfer and model revision. Keep V3 original runnable, add \`V4/\` or scoped \`experimental/\` launcher, don't replace \`V3/scripts/serve.py\`.
2. Declarative host profiles \`t15-64gb\`, \`p16-192gb\`, profile \`fp8-control\`, \`nvfp4-gpu\`, \`nvfp4-cpu\`, \`nvfp4-tiered-fs\`. Verify actual flags with installed parser. Launch rejects invalid combinations/layout/kernel source and unsafe memory settings.
3. Startup instrumentation: checkpoint opening, PLE shared map, main/MTP weights, NVFP4 JIT first build vs cached, GPU KV pool, graph capture, first ready API, first token; report total and phase timers.
4. Prometheus counters/gauges for KV groups and bytes, MTP acceptance, tier read/write/hit, offload recompute, p95 TTFT/TPOT, GPU+host RSS, filesystem bytes, data integrity rejects. Verify metric availability rather than promising fake names.
5. Health probes for ready and generate 1-token/short retrieval; optional extended smoke with known fixed fixture. Fail readiness if selected NVFP4 path missing.
6. One-command \`check\`, \`benchmark\`, \`rollback\` flows that never touch original V3 production and never delete shared cache state.
7. Document Linux/TB4 eGPU constraints, permission and mounts, disk quota, 64GB memory budget, restart policy and latency SLO. Distinct port and container names; no simultaneous full models on same GPU.

## Tests
Config schema/CLI tests, source SHA mismatch refusal, fake missing FlashInfer API, simulated full disk/partial write, CPU cgroup OOM guard, fresh process after stopped experimental worker, exact rollback to V3. Log redaction audit (no prompts/tokens/credentials).

## PASS
Proven rollback, reviewed GPU image, instrumented startup and offload, safe separate container controls, CI green and runtime report.