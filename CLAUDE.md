# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a Kubernetes Helm chart (`home-anthill/`) for deploying the **home-anthill** home automation platform — a distributed microservices system for IoT/sensor management, messaging, and commands.

## Commands

### Validate the Helm chart (CI equivalent)
```bash
cd home-anthill && helm template -f values.yaml home-anthill .
```

### Lint the Helm chart
```bash
helm lint home-anthill/
```

### Render a specific template for inspection
```bash
helm template -f home-anthill/values.yaml home-anthill home-anthill/ -s templates/<filename>.yaml
```

## Cluster Prerequisites

These components must be pre-installed on the K8s cluster before deploying this chart:

| Component | Why required |
|-----------|-------------|
| NGINX Gateway Fabric (NGF) | Provides `Gateway`, `HTTPRoute`, `TCPRoute`, `ClientSettingsPolicy`, `SnippetsFilter` CRDs |
| Gateway API CRDs (Experimental channel) | `TCPRoute` is in the experimental channel; must be installed before NGF |
| cert-manager ≥ 1.15 with `enableGatewayAPI=true` | Issues Let's Encrypt TLS certs for HTTP and MQTT gateways |
| Cilium (with L2 announcements enabled) | Bare-metal load balancer; assigns public IPs via `CiliumLoadBalancerIPPool` + `CiliumL2AnnouncementPolicy` |
| RabbitMQ Kubernetes Operator | Required for `RabbitmqCluster`, `User`, `Permission` CRDs used in `rabbitmq.yaml` |

**NGF install note**: `SnippetsFilter` is alpha — must be explicitly enabled:
```bash
helm install ngf ... --set nginxGateway.snippetsFilters.enable=true
```

## Architecture

The chart deploys ~20 resources to a Kubernetes cluster, organized into these layers:

**Public Web / API tier**
- `gui` — frontend web UI (served at domain root via Gateway)
- `api-server` — main REST API with OAuth2, connects to MongoDB Atlas
- `admission` — gRPC admission service exposed at `/admission`; uses a **dedicated NGINX sidecar** (`admission-nginx`) that proxies HTTP→gRPC, because the main Gateway routes HTTP traffic and cannot forward raw gRPC directly to the admission pod

**Internal cluster API tier**
- `api-devices` — gRPC service for IoT device management
- `register` — Rust/Rocket service for sensor registration (MongoDB)
- `online` — Rust/Rocket service tracking device online status (Redis); exposes FCM token management
- `online-receiver` — MQTT → Redis bridge; **no Service resource** (inbound only via MQTT)
- `online-alarm` — polls Redis for offline devices, sends FCM push notifications

**Messaging tier**
- `rabbitmq` — AMQP broker managed by RabbitMQ Operator (`RabbitmqCluster` CRD); 3 users: `admin` (full access), `producer` (write-only to `^ks89$` queue), `consumer` (read-only from `^ks89$` queue)
- `mosquitto` — MQTT broker; when SSL is enabled, a **`k8s-config-reloader` sidecar** watches the TLS cert Secret and sends SIGHUP to Mosquitto on cert rotation
- `producer` / `consumer` — RabbitMQ publish/consume services

**Data tier**
- `redis` — in-cluster cache; `redis-pv.yaml` provisions a 2Gi `hostPath` PersistentVolume + PVC
- MongoDB — external (Atlas Cloud), referenced by `values.mongodbUrl`
- `mosquitto-pv.yaml` — 2Gi `hostPath` PersistentVolume + PVC for Mosquitto data

**Infrastructure**
- `gateway-webapp.yaml` — `Gateway` + `HTTPRoute` (HTTP→HTTPS 301 redirect when SSL on) + `HTTPRoute` (routes `/admission` → admission-nginx-svc, `/` → gui-svc) + `ClientSettingsPolicy` (100 MiB max request body) + `SnippetsFilter` (100 req/s rate limit) + `Issuer` + `Certificate` for Let's Encrypt
- `gateway-mqtt.yaml` — `Gateway` + `TCPRoute` for raw TCP MQTT/MQTTS; a separate HTTP listener exists solely for the Let's Encrypt HTTP-01 ACME challenge; explicit `Certificate` resource (not the cert-manager Gateway shim) to avoid duplicate `parentRef` errors
- `cilium.yaml` — `CiliumLoadBalancerIPPool` (assigns public IPs from a `/32` block per gateway) + `CiliumL2AnnouncementPolicy` (ARP broadcasts on `eth0` for LoadBalancer IPs); replaces the MetalLB approach
- `network-policy.yaml` — 17–18 `NetworkPolicy` resources implementing default-deny-all with explicit per-service allow rules; controlled by `network.enabled`; see `NETWORK_POLICIES.md` for the full communication matrix and design rationale

## Key Files

| File | Purpose |
|------|---------|
| `home-anthill/Chart.yaml` | Chart version and app version |
| `home-anthill/values.yaml` | All configurable values: domain, image tags, auth mode, secrets |
| `home-anthill/templates/*.yaml` | One file per service (see notable patterns below) |
| `.github/workflows/docker-image.yml` | CI: runs `helm template` to validate chart renders cleanly |
| `MIGRATION.md` | Lessons learned migrating from nginx-ingress to Gateway API |
| `NETWORK_POLICIES.md` | Full communication matrix, per-policy rationale, cert renewal flows, kubelet probe table |

## Values Structure

`values.yaml` top-level keys:

| Key | Purpose |
|-----|---------|
| `namespace` | Target K8s namespace (default: `home-anthill`) |
| `domains.http` / `domains.mqtt` | Domain name, public IP, SSL enable flag for each gateway |
| `letsencrypt` | ACME server URL (prod vs staging) |
| `dhi` | DHI.io hardened image registry credentials (optional) |
| `mosquitto` | Image, service name, data path, MQTT auth credentials |
| `redis` | Image (standard + hardened variants), service name, data path, auth credentials |
| `rabbitmq` | Image, admin/producer/consumer credentials, AMQP HMAC secret |
| `mongodbUrl` | External MongoDB Atlas connection string |
| `gui` / `apiServer` / `apiDevices` / `admission` / `register` / `producer` / `consumer` / `online` / `onlineReceiver` / `onlineAlarm` | Per-service image tag, service name, and service-specific secrets |
| `apiServer` | Also: `singleUserLoginEmail`, JWT/cookie secrets, OAuth2 client IDs for web + Android app, `sensorsEnable` |
| `onlineAlarm` | Also: `fcmServiceAccountKey` (full Firebase JSON, base64-encoded in Secret) |
| `serviceAccounts.enabled` | Creates `ServiceAccount` resources and binds them to pods when `true` (default: `false`) |
| `network.enabled` | Deploys all `NetworkPolicy` resources (default: `true`; set `false` for dev/test clusters) |
| `network.nodesCIDR` | CIDR for kubelet health-probe traffic (default: `10.0.0.0/8`; restrict in production) |
| `network.rabbitmqOperatorNamespace` | Namespace of the RabbitMQ Operator (used in cross-namespace NetworkPolicy) |
| `alpine` / `nginx` / `k8sConfigReloader` | Shared utility images with resource limits |
| `debug.pods.alwaysPullContainers` | Forces `imagePullPolicy: Always` on every pod |
| `debug.pods.sleepInfinity` | Overrides container command with `sleep Infinity` for debugging (register, producer, consumer, online, online-receiver, online-alarm) |

All service images are pulled from Docker Hub (`ks89/<service>:<tag>`).

## Notable Template Patterns

**MQTT TLS init container** — `api-devices.yaml`, `producer.yaml`, `online-receiver.yaml`: when `domains.mqtt.ssl=true`, an init container (`alpine`) copies the cluster TLS cert into a shared `emptyDir` volume before the main container starts.

**Rocket.toml as ConfigMap** — `register.yaml`, `online.yaml`, `online-alarm.yaml`: Rocket framework config (port, address, `secret_key`, database pool URL) is mounted as a ConfigMap subPath at `/app/Rocket.toml`. The `secret_key` comes from `values.yaml` and must be a base64-encoded 256-bit key (`openssl rand -base64 32`).

**RabbitMQ Operator CRDs** — `rabbitmq.yaml` uses `RabbitmqCluster` (not a Deployment), plus `User` and `Permission` resources. The `Permission` resources enforce queue-level ACLs: producer can only write to `^ks89$`, consumer can only read from `^ks89$`.

**cert-manager for MQTT** — `gateway-mqtt.yaml` uses an explicit `Certificate` resource (not the cert-manager Gateway shim) because the MQTT Gateway has no TLS listener (Mosquitto handles TLS termination itself); the explicit cert avoids the Gateway shim trying to add a conflicting `parentRef`.

**Admission NGINX sidecar** — `admission.yaml` defines two Deployments: the Go gRPC service and an NGINX sidecar. The sidecar translates the HTTP requests arriving from the Gateway into gRPC calls to the main container on port 50051.
