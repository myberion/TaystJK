# Netcode rewrite: progress

This folder records where the work stands, so that a new session (human or model) can resume without re-reading everything. One document per finished step of [PLAN.md](../PLAN.md); this page is the summary and is updated at the end of every session.

Last update: 5 November 2026.

## Where we are

| step of the plan | status | result | document |
|---|---|---|---|
| before the plan: study moved into TaystJK, re-checked | done | every number of the study reproduced in this repository; TaystJK is wire-compatible with 1.01 except the pilot player state | [../08-taystjk.md](../08-taystjk.md), [../06-reproduction.md](../06-reproduction.md) section 6.7 |
| 1. prediction modes and rANS trick in the Python codec | **done** | 139.1 to **107.3** real bytes per snapshot (engine 337.6), 0 round-trip failures | [step-01-prediction-codec.md](step-01-prediction-codec.md) |
| 2. field descriptors from one source | **next** | | |
| 3. state recorder in the server | to do | | |
| 4. new recordings | to do, **owner** | can start now with the YBEProxy capture on the 1.01 server | |
| 5 to 13 | to do | | |

## Next actions

1. Step 2 of the plan: generate the field descriptors from the arrays of `codemp/qcommon/msg.cpp`, with a schema version and hash, and decide what to do with `netf_overrides.txt` / `psf_overrides.txt`.
2. Owner, in parallel: record games with the YBEProxy capture on the original 1.01 dedicated server (`proxy_sv_captureNet 1` before the first `map`): several humans together, other maps, other modes (CTF, duel, team FFA), a few minutes each. Keep one or two recordings aside; they must never be used for training.

## Decisions taken (with the date)

| date | decision | where it is written |
|---|---|---|
| 4 Nov 2026 | Measurements stay referenced to the original 1.01 server (the captures) | [../08-taystjk.md](../08-taystjk.md) |
| 4 Nov 2026 | Owner's goals: no loss of precision (values identical to today's or more precise, never less); no compatibility constraint; as few network settings as possible, computed at run time; netcode reusable by other JKA forks | [../README.md](../README.md), "Owner's goals" |
| 4 Nov 2026 | Branch defaults: `sv_fps` 20 (was 40), `sv_clientRate` 25000 (was 90000); `snaps` stays 40, `rate` 25000, `cl_maxpackets` 100; rate and snaps policies and the pilot fix are kept | `codemp/server/sv_init.cpp:991`, `:1014`; [../08-taystjk.md](../08-taystjk.md) section 8.2 |
| 4 Nov 2026 | Do not restore the 1.01 pilot rule (`size - 82`): it was a bug | [../08-taystjk.md](../08-taystjk.md) section 8.1 |
| 4 Nov 2026 | No rounding of positions, neither on the client nor on the server (float32 is finer than any useful grid) | [../04-recommendations.md](../04-recommendations.md) section 4.3 |
| 4 Nov 2026 | Settings to remove and compute at run time; new debug cvars and commands are welcome | [../04-recommendations.md](../04-recommendations.md) section 4.10 |
| 5 Nov 2026 | Position prediction in integer arithmetic (0.5 byte more than float, same bits on every compiler) | [step-01-prediction-codec.md](step-01-prediction-codec.md) |
| 5 Nov 2026 | The plan is a separate document, [PLAN.md](../PLAN.md); progress is recorded here | this folder |

## Open questions for the owner

- Which game module the new game will use: the 1.01 base module (players sent as `TR_LINEAR_STOP`, like the captures) or TaystJK's (JAPRO; players sent as `TR_INTERPOLATE` with `g_smoothClients 1`). The codec handles both; the recordings of step 4 should cover the one that will ship.
- The six transport questions of [../07-transport-study.md](../07-transport-study.md) section 7.7 (encryption, player count, platforms, IPv6, server browser, Apache-2.0 / OpenSSL licence position). Not needed before step 9.

## Known issues found on the way (not fixed)

- With `sv_ratePolicy 2` and `sv_maxRate 0`, the client rate becomes 0 and `SV_RateMsec` divides by zero (`codemp/server/sv_client.cpp:1218`, `codemp/server/sv_snapshot.cpp:782`). Read only, not tested.

## State of the repository

- Nothing of this work is committed unless the owner did it. Check `git status` at the start of a session.
- Branch changes in the engine so far: the two defaults above in `codemp/server/sv_init.cpp`. Everything else is under `docs_rewrite/netcode/`.
- Python caches (`*.pkl`) are rebuilt by the tools and ignored by git.

## How to resume

1. Read this page, then the document of the last finished step, then the next step in [PLAN.md](../PLAN.md).
2. Check that the tools still give the recorded numbers: from `docs_rewrite/netcode/tools/python`, `python run_full.py` (study codec: snapshot mean 139.1, 0 failures) and `python run_pred.py pi --fuzz` (step 1: 107.3, 0 failures). About 2 minutes in all.
3. At the end of the session, write the step document, update the tables above, and record new decisions with their date.
