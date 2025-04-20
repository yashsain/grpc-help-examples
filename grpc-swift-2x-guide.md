# gRPC Swift 2.x Implementation Guide

## Overview and Architecture

gRPC Swift 2.x represents a complete redesign that embraces Swift's modern concurrency model with async/await while providing a more type-safe API. The ecosystem consists of several key packages:

- **grpc-swift (2.1.2+)**: Core gRPC implementation with service abstractions
- **grpc-swift-nio-transport (1.0.3+)**: HTTP/2 transport implementation using SwiftNIO
- **grpc-swift-protobuf (1.2.0+)**: Integration for Protocol Buffers
- **swift-protobuf (1.29.0+)**: Swift Protocol Buffers implementation

## Package Configuration

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "YourApp",
    platforms: [.iOS(.v15), .macOS(.v15)],
    dependencies: [
        .package(url: "https://github.com/grpc/grpc-swift.git", from: "2.1.2"),
        .package(url: "https://github.com/grpc/grpc-swift-nio-transport.git", from: "1.0.3"),
        .package(url: "https://github.com/grpc/grpc-swift-protobuf.git", from: "1.2.0"),
        .package(url: "https://github.com/apple/swift-protobuf.git", from: "1.29.0")
    ],
    targets: [
        .target(
            name: "YourApp",
            dependencies: [
                .product(name: "GRPCCore", package: "grpc-swift"),
                .product(name: "GRPCNIOTransportHTTP2", package: "grpc-swift-nio-transport"),
                .product(name: "GRPCProtobuf", package: "grpc-swift-protobuf"),
                .product(name: "SwiftProtobuf", package: "swift-protobuf")
            ],
            plugins: [
                .plugin(name: "GRPCProtobufGenerator", package: "grpc-swift-protobuf")
            ]
        )
    ]
)
```

## Transport System

### Transport Types

The transport system in gRPC Swift 2.x uses protocols with associated types rather than concrete types:

```swift
// Creating HTTP/2 transport with factory pattern
let transport = try .http2NIOPosix(
    target: .dns(host: "example.com", port: 443),
    transportSecurity: .plaintext
)
```

### Target Configuration

```swift
// DNS resolution (replaces older .hostAndPort syntax)
target: .dns(host: "example.com", port: 443)

// Direct IPv4 address
target: .ipv4(host: "127.0.0.1", port: 50051)

// Unix domain socket
target: .unixDomainSocket(path: "/tmp/grpc.sock")
```

### TLS Configuration

TLS configuration requires strict parameter ordering:

```swift
// With TLS certificate validation
transportSecurity: .tls(.init(
    certificateChain: [],     // Empty for client-only auth
    privateKey: nil,          // Nil for client-only auth
    serverCertificateVerification: .fullVerification,
    trustRoots: .certificates([.file(path: certPath, format: .pem)])
))

// Plaintext (insecure) connection
transportSecurity: .plaintext
```

## Client Management

### Client Creation Pattern

The recommended pattern is using `withGRPCClient` which manages client lifecycle:

```swift
try await withGRPCClient(
    transport: .http2NIOPosix(
        target: .dns(host: host, port: port),
        transportSecurity: transportSecurity
    )
) { client in
    // Use client here
    // Connection closes automatically when this closure completes
}
```

### Service Client Creation

Service clients wrap the base client with strong type checking:

```swift
// Basic service client
let serviceClient = YourService.Client(wrapping: client)

// With interceptors
let clientWithInterceptors = YourService.Client(
    wrapping: client,
    interceptors: [AuthInterceptor()]
)
```

### Persistent Connections

For long-lived connections, you need a task to keep the connection alive:

```swift
// Create and store a task for persistent connection
connectionTask = Task.detached {
    try await withGRPCClient(
        transport: try .http2NIOPosix(
            target: .dns(host: host, port: port),
            transportSecurity: transportSecurity
        )
    ) { client in
        // Connection established
        
        // Keep the connection alive until the task is cancelled
        try await Task.sleep(for: .seconds(86400 * 365)) // Effectively forever
    }
}

// Cleanup to disconnect
func disconnect() {
    connectionTask?.cancel()
    connectionTask = nil
}
```

## Connection State Management

### Connection States

Define an enumeration to track connection states:

```swift
enum ConnectionState: Equatable {
    case disconnected
    case connecting
    case connected
    case failed(Error)
    
    var isConnected: Bool {
        if case .connected = self {
            return true
        }
        return false
    }
    
    static func == (lhs: ConnectionState, rhs: ConnectionState) -> Bool {
        switch (lhs, rhs) {
        case (.disconnected, .disconnected),
             (.connecting, .connecting),
             (.connected, .connected):
            return true
        case (.failed(let lhsError), .failed(let rhsError)):
            return lhsError.localizedDescription == rhsError.localizedDescription
        default:
            return false
        }
    }
}
```

### Actor-Based State Management

Use Swift actors for thread-safe state management:

```swift
actor StateManager {
    private let stateStream: AsyncStream<ConnectionState>
    private let continuation: AsyncStream<ConnectionState>.Continuation
    private var currentState: ConnectionState = .disconnected
    
    var stateUpdates: AsyncStream<ConnectionState> { stateStream }
    
    init() {
        let (stream, continuation) = AsyncStream<ConnectionState>.makeStream()
        self.stateStream = stream
        self.continuation = continuation
    }
    
    deinit {
        continuation.finish()
    }
    
    func updateState(_ state: ConnectionState) {
        // Only update if state changed or transitioning from connecting to connected
        if state != currentState || currentState == .connecting {
            currentState = state
            continuation.yield(state)
        }
    }
    
    func getCurrentState() -> ConnectionState {
        return currentState
    }
}
```

### MainActor Client Manager

Combine actor isolation with MainActor for UI updates:

```swift
@MainActor
final class GRPCClientManager: ObservableObject {
    @Published private(set) var connectionState: ConnectionState = .disconnected
    
    private let logger = Logger(subsystem: "com.app", category: "GRPCConnection")
    private var connectionTask: Task<Void, Never>?
    private let stateManager = StateManager()
    
    // Server connection configuration
    private let host: String
    private let port: Int
    private let useTLS: Bool
    private let caCertPath: String?
    
    init(host: String, port: Int, useTLS: Bool = false, caCertPath: String? = nil) {
        self.host = host
        self.port = port
        self.useTLS = useTLS
        self.caCertPath = caCertPath
        
        startObservingStateChanges()
    }
    
    deinit {
        // Direct cleanup without crossing actor boundaries
        connectionTask?.cancel()
    }
    
    func connect() {
        // Skip if already connected or connecting
        if case .connected = connectionState { return }
        guard connectionTask == nil else { return }
        
        // Capture values locally to avoid capturing self
        let host = self.host
        let port = self.port
        let useTLS = self.useTLS
        let caCertPath = self.caCertPath
        let stateManager = self.stateManager
        
        // Update state to connecting
        self.connectionState = .connecting
        Task { await stateManager.updateState(.connecting) }
        
        // Start detached task to avoid capturing actor context
        connectionTask = Task.detached {
            do {
                // Configure transport security
                let transportSecurity: HTTP2ClientTransport.Posix.TransportSecurity
                
                if useTLS, let certPath = caCertPath {
                    transportSecurity = .tls(.init(
                        certificateChain: [],
                        privateKey: nil,
                        serverCertificateVerification: .fullVerification,
                        trustRoots: .certificates([.file(path: certPath, format: .pem)])
                    ))
                } else {
                    transportSecurity = .plaintext
                }
                
                // Establish gRPC connection
                try await withGRPCClient(
                    transport: try .http2NIOPosix(
                        target: .dns(host: host, port: port),
                        transportSecurity: transportSecurity
                    )
                ) { client in
                    // Update state to connected
                    await stateManager.updateState(.connected)
                    
                    // Keep the connection alive until cancelled
                    try await Task.sleep(for: .seconds(86400 * 365))
                }
            } catch {
                // Update state to failed
                await stateManager.updateState(.failed(error))
            }
        }
    }
    
    func disconnect() {
        connectionTask?.cancel()
        connectionTask = nil
        
        // Update state
        connectionState = .disconnected
        Task { await stateManager.updateState(.disconnected) }
    }
    
    private func startObservingStateChanges() {
        Task {
            for await state in await stateManager.stateUpdates {
                await MainActor.run {
                    self.connectionState = state
                }
            }
        }
    }
}
```

## Executing gRPC Calls

### Unary Calls

```swift
// Simple unary call
let request = YourMessage.with { $0.field = "value" }
let response = try await serviceClient.yourMethod(request)
```

### Server Streaming

```swift
// Server streaming call
try await serviceClient.streamingMethod(request) { responseStream in
    for try await message in responseStream.messages {
        // Process each message as it arrives
    }
}
```

### Client Streaming

```swift
// Client streaming call
let response = try await serviceClient.clientStreamMethod { writer in
    // Send multiple messages
    try await writer.write(.with { $0.field = "message1" })
    try await writer.write(.with { $0.field = "message2" })
}
```

### Bidirectional Streaming

```swift
// Bidirectional streaming
try await serviceClient.bidiStreamMethod { writer in
    // Send messages
    try await writer.write(.with { $0.field = "message1" })
    try await writer.write(.with { $0.field = "message2" })
} onResponse: { responseStream in
    // Process responses
    for try await message in responseStream.messages {
        // Handle each response message
    }
}
```

## Working with Metadata

```swift
// Setting metadata on request
let metadata: Metadata = ["auth-token": "token123", "trace-id": "abc123"]

// With request metadata
try await serviceClient.yourMethod(request, metadata: metadata) { response in
    // Access response metadata
    let initialMetadata = response.metadata
    
    // Access message
    let message = try response.message
    
    // Access trailing metadata
    let trailingMetadata = response.trailingMetadata
}
```

## Error Handling

### RPCError Handling

```swift
do {
    let response = try await serviceClient.yourMethod(request)
} catch let error as RPCError {
    switch error.code {
    case .unavailable:
        print("Service unavailable: \(error.message ?? "")")
    case .unauthenticated:
        print("Authentication failed")
    case .internal:
        print("Internal server error")
    default:
        print("Unknown error: \(error)")
    }
} catch {
    print("Other error: \(error)")
}
```

### Custom Error Types

```swift
struct GRPCConnectionError: Error, LocalizedError {
    enum ErrorType {
        case certificateNotFound
        case connectionFailed
        case transportError(String)
    }
    
    let type: ErrorType
    
    var errorDescription: String? {
        switch type {
        case .certificateNotFound:
            return "TLS certificate not found"
        case .connectionFailed:
            return "Failed to establish connection"
        case .transportError(let message):
            return "Transport error: \(message)"
        }
    }
}
```

## Protocol Buffers Integration

### Proto File Configuration

Create a config file `grpc-swift-proto-generator-config.json` in your Protos directory:

```json
{
  "generate": {
    "clients": true,
    "servers": true,
    "messages": true
  }
}
```

### Using Generated Types

```swift
// Create a message
let message = YourMessage.with { 
    $0.field1 = "value"
    $0.field2 = 42
    $0.nested = .with { $0.nestedField = true }
}

// Serialize message
let data = try message.serializedData()

// Deserialize message
let parsed = try YourMessage(serializedData: data)
```

## Interceptors

### Client Interceptor

```swift
struct AuthInterceptor: ClientInterceptor {
    let token: String
    
    func intercept<Request, Response>(
        request: Request,
        context: ClientInterceptorContext
    ) async throws -> Response {
        var modifiedContext = context
        modifiedContext.metadata["authorization"] = "Bearer \(token)"
        return try await context.next(request, modifiedContext)
    }
}
```

### Using Interceptors

```swift
// Applying interceptors to a client
let serviceClient = YourService.Client(
    wrapping: client,
    interceptors: [
        AuthInterceptor(token: "your-token"),
        LoggingInterceptor()
    ]
)
```

## Concurrency Considerations

### Actor Isolation

```swift
// Avoid crossing actor boundaries
@MainActor
func updateUI(with state: ConnectionState) {
    // UI updates here
}

// Update UI from a background task
Task.detached {
    // Background work
    
    await MainActor.run {
        // Safe UI updates
    }
}
```

### Capturing Variables

```swift
// Avoid capturing self in tasks for better isolation
let localValue = self.someValue
let stateManager = self.stateManager

connectionTask = Task.detached {
    // Use localValue instead of self.someValue
    
    // Update state through actor
    await stateManager.updateState(.connected)
}
```

### Pattern Matching with Conditionals

```swift
// INCORRECT: Can't use || with pattern matching
if case .disconnected = state || case .failed = state {
    // This won't compile
}

// CORRECT: Separate conditions
if case .disconnected = state {
    // Handle disconnected
} else if case .failed = state {
    // Handle failed
}
```

## Common Pitfalls

1. **Parameter Order in TLS Configuration**: The parameters must be in the exact order: `certificateChain`, `privateKey`, `serverCertificateVerification`, `trustRoots`.

2. **Actor Isolation Violations**: Be careful when accessing @MainActor properties from background tasks.

3. **Detached Tasks vs Regular Tasks**: Use `Task.detached` for connections to avoid capturing the current actor context.

4. **Deinit and Actors**: Don't call actor-isolated methods from deinit, directly perform cleanup instead.

5. **Pattern Matching Syntax**: Can't combine case patterns with logical operators.

## Best Practices

1. Use `withGRPCClient` for automatic connection lifecycle management.

2. Store connection tasks to maintain persistent connections.

3. Use actors for thread-safe state management.

4. Keep UI updates on @MainActor.

5. Avoid capturing self in long-running tasks.

6. Track connection state explicitly through an observable model.

7. Use Task.detached for connections to avoid actor context capture.

8. Handle all errors from gRPC operations properly.

9. Be explicit about weak references when needed.
