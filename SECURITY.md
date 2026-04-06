# Security Issues

Audit of the `home-anthill` Helm chart (`home-anthill/`).

## Original Issues

| # | Severity | Issue | Status |
|---|----------|-------|--------|
| ~~#3~~ | — | `allow_anonymous true` when mosquitto auth is disabled | ✅ Not an issue (by design) |
| #1 | Critical | Hardcoded secrets in `values.yaml` | 🔴 Open |
| #2 | Critical | Sensitive values (JWT, OAuth2, DB URLs, passwords) stored in ConfigMaps instead of Secrets | 🔴 Open |
| #4 | Critical | Credentials exposed in probe `exec.command` arrays (mosquitto, redis) | 🔴 Open |
| #5 | High | No container `securityContext` (`allowPrivilegeEscalation`, `capabilities.drop`) | ✅ Fixed — added to all 13 containers + 3 init containers |
| #6 | High | No `resources` limits/requests on any container | ✅ Fixed — added to all containers, configurable in `values.yaml` |
| #7 | High | `hostPath` volumes for Redis and Mosquitto data | 🔴 Open |
| #8 | High | `k8s-config-reloader` sidecar runs as root (`runAsNonRoot: false`) | 🔴 Open |
| #9 | High | Reused Rocket Framework `secret_key` across register, online, online-alarm, online-receiver | 🔴 Open |
| #10 | High | `ks89/producer:develop` — mutable image tag | 🔴 Open |
| #11 | High | No NetworkPolicies | ✅ Fixed — 16 policies added, toggle in `values.yaml` |
| #12 | Medium | No Pod Security Standards labels on namespace | 🔴 Open |
| #13 | Medium | No `ResourceQuota` or `LimitRange` | 🔴 Open |
| #14 | Medium | `GRPC_TLS=false` in api-devices, api-server, admission | 🔴 Open |
| #15 | Medium | Spurious RabbitMQ management port 8080 exposed on `register` Service | ✅ Fixed |
| #16 | Medium | Weak default credentials in `values.yaml` | 🔴 Open |
| #17 | Medium | No liveness/readiness probes on `online-receiver` | ✅ Fixed — Rocket HTTP health server added to Rust source and Helm chart |
| #18 | Medium | Redis `protected-mode no` + `bind 0.0.0.0` | 🔴 Open (partially mitigated by NetworkPolicy) |
| #19 | Low | `debug.pods.sleepInfinity` flag present in `values.yaml` | 🔴 Open |
| #20 | Low | `imagePullPolicy: Always` absent on RabbitMQ | ✅ Fixed |
| — | — | Dedicated ServiceAccounts per workload | ✅ Added — 13 SAs, toggle in `values.yaml` |

## Newly Identified Infrastructure Issues

| # | Severity | Issue | Status |
|---|----------|-------|--------|
| N1 | High | PVC `accessModes: ReadWriteMany` on Redis + Mosquitto (should be `ReadWriteOnce`) | ✅ Fixed |
| N2 | High | No `Recreate` deployment strategy for Redis + Mosquitto (RollingUpdate risks concurrent writes to `hostPath`) | 🔴 Open |
| N3 | High | `init-certs` Alpine containers had no `securityContext` | ✅ Fixed — as part of #5 |
| N4 | Medium | No minimum TLS version enforced on webapp Gateway | ✅ Fixed — TLS 1.2 minimum + AEAD-only ciphers in `ClientSettingsPolicy` |
| N5 | Medium | No `Strict-Transport-Security` (HSTS) header when SSL is enabled | 🔴 Open |
| N6 | Medium | Gateway listeners lack explicit `allowedRoutes` | 🔴 Open |
| N7 | Medium | No liveness/readiness probes on `admission-nginx` | ✅ Fixed |
| N8 | Medium | `RabbitmqCluster` has no `spec.resources` defined | 🔴 Open |
| N9 | Low | `producer` Service exposed AMQP port 5672 (producer is a client, not a server) | ✅ Fixed |

## Summary

**11 fixed · 13 open** (3 Critical · 4 High · 6 Medium · 1 Low remaining)
