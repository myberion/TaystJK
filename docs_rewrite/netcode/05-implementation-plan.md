# 5. Implementation plan: server, client and tooling changes

This plan turns the recommendations of [04](04-recommendations.md) into work packages. File names are those of the original 1.01 tree; the same functions in TaystJK, with their line numbers, are listed in [09-taystjk.md](09-taystjk.md) section 9.5. CleanJK has the same structure, but its paths and line numbers were not re-checked, so confirm each file before editing. Effort labels are rough: S = days, M = weeks, L = months, for one engineer who knows the code.

## 5.1 Principles

1. **The game and cgame modules do not change.** The codec reads and writes `entityState_t`, `playerState_t`, the baselines and the server time. `trap_GetSnapshot` returns the same structures as today, so third-party modules built on the existing structures keep working (if the owner wants to keep them at all).
2. **Equivalence criterion.** For any message, `decode(encode(x))` must equal what the current client would see after decoding the current protocol's version of `x` (truncation to field widths, `-0.0` as `+0.0`). This is the test the study used.
3. **One independent stream per message.** No coder state survives from one packet to the next.
4. **The codec is isolated** from the netchan and from the socket layer, so the transport can change later without touching it.
5. **Keep the legacy path** behind a protocol flag during development, so that results can be compared on the same traffic.
6. **Data-driven schema.** Field lists, widths and tables are data generated from one source of truth, not retyped.

## 5.2 New code

A new directory `qcommon/netcodec/` (name to be decided), with no dependency on the server or client:

| unit | content | notes |
|---|---|---|
| `rans` | 32-bit byte-renormalising rANS encoder and decoder, 12 to 15-bit probabilities | start from ryg_rans (CC0); `tools/python/codec.py` is a working reference for the unit format (symbol with context, run of raw bits) |
| `schema` | descriptors of entity and player-state fields (name, offset, width, float flag), the pilot and vehicle subsets, array fields | generated from the existing `entityStateFields` / `playerStateFields` arrays; a field list version number |
| `tables` | loader for the trained probability tables and per-field modes; uniform fallback | binary format with a version and a hash of the schema |
| `fieldcodec` | value coding: absolute / delta classes, float kinds (zero, integral, angle index, full), angle index conversion | the reference is `put_int`, `put_float`, `get_int`, `get_float` in `codec.py` |
| `msgcodec` | whole-message encode / decode: header, operations, snapshot, gamestate, commands, setgame, download, mapchange | reference: `codec2.py` |
| `escape` | raw-format coding of an entity or a message, and the selection rule | not in the study; see 5.4, phase 4 |
| `tests` | golden-trace round trips, synthetic fuzz, garbage-input fuzz, determinism check | see 5.6 |

Interfaces (shape only):

- encode: `bytes = Encode(baseSnapshot*, header, ops, baselines, tables)`; decode: `ops = Decode(bytes, lookupBase(delta) -> baseSnapshot*, baselines, tables)` where `lookupBase` is called after the first symbol (the delta distance) has been decoded, exactly as in `msg_get` of `codec2.py`.
- All bounds are checked while decoding: entity numbers below `MAX_GENTITIES`, field indices below the table length, string and download lengths below `MAX_MSGLEN`, end of input. A failure returns an error that makes the client drop the message (as `CL_ParseSnapshot` already drops invalid snapshots).

## 5.3 Changes by file

### Server

| file | functions | change |
|---|---|---|
| `server/sv_snapshot.cpp` | `SV_WriteSnapshotToClient`, `SV_EmitPacketEntities`, `SV_UpdateServerCommandsToClient`, `SV_SendClientSnapshot`, `SV_SendMessageToClient` | Under the new protocol, build the message with the codec from the current `clientSnapshot_t` and the chosen base frame; keep the base selection logic (`:110-134`) but make the base the newest acknowledged frame found through the new acknowledgement record (phase 5). Compute the rate delay from the real message size (`SV_RateMsec`). Keep `svc_setgame` handling. Detect `MAX_MSGLEN` overflow and fall back (phase 4). |
| `server/sv_client.cpp` | `SV_SendClientGameState`, `SV_ExecuteClientMessage`, `SV_WriteDownloadToClient`, userinfo parsing | Encode the gamestate and downloads with the codec; shorten the gamestate resend path (phase 5); parse the new acknowledgement record. Reconsider the default `rate` clamp. |
| `server/sv_net_chan.cpp` | `SV_Netchan_Transmit`, `SV_Netchan_Process`, encode / decode | The XOR "encryption" has no security value. Keep it as is or remove it, depending on the encryption decision (04, section 4.7). `svc_EOF` handling (`:140`) must be consistent with the codec's end-of-message operation. |
| `server/sv_main.cpp`, `sv_init.cpp` | cvars, frame loop | New cvars (`net_codec`, table file name, statistics). `sv_padPackets` and `svc_nop` padding must be reproduced or dropped. |
| `server/server.h` | `client_t`, `clientSnapshot_t` | Keep the frame history needed for bases; add the acknowledgement bitfield and counters. |
| `qcommon/msg.cpp` | field tables | Remain the source of truth for the schema generator. Legacy `MSG_*` stays for OOB, the `connect` handshake and the legacy path. |
| `qcommon/net_chan.cpp` | `Netchan_Transmit`, `Process` | Phase 5: fragment-size policy, optional ack bitfield in the packet header, reliable channel for gamestate and commands. |
| `qcommon/huffman.cpp` | | Remains in use for OOB data (the `connect` string). Optional later replacement is a tiny gain (the qbc author reports 10 % on that string; unverified). |

### Client

| file | functions | change |
|---|---|---|
| `client/cl_parse.cpp` | `CL_ParseServerMessage`, `CL_ParseSnapshot`, `CL_ParsePacketEntities`, `CL_DeltaEntity`, `CL_ParseGamestate`, `CL_ParseSetGame`, `CL_ParseDownload` | Under the new protocol decode with the codec into `clSnapshot_t`, `cl.parseEntities` and `cl.entityBaselines`. Reproduce the validity rules (`:243-263`): base valid, matching message number, entities not overwritten. Keep `%` sanitation of strings. |
| `client/cl_net_chan.cpp` | encode / decode | Mirror of the server file. |
| `client/cl_input.cpp` (`CL_WritePacket`) and `client/cl_main.cpp` | acknowledgement fields in client messages; `rate` and `snaps` defaults | Phase 5: send the acknowledgement record; re-evaluate the default `rate` (4000, `cl_main.cpp:2636`). |
| `client/client.h` | `clSnapshot_t`, `clientActive_t` | Keep. |
| cgame, game, ui | | No change (principle 1). Re-run the prediction error counters (`cg_showmiss`) before and after to detect any difference. |

### Tools and build

| item | content |
|---|---|
| state recorder | A server option that writes, for each client, the sequence of snapshot states (player state, entity list, header), the baselines, the commands and the gamestate in a simple binary file. It replaces the capture of the study, which recorded bitstreams and needed a Huffman decoder. |
| trainer | Reads recordings, runs the encoder in counting mode, writes the tables file (per-context counts to probabilities, per-field absolute/delta modes). The study's `choose_modes` and `Tables` are the reference. |
| evaluator | Splits recordings by time, trains on one part, reports mean / p95 / max bytes on the other, as in `bench2.py`. Used for every table update. |
| simulator | Loss, latency and rate simulation over a recording, as `sim_loss.py` and `frag_rate.py`. |
| CI | Round-trip on golden traces, fuzz, sanitizer builds, size regression check against the previous tables. |

## 5.4 Phases

### Phase 0: groundwork (S)

- Move the study's tools into a maintained location with relative paths (they have hard-coded Windows paths, see 07).
- Create one source of truth for the field descriptors and make the schema generator read it.
- Build the state recorder and record a first set of traces from the current server, at the snapshot rate the game will ship with (TaystJK defaults to `sv_fps` 40; the study's captures are at 20, see 09 section 9.2).
- Decide: protocol version number, whether to keep the legacy path, the name and location of the tables file.
- Acceptance: a recording of a game can be replayed through a harness that reads states back and re-encodes them with the legacy writer to the same bytes.

### Phase 1: the codec library, standalone (M)

- Implement `rans`, `schema`, `fieldcodec`, `msgcodec`, `tables` and the tests.
- Port the study's behaviour: gap coding of changed fields, `eType` first, per-field modes, class + mantissa, float kinds including angle indices, header deltas, pilot and vehicle states, commands, gamestate, downloads.
- Add the prediction modes of 04 section 4.2 (time against `serverTime`, position from velocity, second-order angle) and measure them with the real encoder.
- Train tables on the study's recordings (or new ones) and reproduce the study's numbers.
- Acceptance: 0 mismatches on the golden traces and on at least 100,000 synthetic messages; random-byte decoding never crashes under a sanitizer; on a held-out recording of similar content the mean snapshot is within 5 % of 140 bytes (study: 139.6 on the 5-minute test part), the gamestate of the study map is near 4 KB, and encode time per snapshot is of the order of 2 microseconds for 300 symbols on an optimised build.

### Phase 2: integration behind a flag (M)

- Wire encode into `SV_SendClientSnapshot` / `SV_SendClientGameState` and decode into `CL_ParseServerMessage` under the new protocol number. Both sides must refuse a protocol mismatch at connection.
- Make the server and client frame histories hold what the codec needs (full states of the last 32 snapshots; the engine already has `client->frames[PACKET_BACKUP]` and `cl.snapshots[]`).
- Handle bots (never sent anything), spectators (a followed player's state), vehicles, downloads, `svc_setgame`, `svc_mapchange`.
- Acceptance: a normal game plays identically with `net_codec 0` and `1`; the prediction error counters and `cg_showmiss` output show no new corrections; bytes per snapshot reported by a cvar match the study within the tolerance above.

### Phase 3: training pipeline (M)

- Record varied sessions: several maps and modes, several humans (this is the gap of the study), saber and force use, vehicles if the game keeps them, different base ages and a range of ping and loss.
- Establish train/held-out discipline and a size regression check; version the tables with the schema.
- Decide between one global table set and per-mod or per-gametype sets, with measured sizes.
- Acceptance: documented mean / p95 / max bytes on held-out sessions per map and mode; if the held-out sizes are more than 15 % above the training-set sizes, enlarge or diversify the training data before shipping.

### Phase 4: safety and limits (S to M)

- Escape path: per entity or per message raw format, chosen by size, bounded to the raw layout.
- Overflow handling: if the message would exceed the cap, retry with fewer entities or a smaller update set; never send a truncated or empty snapshot silently.
- Decoder hardening and a fuzz corpus in CI.
- Acceptance: the synthetic worst case (every field random, 1000 entities) produces a message no larger than the raw layout plus a small constant, and decodes.

### Phase 5: transport (M to L)

**Exploratory: this phase is a plan to be confirmed after phases 1 to 3.** The decision procedure, requirements and candidate options are in [08-transport-study.md](08-transport-study.md).

Order of preference, each step measurable on its own:

1. Acknowledgement record: add an ack bitfield (last 32 messages) to the client packet, and let the server pick the newest acknowledged base from it. Measure with `sim_loss.py`-style simulation and on a live server. (Judgement: the study modelled the single-number acknowledgement only.)
2. Gamestate and server commands: remove the dependence on the 1000 ms resend for the gamestate; make fragments recoverable (short resend timer or acknowledgement of fragments).
3. Fragmentation policy: avoid fragments for snapshots (size cap and escape), keep them for the gamestate if needed.
4. Encryption decision and, if wanted, adopt an authenticated packet format; evaluate yojimbo and GameNetworkingSockets against 04, section 4.7, including licences, maintenance, dependencies, maximum clients, CPU. Do not adopt a library only for the codec; the codec does not need it.
5. Default `rate` and the rate formula, after the codec's real sizes are known.

### Phase 6: live measurement and tuning (M)

- Run public or playtest servers with the recorder and compare: bytes per client per second, snapshots per second under the default `rate`, prediction errors, CPU per client, percentage of escaped entities.
- Decide the position-rounding question for non-solid entity types after reading `cg_players.c`, `cg_weapons.c`, `cg_event.c`.
- Update the tables from the new recordings.

### Phase 7: beyond the codec (L, optional)

These were not studied and have the largest unexplored potential:

- Priority accumulators and per-client bandwidth budgets; distance-based update frequency; entity culling.
- Motion-model prediction of positions (use `trDelta`, previous deltas) so that residuals are small even with exact floats.
- The client-to-server `usercmd` stream (`MSG_WriteDeltaUsercmdKey`, `msg.cpp:740`).
- Higher snapshot rates with delta chains.

## 5.5 Compatibility and fallback

- New protocol number; refuse a mismatch at the connect handshake (the `connect` and `getinfo` out-of-band messages keep the legacy Huffman).
- During development: cvar `net_codec` (0 legacy, 1 new). The legacy path is deleted once phase 6 is accepted.
- Tables: a missing or wrong-version file selects the uniform fallback (larger packets, still correct) with a warning, never a crash.
- Mods: a mod that adds or widens network fields needs tables for them (see 04, section 4.6); until then its extra fields use uniform contexts.

## 5.6 Test matrix

| test | method | when |
|---|---|---|
| Golden round trip | decode a recorded stream, re-encode, decode again, compare field by field; compare the engine's own legacy decode as well | every build |
| Synthetic fuzz | `fuzz.py` generator ported to C++ (any base age, vehicle and pilot, download, extreme values) | every build |
| Garbage fuzz | random and mutated byte streams into the decoder under ASAN / UBSAN | nightly |
| Determinism | same bytes in, same states out on MSVC, GCC, Clang, x86 and x64, with and without optimisation; angle decode compared bit by bit for all 65,536 indices | every build |
| Size regression | evaluator on held-out recordings; fail if the mean grows by more than 3 % | on table or schema change |
| Loss / latency / rate | simulator over recordings (loss 0 to 20 %, RTT 50 to 400 ms, rate 3000 to 25000) | on transport change |
| Worst case | adversarial snapshots (crowded, random) must stay within the cap and decode | every build |
| CPU | the C benchmark over recorded streams on the target machine | on codec change |
| Live | two or more machines, several humans, packet loss emulation | phase 6 |

## 5.7 Risks

| risk | consequence | mitigation |
|---|---|---|
| Training data too narrow (one map, bots only) | real packets larger than the study's | phase 3 recordings; escape path; held-out check before shipping |
| Table drift when game content or mods change | larger packets, never wrong packets | versioned tables, uniform fallback, retrain step in the release procedure |
| Angle index or float edge case differs between compilers | client-server mismatch of a view angle | bit-exact test over all indices; encoder checks the round trip per value |
| Hidden dependence on exact server floats | the study treats `-0.0` and `+0.0` as equal because the engine does | keep the equivalence criterion explicit; compare with the legacy decode |
| Message size growth under adversarial or unusual states | overflow or fragmentation | escape path, cap, fallback |
| Acknowledgement redesign breaks the base-selection logic | wrong base, corrupted deltas | codec round trip tests with simulated loss, as `sim_loss.py` |
| Transport library adoption cost | time spent on a feature the codec does not need | adopt only if the requirements in 04 section 4.7 are not met more cheaply in the existing netchan |
| Prediction behaviour change | visible corrections | keep positions exact; compare `cg_showmiss` counters |
| Scope creep into priority and culling | schedule | keep phase 7 separate from the codec delivery |

## 5.8 Success criteria taken from the study

These are targets, not guarantees; they hold for the study's content (one map, bots) and must be re-measured on the new recordings:

- Snapshot payload mean about 140 bytes against 337 today (-58 %); p95 about 240; maximum below 500 for ordinary snapshots.
- 15 snapshots per second instead of 8 at the default `rate` 4000.
- Gamestate at about 4 KB instead of 7 KB on the study map.
- Encoding and decoding each below 0.3 % of a core for 64 clients at 20 snapshots per second.
- Zero decode mismatches in every test of 5.6.
