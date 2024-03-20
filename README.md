
# `minio-ansible`

Ansible playbook to configure our Ubuntu 22 servers to run a distributed MinIO S3 service, heavily
inspired by [`mkdevops-se/hq.mkdevops.se`](https://github.com/mkdevops-se/hq.mkdevops.se).


## Getting Started

Clone the repo:

    git clone git@github.com:OwnTube-tv/minio-ansible.git && cd minio-ansible/

Create a virtual environment and install the dependencies:

    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt

Add the Ansible Vault password to a file named `.vault-pass` and restrict readability.

    echo theSecretAnsibleVaultPassword > .vault-pass
    chmod og-r .vault-pass

Verify that the hosts are reachable:

    ansible minio_servers -m ping

Run through the bootstrap playbook in `--check` mode to verify that provisioning can execute:

    ansible-playbook bootstrap.yml --check

