# Kubernetes Request Flow: curl my-service.local/etc/passwd

This document traces the complete request path from a curl command on your host machine through the Kubernetes cluster with Contour/Envoy Ingress Controller to the final pod.

## Architecture Overview

- **Ingress Controller**: Contour (control plane) + Envoy (data plane)
- **CNI**: Calico
- **Cluster**: kind cluster (sam-test-cluster)
- **Service**: my-service (ClusterIP)
- **Pod**: example-pod (Python HTTP server)

## Complete Request Flow

### HOP 1: DNS Resolution (Your Host Machine)

```
curl my-service.local/etc/passwd
    ↓
Your /etc/hosts or DNS resolver
    ↓
my-service.local → 127.0.0.1 (localhost)
```

**Action**: Hostname resolves to localhost

---

### HOP 2: Docker Host → Kind Worker Container (Port Mapping)

```
127.0.0.1:80 (Your host machine)
    ↓
extraPortMappings in cluster.yaml:
  hostPort: 80 → containerPort: 80
    ↓
sam-test-cluster-worker Docker container:80
```

**Action**: Traffic on host port 80 is forwarded to the worker container's port 80 via Docker port mapping

**Configuration** (from `manifests/cluster.yaml`):
```yaml
extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: "0.0.0.0"
```

---

### HOP 3: Worker Container Network → Envoy Pod (hostPort Binding)

```
Worker container network namespace:80
    ↓
Envoy pod (envoy-dzdlm) uses hostPort: 80
  Container port 8080 → Host port 80 (worker container namespace)
    ↓
Envoy proxy receives request on port 80 (internally listening on 8080)
```

**Action**: Envoy binds to port 80 in the worker container's network namespace (mapped from container port 8080)

**Envoy Configuration**:
- **Pod**: `envoy-dzdlm` in `projectcontour` namespace
- **Node**: `sam-test-cluster-worker`
- **hostPort**: 80 (HTTP), 443 (HTTPS)
- **Container Port**: 8080 (HTTP), 8443 (HTTPS)

---

### HOP 4: Envoy → Ingress Resource Lookup

```
Envoy receives HTTP request:
  Host header: my-service.local
  Path: /etc/passwd
    ↓
Envoy checks xDS configuration (provided by Contour controller)
    ↓
Contour controller watches Ingress resources
    ↓
Finds: example-ingress
  Host: my-service.local
  Path: / → Backend: my-service:8080
    ↓
Envoy configured via xDS to route to backend service
```

**Action**: Envoy matches the host and path from the Ingress resource and routes to the backend service

**Components**:
- **Contour Controller**: Watches Kubernetes Ingress resources and HTTPProxy CRDs
- **xDS Protocol**: Contour uses xDS (Envoy Discovery Service) to configure Envoy
- **Ingress Resource**: `example-ingress` in `default` namespace

**Ingress Configuration** (from `manifests/ingress.yaml`):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: my-service.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 8080
```

---

### HOP 5: Envoy → Kubernetes Service (ClusterIP)

```
Envoy makes internal cluster request:
    ↓
DNS lookup: my-service.default.svc.cluster.local
  → 10.96.106.9:8080 (ClusterIP)
    ↓
kube-proxy (using iptables/ipvs) intercepts
    ↓
Load balances/selects endpoint: 10.244.55.134:8080
```

**Action**: Service DNS resolves to ClusterIP; kube-proxy handles routing

**Service Details**:
- **Name**: `my-service`
- **Type**: ClusterIP
- **ClusterIP**: 10.96.106.9
- **Port**: 8080
- **Selector**: `service: example-pod`

---

### HOP 6: Service → Pod Network (kube-proxy + Calico Infrastructure)

```
kube-proxy iptables DNAT rule:
  10.96.106.9:8080 → 10.244.55.134:8080
    ↓
Packet now has destination: 10.244.55.134:8080
    ↓
Node routing table (set up by Calico):
  10.244.55.134 dev cali47b61747c2f scope link
    ↓
Route packet to Calico interface (cali47b61747c2f)
  This interface is in the pod's network namespace
    ↓
Pod network namespace (example-pod)
```

**Action**: 
- **kube-proxy**: Performs DNAT translation from Service IP to Pod IP via iptables
- **Calico**: Provides the network infrastructure (routing table, network interfaces) that allows packets destined for the Pod IP to reach the pod's network namespace

**kube-proxy's Role**:
- Sets up iptables rules that translate Service ClusterIP (10.96.106.9:8080) → Pod IP (10.244.55.134:8080)
- Handles load balancing if multiple pod endpoints exist
- Runs on every node in the cluster

**Calico's Role** (Network Infrastructure Provider):
- **CNI Plugin**: Assigns IP addresses to pods from the pod CIDR (10.244.0.0/16)
- **Network Setup**: Creates pod network namespaces and Calico network interfaces (cali*)
- **Routing**: Sets up routing table entries so packets for pod IPs are routed to the pod's network interface
- **Pod-to-Pod Networking**: Handles pod-to-pod communication (same node or across nodes via VXLAN if needed)
- Does NOT handle Service routing - that's kube-proxy's job

**Configuration**:
- **Pod IP**: 10.244.55.134
- **Pod CIDR**: 10.244.0.0/16
- **Calico Interface**: cali47b61747c2f (network interface in pod's network namespace)
- **Node**: sam-test-cluster-worker
- **Encapsulation**: VXLANCrossSubnet (for cross-node pod-to-pod traffic)

---

### HOP 7: Pod Network → Container → Python HTTP Server

```
Pod receives packet on containerPort: 8080
    ↓
Container network namespace
    ↓
Python process listening on port 8080
  Command: python -m http.server 8080
    ↓
Filesystem access: /etc/passwd
    ↓
HTTP Response: 200 OK + file content
```

**Action**: Python HTTP server serves the file content

**Pod Details**:
- **Name**: `example-pod`
- **IP**: 10.244.55.134
- **Container Port**: 8080
- **Image**: python
- **Command**: `python -m http.server 8080`

---

### HOP 8-12: Response Path (Reverse)

```
Response travels back through all the same hops:
Pod → Calico → Service → Envoy → Worker Container → Docker Host → Your Machine
```

**Action**: HTTP response follows the reverse path back to the client

---

## Visual Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    [Your Host Machine]                          │
│                  curl my-service.local/etc/passwd               │
│                        ↓                                        │
│              DNS: my-service.local → 127.0.0.1                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              [localhost:80 - Docker Port Mapping]               │
│            extraPortMappings: hostPort 80 → containerPort 80   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│           [Kind Worker Container: sam-test-cluster-worker]      │
│                         Port: 80                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                  [Envoy Pod (envoy-dzdlm)]                      │
│              Container Port 8080 → Host Port 80                 │
│              Receives request, checks xDS config                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              [Contour Controller → Ingress Resource]            │
│                  example-ingress                                │
│              Host: my-service.local                             │
│              Path: / → Backend: my-service:8080                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              [Kubernetes Service: my-service]                   │
│              ClusterIP: 10.96.106.9:8080                        │
│              kube-proxy iptables DNAT:                           │
│              10.96.106.9:8080 → 10.244.55.134:8080              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              [Node Routing (Calico Infrastructure)]             │
│              Routing table: 10.244.55.134 dev cali47b61747c2f   │
│              Routes to pod's Calico interface                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              [Pod Network Namespace]                            │
│              Pod IP: 10.244.55.134                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              [example-pod Container]                            │
│              Container Port: 8080                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              [Python HTTP Server]                               │
│              python -m http.server 8080                         │
│              Serves: /etc/passwd                                │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### Contour Architecture

1. **Contour Controller** (`contour` pods):
   - Watches Kubernetes Ingress resources and HTTPProxy CRDs
   - Configures Envoy via xDS (Envoy Discovery Service)
   - Runs in `projectcontour` namespace

2. **Envoy Proxy** (`envoy` pod):
   - The data plane that handles actual traffic
   - Uses `hostPort` to bind to port 80/443
   - Receives configuration from Contour controller
   - Runs in `projectcontour` namespace

### Network Components

- **extraPortMappings**: Maps Docker host ports to kind container ports
- **hostPort**: Envoy binds directly to container port 80 (mapped from host port 80)
- **Service**: ClusterIP service provides internal DNS and load balancing
- **kube-proxy**: Implements Service routing via iptables DNAT (Service IP → Pod IP)
- **Calico CNI**: Provides pod networking infrastructure:
  - Assigns pod IP addresses
  - Creates pod network namespaces and interfaces
  - Sets up routing tables for pod-to-pod communication
  - Handles cross-node pod networking via VXLAN

## Troubleshooting

If the request fails, check:

1. **DNS Resolution**: Ensure `my-service.local` resolves to `127.0.0.1`
   ```bash
   cat /etc/hosts | grep my-service.local
   ```

2. **Port Mapping**: Verify port 80 is mapped correctly
   ```bash
   docker ps --filter "name=sam-test-cluster-worker" --format "table {{.Names}}\t{{.Ports}}"
   ```

3. **Envoy Pod**: Check if Envoy is running and bound to port 80
   ```bash
   kubectl get pods -n projectcontour -l app=envoy
   kubectl describe pod -n projectcontour -l app=envoy | grep -A 5 "Ports"
   ```

4. **Ingress Resource**: Verify the Ingress is configured correctly
   ```bash
   kubectl get ingress example-ingress
   kubectl describe ingress example-ingress
   ```

5. **Service**: Check service endpoints
   ```bash
   kubectl get svc my-service
   kubectl get endpoints my-service
   ```

6. **Pod**: Verify the pod is running and serving on port 8080
   ```bash
   kubectl get pod example-pod
   kubectl logs example-pod
   ```

## Summary

The request flow demonstrates how Kubernetes networking components work together:

1. **Port Mapping**: Docker/kind maps host ports to container ports
2. **Ingress Controller**: Envoy receives traffic and routes based on Ingress rules
3. **Service Discovery**: Kubernetes Service provides DNS and load balancing
4. **Service Routing (kube-proxy)**: iptables DNAT translates Service IP → Pod IP
5. **Pod Networking (Calico CNI)**: Routing infrastructure routes Pod IP → pod network namespace
6. **Application**: Pod receives traffic and serves the request

**Key Distinction**:
- **kube-proxy**: Handles Service routing (Service IP → Pod IP translation via iptables)
- **Calico CNI**: Provides pod networking infrastructure (IP assignment, network namespaces, routing tables)

Each layer adds abstraction and routing capabilities, creating a robust and flexible networking model.
