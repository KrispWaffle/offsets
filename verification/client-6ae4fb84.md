# Client build 6ae4fb84 verification

Verified against the local game on October 30, 2026 (steam.inf PatchVersion 1.41.9.4, VersionDate Oct 30 2026 13:55:25), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only the build identity, RVAs, hashes and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6ae4fb84, SizeOfImage 0x29a2000, disk SHA256 d618bbf8e6afb36baa09b3a282035996a88300458c55d2aa5cb81f0495c0ec54.
- materialsystem2.dll: unchanged, PE timestamp 0x6adbd6f4, SizeOfImage 0x186000, disk SHA256 f96093bbbc3f1b6df0fba856e37af60e774d9049e4b83dfd8aa69cc6ae9ead8e. All 14 material-system span hashes still match.

## What moved

The previous build (6ae217a2) was kept on disk and compared against the new one. Matched client spans moved by 0 (21), +0x20 (4) or +0xe0 (57). All 20 signature patterns are unchanged and still match with the same hit counts as before; SET_ATTRIBUTE, INVALIDATE_ATTRIBUTE, COPY_MATERIALS, HUD_ADDONS and ARMS_REFRESH moved by +0x20/+0xe0, the rest stayed at the same RVA.

## Span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed or shortened.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly +0xe0.
- 4 spans are byte-identical at the same RVA (glove_tint 5, material_retire 0, private_material 13 and 14).
- 3 short cold-tail fragments (material_retire 17 and 28, pulse_client 8) matched at the uniform +0xe0 shift of their neighbours.
- private_material 2 has an identical twin function at 0x14ce4f0; 0x14ce130 was chosen as the entry that sits at the uniform +0xe0 shift of the old 0x14ce050.

Per-span results are in client-6ae4fb84-spans.json.

## Moved data

Located by comparing RIP-relative reference sets between the two builds (same reference count, same referencing functions):

- required_zero_global 0x27058a8 -> 0x27058c8.
- material_cache_manager 0x2705980 -> 0x27059a0.
- resource_global 0x27a1030 -> 0x27a1050; manager_global 0x27a1038 -> 0x27a1058; material_interface_global 0x27a1048 -> 0x27a1068.
- Byte checks: addon_table_lea 0x10d79d1 -> 0x10d79f1 (bytes unchanged), submit_capture_cmp 0x14d24a8 -> 0x14d2588 (new RIP displacement), float3_jump_entry 0x14e1b30 -> 0x14e1c10 (entry now 0x14e1b0f).

Unchanged in place and re-checked against the new image: glove component tables 0x1c8d738 / 0x1d0e350 (slot +8 still 0x95bb60), unserialize slots 0x1c8cf80 / 0x1d0db98 (both still the 0x226550 no-op), econ singleton 0x25f83e0, import slot 0x1ab1998, Source2Client002 interface 0x25635a0, table 0x1c67200 and FrameStageNotify 0xb6ee60 (table index 36), charm signature 0x801910.

## Host validation

The candidate passed the 24 release checks (same suite as 6ae217a2): production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,481 entries); both glove component tables; manager slot; UpdateSkin wrappers; HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The dump files (offsets.json, client_dll.json, info.json and the generated headers) were not regenerated and still describe the previous dump. The 51 schema fields the consumer reads matched the live game. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
