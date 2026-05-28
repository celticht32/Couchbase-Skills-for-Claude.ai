# Couchbase Lite

## Opening a database

```swift
// Swift
let config = DatabaseConfiguration()
config.directory = getDocumentsDirectory().path
let database = try Database(name: "myapp", config: config)
let collection = try database.defaultCollection()
```

```kotlin
// Kotlin
val config = DatabaseConfigurationFactory.newConfig()
val database = Database("myapp", config)
val collection = database.defaultCollection
```

```python
# Python (Desktop / Edge)
from couchbase.lite import Database, DatabaseConfiguration
db = Database("myapp", DatabaseConfiguration())
collection = db.default_collection
```

## CRUD operations

```swift
// Create / Update
let doc = MutableDocument(id: "user::alice")
doc.setString("Alice Smith", forKey: "name")
doc.setString("alice@example.com", forKey: "email")
doc.setInt(30, forKey: "age")
try collection.save(document: doc)

// Read
if let result = try collection.document(id: "user::alice") {
    let name = result.string(forKey: "name")
}

// Update
if let existing = try collection.document(id: "user::alice")?.toMutable() {
    existing.setInt(31, forKey: "age")
    try collection.save(document: existing)
}

// Delete
if let doc = try collection.document(id: "user::alice") {
    try collection.delete(document: doc)
}
```

## Querying with SQL++

Couchbase Lite supports a SQL++ dialect for querying the local database:

```swift
let query = try database.createQuery(
    "SELECT name, email FROM _ WHERE type = 'user' AND age > 25 ORDER BY name"
)
let results = try query.execute()
for result in results {
    print(result.string(at: 0) ?? "", result.string(at: 1) ?? "")
}
```

```kotlin
val query = database.createQuery(
    "SELECT name, email FROM _ WHERE type = 'user' AND age > 25 ORDER BY name"
)
val results = query.execute()
```

The `_` refers to the default collection. For named collections: `SELECT * FROM myScope.myCollection WHERE ...`

## Live queries (change listeners)

```swift
let query = try database.createQuery("SELECT * FROM _ WHERE type = 'message' ORDER BY timestamp DESC LIMIT 50")
let token = query.addChangeListener { change in
    if let results = change.results {
        // Update UI with new results
        updateMessageList(results.allResults())
    }
}
// Remove listener when done
query.removeChangeListener(withToken: token)
```

Live queries re-execute automatically when underlying data changes. Use for reactive UI updates.

## Document change listeners

```swift
let token = collection.addDocumentChangeListener(id: "user::alice") { change in
    // Document was created, updated, or deleted
    print("Document changed:", change.documentID)
}
```

## Replicator configuration

```swift
let appServiceURL = URL(string: "wss://your-app-service.apps.cloud.couchbase.com/your-endpoint")!
var config = ReplicatorConfiguration(target: URLEndpoint(url: appServiceURL))

// Authentication
config.authenticator = BasicAuthenticator(username: "alice", password: "password")
// Or session auth (preferred):
// config.authenticator = SessionAuthenticator(sessionID: sessionToken)

// Replication type
config.replicatorType = .pushAndPull
config.continuous = true

// Filter — only sync certain documents
config.pushFilter = { (document, flags) -> Bool in
    return document.string(forKey: "type") != "draft"  // don't sync drafts
}

// Collection-level config
var collConfig = CollectionConfiguration()
collConfig.channels = ["user.alice", "public"]
config.addCollection(try database.defaultCollection(), config: collConfig)

let replicator = Replicator(config: config)

// Status listener
let token = replicator.addChangeListener { change in
    switch change.status.activity {
    case .connecting: print("Connecting...")
    case .idle: print("Idle — up to date")
    case .busy: print("Syncing...")
    case .offline: print("Offline")
    case .stopped:
        if let error = change.status.error {
            print("Stopped with error:", error)
        }
    }
}

replicator.start()
```

## Conflict resolution

Default: last-write-wins (highest revision wins). Custom resolver:

```swift
class MyConflictResolver: ConflictResolverProtocol {
    func resolve(conflict: Conflict) -> Document? {
        let local = conflict.localDocument
        let remote = conflict.remoteDocument

        // Return nil to delete the document
        // Return local to keep local version
        // Return remote to keep remote version
        // Return a merged MutableDocument for a custom merge

        // Example: keep the version with the higher score
        let localScore = local?.int(forKey: "score") ?? 0
        let remoteScore = remote?.int(forKey: "score") ?? 0
        return localScore >= remoteScore ? local : remote
    }
}

config.conflictResolver = MyConflictResolver()
```

## Vector search in Couchbase Lite (2.0+)

Couchbase Lite supports on-device vector search for semantic search without a server:

```swift
// Create a vector index
let config = VectorIndexConfiguration(expression: "embedding", dimensions: 128, centroids: 8)
try collection.createIndex(withName: "vector-idx", config: config)

// Query by vector similarity
let query = try database.createQuery(
    "SELECT id, title FROM _ WHERE VECTOR_MATCH(\"vector-idx\", $query_vector, 5)"
)
query.parameters = Parameters()
query.parameters?.setArray(MutableArrayObject(data: queryVector), forName: "query_vector")
```

On-device vector search is useful for: privacy-sensitive applications, fully offline semantic search, edge inference pipelines.

## Best practices

**Always close the database when done.** Especially on mobile where the app may be suspended. Use database lifecycle management tied to the app lifecycle (not individual screens).

**Use document IDs deliberately.** Design IDs like server Couchbase: `type::uuid`. This makes filtering by type via key prefix possible.

**Keep documents small.** Couchbase Lite loads documents into memory for operations. Very large documents (>100KB) slow down sync and query performance.

**Use collections.** Sync specific collections rather than the whole bucket. This reduces the amount of data synced to each device.

**Battery and network.** On mobile, use `continuous: false` for background sync and `continuous: true` only in the foreground. Listen for network reachability changes and pause the replicator when offline.
