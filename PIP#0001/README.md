# PIP-0001: Paqus Gateway Peer Discovery

## Status

Draft

## Summary

Introduce `paqus-gateway` as a lightweight bootstrap and peer-discovery service. Nodes register their public P2P address with the gateway, refresh it through periodic heartbeats, and query the gateway for compatible active peers. The gateway does not participate in consensus, block validation, fork choice, mining, or transaction execution.

## Motivation

New Paqus nodes currently need peers from local configuration or from peers they already know. A gateway gives new nodes one stable entry point for finding live peers while keeping blockchain traffic peer-to-peer after discovery.

The first version should solve three problems:

- New nodes can find active peers without a manually maintained peers file.
- Operators can see which nodes are alive on a network.
- Wallets, explorers, and future services can reuse the same gateway as a public entry point later.

## Non-Goals

- The gateway is not a full node.
- The gateway does not validate blocks or transactions beyond lightweight request checks.
- The gateway does not proxy normal block sync traffic.
- The gateway does not decide canonical chain state.
- The gateway does not replace node-to-node P2P.

## Architecture

```text
paqus-node  -->  paqus-gateway  -->  active peer list
     |                 ^
     |                 |
     +-- heartbeat ----+

paqus-node  <-------- direct P2P -------->  paqus-node
```

The node uses the gateway only for discovery:

1. Node starts with a gateway URL.
2. Node registers its public listen address and local network metadata.
3. Gateway stores the peer with an expiration time.
4. Node periodically sends a heartbeat.
5. Node asks the gateway for compatible peers.
6. Node connects directly to returned peers using the existing P2P protocol.

## Gateway Responsibilities

- Keep an in-memory registry of active peers.
- Expire peers that stop sending heartbeats.
- Return a randomized subset of compatible peers.
- Filter peers by network identity and protocol compatibility.
- Provide basic health and operator visibility endpoints.
- Optionally persist peer snapshots later.

## Node Responsibilities

- Know its public P2P listen address.
- Register with the configured gateway after startup.
- Send periodic heartbeat messages.
- Query the gateway when its peer set is low or at startup.
- Continue using direct P2P for handshake, sync, gossip, and transaction propagation.

## Network Identity

Gateway peer registration should include the same compatibility fields used by the node P2P handshake:

- `chain_id`
- `chain_name`
- `protocol_stage`
- `protocol_version`
- `network_magic`

The gateway should only return peers that match the requesting node's network identity. Protocol version should be exact-match for the first version, matching current node behavior.

## HTTP API

All endpoints are versioned under `/v1`.

### `GET /v1/health`

Returns service health.

Response:

```json
{
  "ok": true,
  "service": "paqus-gateway"
}
```

### `POST /v1/node/register`

Registers or refreshes a peer.

Request:

```json
{
  "peer_id": "optional-stable-node-id",
  "address": "203.0.113.10:30333",
  "chain_id": 1,
  "chain_name": "paqus",
  "protocol_stage": "experimental",
  "protocol_version": 1,
  "network_magic": "58505101",
  "best_height": 120,
  "tip_hash": "optional-tip-hash-hex"
}
```

Response:

```json
{
  "accepted": true,
  "expires_in_secs": 180
}
```

### `POST /v1/node/heartbeat`

Refreshes an existing peer and updates its height metadata.

Request:

```json
{
  "peer_id": "optional-stable-node-id",
  "address": "203.0.113.10:30333",
  "best_height": 125,
  "tip_hash": "optional-tip-hash-hex"
}
```

Response:

```json
{
  "accepted": true,
  "expires_in_secs": 180
}
```

### `GET /v1/peers`

Returns compatible active peers.

Query parameters:

- `chain_id`
- `chain_name`
- `protocol_stage`
- `protocol_version`
- `network_magic`
- `limit`, default `32`, max `128`
- `exclude`, optional address or peer id

Response:

```json
{
  "peers": [
    {
      "peer_id": "optional-stable-node-id",
      "address": "203.0.113.10:30333",
      "best_height": 125,
      "last_seen_unix": 1782490000
    }
  ]
}
```

### `GET /v1/peers/stats`

Returns operator-facing registry stats.

Response:

```json
{
  "active_peers": 12,
  "expired_peers": 3,
  "networks": [
    {
      "chain_id": 1,
      "chain_name": "paqus",
      "protocol_stage": "experimental",
      "protocol_version": 1,
      "count": 12
    }
  ]
}
```

## Peer Expiration

Initial policy:

- Peer TTL: 180 seconds.
- Heartbeat interval: 60 seconds.
- Expired peers are not returned.
- Cleanup can run lazily on register/query or on a timer.

## Abuse Controls

Minimum first-version controls:

- Limit peers per source IP.
- Limit max active peers per advertised address.
- Reject invalid socket addresses.
- Reject private or loopback addresses unless gateway is started with `--allow-private-peers`.
- Randomize peer response order.
- Cap `limit`.

Future controls:

- Signed registrations using a persistent node key.
- Peer reputation from failed connection reports.
- Proof-of-reachability checks from gateway to node.
- Bans or temporary quarantine for suspicious peers.

## Crate Layout

```text
paqus-gateway/
  Cargo.toml
  src/
    main.rs
    config.rs
    error.rs
    model.rs
    registry.rs
    routes.rs
```

Suggested dependencies:

- `axum`
- `tokio`
- `serde`
- `serde_json`
- `thiserror`
- `tracing`
- `tracing-subscriber`
- `tower-http`

## Node Integration

Add node CLI options:

```text
--gateway <url>
--public-addr <host:port>
--gateway-heartbeat-secs <secs>
```

Startup behavior:

1. If `--gateway` and `--public-addr` are set, register with the gateway.
2. Query `/v1/peers` and merge returned addresses into the existing peer list.
3. Keep existing `GetPeers` P2P behavior as a second discovery layer.
4. Send heartbeat in the node loop.

If the gateway is unreachable, the node should continue running with local peers.

## Open Questions

- Should `peer_id` exist in the first version, or should address be the registry key?
- Should gateway registration require proof that the advertised address is reachable?
- Should gateway and public RPC live in one crate from day one, or should peer discovery ship first?
- Should private addresses be allowed by default for local testnets?

## Rollout Plan

1. Create `paqus-gateway` with in-memory peer registry and `/v1` endpoints.
2. Add node CLI flags for gateway URL and public address.
3. Add node register/query/heartbeat client.
4. Keep static peers file support as fallback.
5. Add tests for peer expiration, compatibility filtering, and query limits.
