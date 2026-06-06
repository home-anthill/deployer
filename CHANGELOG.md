# Changelog

## 6.1.0

### Features


### Security

- Rendered consumer and online-receiver signed replay-cache Redis settings as `REDIS_REPLAY_URI` against Redis database `/2`.


## 6.0.1

### Security

- The Deployment now uses valueFrom.secretKeyRef for MOSQUITTO_USERS, and the combined MQTT credentials are rendered into a new mosquitto-users-secret Secret instead of a literal env value in the Deployment.


## 6.0.0

### Features

- Added role-based Mosquitto ACLs for authenticated deployments: shared firmware devices can publish telemetry (`sensors/+/+`, `online/+/features/+`) and read device commands (`devices/+/values`), while backend services get separate producer, online-receiver, and api-devices MQTT credentials scoped to their required topics.
- Updated the Mosquitto Helm deployment to pass all broker users through `MOSQUITTO_USERS` and mount a generated `mosquitto-acl` ConfigMap.
- Updated `producer`, `online-receiver`, and `api-devices` Helm configs to use separate MQTT role credentials instead of the former shared broker credential.
- Added shared `apiToken.hashSecret` and `apiToken.encryptionKey` values, rendered as `API_TOKEN_HASH_SECRET` and `API_TOKEN_ENCRYPTION_KEY` into the Secret-backed `.env` files for `api-server`, `admission`, `register`, `api-devices`, `consumer`, and `online-receiver`.
- Added explicit `REFRESH_TOKEN_HASH_SECRET` rendering for `api-server` so refresh/app-login token lookup hashes use a dedicated deployment secret instead of relying on fallback JWT secrets.
- Replaced the API-server GitHub login restriction value with `apiServer.limitToUserEmails`, rendered as `LIMIT_TO_USER_EMAILS`, so deployments can allow a comma-separated list of GitHub email addresses.
- Added `REDIS_URI`, `REDIS_USERNAME`, and `REDIS_PASSWORD` to the consumer Secret-backed runtime config for Redis-backed signed nonce replay protection.
- Added `HTTP_ONLINE_APITOKEN_API=/api-token/` to the API-server runtime config so profile token regeneration can update plaintext token references held by the online service in Redis.
- Replaced MetalLB with Cilium for bare-metal load balancing: added `CiliumLoadBalancerIPPool` and `CiliumL2AnnouncementPolicy`; removed MetalLB `IPAddressPool` and `L2Advertisement` resources.
- Migrated from `nginx-ingress` to Gateway API using NGINX Gateway Fabric (`Gateway`, `HTTPRoute`, `TCPRoute`, `ClientSettingsPolicy`, `SnippetsFilter`).
- Added `ClientSettingsPolicy` to enforce a 100 MiB maximum request body on the webapp Gateway.
- Enabled Redis persistence with RDB snapshots on the mounted `/data` volume, so Redis state now survives pod restarts instead of being treated as disposable cache data.
- Added liveness/readiness probes to `admission-nginx`.
- Added liveness/readiness probes to `online-receiver` (Rocket HTTP health server).
- Added Helm deployment smoke tests for GUI, API server, online service, Redis auth, Mosquitto auth, and RabbitMQ management auth.
- Updated Helm smoke tests to authenticate as the shared device MQTT role and publish to an ACL-allowed sensor topic.

### Security

- Added `runAsNonRoot: true` and `runAsUser`/`runAsGroup` at pod level to all Deployments: Go services run as uid `65534` (`nobody`), `mosquitto` as `1883`, and `redis` as `999`; NGINX pods get `seccompProfile` only since the master process requires root for worker management.
- Added `readOnlyRootFilesystem: true` to all containers; writable `emptyDir` volumes are mounted at `/tmp`, `/app/logs` for the Rust rolling log appender, and nginx cache/run/log paths.
- Added `seccompProfile: RuntimeDefault` at pod level to all 13 Deployments, activating the container runtime's default syscall allowlist.
- Added `allowPrivilegeEscalation: false` and `capabilities.drop: ["ALL"]` to all containers and init containers; `NET_BIND_SERVICE` is re-added only where port 80 is bound, and `KILL` only on `k8s-config-reloader`.
- Made the `k8s-config-reloader` sidecar `runAsUser: 0` explicit because it is required to send SIGKILL across the process namespace; all other containers run as non-root.
- Made the Redis pod `securityContext` (`runAsUser`/`runAsGroup: 999`, `runAsNonRoot`) unconditional; it previously applied only when `dhi.enabled: true`.
- Added `automountServiceAccountToken: false` to all pods.
- Replaced Redis and Mosquitto credential-bearing `exec` probes with TCP socket probes, removing passwords from normal Pod probe command arrays.
- Added 17-18 `NetworkPolicy` resources implementing default-deny-all ingress and egress with explicit per-service allow rules; controlled by `network.enabled` in `values.yaml`.
- Added Redis network access for `consumer` so signed MQTT nonce replay protection can use the shared Redis replay cache (`consumer -> redis:6379`, and Redis ingress from `consumer`).
- Restricted external egress with Cilium FQDN policies for GitHub, MongoDB Atlas, and Google/Firebase destinations, and removed the port-only `443` / `27017` fallbacks from the standard Kubernetes `NetworkPolicy` objects.
- Fixed MongoDB Atlas egress to allow client traffic on ports 27015-27017, added explicit DNS visibility for Cilium FQDN policy, and matched multi-level Atlas `mongodb.net` hostnames used by SRV records and shard targets.
- Fixed online-alarm FCM egress by adding explicit DNS visibility to the Google/Firebase Cilium FQDN policy before allowing HTTPS to `*.googleapis.com` and `accounts.google.com`.
- Added Helm test hook NetworkPolicies for the deployment smoke-test Job so it can reach only the services it validates during `helm test`.
- Added `Strict-Transport-Security: max-age=31536000; includeSubDomains` to all HTTPS `ResponseHeaderModifier` blocks in the webapp Gateway when SSL is enabled.
- Enforced TLS 1.2/1.3 and AEAD cipher suites on HTTPS Gateway listeners using NGF-supported TLS options.
- Moved `Content-Security-Policy` from Gateway `ResponseHeaderModifier` rules into the `gui` and `admission-nginx` NGINX ConfigMaps, keeping CSP close to the HTTP servers that own those responses.
- Tightened CSP by removing `unsafe-inline` from `script-src`.
- Removed the CSP `sandbox` directive because it does not add useful protection for these same-origin application responses and can interfere with normal app behavior.
- Added `Referrer-Policy: same-origin` from the `gui` and `admission-nginx` NGINX ConfigMaps.
- Kept Gateway-level `X-Frame-Options`, `X-Content-Type-Options`, `Permissions-Policy`, and SSL-only HSTS headers in `gateway-webapp.yaml`.
- Added dedicated `ServiceAccount` per workload (13 total); controlled by `serviceAccounts.enabled` in `values.yaml`.
- Added Pod Security Admission labels to the namespace: enforce `baseline`, audit/warn `restricted`.
- Moved sensitive runtime config from ConfigMaps to Secrets for `api-server`, `register`, `online`, `online-alarm`, `producer`, `consumer`, `api-devices`, `admission`, `online-receiver`, and `redis`, while keeping the same mounted file paths so no container rebuild was needed.
- Added namespace guardrails with `ResourceQuota` and `LimitRange` to bound total namespace usage and provide sane default per-container requests/limits.
- Added explicit Gateway listener `allowedRoutes.namespaces.from: Same` on webapp and MQTT Gateways so only same-namespace routes can attach.
- Switched Redis to `protected-mode yes` while keeping ACL auth enabled, so Redis remains service-reachable but is no longer configured in the broadest mode.
- Replaced Redis and Mosquitto `hostPath` PersistentVolumes with `local-path` PVCs, removing direct node-path bindings from the chart while keeping single-node k3s storage simple.

### Bug Fixes

- Fixed `accessModes: ReadWriteMany` to `ReadWriteOnce` on Redis and Mosquitto PersistentVolumeClaims.
- Fixed spurious RabbitMQ management port 8080 exposed on the `register` Service.
- Fixed `producer` Service incorrectly exposing AMQP port 5672; producer is a client, not a server.
- Removed unnecessary Service resources from worker-only `producer` and `consumer` pods.
- Narrowed default `network.nodesCIDR` from `10.0.0.0/8` to `10.0.0.0/24` for kubelet health-probe ingress.
- Fixed `imagePullPolicy: Always` missing on RabbitMQ.
- Fixed `api-server` deployment env drift: OAuth callback URLs now match `/api/oauth/callback` and `/api/oauth/app/callback`, and the default `COOKIE_SECRET` placeholder satisfies the current 32-character startup validation.

### Chores

- Added `resources.requests` and `resources.limits` for CPU and memory to every container, configurable in `values.yaml`.
- Raised the small workload CPU requests from `5m` to `10m` so they satisfy the namespace `LimitRange` minimum and can actually be scheduled.
