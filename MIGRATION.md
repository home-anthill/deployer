Migration from nginx-ingress to gateway apis via Claude.
This is the result after the migration


---
  Summary

  What was migrated

  - ingress-webapp.yaml → gateway-webapp.yaml (Gateway + HTTPRoutes + ClientSettingsPolicy + SnippetsFilter + Certificate + Issuer)
  - ingress-mqtt.yaml → gateway-mqtt.yaml (Gateway + TCPRoute + Certificate + Issuer)

  Issues encountered and fixed

  ┌───────────────────────────────────────┬─────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │                 Issue                 │                           Root Cause                            │                                                    Fix                                                    │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ NGF chart not found                   │ Wrong Helm repo (nginx-stable is for ingress-nginx)             │ OCI registry: ghcr.io/nginx/charts/nginx-gateway-fabric                                                   │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ NGF crashed on startup                │ Gateway API CRDs not installed before NGF                       │ Install CRDs first with --server-side                                                                     │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ CRD annotation too large              │ kubectl apply client-side limit                                 │ kubectl apply --server-side                                                                               │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Wrong external IP assigned            │ metadata.annotations ignored by NGF                             │ Moved MetalLB annotation to spec.infrastructure.annotations                                               │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ SnippetsFilter rejected               │ Invalid context name "location"                                 │ Renamed to "http.server.location"                                                                         │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ webapp-tls cert never created         │ cert-manager Gateway shim not enabled                           │ helm upgrade cert-manager --set config.enableGatewayAPI=true                                              │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ mqtt-tls secret not found             │ No explicit Certificate resource for TCP Gateway                │ Added explicit Certificate to gateway-mqtt.yaml                                                           │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ ACME challenge duplicate parentRefs   │ Gateway shim + explicit solver config both injected a parentRef │ Removed cert-manager.io/issuer annotation from Gateway, added explicit Certificate to gateway-webapp.yaml │
  ├───────────────────────────────────────┼─────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Helm can't adopt existing Certificate │ cert-manager created it without Helm ownership annotations      │ kubectl annotate + kubectl label to adopt it                                                              │
  └───────────────────────────────────────┴─────────────────────────────────────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────┘