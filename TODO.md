# Setup Tasks TODO

## Prerequisites

- [ ] Install Helm: `brew install helm`
- [ ] Install SOPS: `brew install sops`
- [ ] Install Age: `brew install age`

## Bootstrap Manifests

- [ ] Generate Cilium manifest:
  ```bash
  helm repo add cilium https://helm.cilium.io/
  helm repo update
  helm template cilium cilium/cilium \
    --version 1.16.5 \
    --namespace kube-system \
    --values bootstrap/cilium-values.yaml \
    > bootstrap/cilium.yaml
  git add bootstrap/cilium.yaml
  git commit -m "feat: add Cilium CNI bootstrap manifest"
  git push
  ```

## Secrets Setup

- [ ] Generate Age encryption key:
  ```bash
  age-keygen -o age-key.txt
  # Note the public key from output
  ```

- [ ] Update `.sops.yaml` with your Age public key:
  ```bash
  # Replace YOUR_AGE_PUBLIC_KEY_REPLACE_ME with actual key
  sed -i '' 's/YOUR_AGE_PUBLIC_KEY_REPLACE_ME/age1.../g' .sops.yaml
  git add .sops.yaml
  git commit -m "chore: configure SOPS with Age public key"
  git push
  ```

- [ ] Create Tailscale OAuth secret:
  ```bash
  # Get OAuth credentials from Tailscale admin console
  cat > /tmp/tailscale-secret.yaml <<EOF
  apiVersion: v1
  kind: Secret
  metadata:
    name: tailscale-oauth
    namespace: tailscale
  stringData:
    client-id: "YOUR_ACTUAL_CLIENT_ID"
    client-secret: "YOUR_ACTUAL_CLIENT_SECRET"
  EOF
  
  # Encrypt with SOPS
  sops --encrypt /tmp/tailscale-secret.yaml \
    > infrastructure/overlays/prod/tailscale-secret.enc.yaml
  
  # Clean up
  rm /tmp/tailscale-secret.yaml
  
  # Commit
  git add infrastructure/overlays/prod/tailscale-secret.enc.yaml
  git commit -m "feat: add encrypted Tailscale OAuth secret"
  git push
  ```

## Optional Infrastructure

- [ ] Add OCI Cloud Controller Manager manifests (if using LoadBalancer services)
- [ ] Add NVIDIA Device Plugin manifests (if using GPU passthrough)
- [ ] Add Grafana Alloy for monitoring

## Cluster Bootstrap

After completing above tasks:

- [ ] Update Talos machine config with correct GitHub URLs
- [ ] Bootstrap Talos cluster
- [ ] Store Age key in cluster:
  ```bash
  kubectl create secret generic sops-age \
    --namespace=flux-system \
    --from-file=age.agekey=age-key.txt
  ```
- [ ] Verify Flux deployment:
  ```bash
  flux get all
  kubectl get pods -n kube-system
  kubectl get pods -n tailscale
  ```
