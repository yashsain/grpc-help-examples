# gRPC-Swift 2.x Implementation Guide

## Architecture Overview

gRPC-Swift 2.x represents a significant redesign from previous versions, embracing Swift's modern concurrency model with async/await while providing a more type-safe API. This guide captures key implementation patterns and gotchas when working with this newer version.

## Transport System

### Transport Types and Protocol Hierarchy

- `ClientTransport` is now a **protocol** with an associated type for `Bytes`, not a concrete type
- The concrete transport implementation for HTTP/2 is `HTTP2ClientTransport.Posix` from the `GRPCNIOTransportHTTP2` package
- Transport creation uses a factory-style pattern with `.http2NIOPosix(target:transportSecurity:)` rather than direct instantiation

```swift
// INCORRECT: Trying to use a protocol existential
func someFunction(transport: any ClientTransport) { ... }

// CORRECT: Using a concrete transport type
func someFunction<T: ClientTransport>(transport: T) { ... }

// CORRECT: Creating transport with factory pattern
let transport = .http2NIOPosix(
    target: .dns(host: "example.com", port: 443),
    transportSecurity: .plaintext
)
```

### Critical Type System Limitations

- **Existential Types Issue**: Using `any ClientTransport` in generic contexts leads to errors
- Swift doesn't allow protocol existentials (like `any ClientTransport`) where generic constraints are expected 
- All service clients must use the **same concrete transport type** throughout the call chain
- Type erasure must be avoided for the transport generic parameter

```swift
// INCORRECT: This will cause compiler errors
func createClient<T: Sendable>(
    _ operation: @escaping (GRPCClient<any ClientTransport>) async throws -> T
) async throws -> T { ... }

// CORRECT: Using the specific concrete type
func createClient<T: Sendable>(
    _ operation: @escaping (GRPCClient<HTTP2ClientTransport.Posix>) async throws -> T
) async throws -> T { ... }
```

### Target Configuration

- The `.dns(host:port:)` syntax replaces older `.host(_:port:)` or `.hostAndPort(_:_:)` patterns
- Target specification follows the pattern: `target: .dns(host: "example.com", port: 443)`

```swift
// INCORRECT (old API):
let target = .hostAndPort("example.com", 443)

// CORRECT (new API):
let target = .dns(host: "example.com", port: 443)
```

## Client Creation Patterns

### The `withGRPCClient` Pattern

- Clients are created using the `withGRPCClient` function which manages client lifecycle
- This function takes a transport and a closure that receives the client
- The client is automatically shut down when the closure completes
- For keeping connections open (e.g., for streaming), the closure must be kept alive in a task

```swift
// Basic pattern
try await withGRPCClient(
    transport: .http2NIOPosix(
        target: .dns(host: serverHost, port: serverPort),
        transportSecurity: .plaintext
    ),
    interceptors: [AuthInterceptor()]
) { client in
    // Use client here
    // Connection is closed when this closure completes
}
```

### Service Client Creation

- Service clients are instantiated with: `ServiceName.Client(wrapping: client)`
- Service clients require a generic parameter matching the transport type: `ServiceName.Client<HTTP2ClientTransport.Posix>`
- The generic parameter **cannot be abstracted** with `any ClientTransport`
- Type inference helps by propagating the concrete type through the call chain

```swift
// INCORRECT: Missing transport type parameter
func makeServiceClient() async throws -> ServiceName.Client { ... }

// INCORRECT: Using protocol existential
func makeServiceClient() async throws -> ServiceName.Client<any ClientTransport> { ... }

// CORRECT: Using concrete transport type
func makeServiceClient() async throws -> ServiceName.Client<HTTP2ClientTransport.Posix> {
    return try await createClient { client in
        return ServiceName.Client(wrapping: client)
    }
}
```

### Task Management for Persistent Connections

- For bidirectional streaming, client connections must be kept open
- This requires storing references to tasks that maintain the client
- A common pattern is creating a task and storing it in an array to prevent it from being garbage collected

```swift
// Store tasks to keep connections alive
private var clientTasks: [Task<Void, Never>] = []

// Create and store a task to maintain the client
func createAndMaintainClient<T>(_ operation: @escaping (GRPCClient<HTTP2ClientTransport.Posix>) async throws -> T) async throws -> T {
    let task = Task<T, Error> {
        // Client creation logic here
    }
    
    // Store task to prevent it from being garbage collected
    clientTasks.append(Task { 
        do {
            _ = try await task.value
        } catch {
            // Handle error
        }
    })
    
    return try await task.value
}
```

## TLS Configuration

### Certificate Handling

- Manual certificate loading with `NIOSSLCertificate.fromPEMFile()` is not compatible with the new TLS configuration
- Correct approach uses a file-based certificate source: `.certificates([.file(path, .pem)])`
- Type mismatch occurs between `[NIOSSLCertificate]` and `[TLSConfig.CertificateSource]`

```swift
// INCORRECT: This causes a type mismatch
let caCertificates = try NIOSSLCertificate.fromPEMFile(certificatePath)
trustRoots: .certificates(caCertificates)  // Error: Cannot convert [NIOSSLCertificate] to [TLSConfig.CertificateSource]

// CORRECT: Using file-based certificate source
trustRoots: .certificates([.file(certificatePath, .pem)])
```

### TLS Configuration Initializer

- Proper initialization requires these parameters:
  ```swift
  .tls(.init(
      serverCertificateVerification: .fullVerification,
      certificateChain: [],         // Empty for no client auth
      privateKey: nil,              // Nil for no client auth
      trustRoots: .certificates([.file(certificatePath, .pem)])
  ))
  ```
- The parameter label is `serverCertificateVerification`, not `certificateVerification`
- `certificateChain` and `privateKey` are required parameters even for server-only verification

```swift
// INCORRECT: Missing parameters and wrong label
transportSecurity: .tls(.init(
    certificateVerification: .fullVerification,  // Wrong parameter name
    trustRoots: .certificates([...])
))

// CORRECT: All parameters properly specified
transportSecurity: .tls(.init(
    serverCertificateVerification: .fullVerification,
    certificateChain: [],  // Required even if not used
    privateKey: nil,       // Required even if not used
    trustRoots: .certificates([.file(certificatePath, .pem)])
))
```

## Using Service Clients

### Return Types and Generic Parameters

- Service client methods should be declared with the concrete transport type:
  ```swift
  func makeAdminServiceClient() async throws -> Journaling_Admin_AdminService.Client<HTTP2ClientTransport.Posix>
  ```
- Methods should propagate the concrete type without attempting to abstract it
- The same concrete type must be used consistently throughout the implementation

```swift
// Concrete return type with specific transport
func makeAdminServiceClient() async throws -> AdminService.Client<HTTP2ClientTransport.Posix> {
    return try await createClient { client in
        return AdminService.Client(wrapping: client)
    }
}
```

### Making Requests

- Modern Swift concurrency patterns are used throughout:
  ```swift
  let response = try await client.someMethod(request)
  ```
- Options can be provided as a second parameter:
  ```swift
  try await client.someMethod(request, options: ConnectionConfig.quickCallOptions)
  ```

```swift
// Example health check implementation
func testConnection() async -> Bool {
    do {
        let client = try await makeAdminServiceClient()
        let request = HealthRequest()
        
        let response = try await client.getHealth(request)
        
        return response.status == "ok"
    } catch {
        // Error handling
        return false
    }
}
```

## Error Handling

### Error Types

- The older `GRPCStatus` has been replaced with `RPCError`
- Custom errors should be designed to provide meaningful information about failure points
- Transport-level errors should be distinguished from application-level errors

```swift
// Custom error types for different failure scenarios
enum GRPCConnectionError: Error, LocalizedError {
    case certificateNotFound
    case connectionFailed
    case transportError(String)
    
    var errorDescription: String? {
        switch self {
        case .certificateNotFound:
            return "TLS certificate not found in the app bundle"
        case .connectionFailed:
            return "Failed to establish connection to the server"
        case .transportError(let message):
            return "Transport error: \(message)"
        }
    }
}
```

### Propagation Pattern

- Error handling leverages Swift's structured concurrency with do/catch
- Errors should be logged and then propagated up the call stack
- Connection status should be updated to reflect failures

```swift
do {
    // Attempt to create client and make request
    let client = try await makeServiceClient()
    let response = try await client.someMethod(request)
    return response
} catch {
    // Update connection status
    self.connectionStatus = .failed(error)
    
    // Log the error
    ErrorHandler.handle(error: error, context: "Client operation")
    
    // Propagate the error
    throw error
}
```

## Complete Example Structure

Here's a skeleton structure that demonstrates the recommended pattern:

```swift
class GRPCClientManager {
    // Store tasks to keep connections alive
    private var clientTasks: [Task<Void, Never>] = []
    
    // Connection status tracking
    private(set) var connectionStatus: ConnectionStatus = .disconnected
    
    // Helper method for client creation
    private func createClient<T: Sendable>(
        _ operation: @escaping (GRPCClient<HTTP2ClientTransport.Posix>) async throws -> T
    ) async throws -> T {
        self.connectionStatus = .connecting
        
        let task = Task<T, Error> {
            do {
                // Create client with proper transport
                let result = try await withGRPCClient(
                    transport: .http2NIOPosix(
                        target: .dns(host: serverHost, port: serverPort),
                        transportSecurity: .plaintext // or configure TLS
                    ),
                    interceptors: [AuthInterceptor()]
                ) { client in
                    self.connectionStatus = .connected
                    return try await operation(client)
                }
                
                return result
            } catch {
                self.connectionStatus = .failed(error)
                throw error
            }
        }
        
        // Store task to keep connection alive
        clientTasks.append(Task { 
            do {
                _ = try await task.value
            } catch {
                // Error already handled
            }
        })
        
        return try await task.value
    }
    
    // Service client factory method
    func makeServiceClient() async throws -> Service.Client<HTTP2ClientTransport.Posix> {
        return try await createClient { client in
            return Service.Client(wrapping: client)
        }
    }
}
```

This approach ensures type safety while providing the flexibility needed for different types of gRPC operations.
