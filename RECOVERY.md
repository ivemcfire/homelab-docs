# Disaster Recovery — k3master Rebuild Guide

## If k3master crashes and needs to be rebuilt

### Step 1: Get the age decryption key

The key is stored in **two locations**:

| Location | Path |
|----------|------|
| Jumphost (primary backup) | `user@192.168.100.152:~/.age/key.txt` |
| k3master (if disk survives) | `~/.age/key.txt` |

```bash
scp user@192.168.100.152:~/.age/key.txt ~/.age/key.txt
```

### Step 2: Install age

```bash
# Download binary (no sudo needed)
mkdir -p ~/bin
curl -sL https://github.com/FiloSottile/age/releases/download/v1.2.1/age-v1.2.1-linux-amd64.tar.gz \
  | tar -xz -C /tmp
install -m 755 /tmp/age/age /tmp/age/age-keygen ~/bin/
```

### Step 3: Decrypt credentials

```bash
# Clone this repo
git clone https://github.com/ivemcfire/homelab-docs.git

# Decrypt
~/bin/age -d -i ~/.age/key.txt homelab-docs/credentials.md.age > credentials.md
cat credentials.md
```

### Step 4: Restore repos

```bash
git clone https://github.com/ivemcfire/frigate-k8s.git
git clone https://github.com/ivemcfire/gpu-worker.git
git clone https://github.com/ivemcfire/homelab-docs.git
```

### Step 5: Re-deploy Frigate

```bash
cd frigate-k8s
kubectl apply -f namespace.yaml
# Re-create secrets (password from credentials.md)
kubectl create secret generic frigate-rtsp-creds \
  --from-literal=reolink-password='<from credentials.md>' -n frigate
kubectl apply -f configmap.yaml
kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f cloudflared.yaml
kubectl apply -f backup-cronjob.yaml
```

## Key Facts

- **k3master IP:** 192.168.100.52 (wan0)
- **Jumphost IP:** 192.168.100.152
- **Gitea:** http://192.168.100.206 (mirror — repos also on GitHub)
- **GitHub account:** ivemcfire
- **Domain:** ayurforlife.eu (Cloudflare-managed)

## Age Public Key (for reference)

```
age132jcyde7cjuevvprhlglnjcp93w4aqt9jrx3mszmq2l05sjzfsts867lj2
```

This is the public key only — safe to store here. You need the private key from the jumphost to decrypt.
