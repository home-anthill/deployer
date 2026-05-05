# Changelog — AI-Assisted Improvements

Improvements made with GitHub Copilot / Claude.
Issues reference the audit table in [SECURITY.md](SECURITY.md).

---

## Security

### MQTT access control

- Added role-based Mosquitto ACLs for authenticated deployments: shared firmware devices can publish telemetry (`sensors/+/+`, `online/+/features/+`) and read device commands (`devices/+/values`), while backend services get separate producer, online-receiver, and api-devices MQTT credentials scoped to their required topics.
- Updated the Mosquitto Helm deployment to pass all broker users through `MOSQUITTO_USERS` and mount a generated `mosquitto-acl` ConfigMap.
- Updated `producer`, `online-receiver`, and `api-devices` Helm configs to use separate MQTT role credentials instead of the former shared broker credential.
- Updated Helm smoke tests to authenticate as the shared device MQTT role and publish to an ACL-allowed sensor topic.

### Container hardening

- Added `runAsNonRoot: true` + `runAsUser/Group` at pod level to all Deployments — Go services run as uid `65534` (nobody), `mosquitto` as `1883`, `redis` as `999`; NGINX pods get `seccompProfile` only since the master process requires root for worker management
- Added `readOnlyRootFilesystem: true` to all containers; writable `emptyDir` volumes mounted at `/tmp`, `/app/logs` (Rust rolling log appender), and nginx cache/run/log paths
- Added `seccompProfile: RuntimeDefault` at pod level to all 13 Deployments, activating the container runtime's default syscall allowlist
- Added `allowPrivilegeEscalation: false` and `capabilities.drop: ["ALL"]` to all containers and init containers; `NET_BIND_SERVICE` re-added only where port 80 is bound, `KILL` only on `k8s-config-reloader`
- `k8s-config-reloader` sidecar: `runAsUser: 0` made explicit (required to send SIGKILL across process namespace); all other containers run as non-root
- Redis pod `securityContext` (`runAsUser/Group: 999`, `runAsNonRoot`) made unconditional — previously only applied when `dhi.enabled: true`
- Added `automountServiceAccountToken: false` to all pods
- Replaced Redis and Mosquitto credential-bearing `exec` probes with TCP socket probes, removing passwords from normal Pod probe command arrays

### Network

- Added 17–18 `NetworkPolicy` resources implementing default-deny-all ingress + egress with explicit per-service allow rules; controlled by `network.enabled` in `values.yaml`
- Restricted external egress with Cilium FQDN policies for GitHub, MongoDB Atlas, and Google/Firebase destinations, and removed the port-only `443` / `27017` fallbacks from the standard Kubernetes `NetworkPolicy` objects
- Fixed MongoDB Atlas egress to allow client traffic on ports 27015-27017, added explicit DNS visibility for Cilium FQDN policy, and matched multi-level Atlas `mongodb.net` hostnames used by SRV records and shard targets
- Fixed online-alarm FCM egress by adding explicit DNS visibility to the Google/Firebase Cilium FQDN policy before allowing HTTPS to `*.googleapis.com` and `accounts.google.com`
- Added Helm test hook NetworkPolicies for the deployment smoke-test Job so it can reach only the services it validates during `helm test`
- Added `Strict-Transport-Security: max-age=31536000; includeSubDomains` to all HTTPS `ResponseHeaderModifier` blocks in the webapp Gateway (SSL branch only)
- Added `ClientSettingsPolicy` to enforce a 100 MiB maximum request body on the webapp Gateway
- Enforced TLS 1.2/1.3 and AEAD cipher suites on HTTPS Gateway listeners using NGF-supported TLS options
- Moved `Content-Security-Policy` from Gateway `ResponseHeaderModifier` rules into the `gui` and `admission-nginx` NGINX ConfigMaps, keeping CSP close to the HTTP servers that own those responses
- Tightened CSP by removing `unsafe-inline` from `script-src`
- Removed the CSP `sandbox` directive because it does not add useful protection for these same-origin application responses and can interfere with normal app behavior
- Added `Referrer-Policy: same-origin` from the `gui` and `admission-nginx` NGINX ConfigMaps
- Kept Gateway-level `X-Frame-Options`, `X-Content-Type-Options`, `Permissions-Policy`, and SSL-only HSTS headers in `gateway-webapp.yaml`

### Access control

- Added dedicated `ServiceAccount` per workload (13 total); controlled by `serviceAccounts.enabled` in `values.yaml`
- Added `resources.requests` and `resources.limits` (CPU + memory) to every container, configurable in `values.yaml`
- Added Pod Security Admission labels to the namespace: enforce `baseline`, audit/warn `restricted`
- Added explicit `REFRESH_TOKEN_HASH_SECRET` rendering for `api-server` so refresh/app-login token lookup hashes use a dedicated deployment secret instead of relying on fallback JWT secrets
- Added namespace guardrails with `ResourceQuota` and `LimitRange` to bound total namespace usage and provide sane default per-container requests/limits
- Added explicit Gateway listener `allowedRoutes.namespaces.from: Same` on webapp and MQTT Gateways so only same-namespace routes can attach
- Switched Redis to `protected-mode yes` while keeping ACL auth enabled, so Redis remains service-reachable but is no longer configured in the broadest mode
- Replaced Redis and Mosquitto `hostPath` PersistentVolumes with `local-path` PVCs, removing direct node-path bindings from the chart while keeping single-node k3s storage simple

---

## Bug Fixes

- Fixed `accessModes: ReadWriteMany` → `ReadWriteOnce` on Redis and Mosquitto PersistentVolumeClaims
- Fixed spurious RabbitMQ management port 8080 exposed on `register` Service
- Fixed `producer` Service incorrectly exposing AMQP port 5672 (producer is a client, not a server)
- Removed unnecessary Service resources from worker-only `producer` and `consumer` pods
- Narrowed default `network.nodesCIDR` from `10.0.0.0/8` to `10.0.0.0/24` for kubelet health-probe ingress
- Fixed `imagePullPolicy: Always` missing on RabbitMQ
- Fixed `api-server` deployment env drift: OAuth callback URLs now match `/api/oauth/callback` and `/api/oauth/app/callback`, and the default `COOKIE_SECRET` placeholder satisfies the current 32-character startup validation

## Infrastructure

- Replaced MetalLB with Cilium for bare-metal load balancing: added `CiliumLoadBalancerIPPool` and `CiliumL2AnnouncementPolicy`; removed MetalLB `IPAddressPool` and `L2Advertisement` resources
- Migrated from `nginx-ingress` to Gateway API using NGINX Gateway Fabric (`Gateway`, `HTTPRoute`, `TCPRoute`, `ClientSettingsPolicy`, `SnippetsFilter`)
- Enforced web Gateway TLS 1.2/1.3 and AEAD cipher suites through NGF-supported Gateway listener TLS options

---

## Reliability

- Enabled Redis persistence with RDB snapshots on the mounted `/data` volume, so Redis state now survives pod restarts instead of being treated as disposable cache data
- Raised the small workload CPU requests from `5m` to `10m` so they satisfy the namespace `LimitRange` minimum and can actually be scheduled
- Added liveness/readiness probes to `admission-nginx`
- Added liveness/readiness probes to `online-receiver` (Rocket HTTP health server)
- Added Helm deployment smoke tests for GUI, API server, online service, Redis auth, Mosquitto auth, and RabbitMQ management auth
