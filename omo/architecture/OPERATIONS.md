# Deployment, startup, logging, observability and rollback

## Preserve production
Existing \`V3/scripts/serve.py\` runs a Docker container named \`qwen38-v3\` port 8000. **Do not stop or replace that container** as part of a research task. Use a new image tag, container name and port, checking actual GPU availability and avoiding two complete 96GB models on one device. Running both concurrently may OOM; schedule an explicit maintenance/test window.
Use Podman if target host is Podman-managed, adapting bind mounts and NVIDIA device configuration after host preflight. Do not assume a Docker launch works unchanged with Podman/WSL.

## Runtime controls
- Distinct HF/PLE shared-table directories, XDG/Triton/FlashInfer caches, FS tier root and logs for each branch/profile. Never point experimental cache keys into a production namespace.
- Startup checkpoints (UTC monotonic timestamps): process start, Python imports, model weight mmap/load, PLE shared memory attach, MTP head load, kernel warmup/JIT, cache allocation, CUDA graph capture, API ready, first token. Record cold JIT, warm JIT, cold OS page cache and warm page cache separately.
- T15 64GB budget: shared NVIDIA PLE ~26.82GiB (published V1), optional 4/8GiB pinned tier, OS/agent RAM, Python RSS and page cache. Measure \`/proc/meminfo\`, cgroup memory and \`nvidia-smi\`; don't promise 8GB pinned safe.
- TB4 PCIe: measure \`nvidia-smi dmon\`, Nsight/copy profiling and end-to-end restore; avoid attributing transfer stalls to SSD without evidence.

## Core metrics
- \`GPU KV cache size\`, maximum concurrency, all \`KV cache group\` bytes, MTP accepted/draft token counters, \`cached_tokens\`, offload store/load/promotion/hits/misses/bytes, GPU/CPU/FS quota, CPU memory/locked bytes, p50/p95 TTFT & TPOT, graph memory, prompt recompute tokens, request preemptions.
- Verify actual metric names in the pinned engine; add Prometheus dashboards only after the exporter surfaces them. No invented metric names.
- Structured logs must include \`feature\`, \`fmt\`, \`layout\`, \`cache_key\` (hashed), \`request_id\` (sanitized), \`tier\`, \`bytes\`, \`time_ms\`, \`outcome\`, \`reason\`, \`repo_sha\`, \`image_digest\`. Never log raw prompts or auth keys.

## Rollback
1. Stop only the experimental container after draining requests. Never alter the production V3.
2. Retain previous image digest and launch config. Use original V3 unchanged.
3. Do not reuse NVFP4 FS cache namespace with FP8 or a newer format; old data stays isolated until cleanup is explicitly reviewed.
4. Update \`omo/STATUS.md\` and relevant task with failure mechanism, reproducer, logs, manifest checksum and exact rollback action.
5. In high-risk failures (silent corruption, offload stale state, engine crash), disable the specific feature by default until validated fix lands.

## License and security
Keep model weights out of Git. Preserve third-party license headers, exact SHA256 manifests and NVIDIA checkpoint terms. Check custom kernel license when vendoring FlashInfer experimental source. No untrusted downloaded code execution without reviewing pinned commit. Avoid unsafe pickle or executable artifacts from anonymous repos.