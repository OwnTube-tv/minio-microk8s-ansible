# Bitbucket Pipelines Runner Setup

This guide deploys 4 self-hosted Bitbucket Pipelines runners on a MicroK8s
cluster, one per node, using Docker-in-Docker sidecars (required because
MicroK8s uses containerd, not Docker).

References:
[Set up runners for Linux Docker](https://support.atlassian.com/bitbucket-cloud/docs/set-up-and-use-runners-for-linux/),
[Adding a new runner](https://support.atlassian.com/bitbucket-cloud/docs/adding-a-new-runner-in-bitbucket/),
[Announcing v5 runners](https://www.atlassian.com/blog/bitbucket/announcing-v5-self-hosted-runners)

## Runner versions

All self-hosted runner versions (v3, v4, v5) run on your infrastructure using
the same container image. The version is assigned server-side by Atlassian.

- **v3/v4**: Deprecated June 3, 2026 (monthly) / December 3, 2026 (annual)
- **v5**: Current (5.8.1 as of March 2026). Adds volume mounts, resource
  presets (1x-16x CPU), and private cloud storage for caches/artifacts.

## Step 1: Create 4 workspace runners in Bitbucket

Go to **Workspace settings > Pipelines > Workspace runners** and click
**Add runner** four times. For each:

- Type: **Linux (Docker)**
- Labels: e.g. `smithmicro.k8s`
- Note the `ACCOUNT_UUID`, `RUNNER_UUID`, `OAUTH_CLIENT_ID`, and
  `OAUTH_CLIENT_SECRET` from the Docker command Bitbucket provides

Set the workspace **Concurrency limit** to 4 (under "Steps in queue").

## Step 2: Create namespace, secrets, and node labels

```bash
NAMESPACE=bitbucket-smithmicro-runner
microk8s kubectl create namespace $NAMESPACE

# Create one secret per runner (values from step 1)
for i in 1 2 3 4; do
  echo "Enter credentials for runner $i:"
  read -p "  RUNNER_UUID: " RUNNER_UUID
  read -p "  OAUTH_CLIENT_ID: " OAUTH_CLIENT_ID
  read -p "  OAUTH_CLIENT_SECRET: " OAUTH_CLIENT_SECRET
  microk8s kubectl create secret generic bitbucket-runner-creds-$i \
      --namespace=$NAMESPACE \
      --from-literal=ACCOUNT_UUID='{your-account-uuid}' \
      --from-literal=RUNNER_UUID="$RUNNER_UUID" \
      --from-literal=OAUTH_CLIENT_ID="$OAUTH_CLIENT_ID" \
      --from-literal=OAUTH_CLIENT_SECRET="$OAUTH_CLIENT_SECRET"
done

# Label each node so runners are pinned by label, not hostname
# (survives hostname changes)
microk8s kubectl label node a12a.mabl.online  bitbucket-runner=1
microk8s kubectl label node a12b.mabl.online  bitbucket-runner=2
microk8s kubectl label node v1517a.mabl.online bitbucket-runner=3
microk8s kubectl label node v1517b.mabl.online bitbucket-runner=4
```

## Step 3: Deploy the runners

Each runner is a Deployment with a DinD sidecar (`privileged: true` required),
pinned to a specific node via `nodeSelector`. Resources are set to minimal
requests (100m CPU, 256Mi) with generous limits (8 CPU, 8Gi).

```bash
NAMESPACE=bitbucket-smithmicro-runner

for i in 1 2 3 4; do
cat <<EOF | microk8s kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bitbucket-runner-$i
  namespace: $NAMESPACE
  labels:
    app: bitbucket-runner
    runner: "$i"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bitbucket-runner
      runner: "$i"
  template:
    metadata:
      labels:
        app: bitbucket-runner
        runner: "$i"
    spec:
      nodeSelector:
        bitbucket-runner: "$i"
      containers:
      - name: dind
        image: docker:27-dind
        securityContext:
          privileged: true
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: docker-data
          mountPath: /var/lib/docker
        - name: docker-run
          mountPath: /var/run
      - name: runner
        image: docker-public.packages.atlassian.com/sox/atlassian/bitbucket-pipelines-runner
        envFrom:
        - secretRef:
            name: bitbucket-runner-creds-$i
        env:
        - name: RUNTIME_PREREQUISITES_ENABLED
          value: "true"
        - name: WORKING_DIRECTORY
          value: /tmp
        - name: DOCKER_HOST
          value: unix:///var/run/docker.sock
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: docker-run
          mountPath: /var/run
        - name: docker-data
          mountPath: /var/lib/docker
          readOnly: true
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: "8"
            memory: 8Gi
      volumes:
      - name: tmp
        emptyDir: {}
      - name: docker-data
        emptyDir: {}
      - name: docker-run
        emptyDir: {}
EOF
done
```

## Step 4: Verify

```bash
# All 4 runners should show 2/2 Running, one per node
microk8s kubectl get pods -n bitbucket-smithmicro-runner -o wide

# Check runner logs — look for "ONLINE"
microk8s kubectl logs -n bitbucket-smithmicro-runner -l app=bitbucket-runner \
    -c runner --tail=3

# Confirm in Bitbucket UI: Workspace settings > Pipelines > Workspace runners
# All 4 should show status "Online"
```

## Step 5: Use in bitbucket-pipelines.yml

```yaml
pipelines:
  default:
    - step:
        name: Build
        runs-on:
          - self.hosted
          - smithmicro.k8s
        script:
          - echo "Running on self-hosted K8s runner"
```

## Current deployment

| Runner             | Node                 | Label                | Secret                   | CPU req/limit | Mem req/limit |
|--------------------|----------------------|----------------------|--------------------------|---------------|---------------|
| bitbucket-runner-1 | a12a                 | `bitbucket-runner=1` | bitbucket-runner-creds-1 | 100m / 8      | 256Mi / 8Gi   |
| bitbucket-runner-2 | a12b                 | `bitbucket-runner=2` | bitbucket-runner-creds-2 | 100m / 8      | 256Mi / 8Gi   |
| bitbucket-runner-3 | v1517a (future a12c) | `bitbucket-runner=3` | bitbucket-runner-creds-3 | 100m / 8      | 256Mi / 8Gi   |
| bitbucket-runner-4 | v1517b (future a12d) | `bitbucket-runner=4` | bitbucket-runner-creds-4 | 100m / 8      | 256Mi / 8Gi   |

## Architecture notes

- **Why DinD?** MicroK8s uses containerd. The Bitbucket runner agent requires a
  Docker daemon, so each pod runs a `docker:27-dind` sidecar with `privileged: true`.
- **Why share `/tmp` between DinD and runner?** The runner creates working files
  in `/tmp` (its `WORKING_DIRECTORY`), then tells DinD to bind-mount those paths
  into build/clone containers. Without a shared volume, DinD sees its own empty
  `/tmp` and the clone hangs on a named pipe that was never created.
- **Why not a DaemonSet?** Each runner needs a unique `RUNNER_UUID` / OAuth secret.
  DaemonSets apply the same spec to every node.
- **Why `nodeSelector` on labels, not hostnames?** Node labels survive hostname
  changes. When v1517a/b are renamed to a12c/d, the `bitbucket-runner` label
  stays — no deployment changes needed.
- **Why explicit `requests`?** Kubernetes defaults requests to limits if omitted.
  Without explicit small requests, each pod would reserve 8 CPUs.
- **Scaling**: Each runner handles one build at a time. With 4 runners and
  workspace concurrency set to 4, up to 4 builds can run in parallel.

## Useful links

- [Set up runners for Linux Docker](https://support.atlassian.com/bitbucket-cloud/docs/set-up-and-use-runners-for-linux/)
- [Configure your runner in bitbucket-pipelines.yml](https://support.atlassian.com/bitbucket-cloud/docs/configure-your-runner-in-bitbucket-pipelines-yml/)
- [Runners overview](https://support.atlassian.com/bitbucket-cloud/docs/runners/)
- [Announcing v5 self-hosted runners](https://www.atlassian.com/blog/bitbucket/announcing-v5-self-hosted-runners)
