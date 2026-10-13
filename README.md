# freebsd-ports-cardano-artifacts

Config artifacts used by the Cardano FreeBSD ports.

| Component       | Version  | Artifacts                            |
| --------------- | -------- | ------------------------------------ |
| cardano-node    | 11.1.3   | `cardano-node/<network>-configs/`    |
| cardano-db-sync | 13.7.2.1 | `cardano-db-sync/<network>-configs/` |

`<network>` is one of `mainnet`, `preprod` or `preview`.

The files are derived from the official release configuration published at
<https://book.play.dev.cardano.org/environments.html> (source repository:
<https://github.com/input-output-hk/cardano-playground>, directory
`static/book.play.dev.cardano.org/environments/`, deployed for node 11.1.1; the
11.1.2 and 11.1.3 releases carry no configuration changes). Only the logging,
metrics and topology settings are adapted to the ports. The genesis files are
byte-identical to the official ones and their hashes are unchanged.

## cardano-node 11.1.3

What changed compared with the previous (node 9.1.0) artifacts:

* **The legacy logging system is gone.** cardano-node 11.1.0 removed
  `iohk-monitoring` together with its configuration (`TurnOnLogging`,
  `Trace*` switches, `minSeverity`, `setupScribes`, `defaultScribes`,
  `setupBackends`, `rotation`, `hasEKG`, `hasPrometheus`, ...). The
  `TraceOptions` based tracing system is the only one left, and a node can no
  longer write its own log file (see [Logging to a file](#logging-to-a-file)).
* `ConsensusMode` is `GenesisMode` (Ouroboros Genesis). It needs the ledger
  peer snapshot `peer-snapshot.json`, referenced from `topology.json`.
* Lightweight checkpoints (`checkpoints.json`, mainnet and preview only).
  `CheckpointsFileHash` is the blake2b-256 hash of that file, like the genesis
  hashes.
* `LedgerDB` block (UTxO-HD, `V2InMemory` backend and the snapshot policy).
  Ledger state snapshots written by 9.x/10.x are not compatible; the first start
  replays the chain from the local immutable DB.
* `EnableP2P` is gone (P2P is the only networking mode). `PeerSharing` and the
  `TargetNumberOf*` keys are left at their defaults, as in the official files.
* `MinNodeVersion` is `11.1.0`.

### Files per network

| File                     | Purpose |
| ------------------------ | ------- |
| `config.json`            | Node configuration. JSON log lines on **stdout**; Prometheus metrics served by the node on `0.0.0.0:12798`. |
| `config-wtracer.json`    | Same configuration, but the traces are forwarded to `cardano-tracer`, which writes the log files. Prometheus on `0.0.0.0:12798` is kept. |
| `tracer-config.json`     | `cardano-tracer` configuration matching `config-wtracer.json`. |
| `topology.json`          | P2P topology: official bootstrap peers and `useLedgerAfterSlot`, the ledger peer snapshot, local roots with placeholders to replace. |
| `peer-snapshot.json`     | Ledger peer snapshot (big ledger peers) used while the node bootstraps. |
| `checkpoints.json`       | Lightweight checkpoints (mainnet and preview). |
| `*-genesis.json`         | Byron, Shelley, Alonzo and Conway genesis files. |
| `submit-api-config.json` | `cardano-submit-api` configuration (new tracing format since submit-api 10.2). |

The `severity`, `detail` and `maxFrequency` settings under `TraceOptions` are
the official release defaults: root severity `Notice`, with `ChainDB`,
`BlockFetch.Decision`, `Forge.*`, `Net.*` and `Startup.DiffusionInit` raised to
`Info`, noisy ChainDB events rate limited and `Mempool` silenced on mainnet
(`Info` on the test networks).

### Logging to a file

The node's tracing backends are `Stdout MachineFormat`,
`Stdout HumanFormatColoured`, `Stdout HumanFormatUncoloured`, `EKGBackend`,
`PrometheusSimple [suffix|nosuffix] [host] <port>` and `Forwarder`. There is no
file backend and no rotation inside the node. Two ways to get log files:

1. **cardano-tracer** (`config-wtracer.json` + `tracer-config.json`), the
   supported replacement for the old `FileSK` scribe. The node is started with
   `--tracer-socket-path-connect /socket/cnode-tracer.sock`; `cardano-tracer`
   listens on that socket (`AcceptAt`) and writes rotated JSON logs below
   `/logs/cardano-tracer/cnode-<network>/` (`node.json` is a symlink to the
   current file). The rotation keeps the retention of the old `rotation`
   block: 50 MB per file, 10 files, 24 hours, checked every 60 seconds. Add a
   second `logging` entry with `"logFormat": "ForHuman"` for plain-text
   `.log` files as well. The tracer also serves EKG on `127.0.0.1:12788` and
   Prometheus on `0.0.0.0:12808`.

   ```sh
   cardano-tracer --config tracer-config.json
   cardano-node run --config config-wtracer.json --topology topology.json \
       --tracer-socket-path-connect /socket/cnode-tracer.sock ...
   ```

   Start order does not matter: the node reconnects (`maxReconnectDelay` 30 s)
   and buffers up to `queueSize` (128) messages while the tracer is away, older
   messages are dropped. `config-wtracer.json` has no stdout backend, so with
   it the tracer has to run whenever the node runs; add
   `"Stdout MachineFormat"` to the `backends` list to log to both.

2. **stdout redirection** (`config.json`). The node prints one JSON object per
   line to stdout and the rc script captures it, for example with
   `daemon -o /logs/cnode.json` (add `-H` so that newsyslog(8) can rotate the
   file, or use `-S` to log to syslog instead). Nothing rotates the file
   otherwise.

### Metrics ports

| Port  | Served by                                | Config                               |
| ----- | ---------------------------------------- | ------------------------------------ |
| 12798 | cardano-node, Prometheus `/metrics`      | `config.json`, `config-wtracer.json` |
| 12788 | cardano-tracer, EKG (127.0.0.1 only)     | `tracer-config.json`                 |
| 12808 | cardano-tracer, Prometheus               | `tracer-config.json`                 |

The node itself no longer serves an EKG web page: `EKGBackend` only feeds the
in-process metric store that `PrometheusSimple` and the forwarder read from.
Prometheus is bound to `0.0.0.0` as in the previous artifacts; change the host
in the `PrometheusSimple` backend string or in `hasPrometheus` to restrict it.

### cardano-submit-api

`cardano-submit-api` reads the same tracing format as the node. The metrics
port is a command line option (`--metrics-port`, default 8080) and the network
is selected on the command line as well, so `PrometheusPort`, `GenesisHash`
and `RequiresNetworkMagic` are gone from `submit-api-config.json`. It logs
plain text to stdout (`Stdout HumanFormatUncoloured`); use
`Stdout MachineFormat` for JSON lines.

## cardano-db-sync 13.7.2.1

* `config.json` and the genesis files are copies of the node files. db-sync
  reads only the protocol and genesis keys from the node configuration and
  ignores the tracing keys.
* db-sync still uses the `iohk-monitoring` logging configuration, so
  `db-sync-config.json` keeps writing rotated JSON logs to
  `/var/db/cardano_db_sync*/logs/dbsync.json` itself.
* `insert_options`: since db-sync 13.7.0.1 `offchain_pool_data` and the new,
  separate `offchain_vote_data` default to `"disable"`. Both are enabled here
  to keep the data set of the previous artifacts (vote metadata fetching used
  to be part of `governance`).
* Optional settings worth knowing: `"ledger_backend": "lsm"` keeps the UTxO
  set on disk (about 2-3 GB RAM instead of about 16 GB on mainnet,
  experimental), `snapshot_interval`, `disable_epoch` and `stop_at_block`.
  See <https://github.com/IntersectMBO/cardano-db-sync/blob/13.7.2.1/doc/configuration.md>.
* 13.7.2.1 is built against the cardano-node 11.0.1 libraries (see its
  changelog) and talks to the node over the node-to-client protocol.

## References

* cardano-node 11.1.3 release: <https://github.com/IntersectMBO/cardano-node/releases/tag/11.1.3>
* cardano-node 11.1.1 release (legacy tracing removal): <https://github.com/IntersectMBO/cardano-node/releases/tag/11.1.1>
* cardano-tracer documentation: <https://github.com/IntersectMBO/cardano-node/blob/11.1.3/cardano-tracer/docs/cardano-tracer.md>
* trace-dispatcher documentation (tracing configuration): <https://github.com/IntersectMBO/hermod-tracing/blob/3d7aeb85d9ef01df48a44c751e45609b4c6dea00/trace-dispatcher/doc/trace-dispatcher.md>
* New tracing system (developer portal): <https://developers.cardano.org/docs/operators/monitoring/new-tracing-system/new-tracing-system/>
* Topology documentation: <https://developers.cardano.org/docs/operators/node/topology/>
* cardano-db-sync 13.7.2.1 release: <https://github.com/IntersectMBO/cardano-db-sync/releases/tag/13.7.2.1>
