# T05 — Exact GPU token capacity, group accounting and memory optimization
**Status:** TODO. **Dependencies:** T04. **Owner:** profiling implementer; reviewer confirms accounting. **Gate:** stable capacity proof.

## Goal
Measure *effective* FP8 vs NVFP4 pool with NVIDIA FP4-Local+V3 MTP4, not assume scaling by 1.7. Aim 1.2M; 1.5M stretch. Never reduce correctness/throughput unmeasured.

## Steps
1. Record GPU allocator snapshot after model load, after each warmup class, before/after CUDA Graph, and after all groups allocate. Break down main QSA packed K/V/scales, indexer raw/compressed, GDN recurrent, PLE conv, MTP attention/draft, activation and graph; reconcile with total reserved memory.
2. Reproduce original V2 historical **792,224-token FP8 / 14.59GiB** state on matching options if possible; treat as historical only. Compare current installed V3 FP8, V4 NVFP4, MTP-off diagnostic, and FP8/FP16 draft KV as conditional experiment.
3. Compute each group's physical bytes per token (incl alignment/page padding). Show why adding one additional KV group/slot can lower total token pool; identify bottleneck group and allocator fragmentation.
4. Audit merged #58961 profile memory cleanup, #57105 stable indexer scratch, #54890 FP8 indexer and #59853 smaller M-RoPE. Check #60023 MTP-layer BF16 escape; cost acceptance and VRAM explicitly.
5. Audit open #60967 packed Mamba group planning as capacity experiment in T10, NOT silently include.
6. Sweep GPU util 0.90/0.93/0.95/0.96 after verifying no OOM, graph capture sizes, max-num-seqs and PLE host settings in independent arms. Include display/eGPU effects. Keep 2+GiB operational GPU headroom unless GPU tests justify another value; never hardcode mem_fraction from other runtimes.
7. C8 mixed load e.g. four ~230K prefixes + four ~24K = ~1.016M before outputs. Request pool must also cover output/spec/fragmentation; record queueing and preemption.

## Deliverables
\`omo/runs/T05/ALLOCATION.csv\` before/after by group and \`CAPACITY_REPORT.md\` with exact source, checkpoint, effective flags, GPU memory and independent per-request limit. Include 1.2M/1.5M feasibility verdict and gap budget.

## PASS
Stable reproducible number of GPU-resident tokens with all correct hybrid states at MTP4; no artificial count by CPU/SSD tier; no OOM at the tested concurrency. If <1.2M, report quantified bottleneck rather than declaring false success.