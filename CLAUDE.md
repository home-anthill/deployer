# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a Kubernetes Helm chart (`home-anthill/`) for deploying the **home-anthill** home automation platform - a distributed microservices system for IoT/sensor management, messaging, device presence, alarms, and commands.

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

### Run deployment smoke tests
```bash
helm test home-anthill -n default
```

## Cluster Prerequisites

These components must be pre-installed on the K8s cluster before deploying this chart:

| Component | Why required |
|-----------|-------------|
| NGINX Gateway Fabric (NGF) | Provides `Gateway`, `HTTPRoute`, `TCPRoute`, `ClientSettingsPolicy`, `SnippetsFilter` CRDs |
| Gateway API CRDs (Experimental channel) | `TCPRoute` is in the experimental channel; must be installed before NGF |
| cert-manager >= 1.15 with Gateway API support | Issues Let's Encrypt TLS certs for HTTP and MQTT gateways |
| Cilium (with L2 announcements enabled) | Bare-metal load balancer; assigns public IPs via `CiliumLoadBalancerIPPool` + `CiliumL2AnnouncementPolicy` |
| RabbitMQ Kubernetes Operator | Required for `RabbitmqCluster`, `User`, `Permission` CRDs used in `rabbitmq.yaml` |

**NGF install note**: `SnippetsFilter` is alpha — must be explicitly enabled:
```bash
helm install ngf ... --set nginxGateway.snippetsFilters.enable=true
```

## Architecture

The chart deploys dozens of resources to a Kubernetes cluster, organized into these layers:

**Public Web / API tier**
- `gui` - frontend web UI served at domain root via Gateway; NGINX also proxies `/api` to `api-server`
- `api-server` - main REST API with OAuth2, connects to MongoDB Atlas and internal services
- `admission` - admission service exposed at `/admission`; traffic reaches a **dedicated NGINX proxy deployment** (`admission-nginx`) first, then forwards to the admission HTTP service

**Internal cluster API tier**
- `api-devices` - gRPC service for IoT device management and MQTT command publishing
- `register` - Rust/Rocket service for sensor registration (MongoDB)
- `online` - Rust/Rocket service tracking device online status (Redis); exposes FCM token management
- `online-receiver` - MQTT -> Redis bridge; no Service resource (inbound data comes from MQTT; HTTP port is for probes)
- `online-alarm` - polls Redis for offline devices, sends FCM push notifications

**Messaging tier**
- `rabbitmq` - AMQP broker managed by RabbitMQ Operator (`RabbitmqCluster` CRD); 3 users: `admin` (full access), `producer` (scoped to `ks89` / `amq.default` publishing and `ks89` queue access), `consumer` (no writes, scoped to `^ks89$` configure/read)
- `mosquitto` - MQTT broker; when SSL is enabled, a **`k8s-config-reloader` sidecar** watches the TLS cert Secret and signals Mosquitto on cert rotation
- `producer` / `consumer` - RabbitMQ publish/consume services

**Data tier**
- `redis` - in-cluster cache; `redis-pvc.yaml` provisions a 2Gi PVC bound through the chart `storageClassName` value
- MongoDB - external (Atlas Cloud), referenced by `values.mongodbUrl`
- `mosquitto-pvc.yaml` - 2Gi PVC bound through the chart `storageClassName` value for Mosquitto data

**Infrastructure**
- `gateway-webapp.yaml` - `Gateway` + `HTTPRoute` (HTTP->HTTPS 301 redirect when SSL on) + `HTTPRoute` (routes `/admission` -> admission-nginx-svc, `/` -> gui-svc) + Gateway-level security headers (`Strict-Transport-Security` when SSL is on, `X-Frame-Options`, `X-Content-Type-Options`, `Permissions-Policy`) + `ClientSettingsPolicy` (100 MiB max request body) + `SnippetsFilter` (100 req/s rate limit) + `Issuer` + `Certificate` for Let's Encrypt
- `gateway-mqtt.yaml` - `Gateway` + `TCPRoute` for raw TCP MQTT/MQTTS; a separate HTTP listener exists solely for the Let's Encrypt HTTP-01 ACME challenge; explicit `Certificate` resource (not the cert-manager Gateway shim) to avoid duplicate `parentRef` errors; listener `allowedRoutes.namespaces.from: Same` is set on every listener
- `cilium.yaml` - `CiliumLoadBalancerIPPool` (assigns public IPs from a `/32` block per gateway) + `CiliumL2AnnouncementPolicy` (ARP broadcasts on `eth0` for LoadBalancer IPs); replaces the MetalLB approach
- `cilium-egress.yaml` - `CiliumNetworkPolicy` rules for external FQDN egress to GitHub, MongoDB Atlas, and Google/Firebase; DNS visibility is explicitly allowed so FQDN rules can resolve correctly
- `network-policy.yaml` - 17-18 `NetworkPolicy` resources implementing default-deny-all with explicit per-service allow rules; controlled by `network.enabled`; see `NETWORK_POLICIES.md` for the full communication matrix and design rationale
- `namespace-guardrails.yaml` - namespace `ResourceQuota` and `LimitRange` objects to cap total usage and set sane per-container defaults/minimums

## Key Files

| File | Purpose |
|------|---------|
| `home-anthill/Chart.yaml` | Chart version and app version |
| `home-anthill/values.yaml` | All configurable values: domain, image tags, auth mode, secrets |
| `home-anthill/templates/*.yaml` | One file per service (see notable patterns below) |
| `.github/workflows/docker-image.yml` | CI: runs `helm template` to validate chart renders cleanly |
| `SECURITY.md` | Security audit table with fixed and open findings |
| `NETWORK_POLICIES.md` | Full communication matrix, per-policy rationale, cert renewal flows, kubelet probe table |

## Values Structure

`values.yaml` top-level keys:

| Key | Purpose |
|-----|---------|
| `namespace` | Target K8s namespace (default: `home-anthill`) |
| `domains.http` / `domains.mqtt` | Domain name, public IP, SSL enable flag (`ssl.enable`) for each gateway |
| `letsencrypt` | ACME server URL (prod vs staging) |
| `dhi` | DHI.io hardened image registry credentials (optional) |
| `mosquitto` | Image, service name, data path, MQTT auth credentials |
| `redis` | Image (standard + hardened variants), service name, data path, auth credentials |
| `rabbitmq` | Image, admin/producer/consumer credentials, AMQP HMAC secret |
| `mongodbUrl` | External MongoDB Atlas connection string |
| `storageClassName` | Storage class used by Redis and Mosquitto PVCs (`local-path` by default in this repo) |
| `gui` / `apiServer` / `apiDevices` / `admission` / `register` / `producer` / `consumer` / `online` / `onlineReceiver` / `onlineAlarm` | Per-service image tag, service name where applicable, and service-specific secrets |
| `apiServer` | Also: `singleUserLoginEmail`, JWT/cookie secrets, `refreshTokenHashSecret`, OAuth2 client IDs for web + Android app, `sensors.enable` |
| `onlineAlarm` | Also: `firebaseServiceAccount` (full Firebase service account JSON, rendered into a Secret) |
| `serviceAccounts.enabled` | Creates `ServiceAccount` resources and binds them to pods when `true` (default: `true`) |
| `network.enabled` | Deploys all `NetworkPolicy` resources (default: `true`; set `false` for dev/test clusters) |
| `network.nodesCIDR` | CIDR for kubelet health-probe traffic (default: `10.0.0.0/24`; set to the target cluster's actual node subnet) |
| `network.nginxGatewayNamespace` | Namespace of the NGINX Gateway Fabric controller (used in cross-namespace NetworkPolicy) |
| `network.rabbitmqOperatorNamespace` | Namespace of the RabbitMQ Operator (used in cross-namespace NetworkPolicy) |
| `alpine` / `nginx` / `k8sConfigReloader` | Shared utility images with resource limits |
| `smokeTests` | Helm test image and resources for deployment smoke checks |
| `debug.pods.alwaysPullContainers` | Forces `imagePullPolicy: Always` on every pod |
| `debug.pods.sleepInfinity` | Overrides container command with `sleep Infinity` for debugging (register, producer, consumer, online, online-receiver, online-alarm) |

All service images are pulled from Docker Hub (`ks89/<service>:<tag>`).

## Notable Template Patterns

**MQTT TLS init container** — `api-devices.yaml`, `producer.yaml`, `online-receiver.yaml`: when `domains.mqtt.ssl=true`, an init container (`alpine`) copies the cluster TLS cert into a shared `emptyDir` volume before the main container starts.

**Internal plaintext is intentional** — Do not report internal service-to-service HTTP, gRPC, MQTT, AMQP, or Redis traffic as a security issue merely because it lacks TLS. This chart intentionally relies on namespace isolation, Kubernetes NetworkPolicy, service scoping, and broker/application credentials for in-cluster traffic. Only flag plaintext transport when it crosses the cluster boundary or when there is a concrete exposure beyond the trusted in-cluster network model.

**Default values are placeholders** — Do not report sample credentials or secrets in `home-anthill/values.yaml` as a hardcoded-secret issue by themselves. Production deployments override these values through an external private values file passed to Helm. Only flag this area if a real secret is committed outside the placeholder defaults, a template prevents private overrides, or a rendered manifest exposes sensitive values through an inappropriate Kubernetes object.

**Rocket.toml as ConfigMap** — `register.yaml`, `online.yaml`, `online-alarm.yaml`: Rocket framework config (port, address, `secret_key`, database pool URL) is mounted as a ConfigMap subPath at `/app/Rocket.toml`. The `secret_key` comes from `values.yaml` and must be a base64-encoded 256-bit key (`openssl rand -base64 32`).

**Security headers split by layer** — `gateway-webapp.yaml` sets Gateway-level browser hardening headers; `gui.yaml` and `admission.yaml` set `Referrer-Policy` and `Content-Security-Policy` in their NGINX ConfigMaps. CSP is kept at NGINX because the Gateway response-header filter is too coarse for this use case.

**Gateway route scoping** — `gateway-webapp.yaml` and `gateway-mqtt.yaml` set `allowedRoutes.namespaces.from: Same` on all listeners. That keeps route attachment inside the `home-anthill` namespace because this chart does not use cross-namespace routing.

**RabbitMQ Operator CRDs** — `rabbitmq.yaml` uses `RabbitmqCluster` (not a Deployment), plus `User` and `Permission` resources. The `Permission` resources enforce queue-level ACLs: the producer is scoped to `ks89` / `amq.default` publishing and `ks89` queue access; the consumer has no write permission and can configure/read only `^ks89$`.

**cert-manager for MQTT** — `gateway-mqtt.yaml` uses an explicit `Certificate` resource (not the cert-manager Gateway shim) because the MQTT Gateway has no TLS listener (Mosquitto handles TLS termination itself); the explicit cert avoids the Gateway shim trying to add a conflicting `parentRef`.

**Admission NGINX proxy** — `admission.yaml` defines two Deployments: the admission service and an NGINX proxy deployment. The Gateway routes `/admission` to `admission-nginx-svc`, which forwards to `admission-svc` inside the namespace.

**Deployment smoke tests** — `templates/tests/smoke-test.yaml` defines Helm test hooks. Run `helm test home-anthill -n default` after install/upgrade to validate GUI, API server, online service, Redis auth, Mosquitto auth, and RabbitMQ management auth. The test uses a temporary Secret with `secretKeyRef` env vars; credentials are not embedded in the Job command arguments. The hook also creates temporary NetworkPolicies for the smoke-test pod and removes them on success.

**MongoDB FQDN egress** — `cilium-egress.yaml` allows external MongoDB Atlas access by FQDN, not by raw port-only egress. The policy includes DNS visibility for `kube-dns` and Atlas hostname patterns so SRV records and shard targets can resolve correctly.

## Deployment Env Audit

The current Helm env maps match the service config structs and direct `os.Getenv` usage across `admission`, `api-devices`, `api-server`, `register`, `producer`, `consumer`, `online`, `online-receiver`, and `online-alarm`.

Watch these deployment-sensitive details when updating services:

- `api-server` OAuth callbacks must stay aligned with the Gin routes: `/api/oauth/callback` and `/api/oauth/app/callback`.
- `api-server.cookieSecret` must be at least 32 characters; startup rejects shorter values.
- `api-server.refreshTokenHashSecret` is rendered as `REFRESH_TOKEN_HASH_SECRET` for HMAC hashing of opaque refresh/app-login tokens. The code has a fallback, but deployments should set it explicitly.
- `redis` now runs with `protected-mode yes` and uses a PVC backed by `storageClassName`; do not reintroduce `hostPath` PVs in this chart.
- MQTT auth uses role-specific values under `mosquitto.auth.users`: `device`, `producer`, `onlineReceiver`, and `apiDevices`.
- `register.mongo_max_retries`, `online.redis_username`, `online.redis_password`, and `onlineAlarm.fcm_service_account_key_path` have code defaults or chart-provided defaults; absence is intentional only where the service struct marks the field optional/defaulted.
