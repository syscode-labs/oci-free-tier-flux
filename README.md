# OCI Free Tier Flux GitOps

Flux CD manifests for OCI Always Free Tier Kubernetes cluster running on Proxmox + Talos Linux.

## Overview

This repository contains GitOps configuration for:
- **Cilium CNI** (kube-proxy-free mode with Hubble)
- **Flux CD** (GitOps controller)
- **Tailscale Operator** (mesh networking)
- **OCI Cloud Controller Manager** (LoadBalancer integration)
- **NVIDIA Device Plugin** (GPU support)
- **Grafana Alloy** (monitoring agents)

## Repository Structure

```
oci-free-tier-flux/
├── bootstrap/          # Critical bootstrap manifests (Cilium + Flux sync)
├── clusters/           # Cluster-specific configurations
├── infrastructure/     # Infrastructure components (cert-manager, Tailscale, etc.)
└── apps/               # Applications and monitoring
```

## Prerequisites

- Talos Linux cluster deployed on Proxmox
- `kubectl` configured to access cluster
- `flux` CLI installed
- `sops` and `age` installed for secrets management

## Quick Start

### 1. Generate Age Key (One-time)

```bash
# Generate Age encryption key
age-keygen -o age-key.txt

# Output will show your public key:
# Public key: age1qqz7hjxv8qp2kamxl8txqmjvhcj4jz0qc6zk7z...

# Keep age-key.txt PRIVATE - never commit to Git!
```

### 2. Configure SOPS

```bash
# Update .sops.yaml with your Age public key
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: .*\.enc\.yaml$
    encrypted_regex: ^(data|stringData|values)$
    age: YOUR_AGE_PUBLIC_KEY_HERE
EOF
```

### 3. Create and Encrypt Secrets

```bash
# Example: Tailscale OAuth credentials
cat > infrastructure/overlays/prod/tailscale-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-oauth
  namespace: tailscale
stringData:
  client-id: "YOUR_CLIENT_ID"
  client-secret: "YOUR_CLIENT_SECRET"
EOF

# Encrypt with SOPS
sops --encrypt infrastructure/overlays/prod/tailscale-secret.yaml \
  > infrastructure/overlays/prod/tailscale-secret.enc.yaml

# Delete plaintext file
rm infrastructure/overlays/prod/tailscale-secret.yaml
```

### 4. Bootstrap Talos Cluster

Update your Talos machine configuration to reference this repository:

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
  
  externalCloudProvider:
    enabled: true
    manifests:
      # 1. Cilium CNI (networking)
      - https://raw.githubusercontent.com/YOUR_USER/oci-free-tier-flux/main/bootstrap/cilium.yaml
      
      # 2. Flux CD (GitOps)
      - https://github.com/fluxcd/flux2/releases/download/v2.4.0/install.yaml
      
      # 3. Flux sync (watches this repo)
      - https://raw.githubusercontent.com/YOUR_USER/oci-free-tier-flux/main/bootstrap/flux-sync.yaml
```

Apply the configuration:

```bash
talosctl apply-config --nodes <node-ip> --file controlplane.yaml
talosctl bootstrap --nodes <node-ip>
```

### 5. Store Age Key in Cluster

After the cluster is running:

```bash
# Wait for Flux namespace
kubectl wait --for=condition=ready --timeout=5m \
  -n flux-system pod -l app=source-controller

# Store Age key for SOPS decryption
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age-key.txt
```

### 6. Verify Deployment

```bash
# Check Flux components
flux get all

# Check infrastructure deployments
kubectl get pods -n cert-manager
kubectl get pods -n tailscale
kubectl get pods -n kube-system

# Check Cilium status
cilium status
```

## Secrets Management

All secrets are encrypted using **SOPS + Age** before committing to Git.

### Encrypting Secrets

```bash
# Encrypt a secret file
sops --encrypt secret.yaml > secret.enc.yaml

# Edit encrypted file (decrypts in editor, re-encrypts on save)
sops secret.enc.yaml
```

### Secret Files Pattern

- Unencrypted secrets: `*.yaml` (NEVER commit these)
- Encrypted secrets: `*.enc.yaml` (safe to commit)

## Components

### Bootstrap (Critical Path)

- **Cilium**: CNI networking (must be first)
- **Flux**: GitOps controller

### Infrastructure

- **cert-manager**: Certificate management
- **Tailscale Operator**: Mesh networking
- **OCI CCM**: Cloud controller for OCI
- **NVIDIA Device Plugin**: GPU support

### Applications

- **Grafana Alloy**: Monitoring and observability

## Updating Infrastructure

1. Modify manifests in `infrastructure/` or `apps/`
2. Encrypt any new secrets with SOPS
3. Commit and push to GitHub
4. Flux automatically reconciles changes

```bash
# Force immediate reconciliation
flux reconcile source git flux-system
flux reconcile kustomization infrastructure
```

## Troubleshooting

### Flux not syncing

```bash
# Check Flux logs
flux logs

# Check GitRepository status
flux get sources git

# Check Kustomization status
flux get kustomizations
```

### SOPS decryption failing

```bash
# Verify sops-age secret exists
kubectl get secret -n flux-system sops-age

# Check if Age key is valid
kubectl get secret -n flux-system sops-age -o jsonpath='{.data.age\.agekey}' | base64 -d
```

### Cilium networking issues

```bash
# Check Cilium status
cilium status

# Run connectivity test
cilium connectivity test
```

## Security

- ✅ All secrets encrypted with SOPS before committing
- ✅ Age private key stored only in cluster and locally
- ✅ Repository can be public - no secrets exposed
- ✅ HTTPS Git access - no SSH keys needed

## References

- [Flux Documentation](https://fluxcd.io/docs/)
- [SOPS Documentation](https://github.com/getsops/sops)
- [Age Encryption](https://github.com/FiloSottile/age)
- [Cilium Documentation](https://docs.cilium.io/)
- [Talos Linux](https://www.talos.dev/)
