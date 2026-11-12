# Client build 6af4a59c verification

Verified against the local game on November 12, 2026 (steam.inf VersionDate Nov 11 2026 10:20:41), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only build identities, RVAs, hashes and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6af4a59c, SizeOfImage 0x298a000 (was 0x29a3000), disk SHA256 99a6720b29d8b477d55f0a2d7fd4333b5df13f354ba723804e996bead1231aa8.
- materialsystem2.dll: PE timestamp 0x6af4a4ac (was 0x6ae91d7a), SizeOfImage 0x186000, disk SHA256 c256e7a296c68ba742dee6126bc59d967ac4b2060ce20ef63a7a09f9af9ca78e. All 14 span hashes changed (code moved); see below.

## What moved

The 6aebc4da client and the 6ae91d7a materialsystem2 (both kept on disk, with the manifest that matched them) were used as the reference images. This update moves code backwards: matched client spans shifted by between -0xbcf0 and -0x220, mostly -0xba10 (45) and -0xb440 (11). All 20 signature patterns are unchanged and resolve with the same hit counts as before, all shifted.

## Client span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed or shortened.

- 66 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta.
- 4 spans (glove_tint 5, material_retire 17 and 28, pulse_client 8) matched at exactly one neighbouring delta with no residual bytes.
- private_material 2 has an identical twin function; the entry at the neighbour delta (0x14c3050) was chosen.
- material_retire 6 (world-gloves spawn function, 3373 bytes) is the only span with a content change. It was located by its `gloves/paints/` and `CEntitySpawner<class C_WorldModelGloves>::Spawn` string references, which sit at the same offsets from its start (0xc5b100) as before. Its instruction stream is identical to the previous build; the only differences are nine member displacements on the gloves entity, +0x176d -> +0x179d and +0x1770 -> +0x17a0 (all +0x30). The length is unchanged. The consumer does not reference those displacements.
- private_material 13 and 14 are 20-byte virtual-call thunks that are not in the unwind table. They were matched by byte pattern: the pattern has the same number of hits in both builds (8 and 10) in the same order, and both thunks load the same import-area global.
- material_retire 0 is the 3-byte `ret 0` no-op; it moved to 0x226370, identified as the target of the unserialize slots, with identical bytes.

## Material-system span verification

The 14 pulse_material spans were matched the same way against the saved 6ae91d7a image (whose bytes match the previous manifest's hashes). Code before roughly 0x3a000 moved by 0 to +0x80; later code moved by +0x120. Two spans (7 and 8) have eight identical-shaped twins; they were assigned at the same +0x120 delta as their neighbours (0x99f30 and 0x99fd0), keeping their order. The 6424-byte span 6 matches after masking except for 72 bytes of jump-table RVA words, all +0x120. The interface/output function-pointer tables and the expression descriptors/names are at the same RVAs as before (entries moved with their functions); only the module identity and span hashes changed.

Per-span results for client and material-system are in client-6af4a59c-spans.json.

## Moved data

Globals were located by matching RIP-relative reference sets between the two builds (same reference count, same referencing functions); tables by content; the import slot by import name; the callgate table by the FrameStageNotify pointer at index 36; the interface global from the static initializer that constructs it.

- required_zero_global -> 0x26efe38; material_cache_manager -> 0x26eff10; econ_system -> 0x25e2898.
- resource_global -> 0x278b5d0; manager_global -> 0x278b5d8; material_interface_global -> 0x278b5e8.
- Resource import slot -> 0x1aa79b0 (still tier0.dll `?Purge@CBufferString@@QEAAXH@Z`).
- Glove component tables -> 0x1c81238 / 0x1d01e20 (slot +8 still the submission helper); unserialize slots -> 0x1c80a80 / 0x1d01668 (both still the no-op, now 0x226370). The high slot and addon table were chosen from same-shaped candidates by sharing the same shift (-0xd580).
- Callgate: interface 0x254d9f0, table 0x1c5aa90, FrameStageNotify 0xb628b0 (table index 36, unique hit); the charm signature uniquely resolves to 0x7fb160.
- Byte checks: addon_table_lea -> 0x10cee41 (bytes unchanged), submit_capture_cmp -> 0x14c74a8 (new RIP displacement), float3_jump_entry -> 0x14d6b30 (entry now 0x14d6a2f).

## Host validation

The candidate passed all 24 release checks on the first run: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,481 entries); both glove component tables; manager slot; UpdateSkin wrappers; HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The release harness hardcodes the two UpdateSkin wrapper addresses; they moved from 0x8214f0 / 0x821540 to 0x81b030 / 0x81b080. Their bodies are the same except the entity member displacement +0x1bd0 -> +0x1c10; the consumer's wrapper shape check covers only the first 25 bytes and does not include that displacement. The harness was updated before the run.

The dump files at this commit (offsets.json, client_dll.json, info.json build 14197) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
