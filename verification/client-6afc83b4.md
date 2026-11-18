# Client build 6afc83b4 verification

Verified against the local game on November 17, 2026 (steam.inf VersionDate Nov 17 2026, PatchVersion 1.41.9.8), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only the client build identity, RVAs, hashes and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6afc83b4, SizeOfImage 0x298d000 (was 0x298a000), disk SHA256 3074f335c1da390bcc3445f87597b8757016aa3c724852a6afa0393df00050c6.
- materialsystem2.dll: unchanged (PE timestamp 0x6af4a4ac, SizeOfImage 0x186000, disk SHA256 c256e7a296c68ba742dee6126bc59d967ac4b2060ce20ef63a7a09f9af9ca78e). All 14 pulse_material span hashes were re-checked against the installed image and still match.

## What moved

The 6af4a59c client (kept on disk, with the manifest that matched it) was used as the reference image. Code moved forwards: matched client spans shifted by between +0x950 and +0x2d80, mostly +0x2d80 (57 of 82). All 20 signature patterns are unchanged and resolve with the same hit counts as before, all shifted.

## Client span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed or shortened, and no span has a content change beyond relocated operands.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta (+0x2d80).
- 4 spans (glove_tint 5, material_retire 17 and 28, pulse_client 8) matched at exactly one neighbouring delta with no residual bytes.
- private_material 2 has an identical twin function (0x14c5dd0 and 0x14c6190); the entry at the neighbour delta (0x14c5dd0) was chosen.
- material_retire 6 (world-gloves spawn function, 3373 bytes) matched uniquely by operand-masked hash this time (0xc5bba0); its bytes are identical to the previous build after masking.
- private_material 13 and 14 are 20-byte virtual-call thunks that are not in the unwind table. They were matched by byte pattern: the pattern has the same number of hits in both builds (8 and 10), the thunks keep the same index (0 and 1), and both load the import-area global that moved with the resource import slot (+0x2000).
- material_retire 0 is the 3-byte `ret 0` no-op; it moved to 0x225a20, identified as the target of the unserialize slots, with identical bytes.

## Moved data

Globals were located by matching RIP-relative reference sets between the two builds (same referencing functions and near-identical reference counts); tables by content; the import slot by import name; the callgate table by the FrameStageNotify pointer at index 36; the interface global from the static initializer that constructs it.

- required_zero_global -> 0x26f34a8; material_cache_manager -> 0x26f3580; econ_system -> 0x25e5ea8.
- resource_global -> 0x278ec50; manager_global -> 0x278ec58; material_interface_global -> 0x278ec68 (same +8 / +0x18 relation as before).
- Resource import slot -> 0x1aa99b0 (still tier0.dll `?Purge@CBufferString@@QEAAXH@Z`).
- Glove component tables -> 0x1c82eb8 / 0x1d03cc8 (slot +8 still the submission helper, now 0x9563a0); unserialize slots -> 0x1c82700 / 0x1d03510 (both still the no-op, now 0x225a20). The addon table and high slot each had two same-shaped candidates; the pair that shares one shift (+0x1ea8) was chosen, and the addon candidate has the same two referencing-function structure as the previous build.
- Callgate: interface 0x2551080, table 0x1c5c8e0, FrameStageNotify 0xb63350 (table index 36, unique hit); the charm signature uniquely resolves to 0x7fbc40.
- Byte checks: addon_table_lea -> 0x10d04a1 (bytes unchanged), submit_capture_cmp -> 0x14ca228 (new RIP displacement), float3_jump_entry -> 0x14d98b0 (entry now 0x14d97af).

## Host validation

The candidate passed all 24 release checks on the first run: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,487 entries); both glove component tables; manager slot; UpdateSkin wrappers; HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The release harness hardcodes the two UpdateSkin wrapper addresses; they moved from 0x81b030 / 0x81b080 to 0x81bb10 / 0x81bb60. The bodies are unchanged (the same entity member displacement as the previous build). The harness was updated before the run.

The dump files at this commit (offsets.json, client_dll.json, info.json build 14198) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
