# UmbraDB tapes

Recorded Midnight node RPC responses for repeatable UmbraDB tests and offline replay. Tape data is published as GitHub release assets; this repository holds the inventory, checksums, and format documentation.

## Collection version and provenance

The initial collection is **0.10.0-rc.1**, matching UmbraDB's `v0.10.0-rc.1` release at [`8dcc9f30e41fe78c884a05d61ca06e10d7c0f373`](https://github.com/midnightntwrk/umbradb/tree/8dcc9f30e41fe78c884a05d61ca06e10d7c0f373). The package version in that source is `0.10.0`.

The seven original tapes below were recorded on **2026-10-26**, using development snapshots whose package version was **0.9.5**, and carried forward into the rc.1 fixtures. They have not been re-recorded or relabelled as newly recorded by rc.1. Their original node versions, endpoints, recording parameters, original lane commit abbreviations, full integrated source commits, and limitations are in the [versioned manifest](releases/v0.10.0-rc.1/manifest.json). The original lane commits were recorded as `b79ab0d` and `eeaf83f`; their retained integrated counterparts are `0e81e0f0ae96fed19268ba8ca497fe26ee554b53` and `86f88fb4cae10e0a96cbba61289a9d51553a1072`. Integration commits identify retained source, not the exact original lane checkout.

## Available tapes

All block endpoints are **inclusive**. A range starting at zero includes genesis. Filename convention: `{network}-{start}-{end}[-{notes}].tape.jsonl`, with decimal block numbers and no thousands separators.

| Asset | Blocks | Bytes | Purpose |
|---|---:|---:|---|
| `preview-0-10000.tape.jsonl` | 0–10,000 | 61,696,218 | Preview from genesis, ledger v8 |
| `preview-1073620-1075620.tape.jsonl` | 1,073,620–1,075,620 | 10,329,363 | Preview runtime 1.0.300 upgrade |
| `preprod-0-10000.tape.jsonl` | 0–10,000 | 45,658,750 | Preprod from genesis, ledger v8 |
| `mainnet-1774000-1780000.tape.jsonl` | 1,774,000–1,780,000 | 36,412,864 | Mainnet runtime 0.22 to 1.0 upgrade |
| `mainnet-1780000-1790000.tape.jsonl` | 1,780,000–1,790,000 | 87,118,415 | Mainnet continuation with contract activity |
| `stagenet-0-10000.tape.jsonl` | 0–10,000 | 50,052,119 | Stagenet from genesis, ledger v9; third-party source |
| `devnet-0-235-v8-v9.tape.jsonl` | 0–235 | 4,237,638 | Local development chain, ledger v8 to v9 conversion |

Total: **295,505,367 bytes**. The two historical Mainnet ranges intentionally share block 1,780,000. The devnet asset was originally named `devnet-v8-v9-0-235.tape.jsonl`; only its distribution filename changed, and its bytes and SHA-256 are unchanged.

## Download and verify

Download the desired `.tape.jsonl` assets, `manifest.json`, and `SHA256SUMS` from the collection release. The same manifest and checksums are committed under [`releases/v0.10.0-rc.1/`](releases/v0.10.0-rc.1/).

For the full collection, run this in the download directory:

```sh
sha256sum --check SHA256SUMS
```

For a subset, `sha256sum --check --ignore-missing SHA256SUMS` verifies only the downloaded assets. A tape's identity is the lowercase SHA-256 of its **exact bytes**. A renamed file retains its identity; editing, reformatting, or changing line endings does not.

UmbraDB's existing real-tape tests use the directory in `UMBRADB_TAPES`. For those tests at rc.1, also provide the devnet asset under its original filename (copy it or create a local symlink), because the pinned upstream fixture manifest still uses `devnet-v8-v9-0-235.tape.jsonl`.

## Tape format

A tape is UTF-8 **canonical JSON Lines**: one record per line, compact JSON without spaces, and an LF newline after every record, including the last. There is no header record or JSON-RPC envelope: network and provenance belong in the manifest. Field order is fixed for each record kind; nested JSON values follow the pinned writer's serialization rules. Arbitrarily sorting keys or pretty-printing produces a file the canonical reader rejects.

The format has five record kinds:

| Kind | Fields in serialized order | Meaning |
|---|---|---|
| Call | `method`, `params`, exactly one of `result` or `error`, optional `delayMs` | Answer a matching RPC call, optionally delayed |
| Subscription | `subscribe`, `method`, `params` | Answer a subscription call with the recorded ID |
| Notification | `notify`, `result` | Deliver a notification for a preceding subscription in this session |
| Disconnect | `disconnect: true` | End the session and drop the connection |
| Fault | `fault`, `method`, `params`, optional `code` | Make a matching call time out or fail |

Illustrative canonical records, demonstrating all five kinds (not a real chain recording):

```jsonl
{"method":"system_health","params":[],"result":{"isSyncing":false,"peers":1,"shouldHavePeers":true}}
{"subscribe":"heads-1","method":"chain_subscribeNewHeads","params":[]}
{"notify":"heads-1","result":{"number":"0x01"}}
{"fault":"timeout","method":"example_timeout","params":[]}
{"disconnect":true}
```

`params` is an array or object. `method` is a nonempty string. `delayMs` is an integer from 0 to 2,147,483,647. Subscription IDs are strings or safe integers and must be unique within a session; notifications reference an existing ID in that session. A fault is `timeout` or `error`; its optional integer `code` is used for an error fault. Unsupported fields are rejected.

Large JSON integers are preserved losslessly by UmbraDB's reader and writer. An ordinary JavaScript `JSON.parse`/`JSON.stringify` round trip may round integers outside ±(2^53−1), destroying the original data and checksum. Use the pinned tape tooling to read or generate these files.

The normative implementations are [`records.ts`](https://github.com/midnightntwrk/umbradb/blob/8dcc9f30e41fe78c884a05d61ca06e10d7c0f373/tools/tape/records.ts), [`tape.ts`](https://github.com/midnightntwrk/umbradb/blob/8dcc9f30e41fe78c884a05d61ca06e10d7c0f373/tools/tape/tape.ts), and [`json.ts`](https://github.com/midnightntwrk/umbradb/blob/8dcc9f30e41fe78c884a05d61ca06e10d7c0f373/tools/tape/json.ts).

## What a range recording contains

For each recorded height, the range recorder captures `chain_getBlockHash`, `chain_getHeader`, `chain_getBlock`, `state_getRuntimeVersion`, System.Events through `state_getStorage`, and `midnight_ledgerStateRoot`. It captures `state_getMetadata` at the range's first block and runtime changes, rather than at every height. A range including genesis also captures `system_properties` and paged cNight registration storage needed for bootstrap.

The historical recordings end with a 60-second live window: new-head and finalized-head subscriptions, `chain_getFinalizedHead`, its header, and notifications. That finalized head is the head at **recording time** and can lie far beyond the recorded range. A bounded replay must choose its own stop height; the asset does not contain every block up to its captured live head.

Replay matches calls by method and parameters within a session, using occurrence order for repeated matching calls. Notifications are released only after the records preceding them have been consumed. A partial range replay can therefore have quiet subscriptions. The k-th client connection receives the k-th session; these range tapes contain one session. Reconnecting therefore requires restarting the tape server/reloading the fixture, unless an adapter explicitly handles reconnections. The exact behavior, including repeatable last-call responses, is defined by the pinned [replay rules](https://github.com/midnightntwrk/umbradb/blob/8dcc9f30e41fe78c884a05d61ca06e10d7c0f373/tools/tape/rules.ts).

Restarting at an arbitrary cursor can request `state_getMetadata` at that cursor hash. That response may be absent because metadata is recorded only at the range start and runtime changes. Such a replay can need a derived fixture or metadata-aware replay adapter. Preserve the published original and its checksum when making any derived fixture.

Tapes starting at genesis include bootstrap data for replay from genesis. A tape starting mid-chain supports decoding and event/outcome inspection on its own, but complete ledger replay and state-root checking require the matching preceding ledger state, normally restored from an UmbraDB backup. Tape files are RPC fixtures, not database backups. Consult each manifest entry's historical `bounds` and `notes` for network-specific requirements, runtime boundaries, captured event kinds, and events the recording does not demonstrate.
