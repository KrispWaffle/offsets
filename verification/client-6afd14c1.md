# Client build 6afd14c1 verification

Verified against the local game on November 18, 2026 (steam.inf VersionDate Nov 17 2026, PatchVersion 1.41.9.8; the same Steam build label as the previous manifest, but both client.dll and materialsystem2.dll were rebuilt), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only build identities, span RVAs/hashes and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6afd14c1, SizeOfImage 0x298d000 (unchanged), disk SHA256 39de4a6d1881ebf9726a43ffcb1e05ce12d007750053afc52eed5a97b7d21c7d.
- materialsystem2.dll: PE timestamp 0x6afcfdd8 (was 0x6af4a4ac), SizeOfImage 0x186000 (unchanged), disk SHA256 2cd59750aec326ecfc49e93bd8064071e00b718b45600689c848f30090f47ae6. Only ten .rdata bytes differ from the previous image (offsets 0x131054-0x131056, 0x131070-0x131072, 0x13108c-0x13108e, 0x134bec; none are referenced by the manifest); .text, .data, .pdata and .reloc are identical. All 14 span hashes and all pulse_material table/descriptor/name RVAs are unchanged and were re-checked against the installed image.

## What moved

The 6afc83b4 client (kept on disk, with the manifest that matched it) was used as the reference image. This was a small patch: 25 of the 82 client spans are at the same RVA with identical bytes, and the other 57 moved forward by exactly +0x120 (code after roughly 0x14e3a70 shifted by that amount). All 20 signature patterns are unchanged and resolve with the same hit counts; only SKIN_SIG_COPY_MATERIALS moved (0x14e3950 -> 0x14e3a70).

## Client span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed, shortened or changed in content.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta (+0x120).
- 4 spans (glove_tint 5, material_retire 17 and 28, pulse_client 8) matched at exactly one neighbouring delta with no residual bytes.
- private_material 2 has an identical twin function (0x14c5ef0 and 0x14c62b0); the entry at the neighbour delta (0x14c5ef0) was chosen.
- private_material 13 and 14 (20-byte virtual-call thunks outside the unwind table) and material_retire 0 (the 3-byte `ret 0` no-op, the target of both unserialize slots) are at the same RVAs with identical bytes.

## Moved data

Nothing in the data layout moved. Globals were checked by matching RIP-relative reference sets between the two builds (same reference counts and the same or +0x120/+0x20 shifted referencing functions): required_zero_global, material_cache_manager, resource_global, manager_global, material_interface_global, econ_system, the callgate interface global and the callgate table all keep their addresses. The resource import slot is still at 0x1aa99b0 (tier0.dll `?Purge@CBufferString@@QEAAXH@Z`). The glove component tables (slot +8 is still the submission helper, 0x9563a0) and both unserialize slots (still the no-op at 0x225a20) are at the same addresses. Callgate FrameStageNotify is still 0xb63350 (table index 36, unique hit) and the charm signature still resolves uniquely to 0x7fbc40.

Byte checks that moved with their functions: submit_capture_cmp -> 0x14ca348 (bytes 803d5991220100, new RIP displacement), float3_jump_entry -> 0x14d99d0 (entry now 0x14d98cf). addon_table_lea (0x10d04a1) did not move.

## Host validation

The candidate passed all 24 release checks on the first run: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,487 entries); both glove component tables; manager slot; UpdateSkin wrappers (0x81bb10 / 0x81bb60, unchanged); HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The dump files at this commit (offsets.json, client_dll.json, info.json build 14198) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
