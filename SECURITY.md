# Security Issues

Audit of the `home-anthill` Helm chart (`home-anthill/`).

Last reviewed against chart templates and `values.yaml`: 2026-05-05.

## Original Issues

| # | Severity | Issue | Status |
|---|----------|-------|--------|
| ~~#3~~ | — | `allow_anonymous true` when mosquitto auth is disabled | ✅ Not an issue (by design) |
| ~~#1~~ | — | Placeholder credentials in `values.yaml` | ✅ Not an issue — production overrides come from an external private Helm values file |
| #2 | Critical | Sensitive values (JWT, OAuth2, DB URLs, passwords) stored in ConfigMaps instead of Secrets | 🔴 Open |
| #4 | Critical | Credentials exposed in probe `exec.command` arrays (mosquitto, redis) | ✅ Fixed — Redis and Mosquitto probes now use TCP socket checks; authenticated checks moved to Helm smoke tests using Secret-backed env vars |
| #5 | High | No container `securityContext` (`allowPrivilegeEscalation`, `capabilities.drop`) | ✅ Fixed — added to all 13 containers + 3 init containers |
| #6 | High | No `resources` limits/requests on any container | ✅ Fixed — added to all containers, configurable in `values.yaml` |
| #7 | High | `hostPath` volumes for Redis and Mosquitto data | 🔴 Open |
| #8 | High | `k8s-config-reloader` sidecar runs as root (`runAsNonRoot: false`) | ✅ Mitigated — `runAsUser: 0` is now explicit (required for cross-process SIGKILL); all other containers run as non-root |
| #9 | High | Reused Rocket Framework `secret_key` across register, online, online-alarm, online-receiver | ✅ Fixed — `register`, `online`, `online-receiver`, and `online-alarm` now have distinct `rocketSecretKey.release` values in `values.yaml` |
| #11 | High | No NetworkPolicies | ✅ Fixed — 17 policies added, 18 when SSL is enabled, toggle in `values.yaml` |
| #12 | Medium | No Pod Security Standards labels on namespace | ✅ Fixed — namespace enforces `baseline` and audits/warns on `restricted` |
| #13 | Medium | No `ResourceQuota` or `LimitRange` | 🔴 Open |
| ~~#14~~ | — | `GRPC_TLS=false` in api-devices, api-server, admission | ✅ Not an issue — internal plaintext service traffic is intentional and covered by NetworkPolicy/service scoping |
| #15 | Medium | Spurious RabbitMQ management port 8080 exposed on `register` Service | ✅ Fixed |
| ~~#16~~ | — | Weak placeholder credentials in `values.yaml` | ✅ Not an issue — production overrides come from an external private Helm values file; placeholders still satisfy current startup validation |
| #17 | Medium | No liveness/readiness probes on `online-receiver` | ✅ Fixed — Rocket HTTP health server added to Rust source and Helm chart |
| #18 | Medium | Redis `protected-mode no` + `bind 0.0.0.0` | 🔴 Open (partially mitigated by NetworkPolicy) |
| #19 | Low | `debug.pods.sleepInfinity` flag present in `values.yaml` | 🔴 Open |
| #20 | Low | `imagePullPolicy: Always` absent on RabbitMQ | ✅ Fixed |
| — | — | Dedicated ServiceAccounts per workload | ✅ Added — 13 SAs, toggle in `values.yaml` |

## Newly Identified Infrastructure Issues

| # | Severity | Issue | Status |
|---|----------|-------|--------|
| N1 | High | PVC `accessModes: ReadWriteMany` on Redis + Mosquitto (should be `ReadWriteOnce`) | ✅ Fixed |
| N2 | High | No `Recreate` deployment strategy for Redis + Mosquitto (RollingUpdate risks concurrent writes to `hostPath`) | ✅ Fixed |
| N3 | High | `init-certs` Alpine containers had no `securityContext` | ✅ Fixed — as part of #5 |
| N4 | Medium | No minimum TLS version enforced on webapp Gateway | ✅ Fixed — HTTPS listeners set NGF TLS options for TLS 1.2/1.3 and AEAD cipher suites |
| N5 | Medium | No `Strict-Transport-Security` (HSTS) header when SSL is enabled | ✅ Fixed — `max-age=31536000; includeSubDomains` added to all HTTPS `ResponseHeaderModifier` blocks in `gateway-webapp.yaml` |
| N6 | Medium | Gateway listeners lack explicit `allowedRoutes` | 🔴 Open |
| N7 | Medium | No liveness/readiness probes on `admission-nginx` | ✅ Fixed |
| N8 | Medium | `RabbitmqCluster` has no `spec.resources` defined | ✅ Fixed |
| N9 | Low | Worker-only `producer` and `consumer` pods had unnecessary Service resources | ✅ Fixed — Services removed |
| N10 | Medium | Default `network.nodesCIDR` allowed kubelet probe ingress from broad `10.0.0.0/8` range | ✅ Fixed — default narrowed to `10.0.0.0/24`; deployments should set the actual node subnet |
| N11 | Medium | `api-server` deployment env drift after OAuth/refresh-token updates: old callback paths, short cookie placeholder, no explicit token-hash pepper | ✅ Fixed — callbacks now match `/api/oauth/...`, cookie placeholder is >=32 chars, and `REFRESH_TOKEN_HASH_SECRET` is rendered |

## Hardening Improvements (K8s Security Best Practices)

| # | Severity | Issue | Status |
|---|----------|-------|--------|
| H1 | High | All containers ran as root (no `runAsNonRoot`/`runAsUser`) | ✅ Fixed — all pods run as uid 65534 (nobody); mosquitto as uid 1883; redis as uid 999 |
| H2 | High | Writable root filesystem on all containers (no `readOnlyRootFilesystem`) | ✅ Fixed — `readOnlyRootFilesystem: true` on all containers; writable `emptyDir` volumes added for `/tmp`, `/app/logs`, nginx cache/run/log paths |
| H3 | High | No seccomp profile on any pod | ✅ Fixed — `seccompProfile: RuntimeDefault` added to all 13 Deployments |
| H4 | Medium | Redis pod `securityContext` was conditional on `dhi.enabled` | ✅ Fixed — `runAsUser/Group: 999`, `runAsNonRoot: true`, `seccompProfile` now applied unconditionally |

## Summary

**28 fixed / not issues · 6 open** (1 Critical · 1 High · 3 Medium · 1 Low remaining)

## Current Open Issue Review

Reviewed on 2026-05-05:

| # | Status | Evidence |
|---|--------|----------|
| #2 | 🔴 Still open | Sensitive values are still rendered into ConfigMaps, for example `MONGODB_URL`, Redis/MQTT passwords, OAuth secrets, JWT secrets, cookie secret, AMQP URI/HMAC secret, and Rocket `secret_key` in `home-anthill/templates/*.yaml`. |
| #7 | 🔴 Still open | Redis and Mosquitto still use static PVs backed by `hostPath` in `home-anthill/templates/redis-pv.yaml` and `home-anthill/templates/mosquitto-pv.yaml`. |
| #13 | 🔴 Still open | No `ResourceQuota` or `LimitRange` templates are present. Container-level requests/limits exist, but namespace quota/default limit objects are still absent. |
| #18 | 🔴 Still open | `home-anthill/templates/redis.yaml` still renders `protected-mode no` and `bind 0.0.0.0`; this remains partially mitigated by NetworkPolicy only. |
| #19 | 🔴 Still open | `debug.pods.sleepInfinity` still exists in `values.yaml` and is wired into multiple workload templates. Default is `false`, but the debug override remains available. |
| N6 | 🔴 Still open | `Gateway` listeners in `gateway-webapp.yaml` and `gateway-mqtt.yaml` still do not set explicit `allowedRoutes`; routes are therefore not constrained at listener level. |
