# Client build 6aff8722 verification

Verified against the local game on November 19, 2026 (steam.inf VersionDate Nov 19 2026, PatchVersion 1.42.0.0; this is a new patch version, not a rebuild of the same label), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only the client build identity, span RVAs/hashes, callgate and data RVAs, and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6aff8722, SizeOfImage 0x298e000 (was 0x298d000), disk SHA256 07103341bc9e0ae2c596b2929075c5094cb397d7e370ea00986c720ed72414e1.
- materialsystem2.dll: unchanged (PE timestamp 0x6afcfdd8, SizeOfImage 0x186000, disk SHA256 2cd59750aec326ecfc49e93bd8064071e00b718b45600689c848f30090f47ae6). All 14 pulse_material span hashes and table/descriptor RVAs were re-checked against the installed image and still match.

## What moved

The 6afe3e44 client (kept on disk, with the manifest that matched it) was used as the reference image. Code moved forward by small amounts that depend on the region: 61 of the 82 client spans moved by +0x60, 7 by +0x70 (material_retire 2-6 and the two private_material thunks), 2 by +0x10 (glove_tint 0, the submission helper, and material_retire 1), and 12 are at the same RVA. Of the 20 signature patterns, 5 are at the same addresses and 15 shifted; every pattern keeps its hit count.

## Client span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed, shortened or changed in content.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta (+0x60).
- 4 spans (glove_tint 5, material_retire 17 and 28, pulse_client 8) matched at exactly one neighbouring delta with no residual bytes.
- private_material 2 has an identical twin function (0x14c5fb0 and 0x14c6370); the entry at the neighbour delta (0x14c5fb0) was chosen.
- private_material 13 and 14 are 20-byte virtual-call thunks outside the unwind table. They moved +0x70 (to 0xc28340 and 0xc283a0) with identical raw bytes; each still loads the import-area global, which moved +8 (0x1aa9940 -> 0x1aa9948).
- material_retire 0 is the 3-byte `ret 0` no-op, still at 0x225a20 with identical bytes, and is the target of both unserialize slots.

## Moved data

Globals were checked by matching RIP-relative reference sets between the two builds, then verifying each referencing instruction (opcode bytes with the RIP displacement masked, new displacement landing on the candidate):

- required_zero_global 0x26f34a8 -> 0x26f43d8 (+0xf30; 3/3 references verified); material_cache_manager 0x26f3580 -> 0x26f44b0 (+0xf30; 5/5); econ_system 0x25e5ea8 -> 0x25e6dd8 (+0xf30; 127/127).
- resource_global 0x278ec50 -> 0x278fb70, manager_global 0x278ec58 -> 0x278fb78, material_interface_global 0x278ec68 -> 0x278fb88 (all +0xf20; reference sets match in count and function structure, 208/225, 2719/3013 and 82/98 references verified instruction-for-instruction).
- Resource import slot 0x1aa99b0 -> 0x1aa99b8 (+8; still tier0.dll `?Purge@CBufferString@@QEAAXH@Z`).
- Glove component tables: world 0x1c82eb8 -> 0x1c82ef8 (+0x40); addon 0x1d03cc8 -> 0x1d03ca8 (-0x20; one old reference now resolves directly to 0x1d03ca8). Slot +8 of both tables is still the submission helper, now 0x9563b0.
- Unserialize slots: low 0x1c82700 -> 0x1c82740 (+0x40); high 0x1d03510 -> 0x1d034f0 (-0x20, the same -0x7b8 offset from the addon table as in the previous build). Both still point at the no-op at 0x225a20.
- Callgate: interface global 0x2551080 -> 0x2551600 (the constructor caller at 0xd14e4 now loads it with `lea rcx, [rip+...]` resolving to 0x2551600; the constructor moved 0xb51ed0 -> 0xb51f40); table 0x1c5c8f0 -> 0x1c5c930 (the one reference to it is the constructor); FrameStageNotify 0xb63350 -> 0xb633c0, table index 36, unique hit.
- The charm signature still resolves uniquely to 0x7fbc40.

Byte checks that moved: submit_capture_cmp -> 0x14ca408 (bytes 803dc99f220100, new RIP displacement), float3_jump_entry -> 0x14d9a90 (entry now 0x14d998f). addon_table_lea -> 0x10d0551 (bytes unchanged).

## Host validation

The candidate passed all 24 release checks on the first run: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,487 entries); both glove component tables and submission slots; manager slot; UpdateSkin wrappers (0x81bb10 / 0x81bb60, unchanged, no harness edit needed); HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The dump files at this commit (offsets.json, client_dll.json, info.json build 14200) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
