# dgraph-rs

Rust client for the [Dgraph](https://dgraph.io) graph database.

```rust
use dgraph_client::{DgraphClient, Mutation};

let client = DgraphClient::connect("dgraph://groot:password@localhost:9080").await?;
client.set_schema("name: string @index(exact) .").await?;
```

## Layout

| Path | What it is |
|---|---|
| `dgraph-client/` | The client. The only hand-written crate. |
| `dgraph-client/src/utils_test/` | Acceptance fixture, behind the `test-utils` feature. |
| `proto/dgraph/` | Generated tonic/prost bindings for Dgraph's `api.proto`. |
| `build/rust/` | The prost/protoc Bazel toolchains, and the crate that pins the plugins. |

## Building

Both build systems work and both are checked:

```bash
cargo build
bazel build //...
```


## Testing

The client's own unit tests need nothing:

```bash
cargo test
bazel test //... --test_tag_filters=-integration_test
```

The acceptance test drives the real client against a real Dgraph, and is `#[ignore]` so neither
`cargo test` nor a plain `bazel test` runs it by accident:

```bash
cargo test -p dgraph-client --features test-utils --test dgraph_container_test -- --ignored
bazel test //dgraph-client/tests:dgraph_client_integration_test \
  --test_tag_filters=integration_test --nocache_test_results
```

## The `test-utils` feature

`dgraph_client::utils_test` obtains a running Dgraph -- provisioning a `dgraph/standalone`
container, or verifying one that already exists -- and hands back a connection string. It is
useful to anyone testing code that talks to Dgraph, so it ships with the client rather than as a
second crate.

It is **off by default**, because acquiring an instance means a Docker client, an HTTP agent and
a JSON parser that a consumer of the client itself will never call.

```rust
use dgraph_client::utils_test::DgraphInstance;

let dgraph = DgraphInstance::acquire()?;
let client = DgraphClient::connect(dgraph.connection_string()).await?;
```

### Configuration

Every variable has a default, so a workstation with a Docker daemon needs none of them.

| Variable | Default | Meaning |
|---|---|---|
| `DGRAPH_TEST_STRATEGY` | `container` | `container` provisions a `dgraph/standalone`; `existing` verifies a Dgraph this process did not create. An unknown value is refused, not defaulted. |
| `DGRAPH_TEST_HOST` | `127.0.0.1` | Where Dgraph is. |
| `DGRAPH_TEST_PORT` | `9080` | Dgraph's gRPC port. |
| `DGRAPH_TEST_SSLMODE` | `disable` | `disable`, `require` or `verify-ca` -- the same spellings a connection string uses. |
| `DGRAPH_TEST_CA_BUNDLE_URL` | unset | Where this deployment publishes the CA that verifies Dgraph. |
| `DGRAPH_TEST_RUN_ID` | unset | Correlation id for one run. Unset synthesises a local id and says so, rather than borrowing a name that means "the CI run". |
| `DGRAPH_ADMIN_PASSWORD` | unset | ACL password for `groot`. Required by `existing` only; a container this process started uses Dgraph's own published default. |

### Ownership is part of the answer

`DgraphInstance::exclusivity()` reports whether this process created the instance. A container it
started is `Exclusive` and may be wiped; a Dgraph it merely found is `Shared` and must not be.
Callers that destroy data are expected to branch on it, because nothing else can tell the two
apart from the endpoint alone.

## Licence

Apache-2.0. See [LICENSE](LICENSE).
