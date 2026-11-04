# 8. TaystJK: what differs from the 1.01 tree, and where the code is

This study measured traffic from the original 1.01 dedicated server (through the YBEProxy capture, see 01). The analysis in [02](02-engine-code-findings.md) and [07](07-transport-study.md) cites the 1.01 source tree (`E:\Git\jka\jediacademy\codemp`). This repository is TaystJK, an OpenJK-derived client and server that is used for the proof of concept of the rewrite. This document records what was re-checked in the TaystJK source on 4 November 2026, what is identical, what differs, and where each function is in this tree.

**The 1.01 server stays the reference for the measurements.** Where TaystJK differs, the difference is either a default setting (a cvar that can be set back) or one rule of the pilot player state, described below.

## 8.1 Wire compatibility with 1.01: checked

| item | how it was checked | result |
|---|---|---|
| Field tables (`entityStateFields`, `playerStateFields`, `pilotPlayerStateFields`, `vehPlayerStateFields`) | extracted from both `msg.cpp` files with the decoder's extractor and compared entry by entry | identical: 132, 137, 140, 69 entries, same names, same widths. TaystJK writes two widths as macros (`GENTITYNUM_BITS`, `MAX_POWERUPS` = 16); the extractor in `jkadec.py` now accepts both. |
| Huffman frequencies (`msg_hData`) | extracted from both files and compared | identical (256 values). The active table is `codemp/qcommon/msg.cpp:2814`; the one at `:2552` is commented out. |
| Bit layer, delta entity, delta player state, arrays | read; `diff` of the two `msg.cpp` files | same bitstream. TaystJK adds overflow checks (`MSG_WriteBits`, `MSG_ReadBits`, `Huff_offsetReceive` / `Huff_offsetTransmit` take a bound), `lc` range checks in the readers, and replaces `MSG_WriteShort` of the array masks by `MSG_WriteBits(..., MAX_STATS)` etc., which writes the same 16 bits. In OOB mode it fixes the 32-bit write (`msg->bit += 32` instead of `8`); OOB data does not use the delta path. |
| Whole decode of the study's captures with TaystJK's tables | `jkadec.py` run with `SRC` pointing at `codemp/qcommon/msg.cpp` | 7,101 / 7,101 and 1,605 / 1,605 messages decoded, as with the 1.01 tables |
| Gamestate tail | read both | the same: after the checksum feed, a short (0 when there is no random-mission terrain; TaystJK always writes 0, `server/sv_client.cpp:500`) |

The only wire difference found:

- **Pilot player state.** In 1.01 a pilot riding inside a vehicle is written with only `ARRAY_LEN(pilotPlayerStateFields) - 82` = **58** fields (the earlier documents wrongly said 122). TaystJK writes all 140 (`codemp/qcommon/msg.cpp:2141-2148` and the reader at `:2392`). This comes from OpenJK commit `3409151cb` (2015, "Stop clobbering number of fields in playerstate for pilots", reported as a cause of the "seeker bug", OpenJK issue 708). A TaystJK client reads a 1.01 server correctly (it accepts `lc` up to 140). A 1.01 client connected to a TaystJK server fails with `invalid playerState field count` as soon as a field after the 58th changes. The captures contain no pilot state, so no measurement depends on it. `codec2.py` and `fuzz.py` follow the 1.01 rule.

**Decision proposed for the rewrite:** do not restore the `- 82` rule. The new codec sends changed fields only, so the rule saves nothing, and OpenJK removed it because it caused bugs. The legacy path in the branch can stay as TaystJK has it.

## 8.2 Defaults that differ (settings, not format)

These change the conditions under which the 1.01 measurements apply. All are cvars except `PACKET_BACKUP`.

| setting | 1.01 | TaystJK | where |
|---|---|---|---|
| client `rate` | 4000 | 25000 | `client/cl_main.cpp:3419` |
| client `snaps` | 20 | 40 | `client/cl_main.cpp:3420` |
| `cl_maxpackets` | 30, clamped 15..100 | 100, clamped 15..1000 | `client/cl_main.cpp:3347`, `client/cl_input.cpp:1520-1525` |
| `cl_packetdup` | (not re-checked) | 1, clamped 0..5 | `client/cl_main.cpp:3348`, `cl_input.cpp:1600-1605` |
| server `sv_fps` | 20 | 40 upstream; **20 in this branch** | `server/sv_init.cpp:1014` |
| rate policy | client `rate` clamped 1000..90000 | `sv_ratePolicy` 1 (default): every non-LAN client gets `sv_clientRate` = 90000 upstream, **25000 in this branch**; policy 2 clamps the client value 1000..100000 then to `sv_minRate..sv_maxRate` | `server/sv_init.cpp:989-993`, `server/sv_client.cpp:1198-1222` |
| snaps policy | `snaps` clamped 1..30 | `sv_snapsPolicy` 2 (default): `snaps` clamped to `sv_snapsMin..min(sv_fps, sv_snapsMax)`; policy 1 forces `sv_fps` | `server/sv_init.cpp:1010-1012`, `server/sv_client.cpp:1224-1253`, `server/sv_main.cpp:1108-1140` |
| LAN rate | 99999 | 100000 when `sv_lanForceRate` 1 and not `com_dedicated 2` | `server/sv_client.cpp:1202-1204` |
| `PACKET_BACKUP` | 32 | **128 in non-dedicated builds** (client and listen server), 32 in the dedicated server | `qcommon/qcommon.h:117-122` |
| `SV_RateMsec` | `(min(size,1500)+48)*1000/rate` | the same, with `sv_minRate` / `sv_maxRate` and `timescale` applied | `server/sv_snapshot.cpp:755-785` |

Branch decision (4 November 2026): keep TaystJK's code, policies and the pilot fix; set the server's `sv_fps` back to 20 so that the snapshot rate matches the captures (a client asking `snaps` 40 is clamped to `sv_fps`, so it gets 20); set `sv_clientRate` to 25000 (the rate the server imposes under `sv_ratePolicy` 1); keep the client defaults (`rate` 25000, `snaps` 40, `cl_maxpackets` 100). At 25000 bytes per second and 50 ms, a message is rate-delayed from 1,202 bytes (`(size + 48) * 1000 / 25000 >= 50`); on the test part of the 5-minute capture this delays 0.0 % of snapshots for both the engine and the new codec (03, section 3.10, row 25,000).

Consequences for the results of [03](03-results.md) (upstream defaults; the branch removes the 40 Hz part):

1. **Rate table (3.10).** With TaystJK's defaults (rate forced to 90000, 40 snapshots per second) the formula gives `rateMsec` = 17 ms for a 1,500-byte message, below the 25 ms interval: **no snapshot is ever rate-delayed**, whatever the codec. The 3.10 table only describes a 1.01 server, or a TaystJK server set back to `sv_ratePolicy 2` with low client rates.
2. **Base age.** The loss and latency results (3.9) convert RTT to base age with 50 ms per snapshot. At 40 snapshots per second the same RTT gives twice the age (RTT 100 ms = age 4), but each snapshot then covers 25 ms of movement, so a delta of age 2 at 40 Hz spans the same time as age 1 at 20 Hz. The captures are all at 20 Hz; sizes at 40 Hz were **not measured**.
3. **Bytes per second.** At 40 Hz the server sends twice as many snapshots. Each one is smaller than at 20 Hz (less movement per delta), but by how much is unknown. Record at 40 Hz before drawing conclusions on bandwidth.
4. **To stay on the 1.01 conditions** on a TaystJK server, set `sv_fps 20`, `sv_ratePolicy 2`, `sv_maxRate 90000` (see 8.4 for why `sv_maxRate` must not stay 0), and on the client `rate 4000`, `snaps 20`. No code change is needed.

## 8.3 Other differences relevant to the netcode

| item | TaystJK | where |
|---|---|---|
| Challenge | `HMAC-MD5(secret, address + timestamp)`, stateless, valid about 16 s; no longer `rand()` | `server/sv_challenge.cpp:108-185`, called by `SV_GetChallenge` at `server/sv_client.cpp:62` |
| OOB rate limiting | per-address leaky buckets (`SVC_RateLimitAddress`) on the connectionless packets | `server/sv_main.cpp:349-431` |
| `connect` string | still `Huff_Decompress(msg, 12)` | `server/sv_main.cpp:776-778` |
| Downloads | UDP downloads as in 1.01, plus an HTTP download server built on mongoose (`NET_HTTP_*`, `lib/mongoose`, dual GPL v2 / commercial licence) | `qcommon/net_http.cpp`, `client/cl_parse.cpp:745-800` |
| Server-side demos | the server can build and record messages for clients and bots (`sv_autoDemo`, `sv_demoPreRecord`). For bots being recorded, `deltaMessage` is set to the newest sequence; demo constraints can force snapshots without a base | `server/sv_snapshot.cpp:138-220`, `:816-920` |
| Duel isolation | entities of other duels are left out of the snapshot (`DuelCull`), an existing form of interest culling | `server/sv_snapshot.cpp:483`, `:733`, `server/duel_cull.cpp` |
| Ping | `sv_pingFix` 1 stores `Sys_Milliseconds()` as send time | `server/sv_snapshot.cpp:813` |
| IPv6 | still none: `qcommon/net_ip.cpp` opens `AF_INET` sockets only (`:436`) | |
| Clients | `MAX_CLIENTS` 32 | `qcommon/q_shared.h:899` |
| cgame | race mode (JAPRO): movers can be non-solid for racers in prediction, and other players are forced to `TR_INTERPOLATE` in race mode. Prediction still restarts from the received player state with a 0.1-unit error threshold. Racing makes exact positions more important, not less | `cgame/cg_predict.c:300-352`, `:1031`, `:1386`; `cgame/cg_ents.c:3595` |
| Player trajectories | Same cvar value, different behaviour: with `g_smoothClients 1` (the value used in the captures and on servers) TaystJK's game module sends players as `TR_INTERPOLATE` (no `trTime`), and only 2 gives the extrapolated form; the 1.01 base module used `TR_LINEAR_STOP` with `trTime` = `commandTime`. This changes the prediction variants of 03 section 3.12. The game module also snaps `pos.trBase` when it extrapolates a client that missed frames | `game/g_active.c:6283-6323`, `game/g_xcvar.h:154` |
| Licence | 503 files under `codemp/` and `shared/` carry "GNU General Public License version 2 as"; 72 carry "version 2 or any later version". The GPL-v2-only conclusion of 02 section 2.9 holds for this tree | grep, not a legal reading |

## 8.4 A bug found on the way

With `sv_ratePolicy 2` and the default `sv_maxRate 0`, `SV_UserinfoChanged` clamps the client rate with `Com_Clampi(sv_minRate, sv_maxRate, i)` = `Com_Clampi(0, 0, i)`, so the rate becomes 0 (`server/sv_client.cpp:1218`). `SV_RateMsec` then divides by `rate * timescale` = 0 (`server/sv_snapshot.cpp:782`). Not tested; read only. Set `sv_maxRate` when using policy 2, or fix the clamp.

## 8.5 Where the functions of 02 and 07 are in this tree

Paths are relative to `codemp/`. Line numbers as of commit `430aca2ef`.

| function or item (02 / 07 section) | TaystJK location |
|---|---|
| `MSG_WriteBits` / `MSG_ReadBits` (2.1) | `qcommon/msg.cpp:150` / `:244` |
| `Huff_offsetReceive` / `Huff_offsetTransmit` / `Huff_Decompress` / `Huff_Compress` / `Huff_addRef` | `qcommon/huffman.cpp:272` / `:327` / `:333` / `:387` / `:186` |
| `MSG_initHuffman` (two versions, `_USINGNEWHUFFTABLE_`) | `qcommon/msg.cpp:3075`, `:3094` |
| `_NEWHUFFTABLE_` / `_USINGNEWHUFFTABLE_` (commented out) | `qcommon/msg.cpp:29-30` |
| Huffman structures | `qcommon/qcommon.h:1092-1122` |
| `entityStateFields` (2.2), user fields, `FLOAT_INT_BITS` / bias | `qcommon/msg.cpp:845`, `:1023-1034`, `:1039-1040` |
| `assert(numFields + 1 == sizeof(*from)/4)` | `qcommon/msg.cpp:1068` |
| `MSG_WriteDeltaEntity` / `MSG_ReadDeltaEntity` | `qcommon/msg.cpp:1053` / `:1174` |
| `playerStateFields` / `pilotPlayerStateFields` / `vehPlayerStateFields` / unoptimised table (2.3) | `qcommon/msg.cpp:1316` / `:1474` / `:1638` / `:1729` |
| Commented `hyperSpace*` entries | `qcommon/msg.cpp:1427-1457`, `:1615-1621` |
| `MSG_WriteDeltaPlayerstate` (pilot bit, arrays) | `qcommon/msg.cpp:2109` (pilot bit `:2141-2148`, arrays `:2262-2311`) |
| `MSG_ReadDeltaPlayerstate` (arrays) | `qcommon/msg.cpp:2347` (pilot bit `:2392`, arrays `:2465-2520`) |
| `_ONEBIT_COMBO` (commented) / `_OPTIMIZED_VEHICLE_NETWORKING` | `qcommon/q_shared.h:1065` / `:1069` |
| `MAX_WEAPONS`, `MAX_STATS`, `MAX_POWERUPS`, `GENTITYNUM_BITS` | `qcommon/q_shared.h:1050`, `:1047`, `:1049`, `:908` |
| `EF_NODRAW`, `STAT_WEAPONS` | `game/bg_public.h:798`, `:727` |
| `MSG_CheckNETFPSFOverrides`, called from `CL_ParseSetGame` | `qcommon/msg.cpp:1903`; `client/cl_parse.cpp:410-411` |
| `MSG_ReadString` / `MSG_ReadBigString` (`%` to `.`) | `qcommon/msg.cpp:504` / `:543` |
| `MSG_WriteDeltaUsercmdKey` / `MSG_ReadDeltaUsercmdKey` | `qcommon/msg.cpp:732` / `:777` (the reader maps `-128` moves to `-127`) |
| `SV_EmitPacketEntities` / `SV_WriteSnapshotToClient` (2.4) | `server/sv_snapshot.cpp:61` / `:128` (base selection `:147-193`, vehicle state `:270-303`, `sv_padPackets` `:309-314`) |
| `SV_UpdateServerCommandsToClient` | `server/sv_snapshot.cpp:325` |
| `SV_SendClientSnapshot` (setgame `:973-1013`, bots `:1035-1041`, overflow `:1061-1064`) | `server/sv_snapshot.cpp:969` |
| `SV_SendMessageToClient` (rate delay `:936-944`, LAN `:930-933`, not active `:948-956`) | `server/sv_snapshot.cpp:797` |
| `SV_SendClientMessages` (pending fragments `:1089-1096`) | `server/sv_snapshot.cpp:1075` |
| `SV_Netchan_Encode` (key `:64`), `svc_EOF` appended | `server/sv_net_chan.cpp:37`, `:155` |
| `CL_ParseServerMessage` / `CL_ParseSnapshot` (validity `:258-285`) / `CL_ParsePacketEntities` (2.5) | `client/cl_parse.cpp:875` / `:217` / `:105` |
| `CL_ParseGamestate` / `CL_ParseDownload` | `client/cl_parse.cpp:515` / `:648` |
| `svc_*` / `clc_*` | `qcommon/qcommon.h:305-316` / `:323-329` |
| `SV_CreateClientGameStateMessage` / `SV_SendClientGameState` (2.6) | `server/sv_client.cpp:454` / `:514` (`gamestateMessageNum` `:541`) |
| Gamestate resend in `SV_ExecuteClientMessage` | `server/sv_client.cpp:1855-1868` (function at `:1815`) |
| `SV_WriteDownloadToClient` (error `-1` `:794`) | `server/sv_client.cpp:749` |
| Netchan header, `FRAGMENT_SIZE` (2.7) | `qcommon/net_chan.cpp:27-51` |
| `Netchan_TransmitNextFragment` (exact multiple `:140-147`) / `Netchan_Transmit` (stomp `:168-171`, split `:173`) | `qcommon/net_chan.cpp:102` / `:159` |
| `Netchan_Process` (reject old `:272`, fragment check `:308`, final fragment `:370-377`) | `qcommon/net_chan.cpp:222` |
| `PACKET_BACKUP`, `MAX_RELIABLE_COMMANDS`, `MAX_MSGLEN` | `qcommon/qcommon.h:117-122`, `:133`, `:179` |
| `CG_PredictPlayerState` (0.1-unit error `:1386`) (2.8) | `cgame/cg_predict.c:1031` |
| `CG_BuildSolidList` / `CG_ClipMoveToEntities` (mover position `:326`, other entities `:351`) | `cgame/cg_predict.c:67` / `:232` |
| `CG_AdjustPositionForMover` / `CG_CalcEntityLerpPositions` (active copy; the one at `:3523` is commented out) | `cgame/cg_ents.c:3445` / `:3595` |
| `CG_ScanForCrosshairEntity` | `cgame/cg_draw.c:9207` |
| `lerpOrigin` count | 399 uses in 20 files (138 `cg_players.c`, 106 `cg_ents.c`, 62 `cg_draw.c`, 16 `cg_weapons.c`, 16 `cg_event.c`) |
| `BG_PlayerStateToEntityState` / `...ExtraPolate` (player `pos` and `apos`) | `game/bg_misc.c:3024` / `:3174` |
| `CL_ReadyToSendPacket` / `CL_WritePacket` (usercmd key `:1631`) (7.1) | `client/cl_input.cpp:1485` / `:1557` |
| `cl_timeout`, `sv_timeout`, `sv_zombietime` | `client/cl_main.cpp:3320`; `server/sv_init.cpp:1016-1017` |
| `SV_DirectConnect` | `server/sv_client.cpp:131` |
