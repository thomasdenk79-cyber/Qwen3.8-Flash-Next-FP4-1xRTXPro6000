# Patch selection register (engineering decisions, 2026-11-16)

A merged patch is **not automatically absent from the pinned vLLM image**. T01 must compare. This table is a shortlist of work, not a declaration of installed fixes.

| Priority | Upstream | Current review status | Target / conflict | Decision |
|---|---|---|---|---|
| P0 | #55557 FP8 QSA | merged | FP8 baseline | Audit present; never revert |
| P0 | #58961 QSA profile leak | merged | \`common/qsa_cache.py\` | Backport if absent: memory correctness |
| P0 | #58489 pinned PLE graph lifetime | merged | \`nvidia/ngram_embedding.py\` | Backport if absent: graph correctness |
| P0 | #55760 + #55861 MTP hybrid dense prefix retention | merged | engine args/config | Backport if absent before offload |
| P0 | #57097 fused QSA prepare | merged | \`nvidia/qsa.py\` & \`ops/qsa_prepare.py\` | Audit before #64199; preserve FP8 current path |
| P0 | #64199 FlashInfer NVFP4 QSA | **open** | \`nvidia/qsa.py\`, \`qsa_indexer.py\`, warmup | Primary candidate; experimental source dependency |
| P0 | #61225 stock C++ NVFP4 BLNHC bug | **open issue** | writer/reader + layout | Mandatory negative test; refuse incompatible layout |
| P1 | #54890 FP8 QSA indexer | merged | common cache/indexer | Feature-gated separate test |
| P1 | #60023 KV dtype skip per layer | merged | \`nvidia/qsa.py\` | Audit MTP draft acceptance, then enable if proven |
| P1 | #57105 QSA workspace fragmentation | merged | logits workspace | Backport if absent, verify allocation |
| P1 | #66938 mid-block Mamba resume | merged 2026-11-16 | hybrid block manager, dependency #67717 | Port entire dependency chain only after CPU tier proof |
| P1 | #59853 compact M-RoPE | merged | RoPE & model | Small memory/startup win if absent |
| P1 | #58114 PLE metadata | merged | PLE and MTP | A/B TTFT and sync overhead |
| P1 | #58720 expert-mapping load | merged | \`RoutedExperts.load_weights\` | Low-risk potential startup improvement |
| P1 | #59997 NIXL PLE page | merged | disaggregated P/D | Out of scope until P/D; keep regression note |
| P1 | #59102 NIXL host-buffer chunk store | open | NIXL P/D | Not our local FS path; study for generic store boundaries |
| P2 | #59533 fused QSA projections | merged | QSA weights, loader, quant and #64199 | Optional only after NVFP4 stable; SM120 performance must win |
| P2 | #58957 fused HC down+SiLU | merged | HC V1 b12x overlap | Avoid duplicate until V1 vs new A/B |
| P2 | #60967 compact Mamba groups | open | planning and physical group layout | High memory potential; isolated design + test, don't cherry-pick blind |
| P2 | #58863 RecoverSSM GDN+PLE | open | ~34 files; MTP state | High-risk capacity candidate; separate epic |
| P2 | #59366 fused RecoverSSM | open, stacked #58863 | fused MTP | Defer until #58863 qualifies |
| P2 | #66881 exact-end Mamba+GDS | open | scheduler/hybrid checkpoint ~34 files | Later A/B for idle agent reuse |
| P2 | #58413 partial-tail >1 chunks | open | offload chunk alignment | Use chunks=1; defer |
| P2 | #54846 older NVFP4 QSA | open/stale | replaces same QSA paths | Reject as default; reference and fallback only |
| P2 | #66865 specialized GB300 decode | open | GB300 specific | Reject for TP1 SM120 until actual SM120 path |
| P2 | #59869 SM120 skinny plans | closed unmerged | GEMM tuning, overlaps b12x | Experimental only, not production dependency |
| P2 | #66407 multi-boundary partial tail | closed unmerged | offload scheduler | Audit alternate fix and retain crash regression |
| P2 | #53051 short-prefill MTP graph bug | open issue | dispatch/metadata | Include 1..8-token prompt regression gate |
| P2 | #62383 >native-262K crash | open issue | YaRN/chunked prefill | Long-context stage gate, no automatic 500K |
| P2 | #62776 >indexer-budget invisibility | open issue | QSA indexer | 2K/8K/128K long-document regression |
| P3 | SGLang/Lesilva NVFP4 patches | different engine | incompatible runner/cache | Reference only; no direct import |

## Practical sequence
1. Freeze pinned image source manifest and diff against current upstream.
2. Port only missing P0 *correctness* repairs (unit/e2e/rollback each).
3. On a dedicated branch implement #64199 NVFP4 writer+reader and FlashInfer dependency; prove FP8 unchanged.
4. Validate GPU capacity (not deduced from main QSA alone).
5. Add CPU then FS offload; only then revisit Mamba/precision optimization PRs.
6. A/B P1/P2 individually. Keep rejection log showing why patches were skipped.

The register must be revised after T01 (explicit source hashes, file diff overlap, new PR statuses, actual test results).