> Migrated from `docs/superpowers/specs/2026-06-11-sdkwork-connect-device-connect-design.md` on 2026-06-24.
> Owner: SDKWork maintainers

Status: proposed
Date: 2026-06-11
Repository: `sdkwork-connect`
Root: `<workspace-root>/sdkwork-connect`
Domain: `device`
Capability: `connect`

## Context

SDKWork needs a reusable platform-level device interconnect capability that can be integrated by multiple desktop applications, starting with SDKWork Drive PC and Craw Chat PC. The goal is broader than file transfer: applications need a common way to discover trusted devices, pair devices, open peer streams, exchange app-scoped requests, transfer files or objects, report progress, and later use relay or NAT traversal without duplicating P2P runtime code in every Tauri desktop package.

The initial SDKWork Drive inspection showed that its current transfer feature uses Drive App SDK upload sessions and backend download grants, not `libp2p`. Craw Chat has its own Tauri desktop package and native command surface. Both app roots should consume a shared capability rather than embedding app-specific P2P implementations.

SDKWork standards constrain the design:

- Shared modules must expose public contracts and adapters instead of app-local globals.
- Desktop native host code must expose local capabilities only and must not own product business authorization.
- Top-level shared packages must not contain application pages, routes, shells, or product business UI.
- Cross-repository dependencies must use native build-tool workspaces and release dependency refs, not copied source.

## Decision

Create `sdkwork-connect` as an independent SDKWork repository for platform-level device interconnect and P2P runtime capabilities.

Use:

- Repository/product name: `sdkwork-connect`
- User-facing capability name: SDKWork Connect
- Canonical domain: `device`
- Primary capability: `connect`

SDKWork Connect owns device-level P2P and local runtime capabilities. It does not own Drive files, Craw Chat messages, app business permissions, or product data persistence. Consuming applications map SDKWork Connect events and payloads into their own product workflows through their generated SDKs and business services.

## Non-Goals

The first design does not make SDKWork Connect a Drive replacement, an IM backend, a global message bus, or an application shell.

It must not:

- Put Drive or Craw Chat pages under top-level `packages/`.
- Let a consuming app deep-import Rust or TypeScript internals.
- Bypass Drive App SDK upload/download lifecycle for Drive-owned storage.
- Bypass Craw Chat IM SDK/message authorization for chat-owned messages.
- Treat `libp2p` peer identity as SDKWork user authentication.
- Store P2P private keys, runtime state, logs, or user files in source-controlled `.sdkwork/`.

## Repository Layout

Target layout:

```text
sdkwork-connect/
  AGENTS.md
  CODEX.md
  CLAUDE.md
  GEMINI.md
  .sdkwork/
  specs/
  docs/

  crates/
    sdkwork-connect-core/
    sdkwork-connect-libp2p/
    sdkwork-connect-runtime/
    sdkwork-connect-ipc/
    sdkwork-connect-tauri/

  services/
    sdkwork-connect-daemon/

  packages/
    sdkwork-connect-contracts/
    sdkwork-connect-client/
    sdkwork-connect-tauri-client/

  apps/
    sdkwork-connect-pc/

  sdks/
    sdkwork-connect-app-sdk/
    sdkwork-connect-backend-sdk/

  tools/
  scripts/
  package.json
  pnpm-workspace.yaml
  Cargo.toml
```

Only the dictionary, specs, and this design document are created before implementation planning. Language manifests, crates, packages, services, apps, and SDK families are introduced by later implementation plans.

## Directory Ownership

`crates/` owns Rust libraries:

- `sdkwork-connect-core`: public Rust domain types, protocol versions, config, state models, errors, and traits.
- `sdkwork-connect-libp2p`: `libp2p` transport/runtime implementation, peer discovery, connection management, relay support, app protocol routing, and stream handling.
- `sdkwork-connect-runtime`: orchestration of identity, trusted peer store, transfer queue, channel registry, policy checks, and event emission.
- `sdkwork-connect-ipc`: local IPC protocol for apps to communicate with the local service.
- `sdkwork-connect-tauri`: Tauri command registration, event bridge helpers, permissions/capability templates, and embedded-mode integration.

`services/` owns runnable local processes:

- `sdkwork-connect-daemon`: installed per-user local service or sidecar that runs one SDKWork Connect node per OS user, owns the local PeerId/private key, and exposes local IPC to consuming apps.

`packages/` owns application-neutral TypeScript libraries only:

- `sdkwork-connect-contracts`: public TypeScript contracts for device identity, peers, pairing, app channels, streams, transfers, events, and errors.
- `sdkwork-connect-client`: runtime-neutral TypeScript client facade over the SDKWork Connect local API. It can be backed by IPC, localhost loopback, Tauri bridge, or test fakes.
- `sdkwork-connect-tauri-client`: Tauri renderer adapter that calls SDKWork Connect Tauri commands or sidecar bridge. It must not include app pages, routes, or product-specific copy.

`apps/` owns runnable applications:

- `apps/sdkwork-connect-pc` is optional and reserved for a future SDKWork Connect management app. It may include devices, pairing, diagnostics, transfer queue, and local service configuration screens. Consuming apps must not depend on its internals.

`sdks/` owns generated SDK families only when cloud control-plane APIs exist:

- `sdkwork-connect-app-sdk`: app-side SDK for device registry, pairing tickets, bootstrap/rendezvous config, and relay allocation.
- `sdkwork-connect-backend-sdk`: backend/admin SDK for device policy, fleet diagnostics, relay operations, and audit review.

## Runtime Architecture

The long-term runtime model is one local SDKWork Connect node per OS user.

```text
SDKWork Connect Local Daemon
  -> Peer identity and private key
  -> libp2p swarm and connection pool
  -> trusted peer store
  -> pairing state
  -> app channel registry
  -> transfer queue
  -> relay/bootstrap/rendezvous config
  -> local IPC API

Drive PC / Craw Chat PC / future apps
  -> sdkwork-connect-client or sdkwork-connect-tauri-client
  -> app-owned service bridge
  -> product SDK and business workflow
```

During early implementation, apps may use embedded mode through `sdkwork-connect-tauri` to validate protocols. The public contracts must still be daemon-compatible so the runtime can migrate to the single local service without changing Drive or Craw Chat feature code.

## Capability Model

SDKWork Connect exposes platform capabilities, not product workflows:

- `device.identity`: local device identity, PeerId, device metadata, and runtime status.
- `device.pairing`: trusted device pairing through QR, code, or account-assisted confirmation.
- `device.discovery`: LAN discovery, account device directory, rendezvous, bootstrap, and relay hints.
- `device.presence`: peer online/offline and capability presence.
- `peer.connection`: connection status, transport, latency, and reconnect state.
- `peer.capabilities`: protocol and app channel negotiation.
- `stream.open`: app-scoped bidirectional streams.
- `stream.requestResponse`: request/response messages over trusted peers.
- `stream.pubsub`: optional app-scoped topic support when approved.
- `object.transfer`: generic object transfer by manifest and chunks.
- `file.transfer`: file and folder transfer built on the object transfer layer.
- `app.channel`: product-owned protocol namespace registration.
- `diagnostics`: safe local diagnostics and support bundle data.

Product capabilities remain outside SDKWork Connect:

- Drive owns Drive nodes, spaces, upload sessions, download grants, object storage providers, quota, retention, and audit.
- Craw Chat owns conversations, messages, contacts, attachments, IM device semantics, and message delivery rules.

## Protocol Namespace

SDKWork Connect reserves platform protocol namespaces:

```text
/sdkwork/connect/identity/1.0.0
/sdkwork/connect/pairing/1.0.0
/sdkwork/connect/capabilities/1.0.0
/sdkwork/connect/stream/1.0.0
/sdkwork/connect/object-transfer/1.0.0
/sdkwork/connect/file-transfer/1.0.0
```

Consuming applications register app channel namespaces:

```text
/sdkwork/drive/direct-transfer/1.0.0
/sdkwork/chat/nearby-message/1.0.0
/sdkwork/chat/attachment-transfer/1.0.0
/sdkwork/<domain>/<capability>/<semver>
```

SDKWork Connect routes bytes and events. The app that registers a channel owns payload schema validation and product workflow mapping.

## Identity And Trust

SDKWork Connect has three identity layers:

1. Peer identity: `libp2p` keypair and PeerId, generated and stored locally.
2. Device identity: SDKWork Connect device ID, device name, OS, app inventory, and runtime metadata.
3. Account context: optional SDKWork user, tenant, and organization binding.

Rules:

- Peer private keys never leave the user-private runtime directory.
- PeerId is not a login credential and does not authorize product workflows.
- Account binding proves that a peer belongs to a signed-in SDKWork user context, but app authorization remains app/API-owned.
- Pairing creates a trusted peer relation with scoped capabilities and expiration/rotation policy.
- Cross-account pairing requires an explicit policy decision before implementation.

## Local Runtime State

Runtime state belongs in user-private SDKWork runtime directories, not source:

- Peer private key.
- Device registration cache.
- Trusted peer store.
- Transfer queue metadata.
- Temporary incoming chunks.
- Logs and diagnostics.
- Relay/bootstrap cache.

The source repository `.sdkwork/` stores only development metadata.

## TypeScript Contract Shape

`sdkwork-connect-contracts` exposes stable application-neutral types:

```ts
export interface DeviceConnectNodeStatus {
  nodeId: string;
  peerId: string;
  state: 'stopped' | 'starting' | 'online' | 'degraded' | 'offline';
  transports: string[];
  relayMode: 'disabled' | 'available' | 'active';
}

export interface AppChannelRegistration {
  appId: string;
  channel: string;
  protocols: string[];
  capabilities: string[];
}

export interface OfferTransferRequest {
  targetPeerId: string;
  channel: string;
  manifest: TransferManifest;
}
```

The package must not import React, Tauri, Drive SDKs, Craw Chat SDKs, or generated app SDKs.

## Client Contract Shape

`sdkwork-connect-client` exposes a stable facade:

```ts
export interface SdkworkConnectClient {
  getNodeStatus(): Promise<DeviceConnectNodeStatus>;
  listPeers(filter?: PeerFilter): Promise<DevicePeer[]>;
  pairDevice(request: PairDeviceRequest): Promise<PairDeviceResult>;
  trustPeer(request: TrustPeerRequest): Promise<void>;
  registerAppChannel(channel: AppChannelRegistration): Promise<void>;
  openStream(request: OpenPeerStreamRequest): Promise<PeerStreamHandle>;
  sendRequest<TResponse>(request: PeerRequest): Promise<TResponse>;
  offerTransfer(request: OfferTransferRequest): Promise<TransferHandle>;
  acceptTransfer(request: AcceptTransferRequest): Promise<void>;
  cancelTransfer(transferId: string): Promise<void>;
  subscribe(listener: DeviceConnectEventListener): Unsubscribe;
}
```

Concrete transports are adapters. Application code consumes this interface.

## Event Model

SDKWork Connect emits structured events:

```text
sdkwork.connect.node.status.changed
sdkwork.connect.peer.discovered
sdkwork.connect.peer.connected
sdkwork.connect.peer.disconnected
sdkwork.connect.channel.requested
sdkwork.connect.transfer.offered
sdkwork.connect.transfer.progress
sdkwork.connect.transfer.completed
sdkwork.connect.transfer.failed
```

Events must avoid secrets, raw private file contents, auth tokens, refresh tokens, and sensitive payload logs.

## Drive Integration

SDKWork Drive PC consumes SDKWork Connect for direct device transfer and optional P2P acceleration.

Drive may register:

```text
/sdkwork/drive/direct-transfer/1.0.0
```

Drive owns:

- Selecting Drive nodes or local files.
- Mapping received local files into Drive upload workflows.
- Calling Drive App SDK upload sessions when content must enter Drive storage.
- Preserving Drive quota, retention, audit, permission, and node lifecycle.

SDKWork Connect owns:

- Peer connection.
- Transfer offer.
- Chunk transfer.
- Progress/cancel.
- Local file receipt and hash validation.

Drive must not treat P2P receipt as a Drive node until Drive App SDK completes the product-owned upload/import workflow.

## Craw Chat Integration

Craw Chat PC consumes SDKWork Connect for nearby devices, attachment transfer, and future app-scoped streams.

Craw Chat may register:

```text
/sdkwork/chat/nearby-message/1.0.0
/sdkwork/chat/attachment-transfer/1.0.0
```

Craw Chat owns:

- Conversation selection and message authorization.
- Attachment message creation.
- IM/communication SDK integration.
- Message persistence and delivery semantics.

SDKWork Connect owns:

- Peer discovery.
- App-scoped stream.
- Attachment byte transfer.
- Progress/cancel.
- Receipt events.

Craw Chat must not bypass its IM/message authorization model by treating P2P payloads as trusted messages without product-level validation.

## Security Model

Security requirements:

- Private keys are generated locally and stored in user-private runtime directories.
- All peer connections use authenticated encrypted transports.
- App channel registration is explicit and scoped by app ID.
- A consuming app can only open channels and capabilities granted by local policy.
- Incoming transfer offers require trust and user/app policy approval.
- File/object manifests include size, hash, content type, filename, channel, sender peer, expiration, and optional account context.
- Payload validation belongs to the receiving app.
- Logs and diagnostics must not contain tokens, private keys, file contents, or raw message payloads.
- Relay usage and cross-account sharing require explicit policy and review before production enablement.

## Cloud Control Plane

SDKWork Connect can start with LAN/manual pairing. A cloud control plane can be added later for:

- Device registry.
- Account-bound device directory.
- Pairing tickets.
- Bootstrap/rendezvous config.
- Relay allocation.
- Enterprise policy.
- Admin diagnostics and audit metadata.

Cloud APIs, when introduced, require SDK families under `sdks/` and must follow SDKWork API/SDK generation standards. The local daemon must continue to support local-only LAN discovery where policy allows.

## Phased Implementation

Phase 0: repository design and dictionary

- Establish `sdkwork-connect` root dictionary.
- Record this design.
- Do not create implementation crates/packages until the design is reviewed.

Phase 1: embedded runtime MVP

- Create `sdkwork-connect-core`, `sdkwork-connect-libp2p`, and `sdkwork-connect-tauri`.
- Create `sdkwork-connect-contracts`, `sdkwork-connect-client`, and `sdkwork-connect-tauri-client`.
- Support local node identity, LAN discovery, manual pairing, request/response, file/object transfer, progress, cancel, and Tauri event bridge.
- Integrate one consumer first behind a feature flag.

Phase 2: local daemon

- Create `sdkwork-connect-daemon`.
- Move to one local node per OS user.
- Add IPC, trusted peer store, background transfer queue, diagnostics, and app channel policy.
- Migrate consumers from embedded mode to daemon client without changing app business flows.

Phase 3: cloud control plane

- Add device registry, pairing tickets, relay/bootstrap config, and policy APIs.
- Generate app/backend SDK families.
- Add relay/hole punching production support with operator diagnostics.

Phase 4: multi-application platform

- Drive, Craw Chat, and future apps register app-scoped channels.
- Add stronger cross-app transfer queue and user-facing SDKWork Connect PC management app if needed.

## Verification Strategy

Initial design verification:

- Read `AGENTS.md`.
- Parse `specs/component.spec.json`.
- Review this design for package/app boundary violations.

Future implementation verification:

- Rust unit tests for protocol models and transfer state machines.
- Rust integration tests for libp2p request/response and local discovery.
- Tauri command permission tests.
- TypeScript typecheck and client fake tests.
- Static scans proving top-level `packages/` has no app pages, routes, shell, or product SDK imports.
- Consumer integration tests in Drive and Craw Chat proving they use public `sdkwork-connect-*` packages or daemon IPC, not internal source paths.
- Security tests for secret-free logs, local key path hygiene, and transfer manifest validation.

## Open Decisions

- Whether the first consumer should be SDKWork Drive PC or Craw Chat PC.
- Whether phase 1 must include relay/hole punching or only LAN/manual pairing.
- Whether `apps/sdkwork-connect-pc` is needed in phase 1 or should wait until daemon diagnostics exist.
- Which IPC transport should be primary on Windows, macOS, and Linux.
- Whether cross-account transfer is in scope for phase 1.
- Whether cloud control-plane APIs belong in this repo or a separate service repo once introduced.

## Acceptance Criteria

- SDKWork Connect is an independent repository at `<workspace-root>/sdkwork-connect`.
- The canonical domain is `device` and capability is `connect`.
- Top-level `packages/` is restricted to application-neutral contracts, clients, and adapters.
- SDKWork Connect PC application content, if added, is under `apps/sdkwork-connect-pc`.
- Drive and Craw Chat consume public SDKWork Connect contracts/adapters or the local daemon API.
- P2P runtime never replaces product SDK/API authorization.
- The design can evolve from embedded Tauri runtime to one local daemon without changing consumer app business logic.

