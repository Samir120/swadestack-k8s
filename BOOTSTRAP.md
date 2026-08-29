# SwadeStack Kubernetes Bootstrap

This document describes how to rebuild the SwadeStack Kubernetes
infrastructure from a fresh k3s cluster.

## 1. Install k3s

The k3s server is configured with:

```yaml
node-name: homeserver
node-ip: 192.168.0.100

tls-san:
  - 192.168.0.100
  - homeserver

disable:
  - traefik
  - servicelb

write-kubeconfig-mode: "0640"
write-kubeconfig-group: k3s-admin
