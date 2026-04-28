# Ethereum infrastructure

Docker Compose stack to run an Ethereum node (execution + consensus) alongside an observability stack.

## Services

### `nethermind`
Execution layer client. Processes transactions, executes the EVM, and exposes the JSON-RPC and Engine API used by the consensus client.

### `beacon`
Consensus layer client (Prysm). Drives consensus, follows the beacon chain, and instructs `nethermind` what to build/validate via the Engine API. Authenticates to `nethermind` with a shared JWT secret mounted from `./data/${NETWORK}/ipc`.

### `alloy`
Grafana Alloy agent. Tails the beacon node logs and ships them to `loki`.

### `loki`
Log aggregation backend. Stores logs forwarded by `alloy` so they can be queried from Grafana.

### `prometheus`
Metrics database. Scrapes metrics endpoints (Nethermind, beacon monitoring, etc.) defined in `configuration/prometheus.yml`.

### `pyroscope`
Continuous profiling backend. Receives pprof profiles (the beacon node exposes pprof on `:6060`).

### `grafana`
Dashboards and UI for the data stored in Loki, Prometheus, and Pyroscope. Datasources and dashboards are provisioned from `configuration/grafana/`.

## Ports exposed to the world

Only ports bound to `0.0.0.0` (the default when no host IP is given) are reachable from outside the host. Everything bound to `127.0.0.1` is local-only.

| Port | Proto | Service | Reason |
|------|-------|---------|--------|
| 30303 | TCP/UDP | nethermind | Execution layer P2P (devp2p). Required for peer discovery and block/tx gossip. |
| 12000 | UDP | beacon | Consensus layer discv5 discovery. Required to find beacon peers. |
| 13000 | TCP+UDP | beacon | Consensus layer libp2p. Required for block/attestation gossip and sync. |
| 3000 | TCP | grafana | Web UI. Exposed so the dashboards can be reached from a browser. |

All other ports (Nethermind JSON-RPC `8545` and Engine API `8551`; beacon REST `3500`, gRPC `4000`, monitoring `8080`, pprof `6060`; Loki `3100`; Prometheus `9090`; Pyroscope `4040`) are bound to `127.0.0.1` and only reachable from the host itself — typically via SSH port-forwarding.

## `.env` file

`docker-compose.yaml` interpolates the following variables, which must be defined in a `.env` file next to it:

| Variable | Description |
|----------|-------------|
| `NETWORK` | Ethereum network name (e.g. `mainnet`, `sepolia`, `holesky`). Used as the Nethermind `--config` value, the beacon `--<network>` flag, and to namespace the on-disk data directory under `./data/${NETWORK}/`. |
| `NETHERMIND_IMAGE` | Docker image (with tag) to use for the `nethermind` service, e.g. `nethermind/nethermind:1.34.0`. Pinned here so upgrades are explicit. |
| `BEACON_IMAGE` | Docker image (with tag) to use for the `beacon` service, e.g. `gcr.io/prysmaticlabs/prysm/beacon-chain:v6.0.4`. |
| `CHECKPOINT_SYNC_URL` | URL of a trusted beacon checkpoint-sync provider. Lets the beacon node start from a recent finalized state instead of syncing from genesis. |
| `P2P_HOST_IP` | Public IP address of the host, advertised by the beacon node to peers (`--p2p-host-ip`). Required so inbound libp2p connections from the rest of the network can reach this node. |

## Links
- [http://<P2P_HOST_IP>:3000/d/adnmforf/beacon-node](http://<P2P_HOST_IP>:3000/d/adnmforf/beacon-node) - The beacon node Grafana Dashboard
