# `minio-microk8s-ansible` – MinIO S3 Object Storage with a MicroK8s Sidecar

Ansible playbook to configure our Ubuntu 22 servers to run a distributed MinIO S3 service, heavily
inspired by [`mkdevops-se/hq.mkdevops.se`](https://github.com/mkdevops-se/hq.mkdevops.se). MicroK8s
is used here as a sidecar to provide container platform capabilities and handle internet ingress.

## Getting Started

Clone the repo:

    git clone git@github.com:OwnTube-tv/minio-microk8s-ansible.git
    cd minio-microk8s-ansible/

Create a virtual environment and install the dependencies:

    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt

Add the Ansible Vault password to a file named `.ansible_vault_password` and restrict readability:

    echo theSecretAnsibleVaultPassword > .ansible_vault_password
    chmod og-r .ansible_vault_password

Verify that the hosts are reachable:

    ansible minio_microk8s_servers -m ping

Run through the bootstrap playbook in `--check` mode to verify that provisioning can execute:

    ansible-playbook 0-bootstrap.yml --check


## Live Deployment

The setup steps for a live deployment are as follows:

1. Run the `0-bootstrap.yml` playbook to prepare the server baseline for MinIO and MicroK8s setup:

    ```shell
    ansible-playbook 0-bootstrap.yml
    ```

    Follow the instructions in the end of the playbook to establish HA clustering for MicroK8s.

2. Run the `1-microk8s-cluster.yml` playbook to set up MicroK8s add-ons and configure the cluster:

    ```shell
    ansible-playbook 1-microk8s-cluster.yml
    ```

    After the successful completion of the playbook, you can access the Kubernetes dashboard at
    https://k8s-dashboard.owntube.tv/ with a proper certificate and login with a token created from
    one of the MicroK8s cluster nodes:

    ```shell
    kubectl get secret -n kube-system microk8s-dashboard-token \
      -o jsonpath="{.data.token}" | base64 -d
    ```
