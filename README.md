# Ethereum infrastructure

Docker Compose stack to run an Ethereum node (execution + consensus) alongside an observability stack.

## Services

### `jwt-init`
One-shot bootstrap container. Generates the shared Engine API JWT secret at `./data/${NETWORK}/ipc/jwt-secret` if it does not already exist, then exits. The execution client and `beacon` both `depends_on` it (`service_completed_successfully`), so the secret is guaranteed to be present before they start. This is client-agnostic: both Nethermind and Geth would create the secret themselves if missing, but generating it up front guarantees the file is present before either starts.

### `nethermind` / `geth` (execution client)
Execution layer client. Processes transactions, executes the EVM, and exposes the JSON-RPC and Engine API used by the consensus client. The stack ships **two** alternative execution clients — Nethermind and Geth — exactly one runs at a time, selected via a Compose profile (see [Choosing the execution client](#choosing-the-execution-client)). Whichever is active is reachable under the shared network alias `execution`, so the rest of the stack does not care which one runs.

### `beacon`
Consensus layer client (Prysm). Drives consensus, follows the beacon chain, and instructs the execution client what to build/validate via the Engine API (`http://execution:8551`). Authenticates with a shared JWT secret mounted from `./data/${NETWORK}/ipc`.

### `alloy`
Grafana Alloy agent. Tails the beacon node logs and ships them to `loki`.

### `loki`
Log aggregation backend. Stores logs forwarded by `alloy` so they can be queried from Grafana.

### `prometheus`
Metrics database. Scrapes metrics endpoints (Nethermind, beacon monitoring, etc.) defined in `configuration/prometheus.yml`.

### `pyroscope`
Continuous profiling backend. Receives pprof profiles (the beacon node exposes pprof on `:6060`).

### `tempo` (optional, disabled by default)
Distributed tracing backend. Receives OTLP traces from the beacon node (`--tracing-endpoint=http://tempo:4318/v1/traces`). Disabled by default: it only starts when the `tempo` Compose profile is active, and the beacon node only emits traces when `ENABLE_TRACING=true`. To enable it, set both in `.env`:

```
COMPOSE_PROFILES=nethermind,tempo
ENABLE_TRACING=true
```

The Grafana `Tempo` datasource is always provisioned; it simply returns errors while Tempo is not running.

### `peer-geo-exporter`
Small Go service to a geographic location using a local DB-IP Lite City database, and exposes the `beacon_peer_geo` Prometheus gauge that feeds the world-map panel on the beacon dashboard.

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

All other ports (Nethermind JSON-RPC `8545` and Engine API `8551`; beacon REST `3500`, gRPC `4000`, monitoring `8080`, pprof `6060`; Loki `3100`; Prometheus `9090`; Pyroscope `4040`; Tempo `3200`/`4318` when enabled) are bound to `127.0.0.1` and only reachable from the host itself — typically via SSH port-forwarding.

## `.env` file

`docker-compose.yaml` interpolates the following variables, which must be defined in a `.env` file next to it:

| Variable | Description |
|----------|-------------|
| `NETWORK` | Ethereum network name (e.g. `mainnet`, `hoodi`). |
| `COMPOSE_PROFILES` | Comma-separated list of Compose profiles. Must contain exactly one execution client (`nethermind` or `geth`); only the matching service is started. Add `tempo` to also start the Tempo tracing backend (e.g. `nethermind,tempo`). |
| `NETHERMIND_IMAGE` | Docker image (with tag) for the `nethermind` service, e.g. `nethermind/nethermind:1.37.2`. Pinned here so upgrades are explicit. Used only when `COMPOSE_PROFILES=nethermind`. |
| `GETH_IMAGE` | Docker image (with tag) for the `geth` service, e.g. `ethereum/client-go:v1.16.1`. Pinned here so upgrades are explicit. Used only when `COMPOSE_PROFILES=geth`. |
| `BEACON_IMAGE` | Docker image (with tag) to use for the `beacon` service, e.g. `gcr.io/prysmaticlabs/prysm/beacon-chain:v6.0.4`. |
| `P2P_HOST_IP` | Public IP address of the host, advertised by the beacon node to peers (`--p2p-host-ip`). Required so inbound libp2p connections from the rest of the network can reach this node. |
| `HTTP_MEV_RELAY` | MEV relay URL passed to the beacon node (`--http-mev-relay`), e.g. `http://mev-boost:18550`. Leave empty (`HTTP_MEV_RELAY=`) to disable MEV-boost and build blocks locally. |
| `SUGGESTED_FEE_RECIPIENT` | Execution-layer address passed to the beacon node (`--suggested-fee-recipient`) that receives priority fees and MEV payments from proposed blocks. Leave empty (`SUGGESTED_FEE_RECIPIENT=`) to start without one: Prysm logs a warning and falls back to the burn address (validators can still set their own via `prepareBeaconProposer`). |
| `ENABLE_TRACING` | Optional, defaults to `false`. Set to `true` to make the beacon node export traces to Tempo (`--enable-tracing`). Only useful together with the `tempo` profile in `COMPOSE_PROFILES`. |
| `ENABLE_BACKFILL` | Optional, defaults to `false`. Set to `true` to make the beacon node backfill historical blocks before its checkpoint-sync origin (`--enable-backfill`). |


## Links
- [http://<P2P_HOST_IP>:3000/d/adnmforf/beacon-node](http://<P2P_HOST_IP>:3000/d/adnmforf/beacon-node) - The beacon node Grafana Dashboard
