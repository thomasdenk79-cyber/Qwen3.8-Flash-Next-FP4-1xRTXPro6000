# T01 — Upstream semantic patch audit, conflict DAG and pinned base decision
**Status:** TODO. **Dependencies:** T00. **Owner:** research agent (read-only); reviewer independently verifies. **Gate:** architecture signoff.

## Objective
Produce an exact, SHA-backed map of merged and open upstream changes **against the image's actual vLLM source**, not against \`hyudryu/main\` documentation. Determine which fixes are already included, superseded, or safe to backport.

## Procedure
1. Pin upstream vLLM source commit corresponding to the installed base; pin current \`vllm-project/vllm\` main and relevant release tags; do not overwrite image.
2. For each item in \`omo/research/PATCH_REGISTER.md\`, fetch current PR status/merged SHA, changed filenames, dependencies, patch, tests, source license, and verify if identical functionality already exists. Cross-reference base implementation including V1/V3 changes. Record \`present|absent|superseded|conflict|not-relevant\`.
3. Create file-level dependency/conflict graph for QSA: #55557 -> #57097 -> #64199 vs #54846, MTP4 V3 hooks, KV layout, \`common/qsa_cache.py\`, Mamba group manager, CPU offload, FS serializer.
4. Inspect #64199 FlashInfer dependency #6804 (availability in 0.7.3/0.7.4, experimental JIT sources missing from PyPI). Determine available SM120 implementation and source-install commit. Test with a minimal source-only import/probe on isolated container when GPU becomes available.
5. Check merged correctness shortlist: #58961, #58489, #55760/#55861, #60023, #57105, #59853, #66938 dependency #67717 and #54890. Check recent security/correctness regressions before each backport.
6. For #60967, #58863, #59366, #66881, #58413, #66407, #53051, #62383, #62776, #61225: tag blocked/unmerged/test candidates and explain exact relevance.

## Deliverables
- \`omo/runs/T01/PATCH_AUDIT.csv\` with columns \`pr,repo,status,merged_sha,base_present,changed_paths,conflict,dependency,expected_gain,tests,decision,reason\`.
- \`omo/runs/T01/UPSTREAM_LOCK.json\`: current SHA, tested base SHA, FlashInfer commit/tag, model revision.
- \`omo/runs/T01/CONFLICT_GRAPH.mmd\` and exact backport order. Reviewer signs ADR update.
- Initial per-feature branches named \`feat/kv-nvfp4-\`, \`feat/offload-cpu-\`, \`feat/offload-fs-\` only after audit.

## PASS / STOP
No guessed SHA, no patches marked merged based only on a closed state, no duplicate fixes. Do not cherry-pick or merge anything in T01. A conflict without understood semantics stays BLOCKED.