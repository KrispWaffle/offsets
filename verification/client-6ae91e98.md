# Client build 6ae91e98 verification

Verified against the local game on November 2, 2026 (steam.inf VersionDate Nov 02 2026 16:17:08), using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

None. The manifest keeps schema_version 3 / midnight-build-data-v3 and the pinned extraction/private-layout rules; only build identities, RVAs, hashes and three byte checks changed. The single-file consumer does not need rebuilding.

## Binary identities

- client.dll: PE timestamp 0x6ae91e98, SizeOfImage 0x29a2000, disk SHA256 57ee728e15c50881070d847d7516d632d70456e0648d486a53f2528ec53f6ee0.
- materialsystem2.dll: PE timestamp 0x6ae91d7a (was 0x6adbd6f4), SizeOfImage 0x186000, disk SHA256 7aa863a1a17ca42092f0282d7e0607d3cff4a85a659ccaeda01958f8d4b971d1. All 14 material-system span hashes still match byte for byte, so only the module identity changed.

## What moved

The 6ae4fb84 client was no longer on disk, so the 6ae217a2 client (with its matching manifest) was used as the reference image. Matched client spans moved by 0 (1), +0x10 (20), +0x30 (4), +0xf0 (45) or +0x150 (12) relative to 6ae217a2. All 20 signature patterns are unchanged and resolve with the same hit counts as before (UPDATE_SUBCLASS and COLOR_VTABLE stayed at the same RVA; the rest shifted by +0x10 to +0xf0).

## Span verification

All 82 client spans were matched to the new image by hashing the instruction stream with rel32 branch and RIP-relative operands zeroed, then taking the new raw SHA256 at the matched RVA. No span was removed or shortened.

- 67 matched uniquely by operand-masked hash.
- 7 pulse_client spans (3, 4, 7, 11, 12, 14, 16) match after masking except for jump-table entries and table displacements; every residual byte is a 32-bit RVA that moved by exactly the function's delta (+0xf0 or +0x150).
- 7 spans (glove_tint 5, material_retire 17 and 28, private_material 13 and 14, pulse_client 8, and the private_material 2 twin below) matched at exactly one neighbouring delta with no residual bytes.
- material_retire 0 (the 3-byte `ret 0` no-op at 0x226550) is byte-identical at the same RVA.
- private_material 2 has an identical twin function; the entry at the neighbour delta (0x14ce140) was chosen.

Per-span results are in client-6ae91e98-spans.json.

## Moved data

Globals were located by matching RIP-relative reference sets between the two builds (same reference count, same referencing functions); tables were located by content; the interface global by the static initializer that constructs it.

- required_zero_global -> 0x2705948; material_cache_manager -> 0x2705a20; econ_system -> 0x25f8460.
- resource_global -> 0x27a10e0; manager_global -> 0x27a10e8; material_interface_global -> 0x27a10f8. The resource import slot 0x1ab1998 is unchanged.
- Glove component tables -> 0x1c8d748 / 0x1d0e360 (slot +8 still the submission helper); unserialize slots -> 0x1c8cf90 / 0x1d0dba8 (both still the 0x226550 no-op).
- Callgate: interface 0x2563620, table 0x1c67210, FrameStageNotify 0xb6ee70 (table index 36); the charm signature uniquely resolves to 0x801920.
- Byte checks: addon_table_lea -> 0x10d7a01 (bytes unchanged), submit_capture_cmp -> 0x14d2598 (new RIP displacement), float3_jump_entry -> 0x14e1c20 (entry now 0x14e1b1f).

## Host validation

The candidate passed the 24 release checks (same suite as the previous build): production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes (96 spans); glove tint, retirement and private-material guards; pulse interface, float3 converter and five expression descriptors; econ paint map (1,481 entries); both glove component tables; manager slot; UpdateSkin wrappers; HUD addon/removal linkage; charm address; callgate layout (211 slots, index 36).

The release harness hardcodes the two UpdateSkin wrapper addresses used by its wrapper check; they moved from 0x821370 / 0x8213c0 to 0x821380 / 0x8213d0 (identical bodies) and the harness was updated. The first run with the stale addresses failed that check and the dependent HUD check; the rerun passed 24/24.

The dump files at this commit (offsets.json, client_dll.json, info.json build 14195) were already refreshed upstream and were the ones used for the 51-field schema check. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. The MCP process reference and session were released.
