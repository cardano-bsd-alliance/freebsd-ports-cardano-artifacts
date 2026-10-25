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
  longer write its own log file (see [Logging (syslog)](#logging-syslog)).
* `ConsensusMode` is `GenesisMode` (Ouroboros Genesis). It needs the ledger
  peer snapshot `peer-snapshot.json`, referenced from `topology.json`.
* Lightweight checkpoints (`checkpoints.json`). `CheckpointsFileHash` is the
  blake2b-256 hash of that file, like the genesis hashes. The official files
  have checkpoints for mainnet and preview only; preprod gets an empty list so
  that every network has the same set of files.
* `LedgerDB` block (UTxO-HD, `V2InMemory` backend and the snapshot policy).
  Ledger state snapshots written by 9.x/10.x are not compatible; the first start
  replays the chain from the local immutable DB.
* `EnableP2P` is gone (P2P is the only networking mode). `PeerSharing` and the
  `TargetNumberOf*` keys are left at their defaults, as in the official files.
* `MinNodeVersion` is `11.1.0`.

### Files per network

| File                     | Purpose |
| ------------------------ | ------- |
| `config.json`            | Node configuration used by the port. Plain-text log lines on **stdout**, which the rc script sends to **syslog**; Prometheus metrics served by the node on `0.0.0.0:12798`. |
| `config-wtracer.json`    | Optional, not used by the port: the traces are forwarded to `cardano-tracer`, which writes rotated log files. Prometheus on `0.0.0.0:12798` is kept. |
| `tracer-config.json`     | Optional, not used by the port: `cardano-tracer` configuration matching `config-wtracer.json`. |
| `topology.json`          | P2P topology: official bootstrap peers and `useLedgerAfterSlot`, the ledger peer snapshot, local roots with placeholders to replace. |
| `peer-snapshot.json`     | Ledger peer snapshot (big ledger peers) used while the node bootstraps. |
| `checkpoints.json`       | Lightweight checkpoints (empty list on preprod). |
| `*-genesis.json`         | Byron, Shelley, Alonzo and Conway genesis files. |
| `submit-api-config.json` | `cardano-submit-api` configuration (new tracing format since submit-api 10.2). |

The `severity`, `detail` and `maxFrequency` settings under `TraceOptions` are
the official release defaults: root severity `Notice`, with `ChainDB`,
`BlockFetch.Decision`, `Forge.*`, `Net.*` and `Startup.DiffusionInit` raised to
`Info`, noisy ChainDB events rate limited and `Mempool` silenced on mainnet
(`Info` on the test networks).

### Logging (syslog)

A node can no longer write its own log file. Its tracing backends are
`Stdout MachineFormat`, `Stdout HumanFormatColoured`,
`Stdout HumanFormatUncoloured`, `EKGBackend`,
`PrometheusSimple [suffix|nosuffix] [host] <port>` and `Forwarder`; there is no
file backend and no rotation inside the node.

`config.json` therefore logs plain text to stdout
(`Stdout HumanFormatUncoloured`, no ANSI colour codes). The port's rc script
starts the node with `daemon -S -T cardano-node`, so every line goes to syslog
as `daemon.notice` with the tag `cardano-node`. With the stock
`/etc/syslog.conf` it lands in `/var/log/messages`:

```sh
grep cardano-node /var/log/messages
tail -f /var/log/messages | grep cardano-node
```

The node logs far more than the old artifacts sent to syslog (they used the
`FileSK` scribe), and the stock newsyslog rule rotates `/var/log/messages` at
1 MB with 5 archives. A dedicated, larger log file is recommended; it needs no
change to the rc script:

```
# /usr/local/etc/syslog.d/cardano-node.conf
!cardano-node
*.*						/var/log/cardano-node.log
!*
```

```
# /usr/local/etc/newsyslog.conf.d/cardano-node.conf
# logfilename                 mode count size(KB) when flags
/var/log/cardano-node.log     640  10    51200    *    JC
```

```sh
touch /var/log/cardano-node.log && chmod 640 /var/log/cardano-node.log
service syslogd reload
```

This keeps the old retention of 50 MB x 10 files. The lines still reach
`/var/log/messages` too; to keep them out, add `!-cardano-node` on its own line
at the top of `/etc/syslog.conf`, before the `/dev/console` and
`/var/log/messages` rules (the existing `!-devd` line further down resets the
filter).

Notes:

* daemon(8) logs every line at priority `notice`, so syslog cannot filter by
  the node's own severity; use `severity` in `TraceOptions` for that.
* daemon(8) splits lines longer than 4095 bytes.
* For JSON logs, use `Stdout MachineFormat` instead.
* Alternative without syslog: `config-wtracer.json` + `tracer-config.json`
  with `cardano-tracer`, which the port builds but does not start. That needs
  a second process in the rc script.

### rc script fetch list

`service cardano_node fetch` downloads a fixed list of files. In Genesis mode
the node refuses to start without `peer-snapshot.json`, and `config.json`
references `checkpoints.json`, so the port's rc script needs both added:

```sh
cardano_config_files="config byron-genesis shelley-genesis alonzo-genesis conway-genesis topology submit-api-config peer-snapshot checkpoints"
```

The rc script fetches from the `master` branch of this repository.

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
