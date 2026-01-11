# Contour Ingress Controller

## Overview

Contour is an Ingress Controller for Kubernetes that provides advanced Layer 7 (application layer) routing and load balancing capabilities. Contour solves networking challenges at the HTTP/HTTPS level, similar to how CNI (Container Network Interface) plugins solve pod-to-pod networking at Layer 3/4, but operates at a higher abstraction layer.

## Layer 7 Networking

Just as CNI providers (like Calico, Flannel, Weave, etc.) handle Layer 3/4 networking for pods—managing IP address assignment, routing, and pod-to-pod communication—Contour addresses Layer 7 networking challenges for ingress traffic:

- **CNIs (Layer 3/4)**: Handle pod IP assignment, routing tables, pod-to-pod connectivity, and network policies
- **Contour (Layer 7)**: Handles HTTP/HTTPS routing, TLS termination, host-based and path-based routing, load balancing at the application layer, and advanced traffic management

## CNI Provider Agnostic

**Contour works with any CNI provider.** Unlike some networking solutions that are tightly coupled to specific CNI implementations, Contour operates independently at the application layer and integrates seamlessly with:

- **Calico**
- **Flannel**
- **Weave Net**
- **Cilium**
- **Antrea**
- **Any other CNI-compliant networking plugin**

Contour's independence from the underlying CNI is possible because:

1. **Different Layers**: CNIs operate at Layer 3/4 (IP/TCP/UDP), while Contour operates at Layer 7 (HTTP/HTTPS)
2. **Standard Kubernetes APIs**: Contour uses standard Kubernetes Ingress and Gateway API resources, which are CNI-agnostic
3. **Service Abstraction**: Contour routes traffic to Kubernetes Services (ClusterIP), which are handled by kube-proxy and work regardless of the CNI

## Architecture

Contour consists of two main components:

1. **Contour Controller**: Watches Kubernetes Ingress and HTTPProxy resources, configures Envoy via xDS (Envoy Discovery Service)
2. **Envoy Proxy**: The data plane that handles actual HTTP/HTTPS traffic routing and load balancing

## Key Benefits

- **Layer 7 Routing**: Advanced HTTP/HTTPS routing capabilities (host-based, path-based, header-based)
- **TLS Management**: Automated TLS certificate management and termination
- **CNI Independent**: Works with any CNI provider
- **Standards Compliant**: Supports Kubernetes Ingress and Gateway API specifications
- **Production Ready**: Battle-tested in production environments

## See Also

- [Request Flow Documentation](../REQUEST_FLOW.md) - Detailed explanation of how requests flow through Contour and the CNI layer
