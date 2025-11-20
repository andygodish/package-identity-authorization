# Keycloak

## keycloak-waypoint pod and service

Istiod runs the waypoint controller that creates/manages the keycloak-waypoint pod and service. The Keycloak helm chart that ships with UDS Core deploys a Gateway resource (assumes Gateway API CRDs exist: `kubectl get crd | grep gateway.networking.k8s.io`) and a companion config map that describes the desired configuration. The controller reconciles that Gateway resource by creating the waypoint pod and service.

There isn’t a separate “waypoint controller” pod; istiod is doing that work. If you disable ambient (PILOT_ENABLE_AMBIENT=false or remove Gateway API support), the waypoint pod/service would stop being created.

PILOT_ENABLE_AMBIENT=true (and ENABLE_NATIVE_SIDECARS=true) turns on ambient mode in 1.27, which includes the waypoint/Gateway API reconciliation path inside istiod. These are configured as env vars on the istiod deployment.

### Configuration

In the UDS Core repo, `src/istio/values/base-istiod.yaml` sets `profile: ambient` and adds `pilot.env.ENABLE_NATIVE_SIDECARS: true`. With the ambient profile, the Istio chart sets `PILOT_ENABLE_AMBIENT=true` in the istiod Deployment.

### Logs Findings

The Istiod logs will show the ambient/waypoint controller inside istiod is running and reconciling your Keycloak waypoint:

You'll see general logs about how Istio is running in Gateway API/ambient mode and loading Gateway/GatewayClass informers.

- `controllers starting controller=gateway deployment`, then reconciling gateway with controller `istio.io/mesh-controller gateway=keycloak/keycloak-waypoint` — that’s the mesh-controller (ambient) creating/patching the managed waypoint Deployment.

- A retry on Deployment/keycloak-waypoint due to a resource version conflict confirms it’s actively managing that Deployment.

- Later, XDS pushes to a node named keycloak-waypoint-... show the waypoint pod connected and being configured.

> xDS is Istio/Envoy’s control-plane API family (“discovery services”) used to push config to data-plane proxies: LDS (listeners), RDS (routes), CDS (clusters), EDS (endpoints), SDS (secrets), etc. istiod is the xDS server; sidecars, gateways, ztunnel, and waypoints connect to it to receive live config updates.
