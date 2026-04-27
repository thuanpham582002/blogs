---
layout: post
title: "Replacing Nginx Ingress + MetalLB with Cilium Gateway API + L2 Announcements"
date: 2026-04-27
author: "Phạm Tiến Thuận"
tags: [writing-contest-2026-term1, kubernetes, cilium, gateway-api, ingress, metallb, networking, migration, bare-metal]
excerpt: "On bare-metal Kubernetes, Nginx Ingress + MetalLB is the classic stack. With ingress-nginx entering maintenance-only mode in March 2026, we migrated to Cilium Gateway API + L2 Announcements — one CNI replacing two components. Here is the practical walkthrough, including the non-obvious pitfalls."
---

## The Old Stack and Why It Was Painful

For years, the standard recipe for a bare-metal Kubernetes cluster has been:

- **MetalLB** to assign LoadBalancer IPs (since cloud LB controllers do not exist on bare metal)
- **Nginx Ingress Controller** for HTTP/HTTPS routing

Two separate projects, two sets of CRDs, two upgrade cycles, two failure domains. MetalLB needs its own IP pool config and L2/BGP advertisement setup. Nginx Ingress needs its own controller deployment, ConfigMaps, annotations, and a LoadBalancer Service to receive traffic from MetalLB.

Then in March 2026, `kubernetes/ingress-nginx` officially moved to maintenance-only mode. No new features, only security patches. The project itself recommends migrating to [Gateway API](https://gateway-api.sigs.k8s.io/), the next-generation routing standard from Kubernetes SIG-Network.

That was the trigger to rethink the whole stack. Since we already ran Cilium as our CNI, we had two features sitting unused that could replace both components:

- **Cilium L2 Announcements** — assigns LoadBalancer IPs and advertises them via Gratuitous ARP. Replaces MetalLB entirely.
- **Cilium Gateway API** — uses Cilium's built-in Envoy as the data plane. Replaces Nginx Ingress.

One CNI, one upgrade cycle, no extra controllers.

## Step 1: Replace MetalLB with Cilium L2 Announcements

Both MetalLB L2 mode and Cilium L2 Announcements work the same way at the network level: they advertise LoadBalancer IPs through ARP on a chosen interface. The migration is mostly a matter of moving config from MetalLB CRDs to Cilium CRDs.

Enable L2 Announcements (this requires `kubeProxyReplacement` to be enabled — the feature does not work alongside kube-proxy):

```bash
helm upgrade cilium ./cilium --namespace kube-system --reuse-values \
  --set l2announcements.enabled=true --set kubeProxyReplacement=true
```

Create a `CiliumLoadBalancerIPPool` (the equivalent of MetalLB's `IPAddressPool`) with the IP range you want to assign to LoadBalancer Services. Then create the L2 announcement policy — this is where the only real pitfall lives:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: l2-policy
spec:
  interfaces:
    - ^bond-mgmt$
  externalIPs: true
  loadBalancerIPs: true
```

**The non-obvious part:** The `interfaces` field is a regex, not a string match. If your nodes have multiple interfaces, a loose pattern can announce the LoadBalancer IP on the wrong subnet.

Our nodes had two bonded interfaces: `bond-mgmt` (1Gbps, management) and `bond-local` (200Gbps, internal). Using `^bond.*` would match both — Cilium would then announce the LB IP on the 200Gbps internal network where external clients cannot reach it. The result: `arping` from outside gets no reply, the LoadBalancer IP appears unreachable, but `kubectl get svc` shows everything healthy.

Always verify which interfaces are on which subnet with `ip addr show` before writing the regex. Use the most specific pattern that matches only the intended interface — for our setup, `^bond-mgmt$` (anchored both ends) is correct.

Once the policy is applied and verified, decommission MetalLB. Existing LoadBalancer Services keep their IPs as long as the IP falls within the new Cilium pool.

## Step 2: Enable Gateway API

```bash
helm upgrade cilium ./cilium --namespace kube-system --reuse-values \
  --set gatewayAPI.enabled=true
```

Cilium creates the `cilium` GatewayClass automatically and starts watching `Gateway` and `HTTPRoute` resources. Each `Gateway` you apply spawns a dedicated LoadBalancer Service (named `cilium-gateway-<name>`) that gets an IP from the L2 pool you configured in Step 1.

## Step 3: Migrate an Ingress to Gateway + HTTPRoute

The conceptual change: an `Ingress` collapses routing config into one resource per app. Gateway API splits this into a shared `Gateway` (infrastructure: which ports, which TLS certs, which namespaces are allowed to attach) and per-app `HTTPRoute` (application: hostnames and backends).

A typical pattern is one `Gateway` in a shared namespace with `allowedRoutes.namespaces.from: All` on each listener, and one `HTTPRoute` per app pointing back via `parentRefs`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: n8n-route
  namespace: n8n
spec:
  parentRefs:
  - name: app-gateway
    namespace: default
  hostnames: ["n8n.example.com"]
  rules:
  - backendRefs:
    - name: n8n
      port: 80
```

A useful side effect: with Nginx Ingress each namespace owned its own Ingress. With Gateway API, a single shared Gateway can accept routes from any namespace — explicit, auditable, and aligned with platform-vs-app team separation.

## Step 4: cert-manager with Gateway API

cert-manager does not enable Gateway API support by default — challenges are created but never presented unless you explicitly turn it on. The cleanest way is the Helm `config.enableGatewayAPI` value:

```bash
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --set config.enableGatewayAPI=true
```

If you installed cert-manager via static manifests, set the same flag in the `cert-manager` ConfigMap:

```bash
kubectl create configmap cert-manager -n cert-manager \
  --from-literal=enableGatewayAPI=true \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deployment cert-manager -n cert-manager
```

In your `ClusterIssuer`, use the `gatewayHTTPRoute` solver (instead of the `ingress` solver) and point `parentRefs` at the shared Gateway. cert-manager creates a temporary HTTPRoute for the ACME challenge.

One additional trap on Cilium 1.18.x ([cilium/cilium#44920](https://github.com/cilium/cilium/issues/44920), [#45139](https://github.com/cilium/cilium/issues/45139)): Cilium uses `TLSRoute v1alpha2`, but the standard Gateway API CRD bundle (v1.5.0+) only serves `v1`. The Cilium operator crashes with:

```
no matches for kind "TLSRoute" in version "gateway.networking.k8s.io/v1alpha2"
```

Install the experimental CRD bundle instead — it serves both versions:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/experimental-install.yaml
```

Patching the existing CRD with `kubectl patch` to set `served: true` for v1alpha2 will be reverted by the admission policy. Reinstalling from the experimental bundle is the only reliable fix.

## Verifying the Full Stack

```bash
kubectl get svc -A | grep LoadBalancer    # LB IP assigned
arping -c 3 <LB_IP>                       # ARP reachable on the right subnet
kubectl get gateway,httproute -A          # Gateway Programmed, HTTPRoute Accepted
kubectl get certificate -A                # certs Ready
curl -I https://n8n.example.com           # end-to-end
```

## Key Takeaways

1. **Cilium L2 Announcements replaces MetalLB completely** for L2-mode deployments. One CNI, one set of CRDs, one upgrade path.

2. **L2 interface regex must be specific.** `^bond.*` matches all bond interfaces. If you have multiple bonds on different subnets, use an exact match like `^bond-mgmt$` or you will announce LB IPs on the wrong network and external clients will not reach them.

3. **cert-manager Gateway API support is off by default.** Set `config.enableGatewayAPI: true` (Helm) or the equivalent ConfigMap flag — without it, ACME challenges are created but never presented and you wait forever for a certificate that never arrives.

4. **Cilium 1.18.x requires the experimental Gateway API CRD bundle.** The stable bundle lacks `TLSRoute v1alpha2` and the operator will crash. Do not patch the CRD manually — the admission controller reverts it.

## References

- [Cilium Gateway API Documentation](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api.html)
- [Cilium L2 Announcements](https://docs.cilium.io/en/stable/network/lb-ipam/l2-announcements/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [cert-manager Gateway API](https://cert-manager.io/docs/usage/gateway/)
- [ingress-nginx project status](https://github.com/kubernetes/ingress-nginx)
