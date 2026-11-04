# 4. Recommendations

Each recommendation states how strongly the evidence supports it: **measured** (a number in [03-results.md](03-results.md)), **read** (a fact in the code, [02](02-engine-code-findings.md)), or **judgement** (reasoned from the above, not tested). Transport recommendations are mostly judgement because no transport library was evaluated.

## 4.1 Summary

| area | recommendation | evidence |
|---|---|---|
| Entropy coder | Static, offline-trained, per-packet-independent rANS with 12 to 15-bit probabilities. First implementation: a ryg_rans-style 32-bit byte-renormalising rANS (CC0, 319 lines). Keep the coder behind a small interface; FSE (tANS) saves 2 bytes per packet and can replace it later. | measured |
| Do not use | Adaptive coders (qbc, nibrans, adaptive binary), generic compressors (zstd, deflate, lzma), context mixing, anything whose state survives across packets | measured, judgement |
| Message model | Replace the snapshot encoding by the schema in 4.2: one stream per message, delta against an acknowledged base, no `lc` byte, changed fields listed as gaps with context, per-field trained distributions | measured |
| Precision | Keep every value the client sees identical to today. Code view angles and entity angles as 16-bit indices (lossless). Do not round positions. | measured, read |
| Tables | Generate the probability tables offline from recorded states, version them, load them as data, and ship a fallback of uniform contexts | judgement |
| Safety | Add a raw-format escape path and a hard cap on message size | measured (worst case) |
| Transport | Phase 1: keep the existing netchan. Phase 2: improve acknowledgement latency and precision, and the gamestate/reliable path. Evaluate libraries only against the requirements in 4.7 | measured (what matters), judgement (how) |
| Rate | Re-evaluate the default client `rate` once the new codec exists | measured |
| Not studied | Entity priority, interest culling, bandwidth budgets, client to server direction | not measured |

## 4.2 The message model

This is what the experiments actually coded; the codec in `tools/python/codec.py` and `codec2.py` is a working reference.

### Message

1. `delta` (8 bits): distance in message numbers to the base snapshot, 0 for none. It comes first because the decoder needs it to locate the base.
2. `ack` (the engine's `lastClientCommand`): coded as a signed delta from the base's `ack`.
3. A list of operations, each with a 3-bit type and the previous type as context: end, nop, gamestate, server command, snapshot, setgame, download, mapchange.

### Snapshot operation

- `serverTime`: signed delta from the base's `serverTime`.
- Flags, area-mask bytes (as today).
- Player state: a pilot bit (derived exactly as the engine does), then the changed fields.
- Vehicle player state when the player state has a vehicle, against the base's vehicle state (or zero).
- Entities.

### Changed fields (player state and entities)

- Fields are listed in table order as gaps: the symbol is the index of the next changed field, in a context made of the previous changed index (and, for entities, the entity's class); a terminator ends the list. This replaces the `lc` byte and the per-field "changed" flag. It beat one flag per field (168.5 against 173.7 bytes).
- For entities the type `eType` is coded first (a flag and an 8-bit value) because its class is the context for everything else.
- Entity list: gap coding of entity numbers (symbol 0 to 39 direct, 40 = escape plus 10 raw bits, 41 = end), an operation per entity (delta, new from baseline with changes, remove, new equal to baseline), as in the engine.

### Values

Each field has a mode chosen offline: **absolute** or **delta against the base value**. A value is coded as a class (the bit length of the zig-zag value) from a static per-field distribution, followed by the mantissa bits raw. Fields up to 8 bits wide are coded directly as symbols in absolute mode.

Floating-point fields have a kind symbol: zero; integral value within +-4096; **16-bit angle index**; any other float. An integral float is coded as an integer (absolute or delta). An angle index is coded as a delta from the old angle's index. Any other float is coded as the signed difference of the two 32-bit patterns (monotonic for floats of the same sign), or 32 raw bits.

### Why these choices

- The gap list and per-field distributions are where most of the 50 % comes from (measured: 337.6 to 168.5).
- Angles as indices are the second largest item (a further 19 %, 168.5 to 136.9) and cost no precision.
- The delta of 32-bit float patterns is lossless and exploits that nearby positions share exponent and high mantissa bits. It leaves about 20 bits per coordinate. Reducing that further needs a model of motion, not a different coder (see 06, section 6.7).

## 4.3 Precision policy

**Rule: the client must receive exactly the values it receives today.** The study's equality criterion (the round-trip tests) is "equal to what the current client sees", which includes truncation to the declared width and `-0.0` becoming `+0.0`.

Reasons for not rounding positions (read, 02 section 2.8):

- the client prediction restarts from the received player state every snapshot and flags an error at 0.1 unit; the received position, velocity and view angle are the initial conditions of a simulation;
- solid entities used by the prediction (movers, other players) take their positions from the received entity state, so rounding them changes the local player's predicted movement even though the server stays exact;
- `trDelta` (velocity) errors grow with the extrapolation time in `BG_EvaluateTrajectory`.

Measured upside of rounding other entities' positions to 1/16 unit: from 136.9 to 113.9 bytes (a further 17 %). It could be revisited for entity types that are neither solid nor used for aiming (for example cosmetic effects) after reading `cg_players.c`, `cg_weapons.c` and `cg_event.c`, which were counted but not read. It is not part of the recommended design.

Angle decoding must produce the same float as the engine: `(float)(k * (360.0 / 65536))` with `k` a 32-bit integer and the product evaluated in double. `360/65536` is an exact binary fraction, so the product is exact and the rounding to float is the standard one; this is bit-exact on any IEEE-754 compiler. The encoder sends an angle as an index only if re-computing that float gives the same 32 bits; otherwise it falls back to the full-float kind. Validate the equality on the target compilers (06, section 6.4).

## 4.4 Entropy coder

Measured (03, sections 3.4 and 3.5):

- all static coders are within 1 % of the ideal; the real 32-bit rANS costs 3.5 bytes per packet more than the ideal (final state and byte rounding); FSE with a 12-bit state costs 2.0 bytes less than that;
- static Huffman per context is 10 % larger (186.1 against 168.5), because integer code lengths waste bits on skewed distributions;
- CPU is irrelevant to the choice (0.1 to 0.3 % of a core at 64 clients).

Recommendation: rANS 32-bit with byte renormalisation, probabilities of 12 to 15 bits, one independent stream per message. Reasons: a single header, public domain, easy to audit and to test, fastest encoder of those measured (951 ns against 1,524 ns for FSE).

Switch to FSE/tANS if the 2-byte saving matters. It matters when packets are tiny (the solo case is 7 to 9 bytes of payload) and when no other fixed overhead dominates; with 32 bytes of IP, UDP and netchan header, and possibly 16 bytes of authentication tag, it is a small share. Two things to keep in mind for FSE: its tables are large at table log 12 (use smaller logs for contexts with few symbols), and tables of different contexts can share one state only if all use the same table log (the benchmark did this by switching tables on a single state).

The interface should hide the coder: `encode(units) -> bytes` and `decode(bytes, context provider) -> symbols`, where a unit is a symbol with its context or a run of raw bits. `codec.py` is written this way.

Rejected, with measured reasons: static Huffman per context (+10 %), adaptive binary (no gain on trained tables, 2 to 3 times the CPU, about 5 binary decisions per symbol), qbc and nibrans (no better than the engine's own table), zstd, deflate, lzma (no gain, large frame overhead on tiny packets). Draco and fsc (Apache-2.0) and Turbo-Range-Coder (GPL v3) cannot be combined with the GPL-v2-only files of CleanJK without relicensing. Context mixing was not measured; its size benefit over static tables is unknown (06, section 6.7).

## 4.5 Tables and training

- The tables are data, produced offline from recorded snapshot states and not from bitstreams (a state recorder inside the server avoids the Huffman decoding step that this study needed).
- Tables must be versioned together with the protocol and the entity/player-state field list. A mod that changes the field list or the widths needs its own tables or the generic ones.
- Train on varied data: several maps, several game modes, several humans, a range of base ages, and server commands and gamestates. The study trained on one 3.5-minute session on one map with bots: expect the real sizes to be somewhat worse on other content, and measure the gap with held-out recordings before shipping.
- Smoothing: counts get a small additive constant (0.01 worked best on validation). Unseen symbols then cost about log2(N/alpha) bits, so the escape path (4.8) must exist for states far from the training data.
- Table size: the study used dense tables for convenience (316 to 510 contexts, 32,000 entries). A real implementation should store non-zero entries only, with an escape symbol; the study counted about 1,900 non-zero entries (about 7 KB).
- Per-field absolute/delta modes are part of the table data (50 of the 100 distinct field names chose delta).
- Base age: the loss and latency runs use tables trained on a mix of ages; whether age-specific tables (or the age as a context) would be smaller was not measured, because the 0 % loss, 50 ms figure of that experiment (152.2) is not comparable with the age-1 codec figures (different test selection and it includes messages with server commands). Test it before fixing the table design, and keep the age low (4.7).

## 4.6 Mods and game-module compatibility

- The codec should consume and produce `entityState_t` and `playerState_t`, so `game` and `cgame` are unchanged: they see the same `trap_GetSnapshot` data.
- The field list and widths (`entityStateFields`, `playerStateFields`) must remain the single description of what is networked. The codec's tables should be generated from it (the study extracted them from `msg.cpp` by pattern; a build step or code generation from the same arrays is better).
- Runtime width overrides (`netf_overrides.txt`, `psf_overrides.txt`, 02 section 2.3) are used by mods to widen the `user*` fields. Decide explicitly: keep the mechanism (tables regenerated or widths read at start, with the generic tables as a fallback for those fields), or remove it and put the needed width in the field list. Do not silently ignore it.

## 4.7 Transport

**Exploratory. See [08-transport-study.md](08-transport-study.md) for the facts read in the code, the security analysis, the requirements and the candidate libraries as read. Do not act on this section before the codec exists (phases 1 to 3 of 05); the transport choice should be made with the real codec payloads.**

What the measurements say about what the transport must do:

1. **Acknowledgement latency drives size.** The delta base is the newest snapshot the server knows the client received, so every extra snapshot of round trip adds about 20 % at ages 1 to 2 (measured). Loss barely matters because the base is always the latest acknowledged one. The transport should acknowledge in every outgoing client packet and the server should use the newest acknowledged sequence precisely.
2. **A precise acknowledgement record.** Today the client acknowledges one number (the latest received message). An ack bitfield of the last 32 messages lets the server know exactly which of several recent snapshots arrived and pick the newest as base without waiting. This is a judgement; the sizes of the gain were not simulated beyond the loss model above.
3. **Fragmentation avoidance.** The engine drops a whole message when one fragment is lost and delays further snapshots while fragments remain. The new codec makes fragmentation rare (2 of 8,836 messages), but the gamestate (4 fragments, 4,024 bytes on the study map) and server-command bursts still fragment. A reliable channel with its own retransmission, or application-level resend with a short timer, would remove the 1,000 ms gamestate resend wait (measured by model: extra join delay of 0.58 s at 10 % loss, 1.59 s at 20 %).
4. **Reliable commands.** Today every unacknowledged server command is repeated in every message until acknowledged. It works and is cheap when commands are rare (715 of 7,101 messages carried commands in the capture). Keep it unless a transport library provides a reliable channel for free.
5. **Fixed overhead.** Per packet today: 4 bytes netchan + 28 bytes IP/UDP. In the one-player case the new codec payload is 7 to 9 bytes, so headers are 80 % of the packet. An authentication tag (16 bytes in the hand-off's figure, not verified) would make the solo case cost more than today's unencrypted traffic (9.3 against 7.9 kbit/s in the study's table). Decide whether encryption is wanted before choosing.
6. **Licence and scale.** Anything adopted must be compatible with GPL v2 (BSD-3, MIT, zlib, CC0 are; Apache-2.0 and GPL v3 are not for the GPL-v2-only files), support the intended player count (the hand-off notes yojimbo at a maximum of 64 clients), IPv6, and a non-blocking API usable from the existing socket code.

Recommendation: do the codec first on the existing netchan (phase 1 of 05). The codec is independent of the transport and carries the largest measured gains. Then change the acknowledgement path and the gamestate path (phase 2). Evaluate yojimbo and GameNetworkingSockets only against the list above; their licences, maintenance, dependencies and costs were **not** checked in this study, and a single third-party report in the hand-off (GameNetworkingSockets at 8.4 % CPU with 256 clients) was not verified.

## 4.8 Safety

- **Raw-format escape.** In the absolute worst case (every field of every entity random) the new codec is 1.34 times the engine's size. An escape (per entity or per message) that sends the plain field values bounds the message to the size of a simple raw layout. Choose the escape per entity when its coded size would exceed the raw size, with a flag costing a fraction of a bit when unused.
- **Message cap.** `MAX_MSGLEN` is 49,152. The server should detect an overflow and fall back to fewer entities or a full-state retry, not to a dropped snapshot (the engine drops the snapshot and prints a warning, `sv_snapshot.cpp:792`).
- **Fuzzing in the build.** Keep the synthetic corpus (06, section 6.3) as a test: decode(encode(x)) == x on random valid states, plus decoding of random bytes without crashing (decoder bounds checks: array indices, entity numbers, string lengths).
- **Determinism.** The encoder and decoder tables and arithmetic are integer. The only floating-point step is the angle decode above.

## 4.9 Other server-side policy to revisit

- **Default `rate`.** At the client default of 4000 bytes per second the engine delays 93 % of snapshots; the new codec delays 39 %. With the new codec, the default can be raised or the rate formula's 48-byte allowance revisited. Measure after the codec exists.
- **Snapshot rate.** `snaps` 20 and `sv_fps` 20 are the defaults used in the captures. A higher rate is cheaper in bytes per snapshot with the new codec, but the study did not measure its effect on size per second.
- **Not studied (and potentially large):** per-client priority and bandwidth budgets, distance-based update frequency, interest management (culling entities the client does not need), and delta coding of entities relative to each other. The study measured only how well a given set of updates is coded, not which updates to send.
