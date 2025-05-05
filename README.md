1. Unary RPC
Definition: one request message from client → one response message from server.
When to use: simple CRUD-style operations (e.g. “GetUserById”, “CreateOrder”), where you don’t need streaming and the payload is small.
Pros: easy to implement, minimal resource usage, simple error handling.
Cons: not suited for large or continuous data flows—you must buffer everything in memory before returning.


2. Server‐Streaming RPC
Definition: client sends one request → server sends back a stream of responses (zero or more messages), then the stream closes.
When to use: “list all log entries since timestamp X,” “download large file in chunks,” or “subscribe to market data feed.”
Pros: you can push arbitrarily many messages without loading them all at once; backpressure is handled by HTTP/2 flow control.
Cons: client must be prepared to consume—or cancel—the stream; more complex client‐side logic to iterate over the stream.


3. Bidirectional‐Streaming RPC
Definition: both client and server open independent streams and can send messages in any order; neither side has to wait for the other to close.
When to use: real-time chat, interactive gaming, collaborative editing, or any scenario where you need low-latency two-way data flow.
Pros: full duplex, low latency, efficient for frequent message exchange.
Cons: most complex to implement—requires handling concurrency, cancellation, ordering, and backpressure on both sides.


4. Security considerations in Rust gRPC
1) Authentication:
TLS client certificates (rustls / openssl) or JWT/OAuth tokens in metadata;
mutual-TLS for strongest identity guarantees.
2) Authorization:
use gRPC interceptors to validate tokens and enforce RBAC or ACL checks on each RPC method;
embed claims in the request context and check them before business logic runs.
3) Encryption:
enable HTTP/2 over TLS (TLS 1.2+ or QUIC);
ensure certificate rotation and pinning policies;
protect metadata channels as well as payloads.
4) Other:
rate-limit per IP or per-user to prevent DoS;
sanitize logs to avoid leaking PII.


5. Challenges in bidirectional streaming (e.g. chat apps)
1) Concurrency & state: maintaining per-user sessions and routing messages between multiple peers;
2) Backpressure: preventing a fast sender from overwhelming a slow receiver—careful with buffer sizes and channel policies;
3) Ordering & delivery guarantees: you may need sequence numbers or ACKs on top of HTTP/2;
4) Error & cancellation handling: if one side errors or cancels, you must clean up streams on both ends and possibly notify peers;
5) Resource leaks: unfinished streams can hang open—always implement timeouts or heartbeats.


6. Pros & cons of tokio_stream::wrappers::ReceiverStream
- Advantages:
* easily converts a tokio::mpsc::Receiver "T" into a Stream<Item = T> that Tonic can return;
* decouples data production (in a task) from consumption by the gRPC runtime;
* leverages Tokio’s async channel with backpressure signals.
- Disadvantages:
* if your channel’s buffer grows unbounded, you risk OOM;
* limited control over fine-grained flow control—channel capacity is coarse;
* cancelling the client stream doesn’t automatically close the sender side—you need to detect dropped receivers.


7. Structuring Rust gRPC code for reuse & modularity
* Proto crate: put your .proto files and generated Rust code in a standalone crate (e.g. myapp-proto).
* Service traits: define your business logic as Rust traits in a “core” crate—unit-testable without any networking.
* Transport layer: in a separate crate or module, wire up your tonic::transport::Server, interceptors, and TLS config.
* Dependency injection: pass trait-object implementations or generic type parameters into your server builder.
* Shared utilities: common error types, log formatters, auth interceptors as reusable modules.


8. Extending MyPaymentService for complex logic
* Idempotency keys: prevent duplicate charges if client retries;
* Transaction management: wrap DB updates, external API calls in database transactions;
* Retry & compensation: automatic retries on transient failures, plus compensating actions on partial failures;
* Audit logging: record every request/response pair, decision point, to an append-only store;
* Fraud detection hooks: integrate with risk-scoring services or ML models;
* Validation & enrichment: verify account balances, currency conversions, anti-money-laundering checks.


9. gRPC’s impact on distributed architecture
* Strong contracts: service interface defined by Protobuf → cross-language codegen → fewer integration bugs.
* Performance: HTTP/2 multiplexing → lower latency and better resource utilization.
* Polyglot: generate clients/servers in Rust, Go, Java, Python…easy interoperability.
* Operational changes: need HTTP/2-aware load-balancers (Envoy, NGINX), observability for streams, tracing integration.


10. HTTP/2 (gRPC) vs HTTP/1.1 / WebSocket for REST
* HTTP/2 strengths: multiplexed streams over single TCP → lower head-of-line blocking; binary framing + header compression → smaller messages; built-in flow control.
* HTTP/1.1+WebSocket: widely supported by browsers and legacy infra; simpler to debug since it’s text over a TCP socket.
* gRPC downsides: harder to inspect with curl/browsers; requires HTTP/2 support end-to-end; less human-readable payloads.
* REST+WebSocket: you get HTTP/1.1 request/response for most ops and fall back to WebSocket for streaming—but that’s two protocols to maintain.


11. REST’s request‐response vs gRPC bidirectional streaming
* REST (HTTP/1.1): client initiates each request, server sends back exactly one response—good for stateless queries, easy caching, retries.
* gRPC streaming: once the stream is open, both sides can send messages at any time—ideal for real-time telemetry, chat, live dashboards.


12. Protobuf schema vs JSON’s flexibility
- Protobuf (gRPC):
* binary, compact, strictly typed → better perf, smaller payloads, generated code for every language, compile-time checks.
* schema evolution via optional fields, reserved tags → safe versioning.
- JSON (REST):
* human-readable, no build step → very flexible for ad-hoc or loosely coupled clients.
* no compile-time checks → runtime errors if fields missing or of wrong type; larger messages due to text and repeated field names.