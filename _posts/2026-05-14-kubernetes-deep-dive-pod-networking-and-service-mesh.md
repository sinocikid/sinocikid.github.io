---
title: "Kubernetes Deep Dive: Pod Networking and Service Mesh"
date: 2026-05-14
categories: ["Cloud"]
tags: ["Kubernetes", "Cloud", "Networking", "Service Mesh", "Istio", "Cilium"]
read_time: 10
---

Kubernetes networking is one of those topics that looks simple on the surface — pods get IPs, services route traffic — until you're debugging a production outage at 2am and realize you have no idea why a pod can't reach another pod on a different node. This post goes deep into how Kubernetes networking actually works, what CNI plugins do, how service meshes solve problems that Kubernetes leaves unsolved, and why eBPF is changing the game.

## The Kubernetes Networking Model

Kubernetes imposes a flat networking model with four fundamental rules:

1. Every pod gets its own IP address
2. Pods on the same node can communicate without NAT
3. Pods on different nodes can communicate without NAT
4. The IP a pod sees for itself is the same IP other pods see for it

This sounds simple, but it's a constraint on the network, not an implementation. Kubernetes itself doesn't implement this — it delegates to CNI plugins. The flat model means pods behave like VMs on a flat network, which simplifies application code dramatically (no port mapping gymnastics, no NAT surprises).

Each pod gets a network namespace. The container runtime creates a `veth` pair — one end in the pod's namespace, one end on the host. The CNI plugin is responsible for wiring up these interfaces and ensuring cross-node connectivity.

## CNI Plugins: Flannel, Calico, and Cilium

The Container Network Interface (CNI) is a standard that Kubernetes uses to delegate networking to plugins. The three you'll encounter most:

### Flannel

The simplest option. Flannel creates an overlay network using VXLAN by default. Traffic between nodes is encapsulated in UDP packets. It works reliably and is easy to operate, but:

- No NetworkPolicy support (you need to pair it with something else, or use Canal)
- VXLAN encapsulation adds overhead
- Limited visibility into network traffic

Flannel is fine for small clusters where simplicity matters more than features.

### Calico

Calico is the production workhorse for most enterprise Kubernetes deployments. It supports:

- **BGP-based routing**: Instead of overlay encapsulation, Calico can peer with your physical network routers using BGP, giving you native routing performance
- **Full NetworkPolicy support**: Including Calico's extended policies that go beyond the Kubernetes spec
- **WireGuard encryption**: Node-to-node traffic encryption with minimal performance overhead

In BGP mode, there's no encapsulation overhead — pods are routable directly from the physical network. In VXLAN mode, it behaves similarly to Flannel but with policy enforcement.

### Cilium

Cilium is in a different category. It uses eBPF (extended Berkeley Packet Filter) to implement networking, security, and observability at the kernel level. More on this below. Key features:

- **Layer 7 policy**: NetworkPolicy based on HTTP methods, paths, gRPC methods — not just IP/port
- **Transparent encryption** with WireGuard or IPsec
- **Hubble**: A built-in observability platform for real-time network flow visibility
- **Kube-proxy replacement**: Cilium can replace kube-proxy entirely for better performance

For new clusters, Cilium is increasingly the default choice for teams that care about security observability.

## Kubernetes Services: The Four Types

Services provide stable network endpoints for pods, which are ephemeral and get new IPs on restart. The four service types:

**ClusterIP** (default): Creates a virtual IP reachable only inside the cluster. DNS resolves `my-service.my-namespace.svc.cluster.local` to this IP. kube-proxy (or Cilium) uses iptables or eBPF rules to load-balance across healthy pod endpoints.

**NodePort**: Exposes the service on a static port (30000-32767) on every node's IP. Traffic to `<any-node-ip>:<nodeport>` reaches the service. Useful for development or when you control your own load balancer.

**LoadBalancer**: Provisions a cloud provider load balancer (AWS ELB, GCP LB, Azure LB) automatically. The cloud controller manager handles provisioning. This is the standard way to expose services externally.

**ExternalName**: Maps a service name to a DNS name. Returns a CNAME. Useful for routing traffic to external databases or services using Kubernetes DNS names, so you can swap the target without changing application config.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

## Ingress Controllers

Services handle east-west (pod-to-pod) and basic north-south traffic. Ingress handles HTTP/HTTPS routing from outside the cluster, with path-based and host-based routing rules.

An Ingress resource defines routing rules. An Ingress controller (a separate deployment — NGINX, Traefik, Contour, AWS ALB controller) actually implements them:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
```

For more complex routing (traffic splitting, header-based routing, circuit breaking), you need a service mesh or a gateway API implementation.

## NetworkPolicy: The Missing Firewall

By default, all pods can talk to all other pods. NetworkPolicy lets you define ingress and egress rules based on pod labels, namespace labels, and CIDR blocks.

A practical example — restrict the order service to only accept traffic from the API gateway, and only connect to the database:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to: # Allow DNS
    ports:
    - protocol: UDP
      port: 53
```

Note: NetworkPolicy requires a CNI plugin that enforces it. Flannel alone does not. Calico and Cilium both do.

## Service Mesh: What Problem It Solves

Kubernetes services give you basic load balancing and service discovery. But in a microservices environment, you quickly need more:

- **Mutual TLS (mTLS)**: Encrypting and authenticating all service-to-service traffic, not just traffic from outside the cluster
- **Observability**: Distributed tracing, golden signals (latency, error rate, throughput) per service pair, without instrumenting every application
- **Traffic control**: Canary deployments (send 5% of traffic to v2), circuit breakers, retries with budgets, fault injection for testing
- **Policy enforcement**: Which services are allowed to call which other services, enforced at the network layer

A service mesh solves these by injecting a sidecar proxy (typically Envoy) into every pod. The sidecar intercepts all inbound and outbound traffic transparently. The application doesn't change — the proxy handles mTLS, collects telemetry, and enforces policies.

## Istio vs Linkerd

| Feature | Istio | Linkerd |
|---|---|---|
| Proxy | Envoy | Linkerd2-proxy (Rust) |
| Resource overhead | Higher (~100-200MB/sidecar) | Lower (~10-30MB/sidecar) |
| mTLS | Yes (automatic) | Yes (automatic) |
| Tracing | Jaeger, Zipkin, Tempo | Jaeger, Tempo |
| Traffic splitting | Yes (VirtualService) | Yes (HTTPRoute) |
| Circuit breaking | Yes (via Envoy config) | Basic |
| Layer 7 policy | Yes (AuthorizationPolicy) | Yes |
| Learning curve | Steep | Moderate |
| Gateway API support | Yes | Yes |
| FIPS compliance | Via Envoy builds | Yes (native) |

Linkerd's proxy is written in Rust and is significantly lighter than Envoy. If you need the basics — mTLS, observability, traffic splitting — Linkerd gets you there with less complexity. If you need advanced traffic management, multi-cluster federation, or deep Envoy customization, Istio is more capable.

For most teams, Linkerd is the right starting point. Adopt Istio when you have specific requirements that Linkerd can't meet.

## eBPF-Based Networking: Why Cilium Matters

Traditional Linux networking goes through a deep kernel stack: network card → kernel network stack → iptables → socket. Every hop adds latency and CPU overhead. kube-proxy uses iptables rules for service load balancing — at scale (10,000+ services), iptables rule evaluation becomes a performance bottleneck.

eBPF (extended Berkeley Packet Filter) lets you run sandboxed programs inside the Linux kernel, triggered by events like packet arrival, without modifying kernel source or loading kernel modules. Cilium uses eBPF to:

- **Replace iptables for service load balancing**: eBPF-based load balancing in the kernel is faster and doesn't degrade at scale
- **Implement NetworkPolicy at the kernel level**: No overhead from going through the full networking stack for denied connections
- **Enable socket-level load balancing**: For pod-to-pod traffic on the same node, Cilium can short-circuit the network stack entirely and connect sockets directly — near-zero overhead

The observability story is also different. Because Cilium operates at the kernel level, Hubble can provide real-time visibility into every network flow without any application instrumentation — connection establishment, DNS queries, HTTP requests, dropped packets — all visible without touching the application.

For high-traffic clusters, the performance difference is measurable. Cilium + eBPF can sustain significantly higher packet rates with lower CPU usage compared to traditional iptables-based networking.

## Practical Recommendations

- **Start with Calico** for most production clusters if you need reliable NetworkPolicy enforcement and BGP peering with your physical network.
- **Choose Cilium** for new clusters where you want the best observability and performance, or where you need Layer 7 network policies.
- **Use Linkerd** as your service mesh unless you have specific advanced requirements that demand Istio.
- **Always define NetworkPolicy** for production workloads. Default deny for both ingress and egress, then explicitly allow only required traffic.
- **Use Gateway API** (not Ingress) for new deployments — it's the future of Kubernetes traffic management and both Istio and Linkerd support it natively.

## Conclusion

Kubernetes networking is a layered system. The flat IP model makes application development simpler, but the implementation underneath — CNI plugins, iptables/eBPF, service proxies — is complex. Understanding each layer helps you make better architectural decisions and debug faster when things go wrong.

The trajectory is clear: eBPF-based networking (Cilium) is becoming the standard for performance-sensitive deployments, service meshes are graduating from optional to expected in microservices architectures, and the Gateway API is replacing both Ingress and proprietary mesh-specific APIs. Building familiarity with these tools now positions you well for where the ecosystem is heading.
