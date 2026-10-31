# Client build 6ae217a2 verification

Verified against the local game on October 28, 2026, using the Perception MCP endpoint http://127.0.0.1:50755/mcp. All runtime checks were read-only. No callbacks, hooks, target allocations or target writes were performed.

## Consumer migration

This manifest uses schema_version 3 / midnight-build-data-v3. It requires the updated single-file consumer once. Version 2 consumers reject it. Extraction and private-layout rules are now in this pinned manifest, so subsequent supported data-only updates do not require rebuilding the script. Calling-convention, semantic ABI or unsupported decoder changes can still require code.

## Binary identities

- client.dll: PE timestamp 0x6ae217a2, SizeOfImage 0x29a2000, disk SHA256 08a741030c1e569d8b854c5886a87fcafc21d0d0e2f05dfd380e8a351171501d.
- materialsystem2.dll: PE timestamp 0x6adbd6f4, SizeOfImage 0x186000, disk SHA256 f96093bbbc3f1b6df0fba856e37af60e774d9049e4b83dfd8aa69cc6ae9ead8e.

## Span verification

82 client spans and 14 material-system spans were checked against live memory. All 96 matched the candidate manifest. All capability flags remain enabled. No span was removed or shortened, and all 20 signature patterns remain unchanged.

72 client spans independently reproduce the previous reviewed SHA256 after reversing decoded relocation operands. The normalization also handles absolute RVA operands and entries in jump/byte lookup tables. This is stronger than guessing a new address and replacing its hash.

The other 10 entries represent eight distinct functions and received a current-build identity/layout review. Previous-build byte equivalence was not established for these entries. Their identification and script-relevant contracts are recorded below; their actual current bytes are pinned by the new hashes.

| Entries | Current RVA | Identity and relevant evidence |
|---|---|---|
| glove_tint 9 / material_retire 8 | 0x10d7890 | Unique direct caller at 0x10d72a7; references `econ_gloves`, `low-res gloves` and `high-res gloves`. Owner handle +0x520, low-glove handle +0x15f8, addon table +0x1328, key 0x3b3411b3, native pending count +0xac8; call at 0x10d7b83 reaches the independently identified HUD addon spawner. |
| glove_tint 14 | 0x14da290 | Called by the still-matching regeneration path at 0x82b334 and normalized helpers at 0x14ec1be / 0x14eb85c. Returns global 0x2705980; constructs an 0xe0-byte object with CCompositeMaterialManager vtable 0x1d77900. Live singleton's slot 4 points to the reviewed retirement method. |
| material_retire 1 | 0x92b480 | Base-model destructor resets primary and component vtables, including +0x610, tears down its containers, calls material-component cleanup at 0x14eb830, and tail-calls base destruction at 0xae3790. The material cleanup is a normalized span. Complete function ends at 0x92b8f9. |
| material_retire 6 | 0xc66c70 | Direct caller 0xc69e7b; references `CEntitySpawner<class C_WorldModelGloves>::Spawn`, `gloves/paints/`, wear, random-seed and econ-instance keys. Builds low glove material records and submits via the +0x610 material component's slot 1. |
| material_retire 7 | 0x10d5d50 | Reached by the production extraction chain from HUD counter 0x10dadf0 (call 0x10dafce), and by HUD glove refresh. References `CEntitySpawner<class C_CS2HudModelAddon>::Spawn`; creates the addon with parent, attachment, model and render-with-viewmodels properties. |
| material_retire 10 / private_material 8 | 0x14de330 | CCompositeMaterialManager RTTI-confirmed vtable slot 4 at 0x1d77920, corroborated live through the manager singleton. Record stride 0x138 and request stride 0x1b0; request state bytes +0x158/+0x159 and completion handling remain consistent with the consumer. Calls normalized retirement helpers, including 0x14e0ad0 and 0x14e1060. |
| material_retire 31 | 0x15cb2e0 | Three incoming calls from independently normalized spans at 0x15b4e98, 0x15bc9d4 and 0x15c799b. References `CEntityClass::Unserialize( %d:%s:%s )` and the KV3 transfer interface; retains RCX/RDX/R8 inputs and uses the verified KV conversion/cleanup graph. |
| pulse_client 7 | 0x14def80 | Called from 0x14d8dec and recursively; references Shader, texgen and draw-text properties. Dispatch table at 0x14e0660 remains inside the hashed span. Float3 dispatch reaches normalized converter 0x14e1900 via call 0x14df717. Reviewed tagged-value handling and KV helper linkage. |

## Moved data

- Glove component tables: RTTI identifies C_WorldModelGloves +0x610 at 0x1c8d738 and C_CS2HudModelAddon +0x610 at 0x1d0e350. Both live submission slots point to normalized helper 0x95bb60.
- Unserialize slots 0x1c8cf80 and 0x1d0db98 both point to the live 0x226550 no-op.
- Composite-manager singleton 0x2705980 is decoded from its getter; live vtable slot 4 agrees with 0x14de330.
- Econ singleton 0x25f83e0 is read/written in the verified attribute helper's construction path at 0x11a4a40. Live paint-map validation found 1,481 entries.
- Resource/manager/material globals 0x27a1030 / 0x27a1038 / 0x27a1048 and import slot 0x1ab1998 are supported by normalized instruction operands and live interface checks.
- Charm AddKeyChainEntity signature uniquely resolves to 0x801910.
- Source2Client002 interface 0x25635a0, vtable 0x1c67200, FrameStageNotify 0xb6ee60, 211 primary slots and index 36 pass the production shadow-layout check.

## Host validation

The exact candidate manifest passed 24 release checks: production startup; all 51 live schema fields; native targets and live private-layout extraction; color targets; all five groups of code hashes; glove tint, retirement and private-material guards; pulse interface, float3 dispatch and five expression descriptors; econ paint map; both glove component tables; manager slot; UpdateSkin wrappers; HUD addon/removal linkage; charm address; callgate layout.

The first expanded harness run omitted UpdateSkin resolution before the HUD resolver and failed that prerequisite. Resolving and checking the two current wrappers (0x821370 and 0x8213c0, both linked to rebuild 0x803fd0) corrected the harness. The final run passed 24/24 without changing the production guards.

The main and actual root bundle compile in the host. The root and dist bundles match. GUI/rendering, persistent scheduling and in-game cosmetic application were not executed by this read-only verification. Every owned MCP process reference was released and its session closed.
