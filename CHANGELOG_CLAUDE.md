# Changelog — AI-Assisted Improvements

Improvements made with GitHub Copilot (Claude Sonnet).
Issues reference the audit table in [SECURITY.md](SECURITY.md).

---

## Security

### Container hardening

- Added `runAsNonRoot: true` + `runAsUser/Group` at pod level to all Deployments — Go services run as uid `65534` (nobody), `mosquitto` as `1883`, `redis` as `999`; NGINX pods get `seccompProfile` only since the master process requires root for worker management
- Added `readOnlyRootFilesystem: true` to all containers; writable `emptyDir` volumes mounted at `/tmp`, `/app/logs` (Rust rolling log appender), and nginx cache/run/log paths
- Added `seccompProfile: RuntimeDefault` at pod level to all 13 Deployments, activating the container runtime's default syscall allowlist
- Added `allowPrivilegeEscalation: false` and `capabilities.drop: ["ALL"]` to all containers and init containers; `NET_BIND_SERVICE` re-added only where port 80 is bound, `KILL` only on `k8s-config-reloader`
- `k8s-config-reloader` sidecar: `runAsUser: 0` made explicit (required to send SIGKILL across process namespace); all other containers run as non-root
- Redis pod `securityContext` (`runAsUser/Group: 999`, `runAsNonRoot`) made unconditional — previously only applied when `dhi.enabled: true`
- Added `automountServiceAccountToken: false` to all pods

### Network

- Added 17–18 `NetworkPolicy` resources implementing default-deny-all ingress + egress with explicit per-service allow rules; controlled by `network.enabled` in `values.yaml`
- Added `Strict-Transport-Security: max-age=31536000; includeSubDomains` to all HTTPS `ResponseHeaderModifier` blocks in the webapp Gateway (SSL branch only)
- Added `ClientSettingsPolicy` to enforce TLS 1.2 minimum and AEAD-only cipher list on the webapp Gateway

### Access control

- Added dedicated `ServiceAccount` per workload (13 total); controlled by `serviceAccounts.enabled` in `values.yaml`
- Added `resources.requests` and `resources.limits` (CPU + memory) to every container, configurable in `values.yaml`

---

## Bug Fixes

- Fixed `accessModes: ReadWriteMany` → `ReadWriteOnce` on Redis and Mosquitto PersistentVolumeClaims
- Fixed spurious RabbitMQ management port 8080 exposed on `register` Service
- Fixed `producer` Service incorrectly exposing AMQP port 5672 (producer is a client, not a server)
- Fixed `imagePullPolicy: Always` missing on RabbitMQ

---

## Infrastructure

- Replaced MetalLB with Cilium for bare-metal load balancing: added `CiliumLoadBalancerIPPool` and `CiliumL2AnnouncementPolicy`; removed MetalLB `IPAddressPool` and `L2Advertisement` resources
- Migrated from `nginx-ingress` to Gateway API using NGINX Gateway Fabric (`Gateway`, `HTTPRoute`, `TCPRoute`, `ClientSettingsPolicy`, `SnippetsFilter`)

---

## Reliability

- Added liveness/readiness probes to `admission-nginx`
- Added liveness/readiness probes to `online-receiver` (Rocket HTTP health server)

