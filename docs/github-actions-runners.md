# GitHub Actions Runner Setup

This setup guide follows the GitHub Docs on
[GitHub Actions / Self-hosted runners / Actions Runner Controller / Quickstart](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/quickstart-for-actions-runner-controller).

Firstly, we install the Actions Runner Controller in namespace `arc-owntube-systems`:

```bash
export CONTROLLER_NAMESPACE=arc-owntube-systems
microk8s helm3 install arc \
    --namespace "${CONTROLLER_NAMESPACE}" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

And then, we need to set up authentication as a GitHub App owned by the OwnTube.tv organisation per the
[Authenticating ARC with a GitHub App](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api#authenticating-arc-with-a-github-app)
documentation.

```bash
export APP_ID=996240
# from https://github.com/organizations/OwnTube-tv/settings/installations/54793885
export INSTALLATION_ID=54793885
export APP_PRIVATE_KEY="$(cat arc-owntube-runner.2024-09-12.private-key.pem)"
export RUNNER_NAMESPACE=arc-owntube-runner
microk8s kubectl create namespace $RUNNER_NAMESPACE
microk8s kubectl create secret generic pre-defined-secret \
   --namespace=$RUNNER_NAMESPACE \
   --from-literal=github_app_id=$APP_ID \
   --from-literal=github_app_installation_id=$INSTALLATION_ID \
   --from-literal=github_app_private_key="$APP_PRIVATE_KEY"
```

Now, configure a runner scale set:

```bash
export INSTALLATION_NAME="arc-owntube-runner"
export RUNNER_NAMESPACE=arc-owntube-runner
export GITHUB_CONFIG_URL="https://github.com/OwnTube-tv"
microk8s helm install "${INSTALLATION_NAME}" \
    --namespace "${RUNNER_NAMESPACE}" \
    --create-namespace \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret=pre-defined-secret \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Run the following command to check your installation:

```bash
microk8s helm list -A
'NAME              	NAMESPACE          	...	STATUS  	CHART                                	APP VERSION'
'arc               	arc-owntube-systems	...	deployed	gha-runner-scale-set-controller-0.9.3	0.9.3      '
'arc-owntube-runner	arc-owntube-runner 	...	deployed	gha-runner-scale-set-0.9.3           	0.9.3      '
```

Check the manager pod:

```bash
microk8s kubectl get pods -n arc-owntube-systems
'NAME                                     READY   STATUS    RESTARTS   AGE  '
'arc-gha-rs-controller-66c95d466b-wb5rt   1/1     Running   0          3h21m'
'arc-owntube-runner-79d95874-listener     1/1     Running   0          4m7s '
```

Continue with the [Using runner scale sets](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/quickstart-for-actions-runner-controller#using-runner-scale-sets)
verification steps from the GitHub Docs to ensure everything is set up correctly.

When jobs are picked up successfully by the controller, they appear as short-lived pods in
the `arc-owntube-runner` namespace:

```bash
microk8s kubectl get pods -n arc-owntube-runner
'NAME                                    READY   STATUS    RESTARTS   AGE'
'arc-owntube-runner-zrcd7-runner-2dn2k   1/1     Running   0          32s'
```
