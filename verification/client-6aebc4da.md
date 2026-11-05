# Client build 6aebc4da verification

Verified against the local game on November 4, 2026 (steam.inf VersionDate Nov 04 2026 16:01:59), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only the build identity, RVAs, hashes and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6aebc4da, SizeOfImage 0x29a3000 (was 0x29a2000; every section after .text moved up by 0x1000), disk SHA256 9279421852772522121be2f13409dd37436e8c809ff48898c8c3d41b09e77844.
- materialsystem2.dll: unchanged, PE timestamp 0x6ae91d7a, SizeOfImage 0x186000. All 14 material-system span hashes still match.

## What moved

The 6ae91e98 client (with its manifest) was used as the reference image. Matched client spans moved by +0x40 (1), +0x120 (10), +0x170 (10) or +0x920 (61). All 20 signature patterns are unchanged and resolve with the same hit counts as before, all shifted.

## Span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed or shortened.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta (+0x920).
- 6 spans (glove_tint 5, material_retire 17 and 28, private_material 13 and 14, pulse_client 8) matched at exactly one neighbouring delta with no residual bytes.
- private_material 2 has an identical twin function; the entry at the neighbour delta (0x14cea60) was chosen.
- material_retire 0 is the 3-byte `ret 0` no-op; it moved to 0x226590, identified as the target of the unserialize slots, with identical bytes.

Per-span results are in client-6aebc4da-spans.json.

## Moved data

Globals were located by matching RIP-relative reference sets between the two builds (same reference count, same referencing functions); tables by content; the interface global by the static initializer that constructs it.

- required_zero_global -> 0x2706958; material_cache_manager -> 0x2706a30; econ_system -> 0x25f9470.
- resource_global -> 0x27a20e0; manager_global -> 0x27a20e8; material_interface_global -> 0x27a20f8; resource import slot -> 0x1ab2998.
- Glove component tables -> 0x1c8e768 / 0x1d0f3a0 (slot +8 still the submission helper); unserialize slots -> 0x1c8dfb0 / 0x1d0ebe8 (both still the no-op, now 0x226590). The high slot was distinguished from three same-shaped tables by its neighbouring entries moving exactly +0x920.
- Callgate: interface 0x2564620, table 0x1c68230, FrameStageNotify 0xb6efe0 (table index 36); the charm signature uniquely resolves to 0x801a40.
- Byte checks: addon_table_lea -> 0x10d8321 (bytes unchanged), submit_capture_cmp -> 0x14d2eb8 (new RIP displacement), float3_jump_entry -> 0x14e2540 (entry now 0x14e243f).

## Host validation

The candidate passed all 24 release checks on the first run: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,481 entries); both glove component tables; manager slot; UpdateSkin wrappers; HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The release harness hardcodes the two UpdateSkin wrapper addresses; they moved from 0x821380 / 0x8213d0 to 0x8214f0 / 0x821540 (identical bodies) and the harness was updated before the run.

The dump files at this commit (offsets.json, client_dll.json, info.json build 14196) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
