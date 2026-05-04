---
name: grpc-services
description: Build a gRPC service end-to-end — proto layout, code generation, streaming, interceptors (auth/logging/tracing), error handling, gRPC-Gateway for REST/JSON, deadline propagation, and the versioning discipline that keeps clients working through breaking changes. Use when building a gRPC service from scratch, not when adding RPCs to an existing one.
---

# gRPC Services

gRPC excels at internal service-to-service traffic: typed contracts, streaming, fast wire format. The trap is treating proto files like a generated artifact instead of a versioned contract.

## When to use

- Internal service mesh / microservices.
- Mobile clients with poor network conditions (HTTP/2 multiplexing helps).
- Polyglot environments (Go server, TS/Python/Java clients).
- High-throughput RPC where JSON serialization is the bottleneck.

## When NOT to use

- Public web API consumed from browsers — gRPC-Web exists but adds friction. REST or GraphQL is better.
- Browsers as clients in general — limited tooling.
- Simple internal API in a single language — REST/tRPC is faster to ship.

## Proto layout

```
proto/
  acme/
    auth/
      v1/
        auth.proto
    payments/
      v1/
        payments.proto
buf.yaml
buf.gen.yaml
```

**One package per service per version**: `acme.payments.v1`. Versioning lives in the package path, not in field names. When you need v2, copy the file to `v2/` — both versions live side by side.

## buf for proto tooling

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
lint:
  use: [STANDARD]
breaking:
  use: [FILE]
```

```yaml
# buf.gen.yaml — generates Go + TS clients
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: [paths=source_relative]
  - remote: buf.build/grpc/go
    out: gen/go
    opt: [paths=source_relative]
  - remote: buf.build/connectrpc/es
    out: gen/ts
```

CI runs `buf lint`, `buf breaking --against '.git#branch=main'`, `buf generate`. Breaking-change check is the safety net.

## Proto example

```protobuf
syntax = "proto3";
package acme.payments.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";

option go_package = "github.com/acme/proto/acme/payments/v1;paymentsv1";

service PaymentService {
  rpc CreateCharge(CreateChargeRequest) returns (Charge);
  rpc GetCharge(GetChargeRequest) returns (Charge);
  rpc ListCharges(ListChargesRequest) returns (ListChargesResponse);
  rpc StreamChargeUpdates(StreamChargeUpdatesRequest) returns (stream Charge);
}

message Charge {
  string id = 1;
  string customer_id = 2;
  int64  amount_paise = 3;                       // never use double for money
  string currency = 4;
  Status status = 5;
  google.protobuf.Timestamp created_at = 6;

  enum Status {
    STATUS_UNSPECIFIED = 0;                      // mandatory zero value
    STATUS_PENDING = 1;
    STATUS_CAPTURED = 2;
    STATUS_FAILED = 3;
    STATUS_REFUNDED = 4;
  }
}

message CreateChargeRequest {
  string customer_id = 1;
  int64  amount_paise = 2;
  string currency = 3;
  string idempotency_key = 4;                    // mandatory
}

message GetChargeRequest { string id = 1; }

message ListChargesRequest {
  string customer_id = 1;
  int32  page_size = 2;
  string page_token = 3;
}

message ListChargesResponse {
  repeated Charge charges = 1;
  string next_page_token = 2;
}
```

Naming conventions to follow:
- `<Verb><Noun>Request` / `<Verb><Noun>Response` — even for RPCs that "feel like" they don't need wrappers. Adding fields later is then non-breaking.
- Enums always have a `_UNSPECIFIED = 0` zero value.
- Money fields are `int64` paise/cents, never `double`.

## Server (Go)

```go
// internal/grpc/server.go
package grpc

import (
    "context"
    paymentsv1 "github.com/acme/proto/acme/payments/v1"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type PaymentServer struct {
    paymentsv1.UnimplementedPaymentServiceServer
    db DB
}

func (s *PaymentServer) CreateCharge(ctx context.Context, req *paymentsv1.CreateChargeRequest) (*paymentsv1.Charge, error) {
    if req.IdempotencyKey == "" {
        return nil, status.Error(codes.InvalidArgument, "idempotency_key required")
    }
    if req.AmountPaise <= 0 {
        return nil, status.Error(codes.InvalidArgument, "amount must be positive")
    }
    charge, err := s.db.CreateCharge(ctx, req)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "create charge: %v", err)
    }
    return charge, nil
}
```

Always embed `Unimplemented*Server` — when proto adds new RPCs, server keeps compiling (forward compat).

## Interceptors (cross-cutting concerns)

```go
import (
    grpc_zap "github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/logging/zap"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        otelgrpc.UnaryServerInterceptor(),       // tracing
        recoveryInterceptor(),                   // panic recovery
        authInterceptor(),                       // auth
        loggingInterceptor(),                    // structured logs
        rateLimitInterceptor(),                  // per-method limits
    ),
)
```

Order matters: tracing first (so all sub-spans are children), recovery second (catches panics from later interceptors), auth third (early reject), logging fourth (logs auth result).

## Deadlines and cancellation

Clients set deadlines; servers propagate `ctx`:

```go
// Client
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
charge, err := client.CreateCharge(ctx, req)

// Server — DB calls take ctx
row, err := s.db.QueryContext(ctx, "SELECT ...")
```

Without `ctx` propagation, a 5s client deadline doesn't actually stop the DB query at 5s — it keeps running. Pool exhaustion follows.

## Streaming (server-side)

```go
func (s *PaymentServer) StreamChargeUpdates(req *paymentsv1.StreamChargeUpdatesRequest, stream paymentsv1.PaymentService_StreamChargeUpdatesServer) error {
    sub := s.subscribe(req.CustomerId)
    defer sub.Close()
    for {
        select {
        case <-stream.Context().Done():
            return stream.Context().Err()
        case ch := <-sub.Channel():
            if err := stream.Send(ch); err != nil {
                return err
            }
        }
    }
}
```

Clients should retry on `Unavailable` errors with exponential backoff.

## Errors: use `status.Error` with proper codes

| Code | When |
|---|---|
| `InvalidArgument` | bad input from caller |
| `NotFound` | resource doesn't exist |
| `AlreadyExists` | duplicate |
| `PermissionDenied` | authenticated but not authorized |
| `Unauthenticated` | no/bad credentials |
| `ResourceExhausted` | rate-limited / quota |
| `FailedPrecondition` | system state wrong (e.g. idempotency conflict) |
| `Aborted` | concurrency conflict |
| `Internal` | bug on our side |
| `Unavailable` | transient — retry with backoff |
| `DeadlineExceeded` | client deadline hit |

Don't return `Internal` for everything. Granular codes drive client retry behavior.

## gRPC-Gateway (for REST/JSON consumers)

Auto-generate a JSON↔gRPC proxy from annotations:

```protobuf
import "google/api/annotations.proto";
service PaymentService {
  rpc GetCharge(GetChargeRequest) returns (Charge) {
    option (google.api.http) = { get: "/v1/charges/{id}" };
  }
}
```

`buf` plugin emits Go code that serves both gRPC and REST on the same binary. Useful when 80% of consumers are gRPC but the support team's `curl` scripts need JSON.

## Anti-patterns

- **Reusing a field number after deletion** — silent wire-format corruption. `reserved` removed numbers in proto.
- **No `_UNSPECIFIED` zero on enums** — proto3 default is 0; without unspecified, you can't tell "default" from "explicitly the first value."
- **`double` for money** — float precision loss, regulatory issue. `int64` paise/cents/satoshi.
- **Field renames without `reserved`** — clients on old protos blow up.
- **No `idempotency_key` on `Create*` RPCs** — retries cause duplicates.
- **No deadline at the client** — cascading timeouts during overload.
- **Server doesn't propagate `ctx` to DB** — work continues past client cancel.
- **Returning `error.Error()` strings** — caller can't programmatically retry. `status.Error(code, msg)` always.
- **One proto file with everything** — package per service, not per company.
- **Hand-rolling generated code in repo** — wastes review time. Generate at build, gitignore output (or commit but mark generated).
- **Skipping `buf breaking` in CI** — breaking changes ship and downstream services 5xx.

## Verify it worked

- [ ] `buf lint` and `buf breaking --against '.git#branch=main'` pass in CI.
- [ ] Generated Go and TS clients work against the server.
- [ ] Tracing: a client RPC produces a parent span; server span is a child; DB query is a grandchild.
- [ ] Killing the client mid-RPC propagates cancellation to server-side DB query (verify via `pg_stat_activity` shows the query terminated).
- [ ] Renaming a field is caught by `buf breaking`.
- [ ] Replaying the same `CreateCharge` with same `idempotency_key` returns the original charge, doesn't create a duplicate.
- [ ] Errors use `status.Error` with appropriate codes (not `Internal` for everything).
- [ ] Stream RPCs handle client cancel cleanly (no leaked goroutines).
- [ ] gRPC-Gateway endpoint serves the same data as gRPC.
- [ ] Server has interceptor chain: trace → recovery → auth → logging → rate-limit.
