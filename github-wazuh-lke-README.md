# Wazuh on Linode Kubernetes Engine (LKE) — Production Deployment Guide

[![Wazuh](https://img.shields.io/badge/Wazuh-4.14.8-3585F9?style=flat&logo=wazuh)](https://documentation.wazuh.com/current/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.35-326CE5?style=flat&logo=kubernetes)](https://kubernetes.io/)
[![LKE](https://img.shields.io/badge/Akamai-LKE-189AD6?style=flat)](https://techdocs.akamai.com/cloud-computing/docs/linode-kubernetes-engine)
[![License](https://img.shields.io/badge/License-GPLv2-green.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)

A complete, step-by-step guide for deploying a production-grade Wazuh SIEM/XDR platform on Akamai Cloud's Linode Kubernetes Engine (LKE).

> **📄 Companion Article:** For the strategic overview, architecture rationale, and cost analysis, see the [LinkedIn article](https://www.linkedin.com/pulse/architecting-enterprise-security-scale-deploying-wazuh-prabhjot-singh-aji1e/).

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [1. Deploying the LKE Cluster](#1-deploying-the-lke-cluster)
- [2. Preparing Kubernetes for Wazuh](#2-preparing-kubernetes-for-wazuh)
- [3. Deploying Wazuh on Kubernetes](#3-deploying-wazuh-on-kubernetes)
- [4. Configuring Persistent Storage on LKE](#4-configuring-persistent-storage-on-lke)
- [5. Exposing Wazuh Securely](#5-exposing-wazuh-securely)
- [6. Installing Wazuh Agents](#6-installing-wazuh-agents)
- [7. Monitoring Kubernetes Workloads](#7-monitoring-kubernetes-workloads)
- [8. Security Hardening Best Practices](#8-security-hardening-best-practices)
- [9. High Availability and Scaling](#9-high-availability-and-scaling)
- [10. Troubleshooting](#10-troubleshooting)
- [References](#references)
- [Disclaimer](#disclaimer)

---

## Overview

This repository contains the deployment guide and configuration examples for running Wazuh 4.14.8 on Akamai Cloud's Linode Kubernetes Engine. The deployment uses the official [wazuh-kubernetes](https://github.com/wazuh/wazuh-kubernetes) repository with a custom Kustomize overlay for LKE.

> **⚠️ Security Note:** Wazuh 4.14.8 (September 2026) includes critical security fixes from the 4.14.6–4.14.8 release cycle: a manager deadlock fix under high event volume, a heap buffer write vulnerability patch, improved API token invalidation across clusters, and a remote command validation bypass fix. If you are running 4.14.5 or earlier, upgrading is strongly recommended.

**Stack:**
- **Wazuh** 4.14.8 (Manager, Indexer, Dashboard, Agents)
- **Kubernetes** 1.35 on LKE (1.36 also supported; 1.33 deprecated)
- **Linode Block Storage** CSI Driver for persistent volumes
- **NGINX Ingress Controller** for dashboard HTTPS access
- **cert-manager** v1.21.0 with Let's Encrypt for automated TLS
- **Linode NodeBalancer** for agent-facing services

> **🆕 Alternative:** A community-driven [Wazuh Kubernetes Operator](https://github.com/wazuh/wazuh-kubernetes) entered preview in August 2026. The Kustomize-based approach in this guide remains the recommended production method, but teams familiar with the Operator pattern may want to evaluate it.

---

## Prerequisites

### Akamai Cloud Account

An active Akamai Cloud (Linode) account with permissions to create LKE clusters, Block Storage volumes, NodeBalancers, and DNS records. Generate a Personal Access Token with Read/Write scopes for Kubernetes, Linodes, NodeBalancers, Volumes, and Domains.

### Local Tooling

| Tool | Minimum Version | Purpose |
|---|---|---|
| `kubectl` | 1.29+ | Kubernetes cluster management |
| `kustomize` | 5.0+ (or use `kubectl -k`) | Manifest overlay and deployment |
| `helm` | 3.14+ | Installing ingress controller and cert-manager |
| `git` | 2.x | Cloning Wazuh Kubernetes repository |
| `openssl` | 1.1+ | Certificate generation and verification |
| `linode-cli` | Latest | LKE cluster provisioning (optional) |

### DNS and Domain Requirements

A domain name with DNS management access is required for TLS-secured dashboard access. You will create an A record pointing to the NGINX Ingress Controller's external IP. Example: `wazuh.security.yourdomain.com`.

### Cluster Sizing (100–500 endpoints)

| Node Role | Plan Type | vCPUs | RAM | Count | Purpose |
|---|---|---|---|---|---|
| Indexer/Manager | Dedicated 8GB | 4 | 8 GB | 3 | Wazuh Indexer + Manager StatefulSets |
| Ingress/Utility | Shared 4GB | 2 | 4 GB | 2 | NGINX Ingress, cert-manager, Dashboard |

**Total minimum:** 5 nodes, ~36 GB RAM, 18 vCPUs.

For smaller environments (<50 endpoints), a 3-node cluster using Dedicated 4GB instances is viable for non-production use.

### Storage Requirements

| Component | Minimum PVC Size | Recommended | Notes |
|---|---|---|---|
| Wazuh Indexer (per node) | 50 Gi | 100–500 Gi | Scales with log retention period |
| Wazuh Manager Master | 10 Gi | 50 Gi | Stores rules, decoders, agent keys |
| Wazuh Manager Worker | 10 Gi | 50 Gi | Event queue and local state |

---

## 1. Deploying the LKE Cluster

### Creating the Cluster via Linode CLI

```bash
# Install and configure the Linode CLI
pip3 install linode-cli --upgrade
linode-cli configure

# Create the LKE cluster
linode-cli lke cluster-create \
  --label wazuh-production \
  --region us-ord \
  --k8s_version 1.35 \
  --node_pools '[
    {
      "type": "g6-dedicated-8",
      "count": 3
    },
    {
      "type": "g6-dedicated-4",
      "count": 2
    }
  ]'
```

### Region Selection

Choose a region close to your monitored endpoints to minimize agent-to-manager latency. All LKE regions support Block Storage.

- **North America:** `us-ord` (Chicago), `us-iad` (Washington DC), `us-lax` (Los Angeles)
- **Europe:** `eu-west` (London), `de-fra-2` (Frankfurt)

### Dedicated vs. Shared CPU

Use **Dedicated CPU** for indexer and manager nodes. OpenSearch is CPU-intensive during indexing and search, and the Wazuh manager performs continuous rule evaluation. Shared CPU instances are acceptable for NGINX Ingress and cert-manager pods.

### Downloading Kubeconfig

```bash
# Get the cluster ID
linode-cli lke clusters-list

# Download kubeconfig
linode-cli lke kubeconfig-view <CLUSTER_ID> --text | base64 -d > ~/.kube/wazuh-kubeconfig.yaml

# Set as active context
export KUBECONFIG=~/.kube/wazuh-kubeconfig.yaml

# Verify connectivity
kubectl get nodes -o wide
kubectl cluster-info
```

### Verifying the Cluster

```bash
# Confirm all nodes are Ready
kubectl get nodes -o wide

# Verify the CSI driver is running (pre-installed on LKE)
kubectl get pods -n kube-system | grep csi

# Verify storage classes
kubectl get storageclass
# Expected: linode-block-storage-retain (default on LKE)

# Verify the Cloud Controller Manager
kubectl get pods -n kube-system | grep ccm
```

---

## 2. Preparing Kubernetes for Wazuh

### Namespace Creation

The Wazuh Kubernetes manifests create the `wazuh` namespace automatically. To create it explicitly:

```bash
kubectl create namespace wazuh
kubectl label namespace wazuh app=wazuh
```

### Storage Class Validation

```bash
kubectl get sc
kubectl describe sc linode-block-storage-retain
# Should show: provisioner: linodebs.csi.linode.com
# reclaimPolicy: Retain
```

The `Retain` policy prevents data loss if a PVC is accidentally deleted. For encryption at rest, use `linode-block-storage-retain-encrypted`.

### Installing NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer \
  --set controller.admissionWebhooks.enabled=true

# Wait for the external IP (NodeBalancer)
kubectl get svc -n ingress-nginx ingress-nginx-controller -w
```

Note the `EXTERNAL-IP` — your DNS A record should point here.

### Installing cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.21.0 \
  --set crds.enabled=true

# Verify
kubectl get pods -n cert-manager
```

### Creating a ClusterIssuer for Let's Encrypt

```yaml
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: security-team@yourdomain.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

```bash
kubectl apply -f cluster-issuer.yaml
kubectl get clusterissuer letsencrypt-prod
```

### DNS Configuration

```
wazuh.security.yourdomain.com  →  A  →  <NGINX_EXTERNAL_IP>
```

Allow time for DNS propagation before deploying the Ingress resource.

---

## 3. Deploying Wazuh on Kubernetes

### Cloning the Repository

```bash
git clone https://github.com/wazuh/wazuh-kubernetes.git -b v4.14.8 --depth=1
cd wazuh-kubernetes
```

### Repository Structure

```
wazuh-kubernetes/
├── wazuh/
│   ├── base/
│   │   ├── storage-class.yaml
│   │   └── wazuh-ns.yaml
│   ├── certs/
│   │   ├── indexer_cluster/
│   │   │   └── generate_certs.sh
│   │   └── dashboard_http/
│   │       └── generate_certs.sh
│   ├── indexer_stack/
│   ├── wazuh_managers/
│   ├── kustomization.yml
│   └── wazuh-certificates/
├── envs/
│   ├── eks/
│   └── local-env/
├── instructions.md
└── local-environment.md
```

### Generating SSL Certificates

```bash
# Generate indexer cluster certificates
wazuh/certs/indexer_cluster/generate_certs.sh

# Generate dashboard HTTPS certificates
wazuh/certs/dashboard_http/generate_certs.sh
```

Certificates are placed in `wazuh/wazuh-certificates/` and referenced by `secretGenerator` in the Kustomize configuration.

> **Production note:** Consider replacing self-signed certificates with ones from your internal CA or cert-manager for production.

### Creating the LKE Environment Overlay

```bash
mkdir -p envs/lke
```

**envs/lke/kustomization.yml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../wazuh
patches:
  - path: storage-class.yaml
  - path: indexer-resources.yaml
  - path: dashboard-resources.yaml
  - path: wazuh-master-resources.yaml
  - path: wazuh-worker-resources.yaml
```

**envs/lke/storage-class.yaml**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wazuh-storage
provisioner: linodebs.csi.linode.com
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**envs/lke/indexer-resources.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wazuh-indexer
  namespace: wazuh
spec:
  template:
    spec:
      containers:
        - name: wazuh-indexer
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

**envs/lke/dashboard-resources.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wazuh-dashboard
  namespace: wazuh
spec:
  template:
    spec:
      containers:
        - name: wazuh-dashboard
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1"
              memory: "2Gi"
```

**envs/lke/wazuh-master-resources.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wazuh-manager-master
  namespace: wazuh
spec:
  template:
    spec:
      containers:
        - name: wazuh-manager
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

**envs/lke/wazuh-worker-resources.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wazuh-manager-worker
  namespace: wazuh
spec:
  template:
    spec:
      containers:
        - name: wazuh-manager
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

### OpenSearch vm.max_map_count

The Wazuh Indexer requires `vm.max_map_count` of at least 262144. Verify on LKE nodes:

```bash
kubectl debug node/<NODE_NAME> -it --image=busybox -- sysctl vm.max_map_count
```

The upstream manifests include an init container to set this value.

### Deploying the Stack

```bash
# Apply the LKE overlay
kubectl apply -k envs/lke/

# Monitor deployment
kubectl get pods -n wazuh -w

# Wait for all pods to reach Running (3-5 minutes)
```

### Verifying the Deployment

```bash
# Check all pods
kubectl get pods -n wazuh
# Expected:
# wazuh-indexer-0            1/1     Running
# wazuh-dashboard-xxx        1/1     Running
# wazuh-manager-master-0     1/1     Running
# wazuh-manager-worker-0     1/1     Running

# Check services
kubectl get svc -n wazuh

# Check persistent volumes
kubectl get pvc -n wazuh

# Retrieve indexer credentials
for type in username password; do
  secret=$(kubectl get secret -n wazuh indexer-cred -o yaml | grep "$type:" | awk '{print $2}')
  echo "$type: $(echo $secret | base64 -d)"
done
```

### Secrets Management

For production:
- Rotate default indexer credentials immediately after deployment
- Use HashiCorp Vault, Sealed Secrets, or External Secrets Operator instead of plain Kubernetes Secrets
- The `indexer-cred` secret contains admin username/password for Indexer/Dashboard
- Certificate secrets (`indexer-certs`, `dashboard-certs`) contain TLS certificates

---

## 4. Configuring Persistent Storage on LKE

### How Linode Block Storage Works with LKE

Key characteristics:
- **Minimum volume size:** 10 Gi
- **Access mode:** ReadWriteOnce (RWO) — correct for StatefulSets
- **Volume expansion:** Supported online without downtime
- **Maximum attachments:** Scales with instance memory (8 volumes for ≤4 GB RAM, up to 64 for larger instances)

### Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wazuh-storage
provisioner: linodebs.csi.linode.com
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

`WaitForFirstConsumer` delays volume provisioning until a pod is scheduled, ensuring the volume is created in the same zone as the node.

### StatefulSet Persistence

The upstream manifests use `volumeClaimTemplates` for per-replica PVCs:

```yaml
# Defined in upstream manifests — shown for reference
volumeClaimTemplates:
  - metadata:
      name: wazuh-indexer
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: wazuh-storage
      resources:
        requests:
          storage: 50Gi
```

### Backup Recommendations

1. **OpenSearch Snapshot API** → Linode Object Storage (S3-compatible) — recommended for indexer data
2. **Volume cloning** via CSI driver for offline backup
3. **Velero** with Linode Object Storage plugin for Kubernetes-native backup/restore

```bash
# Configure OpenSearch snapshot repository
curl -X PUT "https://localhost:9200/_snapshot/wazuh_backup" \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "wazuh-backups",
      "endpoint": "us-ord-1.linodeobjects.com",
      "protocol": "https"
    }
  }'
```

---

## 5. Exposing Wazuh Securely

### Dashboard Access via NGINX Ingress

```yaml
# wazuh-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wazuh-dashboard-ingress
  namespace: wazuh
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - wazuh.security.yourdomain.com
      secretName: wazuh-dashboard-tls
  rules:
    - host: wazuh.security.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dashboard
                port:
                  number: 443
```

```bash
kubectl apply -f wazuh-ingress.yaml

# Verify certificate issuance
kubectl get certificate -n wazuh
kubectl describe certificate wazuh-dashboard-tls -n wazuh
```

> **Note:** The `backend-protocol: "HTTPS"` annotation is required because the Wazuh Dashboard runs on HTTPS (port 443) internally.

### Agent-Facing Services

```bash
# Get external IPs for agent connectivity
kubectl get svc -n wazuh wazuh wazuh-workers

# Note EXTERNAL-IP values:
# wazuh        → port 1515 (registration) + 55000 (API)
# wazuh-workers → port 1514 (event transport)
```

### Security Best Practices

- **Never expose the Wazuh Indexer API (port 9200) externally**
- **Restrict dashboard access** with IP allowlisting:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,203.0.113.50/32"
```

- **Use Linode Cloud Firewall** to restrict NodeBalancer access to known agent IP ranges
- **HTTPS everywhere** — agent-to-manager uses Wazuh's built-in TLS

---

## 6. Installing Wazuh Agents

### Ubuntu/Debian

```bash
# Import GPG key
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && \
  chmod 644 /usr/share/keyrings/wazuh.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  tee /etc/apt/sources.list.d/wazuh.list

# Install agent
apt-get update
WAZUH_MANAGER="<WAZUH_WORKERS_EXTERNAL_IP>" apt-get install wazuh-agent=4.14.8-1

# Enable and start
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### RHEL/CentOS

```bash
# Import GPG key
rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

# Add repository
cat > /etc/yum.repos.d/wazuh.repo << 'EOF'
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1
EOF

# Install agent
WAZUH_MANAGER="<WAZUH_WORKERS_EXTERNAL_IP>" yum install wazuh-agent-4.14.8

# Enable and start
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### Kubernetes DaemonSet

```yaml
# wazuh-agent-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: wazuh-agent
  namespace: wazuh
  labels:
    app: wazuh-agent
spec:
  selector:
    matchLabels:
      app: wazuh-agent
  template:
    metadata:
      labels:
        app: wazuh-agent
    spec:
      containers:
        - name: wazuh-agent
          image: wazuh/wazuh-agent:4.14.8
          env:
            - name: WAZUH_MANAGER
              value: "wazuh-manager-master-0.wazuh-cluster.wazuh.svc.cluster.local"
            - name: WAZUH_AGENT_GROUP
              value: "kubernetes"
          securityContext:
            privileged: true
          volumeMounts:
            - name: host-root
              mountPath: /host
              readOnly: true
            - name: host-var-log
              mountPath: /var/log
              readOnly: true
            - name: agent-state
              mountPath: /var/ossec/etc
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
      volumes:
        - name: host-root
          hostPath:
            path: /
        - name: host-var-log
          hostPath:
            path: /var/log
        - name: agent-state
          emptyDir: {}
      tolerations:
        - effect: NoSchedule
          operator: Exists
      hostNetwork: false
      hostPID: true
```

### Sidecar Agent

```yaml
# Add to your application pod spec
spec:
  containers:
    - name: my-application
      image: my-app:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: wazuh-agent
      image: wazuh/wazuh-agent:4.14.8
      env:
        - name: WAZUH_MANAGER
          value: "wazuh-manager-master-0.wazuh-cluster.wazuh.svc.cluster.local"
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
          readOnly: true
  volumes:
    - name: shared-logs
      emptyDir: {}
```

### Connectivity Notes

- **External agents:** Use NodeBalancer external IP — port 1515 (registration), port 1514 (events)
- **In-cluster agents (DaemonSet/sidecar):** Use internal service DNS names to avoid traversing the NodeBalancer

---

## 7. Monitoring Kubernetes Workloads

### What the DaemonSet Agent Monitors

- **System-level:** File integrity monitoring on node filesystems, process monitoring, log collection (`/var/log/syslog`, `/var/log/auth.log`, container runtime logs)
- **Container visibility:** With `hostPID: true`, the agent detects container escape attempts, privilege escalation, and anomalous process execution
- **Runtime:** Process creation/termination, unauthorized outbound connections, file modifications, user authentication events, package changes

### CIS Benchmark Assessment

Wazuh SCA policies for Kubernetes CIS benchmarks automatically check:
- Kubelet configuration security
- API server argument validation
- Network policy enforcement
- RBAC configuration assessment
- Pod security standards compliance

Results appear in the Wazuh Dashboard under the SCA module.

### Kubernetes Audit Logging

On LKE, the control plane is managed by Akamai, so direct API server configuration is not available. Use a log forwarding pod to collect audit events and forward them to the Wazuh manager.

> **Note:** As of August 2026, LKE uses Konnectivity (instead of WireGuard) for control plane to node communication. This is transparent to Wazuh deployments but worth noting for network policy planning — Konnectivity traffic traverses the standard Kubernetes API path rather than a separate tunnel.

---

## 8. Security Hardening Best Practices

### RBAC

```yaml
# wazuh-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: wazuh-reader
  namespace: wazuh
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["indexer-cred", "indexer-certs", "dashboard-certs"]
```

### Network Policies

```yaml
# Deny all ingress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: wazuh
spec:
  podSelector: {}
  policyTypes:
    - Ingress

---
# Allow indexer cluster communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-indexer-cluster
  namespace: wazuh
spec:
  podSelector:
    matchLabels:
      app: wazuh-indexer
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: wazuh-indexer
      ports:
        - port: 9300
          protocol: TCP
    - from:
        - podSelector:
            matchLabels:
              app: wazuh-dashboard
        - podSelector:
            matchLabels:
              app: wazuh-manager
      ports:
        - port: 9200
          protocol: TCP

---
# Allow dashboard access from ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dashboard-ingress
  namespace: wazuh
spec:
  podSelector:
    matchLabels:
      app: wazuh-dashboard
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 5601
          protocol: TCP
```

### Linode Cloud Firewall

- Allow TCP 1514, 1515 from known agent IP ranges only
- Allow TCP 443 from administrator IP ranges
- Allow TCP 10250 from cluster CIDR only
- Deny all other inbound traffic

### Pod Security

```bash
kubectl label namespace wazuh \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

> **Note:** The agent DaemonSet requires `privileged: true` and `hostPID: true`. If your policy prohibits this, deploy agents as traditional packages on Linode instances directly.

### OpenSearch Hardening

1. Rotate default admin credentials in `indexer-cred` immediately
2. Disable demo certificates when using custom certs
3. Configure OpenSearch RBAC to restrict index access
4. Enable OpenSearch audit logging

---

## 9. High Availability and Scaling

### Indexer Scaling

| Deployment Size | Indexer Replicas | Shard Config | RAM per Node | Storage per Node |
|---|---|---|---|---|
| Small (≤100 agents) | 1 | 1 primary, 0 replica | 4 Gi | 50 Gi |
| Medium (100–500) | 3 | 1 primary, 1 replica | 4–8 Gi | 100 Gi |
| Large (500–2000) | 5 | 2 primary, 1 replica | 8–16 Gi | 250 Gi |
| Enterprise (2000+) | 7+ | 3 primary, 1 replica | 16–32 Gi | 500 Gi+ |

```bash
kubectl scale statefulset wazuh-indexer -n wazuh --replicas=3
```

### Manager Scaling

| Agent Count | Worker Replicas | CPU per Worker | RAM per Worker |
|---|---|---|---|
| ≤200 | 1 | 2 cores | 4 Gi |
| 200–1000 | 2 | 2–4 cores | 4–8 Gi |
| 1000–5000 | 3–5 | 4 cores | 8 Gi |
| 5000+ | 5+ | 4–8 cores | 8–16 Gi |

```bash
kubectl scale statefulset wazuh-manager-worker -n wazuh --replicas=3
```

### Dashboard Scaling

```bash
kubectl scale deployment wazuh-dashboard -n wazuh --replicas=2
```

### Pod Anti-Affinity

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - wazuh-indexer
        topologyKey: "kubernetes.io/hostname"
```

### Performance Tips

- **JVM heap:** Set to ≤50% of container memory limit (4 Gi container → `-Xms2g -Xmx2g`)
- **Shard sizing:** Target 10–50 Gi per shard
- **Index lifecycle:** Implement rollover, shrink, and delete policies based on age/size

---

## 10. Troubleshooting

### Common Commands

```bash
# Pod status and events
kubectl get pods -n wazuh -o wide
kubectl describe pod <POD_NAME> -n wazuh

# Logs
kubectl logs <POD_NAME> -n wazuh
kubectl logs <POD_NAME> -n wazuh --previous
kubectl logs -f <POD_NAME> -n wazuh

# Shell into pod
kubectl exec -it <POD_NAME> -n wazuh -- /bin/bash

# Resource usage
kubectl top pods -n wazuh
kubectl top nodes
```

### Pod Issues

**Pending:** Check resources and PVC provisioning:
```bash
kubectl describe pod <POD_NAME> -n wazuh | grep -A 10 Events
```

**CrashLoopBackOff:** Check logs for startup errors:
```bash
kubectl logs <POD_NAME> -n wazuh --previous
# Common: certificate issues, config errors, memory limits too low
```

### Storage Issues

**PVC stuck in Pending:**
```bash
kubectl describe pvc <PVC_NAME> -n wazuh
# Check: provisioning failures, storage class issues, volume attachment limits
```

### TLS Issues

```bash
# Check indexer certificate
kubectl exec -it wazuh-indexer-0 -n wazuh -- \
  openssl x509 -in /usr/share/wazuh-indexer/certs/indexer.pem -noout -dates

# Let's Encrypt not issuing
kubectl describe certificate wazuh-dashboard-tls -n wazuh
kubectl logs -n cert-manager -l app=cert-manager
```

### OpenSearch Issues

```bash
# Cluster health
kubectl exec -it wazuh-indexer-0 -n wazuh -- \
  curl -sk -u admin:<PASSWORD> https://localhost:9200/_cluster/health?pretty

# Node status
kubectl exec -it wazuh-indexer-0 -n wazuh -- \
  curl -sk -u admin:<PASSWORD> https://localhost:9200/_cat/nodes?v

# Index status
kubectl exec -it wazuh-indexer-0 -n wazuh -- \
  curl -sk -u admin:<PASSWORD> https://localhost:9200/_cat/indices?v

# Disk usage
kubectl exec -it wazuh-indexer-0 -n wazuh -- \
  curl -sk -u admin:<PASSWORD> https://localhost:9200/_cat/allocation?v
```

### Agent Connectivity

```bash
# Test TCP connectivity from agent host
nc -zv <WAZUH_WORKERS_EXTERNAL_IP> 1514
nc -zv <WAZUH_EXTERNAL_IP> 1515

# Check agent logs
tail -f /var/ossec/logs/ossec.log

# List registered agents on manager
kubectl exec -it wazuh-manager-master-0 -n wazuh -- \
  /var/ossec/bin/manage_agents -l
```

---

## References

| Resource | URL |
|---|---|
| Wazuh Official Documentation | https://documentation.wazuh.com/current/ |
| Wazuh Kubernetes Deployment | https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/index.html |
| Wazuh Kubernetes GitHub | https://github.com/wazuh/wazuh-kubernetes |
| Wazuh 4.14.8 Release Notes | https://documentation.wazuh.com/current/release-notes/release-4-14-8.html |
| LKE Documentation | https://techdocs.akamai.com/cloud-computing/docs/linode-kubernetes-engine |
| LKE Enterprise | https://techdocs.akamai.com/cloud-computing/docs/lke-enterprise |
| LKE Versioning & Lifecycle | https://techdocs.akamai.com/cloud-computing/docs/lke-versioning-and-life-cycle-policy |
| Linode Block Storage CSI | https://github.com/linode/linode-blockstorage-csi-driver |
| OpenSearch Documentation | https://opensearch.org/docs/latest/ |
| cert-manager Documentation | https://cert-manager.io/docs/ |
| NGINX Ingress Controller | https://kubernetes.github.io/ingress-nginx/ |
| CIS Kubernetes Benchmark | https://www.cisecurity.org/benchmark/kubernetes |
| MITRE ATT&CK Framework | https://attack.mitre.org/ |

---

## Disclaimer

The views and opinions expressed in this repository are entirely my own and do not represent the positions, strategies, or endorsements of my employer, any affiliated organization, or any vendor mentioned herein. This content is provided for informational and educational purposes only and should not be construed as professional security advice, architectural certification, or a guarantee of any specific security outcome. Technology decisions should be evaluated independently based on your organization's requirements, risk profile, and regulatory obligations. While every effort has been made to ensure accuracy at the time of writing, software versions, pricing, features, and platform capabilities are subject to change. I assume no liability for any damages, security incidents, data loss, or operational disruptions arising from the use or implementation of any tools, configurations, or recommendations discussed in this repository. Always consult with qualified security professionals and conduct thorough testing in non-production environments before deploying any solution. References to specific products or companies are for context only and do not constitute an endorsement or partnership.

---

## License

This guide is provided under the [MIT License](LICENSE). Wazuh itself is licensed under [GPLv2](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html).
