# T04 — Model correctness, MTP4, hybrid states, CUDA Graph and long-context gate
**Status:** TODO. **Dependencies:** T03 passed. **Owner:** reviewer-driven validation, implementation agent fixes only isolated failures. **Gate:** G2 + G3.

## Inputs
New V4 NVFP4 image from T03, byte-identical V3 FP8 control, NVIDIA checkpoint exact revision, baseline V3 draft-vocabulary state, \`V3/tests/dynamic_vocab_kernels.py\`, \`V2/benchmarks/functional.py\`, \`V2/benchmarks/compare_logits.py\`, \`V3/benchmarks/bench_general.py\`.

## Test batches
A. 7+ basic semantic probes, long JSON/schema, tool-call streaming, reasoning parser, multilingual, code fixes with test execution and finite-number logits; temperature 0/0.7, cancellation/EOS.
B. Prompt lengths \`1,2,3,4,5,6,7,8,16,63,64,65,1023,1024,2047,2048,2049,8192,32768\` against issue #53051 spec misdispatch. Check GDN state is materialized on first prefill regardless of matching MTP verify-shape.
C. Needle and content-at-end for 8K,32K,128K,200K,250K,262K; marker positions 5/50/95%; use exact tokenized prompts and output fidelity, not random text-only prompts. Include the #62776 "after indexer_budget invisible" regression with 2K,8K,32K.
D. MTP4 accepted/drafted tokens, ring dynamic 1024, V3 static vocab, draft-KV format, partial accept/reject/rollback, graph capture sizes 5/10/15/20, mixed decoding across C1/C4/C8. Run MTP-off as diagnostic only, never claim production parity from it.
E. Sequential tool turns whose prefix is re-used, mutated request/independent sessions, cancellation, duplicate prefixes and multiple concurrent resumed requests. Verify indexer/GDN state identity on unquantized internal restoration; measure NVFP4 vs FP8 quality statistically.
F. **Do not enable YaRN/500K automatically**. Separate 262,143/262,144/262,145 boundary test, only after correct YaRN config and #62383 reviewed. 350K/500K require explicit supported \`hf_overrides\` and step-by-step proof; risk of CUDA illegal access.

## Numerical checks
Where comparing different precision, use FP8-vs-FP8 noise distribution, logprob/NLL, deterministic tasks, long retrieval and accepted draft rate. For same-dtype offload/restore copy, use exact raw-state/scale checksums. Wrong target/draft output, NaN or acceptance collapse blocks release.

## Deliverables
Versioned fixture set and runner, request/response summary with no private prompt data, tests and evidence per batch, specific failed-case reproducer, \`omo/runs/T04/QUALITY_REPORT.md\`.

## PASS
All mandatory G2/G3 cases supported by the model pass; no corrupt state or crash. If 500K isn't proven, explicitly mark extended per-request context unsupported; aggregate 1.5M remains a separate goal.