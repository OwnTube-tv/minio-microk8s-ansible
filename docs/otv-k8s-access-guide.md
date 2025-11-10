# OwnTube.tv Kubernetes Access Guide

## ✅ Setup Complete!

Your local `kubectl` and `k9s` are now configured to access your MicroK8s cluster via SSH tunnel.

---

## Quick Start

### 1. Start the SSH tunnel (if not already running)

```bash
# Start tunnel in background
ssh -f -N -L 16443:192.168.1.6:16443 -p 622 owntube_ansible@83.233.237.206

# Or in foreground (Ctrl+C to stop)
ssh -N -L 16443:192.168.1.6:16443 -p 622 owntube_ansible@83.233.237.206
```

### 2. Switch to the cluster context

```bash
kubectl config use-context otv-k8s
```

### 3. Use `kubectl` or `k9s`

```bash
# kubectl commands
kubectl get nodes
kubectl get pods --all-namespaces
kubectl logs -n arc-systems <pod-name>

# Launch k9s TUI
k9s
```

---

## Managing the SSH Tunnel

### Check if tunnel is running:
```bash
ps aux | grep "ssh.*16443" | grep -v grep
```

### Find and kill the tunnel:
```bash
# Find the PID
ps aux | grep "ssh.*16443" | grep -v grep

# Kill it
kill <PID>

# Or kill all SSH tunnels on port 16443
pkill -f "ssh.*16443"
```

### Restart the tunnel:
```bash
# Kill existing
pkill -f "ssh.*16443"

# Start new one
ssh -f -N -L 16443:192.168.1.6:16443 -p 622 owntube_ansible@83.233.237.206
```

---

## k9s Quick Reference

Once you run `k9s`:

| Key | Action |
|-----|--------|
| `:` | Command mode |
| `:pods` | View pods |
| `:deploy` | View deployments |
| `:svc` | View services |
| `:ns` | View namespaces |
| `Ctrl+a` | Show all namespaces |
| `0` | Show all namespaces (shortcut) |
| `/` | Filter/search |
| `l` | View logs of selected pod |
| `d` | Describe selected resource |
| `e` | Edit selected resource |
| `?` | Help |
| `:q` or `Ctrl+c` | Quit |

**Navigate:**
- `↑/↓` or `j/k` - Move selection
- `Enter` - Drill down into resource
- `Esc` - Go back up

---

## Useful kubectl Commands

### Cluster info:
```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl top nodes  # Requires metrics-server
```

### Pods:
```bash
kubectl get pods -A
kubectl get pods -n arc-systems
kubectl logs -n arc-systems <pod-name>
kubectl logs -f -n arc-systems <pod-name>  # Follow logs
kubectl describe pod -n arc-systems <pod-name>
```

### Deployments:
```bash
kubectl get deployments -A
kubectl get deploy -n arc-systems
kubectl scale deployment <name> --replicas=3 -n <namespace>
```

### Services & Ingress:
```bash
kubectl get svc -A
kubectl get ingress -A
kubectl describe ingress -n minio minio-console
```

### Certificates (cert-manager):
```bash
kubectl get certificates -A
kubectl describe certificate -n minio minio-console-secret
```

---

## Switching Between Clusters

You have multiple clusters configured:

```bash
# List all contexts
kubectl config get-contexts

# Switch to OwnTube.tv MicroK8s
kubectl config use-context otv-k8s

# Switch to AWS EKS
kubectl config use-context eks-cluster-dev

# Switch to Docker Desktop
kubectl config use-context docker-desktop
```

---

## Troubleshooting

### "Unable to connect to the server"

**Check 1:** Is the SSH tunnel running?
```bash
ps aux | grep "ssh.*16443" | grep -v grep
```

**Fix:** Start the tunnel:
```bash
ssh -f -N -L 16443:192.168.1.6:16443 -p 622 owntube_ansible@83.233.237.206
```

---

### "x509: certificate is valid for ... not 127.0.0.1"

**Check:** Are you using the right kubeconfig?
```bash
kubectl config view | grep "server: https://127.0.0.1:16443"
```

**Fix:** Make sure you're using the modified kubeconfig (should be already set up).

---

### Tunnel died or won't start

```bash
# Kill any existing tunnels
pkill -f "ssh.*16443"

# Check if port is in use
lsof -i :16443

# Restart tunnel
ssh -f -N -L 16443:192.168.1.6:16443 -p 622 owntube_ansible@83.233.237.206
```

---

## Backup Information

- **Original kubeconfig backup:** `~/.kube/config.backup-20251110-161843`
- **Current kubeconfig:** `~/.kube/config`
- **Context name:** `otv-k8s`
- **Cluster:** `microk8s-cluster`
- **User:** `admin`

---

## Security Notes

✅ **Tunnel uses SSH authentication** - Your existing SSH key for port 622
✅ **API not exposed to internet** - Only accessible via tunnel
✅ **Admin access** - Full cluster access through kubectl/k9s
⚠️ **Keep tunnel private** - Don't share tunnel access

---

## Auto-start Tunnel (Optional)

If you want the tunnel to start automatically, add to your `~/.zshrc` or `~/.bashrc`:

```bash
# Function to start OTV k8s tunnel
otv-tunnel() {
    # Check if already running
    if ps aux | grep -v grep | grep "ssh.*16443.*owntube_ansible@83.233.237.206" > /dev/null; then
        echo "✅ OTV k8s tunnel already running"
    else
        ssh -f -N -L 16443:192.168.1.6:16443 -p 622 owntube_ansible@83.233.237.206
        echo "✅ OTV k8s tunnel started"
    fi
}

# Auto-start on shell startup (optional)
# otv-tunnel
```

Then reload your shell:
```bash
source ~/.zshrc  # or ~/.bashrc
```

And use:
```bash
otv-tunnel  # Start tunnel
```

---

## Next Steps

1. ✅ Run `k9s` and explore your cluster visually
2. ✅ Check your running pods: `kubectl get pods -A`
3. ✅ Monitor ARC runners: `kubectl logs -f -n arc-systems <controller-pod>`
4. ✅ View certificates: `kubectl get certificates -A`

Enjoy your cluster access! 🎉
