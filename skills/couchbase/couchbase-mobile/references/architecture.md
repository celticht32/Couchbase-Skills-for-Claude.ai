# Mobile architecture

## Component overview

```
Mobile / Edge Device
  └── Couchbase Lite (embedded database)
        └── Replicator (sync protocol)
              ↕ WebSocket (blip protocol)
        Sync Gateway / Capella App Services
              ↕ REST / internal
        Couchbase Server / Capella Cluster
              └── Buckets / Scopes / Collections
```

**Couchbase Lite** — the embedded NoSQL database that runs inside your mobile or edge app. Stores data locally, enables offline operation, runs queries, and manages replication to/from Sync Gateway.

**Sync Gateway** — the middle tier. Handles authentication, channel-based access control, conflict resolution, and the WebSocket replication protocol. Does not store data permanently — it's a gateway into Couchbase Server.

**Capella App Services** — fully managed Sync Gateway hosted by Couchbase on Capella.

## When you need each component

**Couchbase Lite alone (no sync):** offline-only app with no server synchronization. Local storage and queries only. Use as an embedded database for settings, cached data, or truly offline scenarios.

**Couchbase Lite + Sync Gateway / App Services:** the standard mobile stack. Devices sync with the server. Multiple devices share data. Works offline; syncs when connected.

**Sync Gateway without Couchbase Lite:** some IoT or edge scenarios use the REST API directly on devices that can't embed Couchbase Lite. Less common.

## Replication modes

**Push:** device sends local changes to the server. Use for: data capture devices, sensor uploads, form submissions.

**Pull:** device receives changes from the server. Use for: read-only displays, configuration distribution, content delivery.

**Push + Pull (bidirectional):** device both sends and receives. Use for: collaborative apps, field worker apps, anything where the device both reads and writes shared data.

```swift
// Swift — configure replicator
let config = ReplicatorConfiguration(target: URLEndpoint(url: appServiceURL))
config.replicatorType = .pushAndPull     // or .push / .pull
config.continuous = true                 // continuous: stay connected and sync in real time
                                         // false: one-shot sync then disconnect

let replicator = Replicator(config: config)
replicator.start()
```

## Continuous vs one-shot replication

**Continuous:** maintains a persistent WebSocket connection. Syncs changes as they happen. Appropriate for: real-time collaborative apps, field workers who need live updates.

**One-shot:** connects, syncs all pending changes, disconnects. Appropriate for: periodic sync on a schedule, battery-sensitive devices, background sync tasks.

For most mobile apps, use continuous replication while the app is in the foreground and one-shot during background refresh.

## Authentication

**Basic auth (username + password):**
Most straightforward. Credentials are managed in Sync Gateway's user database or via OIDC.

```swift
config.authenticator = BasicAuthenticator(username: "alice", password: "password")
```

**Session auth (recommended for production):**
App authenticates against your backend, backend creates a Sync Gateway session, app uses the session token. Avoids storing Couchbase credentials on the device.

```swift
config.authenticator = SessionAuthenticator(sessionID: sessionToken)
```

**OpenID Connect (OIDC):**
Sync Gateway acts as an OIDC relying party. App authenticates via your identity provider (Google, Auth0, Okta, etc.) and Sync Gateway validates the JWT. Best for apps with existing SSO infrastructure.

## Collection-aware sync (App Services / Sync Gateway 3.1+)

Older Sync Gateway synced at bucket level. Modern versions sync at collection level:

```swift
// Configure which collections to sync
let collection = try database.defaultCollection()
let collectionConfig = CollectionConfiguration()
collectionConfig.channels = ["user.\(userId)", "public"]

config.addCollection(collection, config: collectionConfig)
```

On the server side, each collection has its own sync function and channel namespace. This enables fine-grained sync partitioning: one collection for user-specific data, another for shared read-only content.

## Peer-to-peer sync

Couchbase Lite supports direct device-to-device sync without a server (using a local WebSocket listener). Use for: offline field teams, device-to-device transfer, local network sync without internet.

```swift
// Device acting as listener (passive side)
let listenerConfig = URLEndpointListenerConfiguration(collections: [collection])
listenerConfig.port = 4984
let listener = URLEndpointListener(config: listenerConfig)
try listener.start()

// Device acting as replicator (active side) — same ReplicatorConfiguration
// but pointing to the listener's URL instead of Sync Gateway
```

P2P sync uses the same conflict resolution and document routing as server-based sync.
