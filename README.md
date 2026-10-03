# quarkus-openid-ssf-test

Example application for exercising releases of the [Quarkiverse Quarkus OpenID SSF extension](https://github.com/quarkiverse/quarkus-openid-ssf).

It wires up `quarkus-openid-ssf-receiver` against a transmitter (e.g. Keycloak or caep.dev), captures incoming Shared Signals Framework (SSF) events into an in-memory ring buffer, and exposes them over a small REST API.

## What it tests

- Receiver registration against an SSF transmitter
- `POLL` and `PUSH` delivery modes (both shown in [`application.properties`](src/main/resources/application.properties); `POLL` is the active default)
- OAuth2 `client_credentials` outbound auth via `quarkus-oidc-client`
- Custom `SsfEventHandler` implementation ([`CapturingSsfEventHandler`](src/main/java/com/github/thomasdarimont/training/CapturingSsfEventHandler.java)); since 0.2.0 the handler types are those of [easyssf](https://github.com/easyssf/easyssf), the receiver library the extension is built on
- Prometheus metrics exposed at `/q/metrics` via `quarkus-micrometer-registry-prometheus` (`easyssf.receiver.*` meters)

Subscribed event types: `CaepSessionRevoked`, `CaepCredentialChange`.

## Versions

| Dependency                       | Version  |
|----------------------------------|----------|
| `quarkus-openid-ssf-receiver`    | `0.2.0`  |
| Quarkus platform                 | `3.35.2` |
| Java                             | `21`     |

## Configuration

The app reads its transmitter and OIDC client coordinates from environment variables — set these before starting it:

| Variable                          | Purpose                                                                  |
|-----------------------------------|--------------------------------------------------------------------------|
| `SSF_RECEIVER_TRANSMITTER_ISSUER` | Issuer URL of the SSF transmitter to register against                    |
| `OIDC_ISSUER_URL`                 | OIDC issuer used to obtain access tokens for outbound calls              |
| `SSF_RECEIVER_CLIENT_ID`          | OAuth2 client id with `ssf.read` and `ssf.manage` scopes                 |
| `SSF_RECEIVER_CLIENT_SECRET`      | OAuth2 client secret                                                     |
| `SSF_RECEIVER_PUSH_AUTH_TOKEN`    | Optional — bearer token expected on inbound PUSH deliveries (`push.expected-auth-header`) |

### Delivery modes

The example ships with two delivery configurations:

- **`POLL`** (active default) — the receiver polls the transmitter on a fixed interval (`10s`), fetches up to 50 events per call and keeps fetching while the transmitter reports more, and auto-starts at boot. The transmitter assigns the delivery endpoint itself, so no receiver URL has to be reachable from outside.
- **`PUSH`** (commented-out example) — the transmitter pushes SETs to a receiver-hosted endpoint. The URL set via `quarkus.openid-ssf.receiver.push.delivery-endpoint-url` is advertised to the transmitter on `createStream` and must be reachable from it (public URL / ngrok / VPN). An optional `push.expected-auth-header` is registered with the stream and lets the receiver verify the `Authorization` header of inbound pushes.

To switch to PUSH, comment out the POLL block and uncomment the PUSH block in [`application.properties`](src/main/resources/application.properties).

## Running in dev mode

```shell script
./mvnw quarkus:dev
```

Quarkus Dev UI: <http://localhost:8080/q/dev/>.

## REST endpoints

| Method | Path                    | Description                                |
|--------|-------------------------|--------------------------------------------|
| `GET`  | `/events/recent-events` | Up to 50 most recently captured SSF events |
| `GET`  | `/events/latest`        | The single most recent captured event      |
| `GET`  | `/q/metrics`            | Prometheus metrics, incl. SSF receiver     |

## Packaging

JVM build:

```shell script
./mvnw package
```

Produces `target/quarkus-app/quarkus-run.jar`, runnable via `java -jar target/quarkus-app/quarkus-run.jar`.

Über-jar:

```shell script
./mvnw package -Dquarkus.package.jar.type=uber-jar
```

## Native executable

```shell script
./mvnw package -Dnative
```

Or, without a local GraalVM installation:

```shell script
./mvnw package -Dnative -Dquarkus.native.container-build=true
```

Then run `./target/quarkus-openid-ssf-test-1.0-SNAPSHOT-runner`. See <https://quarkus.io/guides/maven-tooling> for details.

## References

- Extension: <https://github.com/quarkiverse/quarkus-openid-ssf>
- OpenID Shared Signals Framework: <https://openid.net/specs/openid-sharedsignals-framework-1_0.html>
- Quarkus: <https://quarkus.io/>
