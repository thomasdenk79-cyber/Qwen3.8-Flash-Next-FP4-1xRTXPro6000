# T13 — Independent architectural release audit and candidate signoff
**Status:** TODO. **Dependencies:** T12. **Owner:** reviewer, no editing of implementation branch until review finding accepted. **Gate:** all G0–G7.

## Review checklist
- [ ] Exact NVIDIA checkpoint revision and output semantics unchanged except documented NVFP4 KV numeric approximation; V3 MTP4 dynamic/static vocab still works.
- [ ] V1+V3 manifest and dependency hashes reproducible; no downloaded unpinned binaries or silently altered source.
- [ ] Physical NVFP4 data/scale placement tested with **real GPU allocator**. BLNHC corruption #61225 negative regression and chosen writer+reader positive regression.
- [ ] Correct main QSA and independent indexer/GDN/PLE/MTP auxiliary state, graph and speculative rollback.
- [ ] CUDA Graph and eager, cold and warm, multi-turn, C1/C4/C8 and 128K/200K/262K retrieval pass.
- [ ] Host CPU tier external hits and FS tier external hits proven by bytes, not just names. Reuse after restart tested with compatible config, failure injection logged.
- [ ] Short prompts 1..8 and native-context boundary tested for known upstream issues #53051/#62383.
- [ ] Exact GPU KV size published with complete group byte budget; no claim of 1.5M if not measured.
- [ ] No unauthorized production changes, safe rollback, separate cache namespace.
- [ ] Upstream patch register updated with status, diff, tests and rejected/deferred candidates. Every merged patch in runtime has exact SHA and license.
- [ ] 64GB host memory and TB4 stage costs verified; no unmeasured default 8GiB pinned allocation.
- [ ] C1/C4/C8 performance tested against current baseline under same workload and actual completed tasks; negative results transparent.
- [ ] User-visible README, deployment, known issues, numerical tolerances, failure paths and restart limitations documented.

## Final artifacts
\`omo/release/RELEASE_EVIDENCE.md\`, \`VERSIONS.json\`, \`KNOWN_LIMITATIONS.md\`, \`TEST_MATRIX_RESULTS.csv\`, \`ROLLBACK.md\` and signed reviewer note. A no-go or partial release must clearly label unsafe features default-off. Production activation is a separate explicit operator action.