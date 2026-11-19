# Client build 6afe3e44 verification

Verified against the local game on November 18, 2026 (steam.inf VersionDate Nov 18 2026, PatchVersion 1.41.9.9), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only the client build identity, span RVAs/hashes, the callgate table RVA and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6afe3e44, SizeOfImage 0x298d000 (unchanged), disk SHA256 9e0fd6aa19a8cb491c45ae5e9ce3d13716892a42c578e0c4911cf9c3ffaa67ee.
- materialsystem2.dll: unchanged (PE timestamp 0x6afcfdd8, SizeOfImage 0x186000, disk SHA256 2cd59750aec326ecfc49e93bd8064071e00b718b45600689c848f30090f47ae6). All 14 pulse_material span hashes and table/descriptor RVAs were re-checked against the installed image and still match.

## What moved

The 6afd14c1 client (kept on disk, with the manifest that matched it) was used as the reference image. This was a small patch: 21 of the 82 client spans are at the same RVA with identical bytes, 57 moved forward by +0x60 and 4 by +0x50. All 20 signature patterns resolve with the same hit counts as before; 13 are at identical addresses and 7 have shifted hits (SET_ATTRIBUTE, INVALIDATE_ATTRIBUTE, HUD_ADDONS and ARMS_REFRESH +0x50, COPY_MATERIALS +0x60, and the last hit of REMOVE_ATTRIBUTE and the last three of COLOR_VTABLE +0x50).

## Client span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed, shortened or changed in content.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta (+0x60).
- 4 spans (glove_tint 5, material_retire 17 and 28, pulse_client 8) matched at exactly one neighbouring delta with no residual bytes.
- private_material 2 has an identical twin function (0x14c5f50 and 0x14c6310); the entry at the neighbour delta (0x14c5f50) was chosen.
- private_material 13 and 14 (20-byte virtual-call thunks outside the unwind table) and material_retire 0 (the 3-byte `ret 0` no-op, the target of both unserialize slots) are at the same RVAs with identical bytes.

## Moved data

Globals were checked by matching RIP-relative reference sets between the two builds (same reference counts at the same addresses): required_zero_global, material_cache_manager, resource_global, manager_global, material_interface_global, econ_system and the callgate interface global (0x2551080, still constructed by the same static initializer) keep their addresses. The resource import slot is still at 0x1aa99b0 (tier0.dll `?Purge@CBufferString@@QEAAXH@Z`). The glove component tables (slot +8 is still the submission helper, 0x9563a0) and both unserialize slots (still the no-op at 0x225a20) are at the same addresses.

The only data item that moved is the callgate vtable: 0x1c5c8e0 -> 0x1c5c8f0 (+0x10). FrameStageNotify is still 0xb63350 at table index 36 (unique hit; every entry of the table is unchanged) and the charm signature still resolves uniquely to 0x7fbc40.

Byte checks that moved with their functions: submit_capture_cmp -> 0x14ca3a8 (bytes 803df990220100, new RIP displacement), float3_jump_entry -> 0x14d9a30 (entry now 0x14d992f), addon_table_lea -> 0x10d04f1 (bytes unchanged).

## Host validation

The candidate passed all 24 release checks on the first run: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,487 entries); both glove component tables; manager slot; UpdateSkin wrappers (0x81bb10 / 0x81bb60, unchanged, no harness edit needed); HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The dump files at this commit (offsets.json, client_dll.json, info.json build 14199) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
