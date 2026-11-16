# T08 — SSD restart reuse, fault injection, cache integrity and 64GB pressure
**Status:** TODO. **Dependencies:** T07. **Owner:** test implementer plus reviewer. **Gate:** G5 + G7.

## Objective
Prove persisted cache is genuinely reusable after process restart with same source/model/format and bounded host RAM, without fabricated hit claims.

## Atomic steps
1. Run baseline FP8+MTP4, generate 250K prefix, write FS chunks, record hash/digest, disk files, store bytes, scheduler events and completion time. Stop experimental engine cleanly; restart same pinned image and model config, request same prefix continuation and verify nonzero FS read bytes and fewer recompute tokens.
2. Repeat with NVFP4+MTP4 under separate namespace. Capture exact K/V packed data and E4M3 scale bytes, QSA side state, GDN recurrence and PLE short-conv before eviction, after FS write and after restore. Validate independent no-cache recompute semantic oracle. Use hash-based identity only if implementation reliably keys **all** dependent states.
3. Test 64/65, 127/128, 2047/2048/2049 and MTP proof boundaries; 200K and 250K prefixes; 350K/500K only if T04 extended-context gate previously passed.
4. Inject missing cache page, truncated page, corrupted block, wrong model hash, wrong KV dtype, wrong engine version, fake scale metadata, concurrent read/write, disk-full, kill -9 during write, permission denied. Expected: logged failure, no stale hit, recompute or explicit fail-closed exception; **never incorrect output while HTTP 200**.
5. Start under 64GB cgroup bound with NVIDIA shared PLE plus 4GiB CPU tier first. Record \`memory.current\`, \`memory.max\`, \`memory.events\`, VmLck, MemAvailable, total pinned bytes, /dev/shm, file cache, swap. Repeat 8GiB only if headroom exists.
6. Add throughput/cost model: SSD cold-read speed, CPU tier stage speed, TB4 H2D copy speed, TTFT per restored 100K, recomputation t/s, eviction churn. Surface if offloading is slower than compute on this host.

## Security
Use test-only FS path and controlled artificial files. No root deletion, no test data poisoning of production. Never run fault injection on a live user container.

## PASS
The same persisted prefix (correct hash, dtype and complete state) restores after restart with nonzero FS read and validated continuation, robust failure modes, 64GB memory budget and no silent correctness failure. If not, FS cache remains experimental/off.