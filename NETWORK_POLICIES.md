# Network Policies

This document describes the Kubernetes NetworkPolicy resources deployed by the `home-anthill` Helm chart. All policies live in `home-anthill/templates/network-policy.yaml` and are controlled by the `network.enabled` value (default: `true`).

## Design Principles

1. **Default-deny**: A blanket `default-deny-all` policy blocks all ingress and egress for every pod in the namespace. Each service then receives an explicit allow-list policy.
2. **Least privilege**: Every egress and ingress rule is scoped to specific pods and ports. No service has broader access than it needs.
3. **Conditional SSL rules**: When MQTT or HTTP SSL is enabled, additional rules are rendered to support external MQTT connections and ACME certificate challenges. These rules have zero effect when SSL is disabled.
4. **Namespace awareness**: Cross-namespace traffic (RabbitMQ Operator, cert-manager solver pods) uses `namespaceSelector` to restrict access to specific operator namespaces.

## Configuration Values

| Value | Default | Purpose |
|-------|---------|---------|
| `network.enabled` | `true` | Enables/disables all NetworkPolicy resources |
| `network.nodesCIDR` | `10.0.0.0/8` | CIDR for kubelet health-probe traffic. Restrict to your actual node IP range in production. |
| `network.nginxGatewayNamespace` | `nginx-gateway` | Namespace where the NGF controller runs (used for reference; data-plane pods run in `home-anthill`) |
| `network.rabbitmqOperatorNamespace` | `rabbitmq-system` | Namespace where the RabbitMQ Kubernetes Operator runs |
| `domains.http.ssl.enable` | `false` | Enables HTTP SSL conditional rules (cert-manager solver) |
| `domains.mqtt.ssl.enable` | `false` | Enables MQTT SSL conditional rules (cert-manager solver + external MQTT IP egress) |
| `domains.mqtt.publicIp` | — | Public IP of the MQTT gateway; used in SSL-mode egress rules (pinned to `/32`) |

## Policy Summary

The chart deploys 17 policies with SSL disabled, or 18 when SSL is enabled.

| # | Policy Name | Pod Selector | Ingress From | Egress To |
|---|------------|-------------|-------------|----------|
| 1 | `default-deny-all` | all pods | deny all | deny all |
| 2 | `allow-dns-egress` | all pods | — | any:53 UDP/TCP |
| 3 | `allow-ngf-dataplane` | `app.kubernetes.io/managed-by: ngf-nginx` | 0.0.0.0/0 on 80,443,1883,8883; (SSL) cert-manager ns on 80 | NGF controller:8443, gui:80, admission-nginx:80, mosquitto:1883/8883, (SSL: solver:8089) |
| 4 | `allow-gui` | `app: gui` | NGF + kubelet on 80 | api-server:80 |
| 5 | `allow-api-server` | `app: api-server` | gui + kubelet on 80 | api-devices:50051, register:80, online:80, ext:27017, ext:443 |
| 6 | `allow-admission-nginx` | `app: admission-nginx` | NGF + kubelet on 80 | admission:80 |
| 7 | `allow-admission` | `app: admission` | admission-nginx + kubelet on 80 | api-devices:50051, register:80, ext:27017 |
| 8 | `allow-api-devices` | `app: api-devices` | api-server + admission + kubelet on 50051 | mosquitto:1883, ext:27017 |
| 9 | `allow-register` | `app: register` | admission + api-server + kubelet on 80 | ext:27017 |
| 10 | `allow-online` | `app: online` | api-server + kubelet on 80 | redis:6379 |
| 11 | `allow-online-receiver` | `app: online-receiver` | kubelet on 80 | mosquitto:1883, redis:6379 |
| 12 | `allow-online-alarm` | `app: online-alarm` | kubelet on 80 | redis:6379, ext:443 |
| 13 | `allow-producer` | `app: producer` | none | mosquitto:1883, rabbitmq:5672 |
| 14 | `allow-consumer` | `app: consumer` | none | rabbitmq:5672, ext:27017 |
| 15 | `allow-mosquitto` | `app: mosquitto` | NGF + api-devices + producer + online-receiver on 1883/8883 | none |
| 16 | `allow-redis` | `app: redis` | online + online-receiver + online-alarm on 6379 | none |
| 17 | `allow-rabbitmq` | `app.kubernetes.io/name: rabbitmq` | producer + consumer on 5672, operator on 5672/15672 | none |
| 18 | `allow-cert-manager-solver` (SSL only) | `acme.cert-manager.io/http01-solver: "true"` | NGF on 8089 | none |

## Communication Matrix

```
                        ┌──────────────────────────────────────────────────────────────┐
                        │                      home-anthill namespace                   │
                        │                                                              │
  Browser ─────────┐    │  ┌─────────────────┐     ┌─────┐     ┌────────────┐          │
                   │    │  │ webapp-gateway   │────▶│ gui │────▶│ api-server │          │
                   ▼    │  │ (NGF dataplane)  │     └─────┘     └──────┬─────┘          │
              ┌────────┐│  │                  │          :80       :80 │                │
              │CiliumL2├┼─▶│  :80 / :443      │                       │                │
              └────────┘│  │                  │     ┌─────────────┐    │:50051   :80    │
                   ▲    │  │                  │────▶│ admission   │◀───┼───────┐        │
  ESP32 ───────────┘    │  │                  │     │   -nginx    │    │       │        │
                   │    │  └─────────────────┘     └──────┬──────┘    │       │        │
                   ▼    │                              :80│           │       │        │
              ┌────────┐│  ┌─────────────────┐     ┌─────▼─────┐     │       │        │
              │CiliumL2├┼─▶│ mqtt-gateway    │     │ admission │─────┼───────┤        │
              └────────┘│  │ (NGF dataplane) │     └─────┬─────┘     │       │        │
                        │  │                 │       :80  │:50051     │       │        │
                        │  │  :1883 / :8883  │           │           ▼       ▼        │
                        │  └────────┬────────┘     ┌─────▼──────┐  ┌────────────┐     │
                        │       :1883│:8883        │ api-devices │  │  register  │     │
                        │           ▼              └──────┬──────┘  └─────┬──────┘     │
                        │     ┌───────────┐          :1883│:8883         │             │
                        │     │ mosquitto │◀──────────────┘              │             │
                        │     └─────┬─────┘                              │             │
                        │       ▲   ▲                                    │             │
                        │  :1883│   │:1883                               │             │
                        │  :8883│   │:8883                               │             │
                        │       │   │                                    │             │
                        │  ┌────┘   └────────┐                           │             │
                        │  │                 │                           │             │
                        │  │   ┌──────────┐  │  ┌───────────────────┐    │             │
                        │  │   │ producer │──┼─▶│    rabbitmq       │    │             │
                        │  │   └──────────┘  │  │  :5672 / :15672   │    │             │
                        │  │            :5672│  └─────────┬─────────┘    │             │
                        │  │                 │        :5672│              │             │
                        │  │                 │   ┌────────▼────────┐     │             │
                        │  │                 │   │    consumer     │     │             │
                        │  │                 │   └─────────────────┘     │             │
                        │  │                 │                           │             │
                        │  │  ┌──────────────┴──┐                       │             │
                        │  │  │ online-receiver │                       │             │
                        │  │  └────────┬────────┘                       │             │
                        │  │       :6379│                                │             │
                        │  │           ▼                                │             │
                        │  │     ┌──────────┐    ┌──────────┐           │             │
                        │  │     │  redis   │◀───│  online  │           │             │
                        │  │     └────┬─────┘    └──────────┘           │             │
                        │  │      :6379│               ▲                │             │
                        │  │          ▼                │:80             │             │
                        │  │   ┌──────────────┐        │                │             │
                        │  │   │ online-alarm │   api-server            │             │
                        │  │   └──────────────┘                         │             │
                        │  │          │                                  │             │
                        │  │          │:443                              │:27017       │
                        └──┼──────────┼──────────────────────────────────┼─────────────┘
                           │          │                                  │
                           │          ▼                                  ▼
                  External │   Google FCM                          MongoDB Atlas
                  services │   (:443)                              (:27017)
                           │
                           │   GitHub OAuth2
                           │   (:443)
                           ▼
```

## Per-Service Policy Details

### Infrastructure Policies

#### 1. `default-deny-all`

Applies to every pod in the namespace. Blocks all ingress and egress. All other policies are additive exceptions to this baseline.

#### 2. `allow-dns-egress`

Allows all pods to reach any destination on port 53 (UDP and TCP) for DNS resolution. This is necessary for Kubernetes service discovery via kube-dns. The rule intentionally does not restrict the destination to kube-system pods, as the kube-dns ClusterIP varies across clusters.

#### 3. `allow-ngf-dataplane`

NGINX Gateway Fabric deploys data-plane nginx pods in the `home-anthill` namespace (one per Gateway resource: `webapp-gateway-nginx` and `mqtt-gateway-nginx`). These pods are covered by `default-deny-all` and need explicit rules.

- **Ingress**: Accepts traffic from the internet (`0.0.0.0/0`) on ports 80 (HTTP), 443 (HTTPS), 1883 (MQTT), and 8883 (MQTTS).
- **Ingress from cert-manager** (SSL only): An additional ingress rule explicitly allows traffic from the `cert-manager` namespace on port 80. This is required by Cilium's identity model: `ipBlock: 0.0.0.0/0` only matches non-pod (external) traffic. Pod-to-pod traffic carries a Cilium identity, so cert-manager's in-cluster ACME self-check must be matched via `namespaceSelector`, not `ipBlock`.
- **Egress to NGF controller**: The nginx agent inside each data-plane pod must connect back to the NGF controller (`ngf-nginx-gateway-fabric.nginx-gateway.svc`) via gRPC to receive its routing configuration. The service exposes port 443 (named `agent-grpc`) which maps to **targetPort 8443** on the controller pod. Because Kubernetes NetworkPolicy evaluates egress against the post-DNAT target port, the egress rule must use port **8443** (not 443), scoped to the `nginx-gateway` namespace via `namespaceSelector`.
- **Egress to backends**: Can only reach the three direct backend pods — `gui:80`, `admission-nginx:80`, and `mosquitto:1883/8883`. Note that `api-server` is NOT a direct NGF backend; traffic reaches it through the gui nginx proxy (`NGF → gui → api-server`).
- **SSL conditional**: When SSL is enabled, an additional egress rule allows NGF to reach cert-manager ACME HTTP-01 solver pods on port 8089 (see [Certificate Renewal](#certificate-renewal-flows)).

Both webapp and mqtt data-plane pods share the same label (`app.kubernetes.io/managed-by: ngf-nginx`), so a single policy covers both. This means the webapp-gateway pod can technically reach mosquitto and vice versa, but both are trusted infrastructure pods.

### Public Web / API Tier

#### 4. `allow-gui`

The React frontend served by nginx.

- **Ingress**: From NGF data-plane pods and kubelet (health probes) on port 80.
- **Egress**: Only to `api-server:80` — nginx proxies `/api` requests to the API server. All external API calls are made by the browser, not the pod.

#### 5. `allow-api-server`

Central REST API handling OAuth2, homes, rooms, devices, and profiles.

- **Ingress**: From `gui` (nginx proxy) and kubelet on port 80.
- **Egress**: To `api-devices:50051` (gRPC), `register:80` (HTTP), `online:80` (HTTP), MongoDB Atlas (port 27017), and GitHub OAuth2 (port 443).
- **External egress**: Uses a bare `ports` rule (no `to` selector) for MongoDB and GitHub, allowing any destination on those ports. This is necessary because MongoDB Atlas and GitHub IP ranges change over time.

#### 6. `allow-admission-nginx`

NGINX sidecar that translates HTTP requests from the Gateway into gRPC calls for the admission service.

- **Ingress**: From NGF data-plane pods and kubelet on port 80.
- **Egress**: Only to `admission:80`.

#### 7. `allow-admission`

gRPC admission service for device/sensor registration.

- **Ingress**: From `admission-nginx` and kubelet on port 80.
- **Egress**: To `api-devices:50051` (gRPC), `register:80` (HTTP), and MongoDB Atlas (port 27017).

### Internal Cluster API Tier

#### 8. `allow-api-devices`

gRPC service for IoT device management and MQTT command publishing.

- **Ingress**: From `api-server`, `admission`, and kubelet on port 50051 (gRPC).
- **Egress**: To `mosquitto:1883` (MQTT publish, always via internal service) and MongoDB Atlas (port 27017).

#### 9. `allow-register`

Rust/Rocket service for sensor registration and data retrieval.

- **Ingress**: From `admission`, `api-server`, and kubelet on port 80.
- **Egress**: To MongoDB Atlas (port 27017).

#### 10. `allow-online`

Rust/Rocket service tracking device online status and managing FCM tokens.

- **Ingress**: From `api-server` and kubelet on port 80.
- **Egress**: Only to `redis:6379`.

#### 11. `allow-online-receiver`

MQTT subscriber that bridges device presence messages into Redis.

- **Ingress**: Kubelet only on port 80 (no service callers — inbound data comes via MQTT).
- **Egress**: To `mosquitto:1883` (MQTT subscribe, always via internal service) and `redis:6379`.

#### 12. `allow-online-alarm`

Polls Redis for offline devices and sends FCM push notifications.

- **Ingress**: Kubelet only on port 80 (no service callers).
- **Egress**: To `redis:6379` and Google FCM (port 443). The FCM egress uses a bare `ports` rule because Google's IP ranges are dynamic.

### Messaging Tier

#### 13. `allow-producer`

Rust service bridging MQTT messages to RabbitMQ.

- **Ingress**: None — this is a pure outbound worker with no health probes.
- **Egress**: To `mosquitto:1883` (MQTT subscribe, always via internal service) and `rabbitmq:5672` (AMQP publish).

#### 14. `allow-consumer`

Rust service consuming from RabbitMQ and persisting sensor data to MongoDB.

- **Ingress**: None — pure outbound worker with no health probes.
- **Egress**: To `rabbitmq:5672` (AMQP consume) and MongoDB Atlas (port 27017).

### Data Tier

#### 15. `allow-mosquitto`

Eclipse Mosquitto MQTT broker.

- **Ingress**: From NGF (external ESP32 device traffic via TCPRoute) and from internal MQTT clients (`api-devices`, `producer`, `online-receiver`) on ports 1883/8883.
- **Egress**: None — Mosquitto is a pure broker that only responds to incoming connections.

When SSL is enabled, a `k8s-config-reloader` sidecar runs alongside Mosquitto. It monitors the TLS cert files via filesystem inotify (no network access required) and sends SIGHUP to Mosquitto for cert reload. The pod sets `shareProcessNamespace: true` and the sidecar has `capabilities: add: ["KILL"]` to enable cross-container signaling.

#### 16. `allow-redis`

In-cluster Redis cache for device online status.

- **Ingress**: From `online`, `online-receiver`, and `online-alarm` on port 6379.
- **Egress**: None — Redis is a pure cache that only responds to incoming connections.

#### 17. `allow-rabbitmq`

AMQP message broker managed by the RabbitMQ Kubernetes Operator.

- **Ingress**: From `producer` and `consumer` on port 5672 (AMQP). Also from the RabbitMQ Operator namespace on ports 5672 and 15672 (management API) for health checks and reconciliation.
- **Egress**: None — RabbitMQ only responds to incoming connections. DNS for internal Erlang operations uses localhost (not affected by NetworkPolicy).

The Operator namespace selector uses `kubernetes.io/metadata.name`, a built-in label that Kubernetes automatically applies to all namespaces.

### SSL-Only Policies

#### 18. `allow-cert-manager-solver`

Only rendered when `domains.http.ssl.enable` or `domains.mqtt.ssl.enable` is `true`.

During certificate issuance/renewal, cert-manager creates temporary solver pods labeled `acme.cert-manager.io/http01-solver: "true"` in the `home-anthill` namespace. Without this policy, `default-deny-all` would block all traffic to these pods, causing ACME challenges to fail.

- **Ingress**: From NGF data-plane pods on port 8089 (the default cert-manager solver port).
- **Egress**: None — solver pods only serve challenge responses.

## Certificate Renewal Flows

The chart supports TLS certificates for both HTTP and MQTT gateways, issued by Let's Encrypt via cert-manager ACME HTTP-01 challenges.

### HTTP Certificate Renewal

```
Let's Encrypt                  home-anthill namespace
     │
     │ HTTP GET /.well-known/acme-challenge/<token>
     │
     ▼
 HTTP public IP
     │
     ▼ (Cilium L2)
 webapp-gateway ──────────────▶ solver pod (:8089)
 (NGF, port 80)                 (temporary)
     │
     │  Note: the HTTP→HTTPS redirect HTTPRoute
     │  does NOT interfere — the solver's Exact
     │  path match (/.well-known/acme-challenge/*)
     │  takes precedence over the redirect's
     │  catch-all PathPrefix (/) per Gateway API
     │  path specificity rules.
     │
     ▼
 webapp-gateway ◀─── cert-manager updates
 (port 443)          webapp-tls Secret
                     │
                     └─▶ NGF auto-reloads nginx config
```

**Resources involved:**
- `Certificate` (`webapp-tls`) with `dnsNames: [domain, www.domain]`
- `Issuer` (`cert-issuer-webapp`) using `gatewayHTTPRoute` solver with `sectionName: http`
- The Gateway's HTTP listener (port 80) serves both the redirect and ACME challenges; path specificity ensures no conflict

**Network policies involved:**
- `allow-ngf-dataplane` (#3): egress to solver pods on 8089
- `allow-cert-manager-solver` (#18): ingress from NGF on 8089

### MQTT Certificate Renewal

```
Let's Encrypt                  home-anthill namespace
     │
     │ HTTP GET /.well-known/acme-challenge/<token>
     │
     ▼
 MQTT public IP
     │
     ▼ (Cilium L2)
 mqtt-gateway ──────────────▶ solver pod (:8089)
 (NGF, port 80)               (temporary)
     │
     │  The mqtt-gateway's HTTP listener on port 80
     │  exists solely for ACME challenges.
     │  No other HTTPRoutes compete for this listener.
     │
     ▼
 cert-manager updates
 mqtt-tls Secret
     │
     ▼
 kubelet syncs mounted Secret volume
     │
     ▼
 k8s-config-reloader sidecar
 detects file change (inotify)
     │
     ▼
 sends SIGHUP to mosquitto
 (shareProcessNamespace: true)
     │
     ▼
 mosquitto reloads TLS cert
```

**Resources involved:**
- `Certificate` (`mqtt-tls`) with `dnsNames: [mqtt-domain]`
- `Issuer` (`cert-issuer-mqtt`) using `gatewayHTTPRoute` solver
- The Gateway has a dedicated HTTP listener (port 80) solely for ACME challenges, plus a TCP listener (port 8883) for MQTTS
- The `mqtt-tls` Secret is mounted into the mosquitto pod at `/etc/mosquitto/certs/` with mode 0600
- The `k8s-config-reloader` sidecar watches the cert files and sends SIGHUP to mosquitto for hot-reload

**Network policies involved:**
- `allow-ngf-dataplane` (#3): egress to solver pods on 8089
- `allow-cert-manager-solver` (#18): ingress from NGF on 8089

**Key difference from HTTP renewal:** NGF terminates TLS for the webapp gateway, so it auto-reloads when the Secret changes. For MQTT, Mosquitto terminates TLS internally, so the `k8s-config-reloader` sidecar is needed to signal a cert reload.

## Kubelet Health Probes

Most services expose HTTP health endpoints that kubelet checks for readiness and liveness. The `nodesCIDR` value controls which IP range is allowed for these probes.

| Service | Probe Port | Probe Path | Protocol |
|---------|-----------|------------|----------|
| gui | 80 | `/keepalive` | HTTP |
| api-server | 80 | `/api/keepalive` | HTTP |
| admission-nginx | 80 | `/admission/keepalive` | HTTP |
| admission | 80 | `/admission/keepalive` | HTTP |
| api-devices | 50051 | — | gRPC |
| register | 80 | `/keepalive` | HTTP |
| online | 80 | `/keepalive` | HTTP |
| online-receiver | 80 | `/keepalive` | HTTP |
| online-alarm | 80 | `/keepalive` | HTTP |
| mosquitto | — | — | exec (`mosquitto_pub`) |
| redis | — | — | exec (`redis-cli ping`) |
| producer | — | — | none |
| consumer | — | — | none |

Services with exec probes (mosquitto, redis) do not need kubelet network access for probes — the check runs inside the container. Services with no probes (producer, consumer) rely on container exit codes for restart decisions.

## Cross-Namespace Traffic

Only two policies reference pods in other namespaces:

| Policy | Remote Namespace | Direction | Purpose |
|--------|-----------------|-----------|---------|
| `allow-ngf-dataplane` (#3) | `nginx-gateway` | Egress | nginx agent → NGF controller gRPC on targetPort 8443 |
| `allow-ngf-dataplane` (#3) | `cert-manager` | Ingress | cert-manager ACME self-check HTTP on port 80 (requires `namespaceSelector`; `ipBlock` doesn't match pod traffic in Cilium) |
| `allow-rabbitmq` (#17) | `rabbitmq-system` | Ingress | RabbitMQ Operator health checks and management API on ports 5672/15672 |
| `allow-cert-manager-solver` (#18) | (same namespace) | Ingress | cert-manager creates solver pods directly in `home-anthill` |

The NGF data-plane pods run in `home-anthill` (not `nginx-gateway`), so backend egress rules (gui, admission-nginx, mosquitto) use same-namespace `podSelector` without `namespaceSelector`. However, the nginx agent must connect to the NGF controller in the `nginx-gateway` namespace via gRPC to receive its configuration — this is the one cross-namespace egress rule in this chart.

> **Port note**: The `ngf-nginx-gateway-fabric` service exposes port 443 (named `agent-grpc`) which maps to `targetPort: 8443` on the controller pod. Kubernetes NetworkPolicy evaluates egress against the **target port** (post-DNAT), so the egress rule must specify port `8443`, not `443`. Using the service port (443) would be silently dropped.

## External Egress Rules

Several policies allow egress to external services using bare `ports` rules (no `to` selector). This permits reaching any host on the specified port.

| Service | External Port | Destination |
|---------|:------------:|-------------|
| api-server | 27017 | MongoDB Atlas |
| api-server | 443 | GitHub OAuth2 |
| admission | 27017 | MongoDB Atlas |
| api-devices | 27017 | MongoDB Atlas |
| register | 27017 | MongoDB Atlas |
| consumer | 27017 | MongoDB Atlas |
| online-alarm | 443 | Google FCM |

These rules are intentionally broad because the IP ranges for cloud services (MongoDB Atlas, GitHub, Google) are dynamic and change over time. Pinning them to specific CIDRs would require ongoing maintenance and risk breaking connectivity.
