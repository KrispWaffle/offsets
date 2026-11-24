# Client build 6b04b843 verification

Verified against the local game on November 23, 2026 (steam.inf VersionDate Nov 23 2026, PatchVersion 1.42.0.1), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None in code. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules. The callgate `primary_slots` value changed from 211 to 212 (see below); the consumer reads that value from the manifest (`build_data` parsing) and sizes its callgate shadow from it, so no consumer change is needed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6b04b843, SizeOfImage 0x2996000 (was 0x298e000, +0x8000), disk SHA256 afe9cf1ffd9770831bedb0e6fc51e3f217b77358b8b537539ba1a1cd38502493.
- materialsystem2.dll: unchanged (PE timestamp 0x6afcfdd8, SizeOfImage 0x186000, disk SHA256 2cd59750aec326ecfc49e93bd8064071e00b718b45600689c848f30090f47ae6). All 14 pulse_material span hashes and table/descriptor RVAs were re-checked against the installed image and still match.

## What moved

The 6aff8722 client (kept on disk, with the manifest that matched it) was used as the reference image. This patch moved code by several different amounts depending on region: 57 of the 82 client spans by +0x5600, 8 by +0x3d10, 4 by +0x5510, 4 by +0x4460, 3 by +0x3f30, 3 by +0x4c90 (the two thunks and material_retire 6), and a few small offsets (+0x42f0, +0x4300, +0x510). Every one of the 20 signature patterns moved and every pattern keeps its hit count.

## Client span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed, shortened or changed in content.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta (+0x5600).
- 4 spans (glove_tint 5, material_retire 17 and 28, pulse_client 8) matched at exactly one neighbouring delta with no residual bytes.
- private_material 2 has an identical twin function; the entry at the neighbour delta (0x14c5fb0 + 0x5600 = 0x14cb5b0) was chosen.
- material_retire 0 is the 3-byte `ret 0` no-op. It moved to 0x225f30 (+0x510) with identical bytes, and is the target of both unserialize slots.
- private_material 13 and 14 are 20-byte virtual-call thunks outside the unwind table. Their raw bytes changed only in the RIP displacement, so they were matched by a masked byte pattern: the pattern has the same hit count and order as before (8 and 10). They moved to 0xc2cfd0 and 0xc2d030 (+0x4c90) and still load the import-area global, which moved +0x6000 (0x1aa9948 -> 0x1aaf948).

## Moved data

Globals were located by matching RIP-relative reference sets between the two builds and then verifying each referencing instruction (opcode bytes with the RIP displacement masked, new displacement landing on the candidate). All six pulse/private/material globals verified at every reference, all at +0x7000:

- required_zero_global 0x26f43d8 -> 0x26fb3d8 (3/3); material_cache_manager 0x26f44b0 -> 0x26fb4b0 (5/5); econ_system 0x25eddd8 (127/127, old 0x25e6dd8 -> 0x25eddd8).
- resource_global 0x278fb70 -> 0x2796b70 (225/225); manager_global 0x278fb78 -> 0x2796b78 (3013/3013); material_interface_global 0x278fb88 -> 0x2796b88 (98/98).
- Resource import slot 0x1aa99b8 -> 0x1aaf9b8 (+0x6000; still tier0.dll `?Purge@CBufferString@@QEAAXH@Z`).
- Glove component tables: world 0x1c82ef8 -> 0x1c89028 (+0x6130; 2/2 references verified); addon 0x1d03ca8 -> 0x1d09df8 (+0x6150; 2/2). Slot +8 of both tables is the submission helper, now 0x95a6a0.
- Unserialize slots: low 0x1c82740 -> 0x1c88870 (+0x6130); high 0x1d034f0 -> 0x1d09640 (+0x6150; the same -0x7b8 offset from the addon table as before). Both point at the no-op at 0x225f30.
- Callgate: interface global 0x2551600 -> 0x2558600 (+0x7000; its constructor caller at 0xd14e4 still loads it with `lea rcx, [rip+...]`); table 0x1c5c930 -> 0x1c62980 (+0x6050; the one reference to it is the constructor, 1/1); FrameStageNotify 0xb633c0 -> 0xb67ea0, table index 36 (unique hit).
- The charm signature moved to 0x7ffb70 (unique hit).

Byte checks that moved: submit_capture_cmp -> 0x14cfa08 (bytes 803dc9b9220100, new RIP displacement), float3_jump_entry -> 0x14df090 (entry now 0x14def8f). addon_table_lea -> 0x10d5a61 (bytes unchanged).

## Callgate layout change: primary slots 211 -> 212

The primary vtable of the callgate interface grew by one entry. In the 6aff8722 build the first non-code entry (the RTTI locator that marks the secondary vtable) sits at index 211; in this build code entries run through index 211 and the locator is at index 212. The FSN pointer is still at index 36, and the secondary vtable and hierarchy (11 entries, 1 attribute) are unchanged. The manifest `callgate.primary_slots` was therefore updated from 211 to 212. The exact insertion point of the new slot was not pinned down (masked prefixes differ for many functions, so an alignment against the previous table was inconclusive); the live shadow-layout check reads the whole primary table and confirms 212 slots with FSN at index 36.

## Host validation

The candidate passed all 24 release checks: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,487 entries); both glove component tables and submission slots; manager slot; UpdateSkin wrappers; HUD addon/removal linkage; charm address; callgate shadow layout (212 slots, index 36).

The first run of the callgate check failed with the 211 value, because the live table has 212 primary slots. That failure is why the layout change above was found. The check passed on the second run after `primary_slots` was set to 212.

The UpdateSkin wrappers moved from 0x81bb10 / 0x81bb60 to 0x81fa40 / 0x81fa90 (+0x3f30, same body shape), so the local harness constants in `tests/patch-release-live.em` were updated before the run. That file is local and not part of this commit.

The dump files at this commit (offsets.json, client_dll.json, info.json build 14201) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
