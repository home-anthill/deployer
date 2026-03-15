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

## Architecture

The chart deploys ~15 services to a Kubernetes cluster, organized into these layers:

**Public Web / API tier**
- `gui` — frontend web UI (served at domain root via Ingress)
- `api-server` — main REST API with OAuth2, connects to MongoDB Atlas
- `admission` — gRPC admission service, exposed at `/admission` via an NGINX sidecar proxy

**Internal cluster API tier**
- `api-devices` — gRPC service for IoT device management
- `register` — Rust (Rocket) service for sensor registration
- `online` / `online-receiver` / `online-alarm` — presence tracking and FCM push notifications

**Messaging tier**
- `rabbitmq` — AMQP message broker (v4.x)
- `mosquitto` — MQTT broker for IoT devices (v2.x), with auth via ConfigMap
- `producer` / `consumer` — services that publish/consume messages from RabbitMQ

**Data tier**
- `redis` — in-cluster cache with a PersistentVolume
- MongoDB — external (Atlas Cloud), referenced by connection string in `values.yaml`

**Infrastructure**
- `gateway-webapp` — NGINX Gateway Fabric (Gateway API): `Gateway` + `HTTPRoute` for HTTP→HTTPS redirect and routing (`/admission` → admission NGINX sidecar, `/` → gui); `ClientSettingsPolicy` for request body size; `SnippetsFilter` for rate limiting; cert-manager `Issuer` + `Certificate` for Let's Encrypt TLS
- `gateway-mqtt` — NGINX Gateway Fabric (Gateway API): `Gateway` + `TCPRoute` for raw TCP MQTT/MQTTS traffic; cert-manager `Issuer` + `Certificate` for Let's Encrypt TLS (HTTP-01 challenge via a dedicated HTTP listener); MetalLB annotation pins each gateway to its public IP
- `metalb` — MetalLB bare-metal load balancer config
- `namespace` — creates the target namespace
- `k8s-config-reloader` — watches ConfigMaps/Secrets and reloads affected pods

## Key Files

| File | Purpose |
|------|---------|
| `home-anthill/Chart.yaml` | Chart version and app version |
| `home-anthill/values.yaml` | All configurable values: domain, image tags, auth mode, secrets |
| `home-anthill/templates/*.yaml` | One file per service; most contain ConfigMap + Deployment + Service |
| `.github/workflows/docker-image.yml` | CI: runs `helm template` to validate chart renders cleanly |

## Values Structure

`values.yaml` controls:
- **Domain** — base domain and public IPs for Gateway routes (HTTP and MQTT)
- **Image tags** — each service has its own image and tag (all from `ghcr.io/home-anthill/`)
- **Auth mode** — OAuth2 vs single-user (`api-server` ConfigMap)
- **Secrets** — MongoDB URI, RabbitMQ credentials, JWT secrets, Firebase config (base64-encoded in templates)
- **Resource limits** — CPU/memory per service
- **Mosquitto auth** — username/password written into a ConfigMap
