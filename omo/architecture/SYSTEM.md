# System architecture and contracts

## Functional topology

~~~mermaid
flowchart TD
  OMO[OMO / OpenCode C1..C8] --> API[OpenAI-compatible vLLM API]
  API --> SCH[Scheduler + prefix cache + hybrid group manager]
  SCH --> RUN[V2 model runner + V1 b12x + V3 MTP4]
  RUN --> QSA[QSA main KV, FP8 reference / NVFP4 experimental]
  RUN --> IDX[QSA indexer raw key / compressed keys]
  RUN --> GDN[GDN / Mamba recurrent + conv]
  RUN --> PLE[CPU mapped NVIDIA PLE + short conv state]
  RUN --> DRAFT[MTP draft KV + proof / verify / commit]
  SCH <--> OFF[OffloadingConnector]
  OFF <--> RAM[CPUOffloadingSpec / bounded pinned RAM]
  RAM <--> FS[TieringOffloadingSpec FS backend]
  FS <--> NVME[/srv/llm/kv-cache versioned namespace]
~~~

## Invariants
1. The target model and served output distribution are unchanged by the draft-vocabulary optimization; retain V3 speculative rejection-sampling correctness. NVFP4 KV is a numeric approximation and must undergo *separate* quality validation.
2. Never confuse main QSA KV with indexer KV/position ring, recurrent/GDN state, PLE short-convolution state, and MTP verifier/draft state. Profile all pools independently.
3. The owner of KV page geometry, quantized packed-data placement, scale placement and dtype must be a single backend implementation. No asymmetric C++ writer plus FlashInfer reader.
4. Prefix-cache key, page/chunk alignment, model revision, quant schema and all auxiliary state must agree on restore. A partial hit cannot be advertised as a complete hit.
5. Store and promote **only complete proved prefixes**. A resumed speculative turn must retain MTP proof lookahead and correct rollback of rejected drafts.
6. Keep CUDA Graph warmup/capture and eager execution equivalent; unsupported layouts fail with explicit errors.
7. Preserve single-GPU TP1 with external TB4: measure host-to-GPU transfer bandwidth and time independently of NVMe I/O.
8. Preserve vLLM API semantics, streaming, tool calling, reasoning parser, structured output, request cancellation, stop sequences and normal EOS.

## API/contracts
- Config toggles (descriptive names, **not fabricated current CLI flags**): \`kv_format=fp8|nvfp4\`, \`kv_offload=off|cpu|tiered_fs\`, \`kv_layout=head_major|backend_specific\`, \`runtime_profile=c1|c4|c8\`. Implement only with actual supported upstream flags after T00 source inspection; any new flag must be explicit, validated and off by default.
- Runtime introspection must report effective target KV dtype, draft KV dtype, indexer dtype, group sizes/bytes, host chunk geometry, CUDA graph allocated bytes, PLE bytes and final GPU token pool. Prefer log plus metrics; no unpublished magic numbers.
- Offload key includes effective model revision and cache namespace; all scale buffers and hybrid state are checked against a versioned physical manifest. If upstream already provides that mechanism, use it without duplicating format identity.

## Out of scope initially
Model re-quantization, ThinkingCap overlay, multi-GPU TP/EP, prefill/decode disaggregation, speculative model swap, NVIDIA megakernel, modified route semantics or approximation, provider-hosted cache service. Opening these options before KV correctness is a scope failure.

## ADR-A: Prefer V3 as performance baseline
V3 has measured C8 ~965.6 aggregate on short-code output and known working b12x QAD weight loader. A modern vLLM full rebase could break exact source SHA and draft hooks. Build isolated modern upstream probe, then select minimal backports. Do not move production baseline before equivalent tests.

## ADR-B: Prefer #64199 as NVFP4 QSA design candidate
#64199 uses FlashInfer-owned NVFP4 QSA read AND writer; it is open and relies on experimental FlashInfer source/nightly, not stock release wheel. No SM120 end-to-end proof in the PR. Alternative #54846 predates fused #57097 and needs major rebase; it exposed stock writer/layout concerns. Port #64199 only after T02/T03 risk gates.

## ADR-C: Conservative offload
Use vLLM's OffloadingConnector and tiered FS backend, not SGLang HiCache code. Enable CPU only after GPU KV is stable, then filesystem tier. Advanced cache policies and open hybrid checkpoint PRs are independent experiments, never prerequisites to a correct baseline.