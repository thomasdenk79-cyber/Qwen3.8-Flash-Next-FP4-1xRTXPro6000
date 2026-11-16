# T00 — Immutable V3 baseline, image provenance and memory accounting
**Status:** TODO. **Dependencies:** none. **Owner:** implementer; reviewer independent. **Gate:** G0.

## Objective
Establish exact reproducible baseline on T15/RTX PRO 6000 TP1 without altering production; determine deployed vLLM source SHA, installed FlashInfer version, model SHA, CPU PLE map and all GPU KV group allocations. No NVFP4 code yet.

## Read
\`V1/Dockerfile\`, \`V1/patches/manifest.json\`, \`V3/Dockerfile\`, \`V3/scripts/serve-args.json\`, \`V3/scripts/serve.env\`, \`V3/patches/manifest.json\`, \`V2/results/startup-memory.json\`, \`omo/architecture/MEMORY_BUDGET.md\`.

## Atomic actions
1. Create \`feat/t00-baseline\` from the reviewed planning branch or original \`main\` plus docs. Record git status/HEAD and upstream hash. Preserve production container.
2. Record driver, GPU PCIe/TB4 link, GPU power/temperature, host RAM, cgroup max/current, kernel, NVMe, OS, Python/Torch/vLLM/FlashInfer/NIXL/transformers versions and all binary image digests.
3. For actual running container, inspect resolved \`vllm\`, \`flashinfer\` package installation paths and pinned vLLM \`__version__\`/commit metadata. **Do not guess source commit from a Docker tag**. Extract target-source SHA256 for all V1/V3 manifest targets.
4. Execute read-only strict patch checks *inside a throwaway copy of the same image*; verify original and modified hashes and preserve exact V3 server args/env (redact credentials).
5. Start an isolated benchmark deployment only in a planned GPU-exclusive test window. Collect cold/warm engine startup timestamps and complete allocation log including QSA, indexer, Mamba/GDN, PLE state, draft KV, activations, CUDA graph and real GPU token pool.
6. Collect 3 repeat measurements at C1/C4/C8 using existing \`V3/benchmarks/bench.py\` and \`bench_general.py\`, with counts, MTP acceptance, full completions and output quality. No numerical claim until repeats finish.

## Commands (adapt container runtime)
~~~bash
git rev-parse HEAD
git status --porcelain=v1
sha256sum V1/Dockerfile V1/patches/manifest.json V3/patches/manifest.json
nvidia-smi --query-gpu=name,memory.total,memory.used,power.draw,temperature.gpu,pci.link.gen.current,pci.link.width.current --format=csv
docker inspect qwen38-v3 --format '{{.Image}} {{.State.Status}}'
docker logs qwen38-v3 2>&1 | grep -Ei 'GPU KV cache size|Maximum concurrency|CUDA graph pool|Mamba|PLE|FlashInfer|VRAM|version'
# For Podman-managed hosts replace docker with podman. Record a separately inspected actual image digest.
~~~
Do not execute install.py without verifying its invocation and target site-packages; \`--check\` is intended to be non-mutating.

## Deliverables
\`omo/runs/T00/SUMMARY.md\` (sanitized), \`omo/runs/T00/ENV.json\` (no secrets), resolved source hashes, per-group memory table, tagged FP8 reference metrics, start time profile, baseline rollback instructions. Update \`omo/STATUS.md\`.

## PASS
Pinned source found, manifests verified; FP8+MTP4 serves correct short/long requests and measurements are repeatable. If no GPU access, status BLOCKED with exact remaining commands. Never alter serving source or delete runtime caches here.