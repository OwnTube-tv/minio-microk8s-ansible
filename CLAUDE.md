# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This Ansible project deploys and manages the S3 object storage infrastructure that powers the OwnTube.tv video platform ecosystem. It provisions a distributed MinIO S3 cluster across 4 physical servers in Sweden, with MicroK8s providing Kubernetes capabilities for load balancing, ingress, cert-manager (Let's Encrypt), and self-hosted GitHub Actions runners.

**Context in the OwnTube.tv Ecosystem:**

- **OwnTube.tv** ([github.com/OwnTube-tv/web-client](https://github.com/OwnTube-tv/web-client)) is a portable video client for PeerTube (decentralized video hosting platform) built with React Native/Expo
- **This repository** provides the S3 storage backend infrastructure for video content and static assets
- **Infrastructure owner:** OwnTube Nordic AB (Swedish org. number: 559517-7196), Stockholm, Sweden
- **Physical location:** 4 servers at two sites in Sweden (a12 and v1517)
- **Total storage capacity:** 64 TB raw across 4 nodes (32 TB usable with MinIO distributed mode)

## Development Environment Setup

1. Create Python virtual environment and install dependencies:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

2. Create Ansible Vault password file (never commit this file):
   ```bash
   echo theSecretAnsibleVaultPassword > .ansible_vault_password
   chmod og-r .ansible_vault_password
   ```

3. Verify host connectivity:
   ```bash
   ansible minio_microk8s_servers -m ping
   ```

## Common Commands

### Running Playbooks

Test a playbook in check mode (dry-run):
```bash
ansible-playbook <playbook>.yml --check
```

Run the bootstrap playbook (initial server setup):
```bash
ansible-playbook 0-bootstrap.yml
```

Run the MicroK8s cluster setup:
```bash
ansible-playbook 1-microk8s-cluster.yml
```

Run the MinIO server setup (requires secrets):
```bash
ansible-playbook -e @secrets.yml 2-minio-servers.yml
```

Run the MinIO OIDC configuration (Auth0 integration):
```bash
ansible-playbook -e @secrets.yml 3-minio-oidc.yml
```

### Selective Execution

Run with specific tags:
```bash
ansible-playbook <playbook>.yml --tags <tag_name>
```

Run on specific hosts:
```bash
ansible-playbook <playbook>.yml --limit node-1
```

Check syntax:
```bash
ansible-playbook <playbook>.yml --syntax-check
```

### Kubernetes Operations

Access the Kubernetes dashboard:
```bash
# Generate a token valid for 1 year (Kubernetes 1.24+)
microk8s kubectl create token default -n kube-system --duration=8760h

# Or for a short-lived token (1 hour, default)
microk8s kubectl create token default -n kube-system
```

Check runner pods:
```bash
microk8s kubectl get pods -n arc-owntube-runner
```

View helm deployments:
```bash
microk8s helm list -A
```

## Architecture

### Physical Infrastructure

Four physical servers (see `docs/hardware.md` for complete specs):

- **minio1 (a12a.mabl.online):** Intel i5-1340P, 64GB RAM, 16TB SSD (8TB M.2 + 8TB SATA)
- **minio2 (a12b.mabl.online):** Intel i5-1340P, 64GB RAM, 16TB SSD (8TB M.2 + 8TB SATA)
- **minio3 (v1517a.mabl.online):** AMD Ryzen 9, 64GB RAM, 16TB SSD (8TB M.2 + 8TB SATA)
- **minio4 (v1517b.mabl.online):** AMD Ryzen 7, 64GB RAM, 16TB SSD (8TB M.2 + 8TB SATA)

Each server has two LVM Volume Groups:
- `ubuntu-vg` (8TB): M.2 NVMe storage for OS and fast access
- `minio-vg` (8TB): SATA SSD storage for MinIO distributed object storage

### Deployment Phases

The deployment follows a sequential 4-phase approach that must be run in order:

#### Phase 0 - Bootstrap (`0-bootstrap.yml`)

Establishes baseline server configuration:
- Configures sysadmin and Ansible service accounts with SSH key authentication
- Hardens SSH (disables root login, restricts max auth tries to 2, removes password auth for sudo users)
- Sets timezone to Europe/Stockholm
- Installs base packages (acl, tree, jq, emacs, python3-kubernetes, etc.)
- Removes firewalld (conflicts with Calico networking)
- Installs MicroK8s via snap
- Prints manual clustering instructions for MicroK8s HA setup

**Important:** After this phase, manual intervention is required to join the MicroK8s nodes into a cluster. The playbook prints instructions showing the exact commands to run on each node.

#### Phase 1 - MicroK8s Cluster (`1-microk8s-cluster.yml`)

Configures Kubernetes capabilities:
- Enables MicroK8s add-ons on designated master (first node alphabetically): dns, helm3, ingress, cert-manager, metallb
- Creates helm3 snap alias
- Configures user access (kubectl config for microk8s_users)
- Deploys Kubernetes resources:
  - Let's Encrypt ClusterIssuer for automatic TLS certificates
  - Kubernetes dashboard with ingress at https://k8s-dashboard.owntube.tv/
  - CoreDNS configuration with fallthrough for host access

**Designated Master Pattern:** The first node alphabetically (`(groups[microk8s_servers_group] | sort)[0]`) acts as the designated master for initial cluster operations.

#### Phase 2 - MinIO Servers (`2-minio-servers.yml`)

Sets up distributed MinIO S3 object storage:
- Creates minio-user account (UID 1010)
- Configures MinIO storage disks from minio-vg LVM volume group
- Sets MinIO environment variables (root credentials, cluster endpoints)
- Deploys MinIO as systemd service (not in Kubernetes)
- Configures MinIO client (`mc`) with:
  - Local MinIO cluster alias
  - External S3 provider aliases (AWS S3, Backblaze B2) for data migration/replication
- Creates Kubernetes ingress for external HTTPS access at https://minio.owntube.tv/

**Design Note:** MinIO runs as a systemd service on bare metal (not containerized) for maximum performance, while Kubernetes provides ingress and load balancing.

#### Phase 3 - MinIO OIDC (`3-minio-oidc.yml`)

Integrates Auth0 OpenID Connect for authentication:
- Configures MinIO with Auth0 OAuth (config URL, client ID, client secret)
- Enables policy-based access control via Auth0 custom claims
- Supports user-to-policy mapping in Auth0 Action
- Requires manual policy creation in MinIO console (noaccess, swt-readwrite, ot-readwrite)

See README.md section "Add OpenID Connect using Auth0" for complete setup instructions including Auth0 tenant configuration and policy JSON.

### Role Structure

**`common`** - Base system hardening and packages
- Configures sysadmin accounts and Ansible service account
- Enforces strict SSH security (no root login, AllowUsers whitelist, no password auth for sudo)
- Removes ~/.ssh/authorized_keys for root user
- Sets Europe/Stockholm timezone
- Installs essential packages
- Removes firewalld (conflicts with Calico)

**`microk8s-node`** - MicroK8s installation
- Installs MicroK8s via snap (stable channel)
- Creates kubectl snap alias
- Adds users to microk8s group
- Prints clustering instructions (manual execution required)

**`microk8s-cluster`** - Kubernetes configuration
- Enables add-ons on designated master first, then other nodes
- Configures user kubectl access
- Deploys cert-manager, dashboard ingress, CoreDNS settings
- Task files: `user-configurations.yml`, `k8s-configurations.yml`

**`minio-server`** - MinIO S3 service
- Creates minio-user account
- Configures storage from LVM volume groups
- Sets up MinIO environment and systemd service
- Configures `mc` client with local and external aliases
- Deploys Kubernetes ingress
- Task files: `minio-disks.yml`, `minio-env.yml`, `minio-server.yml`, `minio-client.yml`

**`minio-oidc`** - Auth0 integration
- Configures MinIO OpenID Connect settings
- Integrates with Auth0 tenant for authentication
- Task files: `minio-oidc.yml`

### GitHub Actions Runner Setup

Self-hosted GitHub Actions runners deployed on MicroK8s using Actions Runner Controller (ARC):

- **Architecture:** Single shared controller managing multiple runner scale sets for different GitHub organizations
- **Deployment:** Ephemeral runner pods spawn on-demand when workflow jobs are triggered

See `docs/github-actions-runners.md` for installation and verification steps.

### Configuration Files

- **`ansible.cfg`:** Configures roles path, inventory file, vault password file, and display settings
- **`hosts.yml`:** Defines `minio_microk8s_servers` inventory group with 4 nodes (custom SSH port 622, owntube_ansible user)
- **`secrets.yml`:** Ansible Vault encrypted file containing:
  - MinIO root credentials
  - External S3 provider credentials (AWS, Backblaze B2)
  - Auth0 OAuth settings (config URL, client ID, client secret)
  - User-to-policy mappings for Auth0 integration

## Important Implementation Notes

### Ansible Best Practices

- Always use FQCN (Fully Qualified Collection Name) for modules: `ansible.builtin.user`, `community.general.snap`, `ansible.posix.authorized_key`
- All roles use extensive tagging for selective execution (e.g., `common`, `microk8s-node`, `minio-server`)
- Complex roles break down tasks using `import_tasks` or `include_tasks` for maintainability
- Role variables passed from playbooks (e.g., `microk8s_servers_group`, `microk8s_users`, `minio_servers_group`)

### Security Considerations

- **Secrets management:** All sensitive data stored in Ansible Vault (`secrets.yml`). Never commit unencrypted secrets.
- **SSH hardening:** Root login disabled, authorized_keys removed, max auth tries limited to 2, AllowUsers whitelist enforced
- **Service account:** Ansible service account (`owntube_ansible`) has passwordless sudo for automation
- **Network ports:** Custom SSH port 622 (non-standard port reduces automated attacks)
- **TLS certificates:** Automatic Let's Encrypt certificates via cert-manager for all ingresses

### Infrastructure Dependencies

- **MicroK8s clustering:** Manual intervention required after bootstrap to join nodes into HA cluster
- **Firewalld removal:** Mandatory - conflicts with Calico iptables networking used by MicroK8s
- **Designated master:** First node alphabetically always acts as initial master for add-on enablement
- **LVM volume groups:** Assumes pre-existing `ubuntu-vg` and `minio-vg` on all nodes
- **DNS records:** Assumes external DNS configured for `*.owntube.tv` pointing to cluster ingress

### Known Hardware Quirks

See `docs/hacks-and-troubleshooting.md` for details:

**Realtek r8125 NIC drivers (minio3 and minio4):**
- Servers at v1517 site have 2.5 GbE NICs with problematic Ubuntu drivers
- Custom systemd service `ar9708-r8125-hack.service` reinstalls Realtek drivers on each boot
- Without this hack, NICs freeze after several days of uptime
- Driver source: `/root/r8125-9.012.04/` (extracted from Realtek tarball)
- Verification: `lsmod | grep r8125` should show driver loaded

### External Integration Points

**MinIO Client Aliases:**
- Local cluster: `mc alias set minio https://minio.owntube.tv/`
- AWS S3 (multiple accounts): Used for data migration and replication workflows
- Backblaze B2: Alternative cloud storage provider

**Auth0 Integration:**
- OpenID Connect authentication for MinIO web console
- Custom Auth0 Action adds policy claims to JWT tokens
- User-to-policy mapping in Auth0 secrets (see README.md for JSON format)
- Manual policy creation required in MinIO console after OIDC setup

## OwnTube.tv Ecosystem Context

This infrastructure repository is one component of the broader OwnTube.tv platform:

**Related Repositories:**
- **[OwnTube-tv/web-client](https://github.com/OwnTube-tv/web-client):** Main video client (React Native/Expo) for PeerTube
- **[OwnTube-tv/cust-app-template](https://github.com/OwnTube-tv/cust-app-template):** Template for creating branded video apps
- **[OwnTube-tv/www.owntube.tv](https://github.com/OwnTube-tv/www.owntube.tv):** Marketing website
- **Branded apps:** cust-app-blender, cust-app-xrtube, cust-app-privacytube, cust-app-basspistol

**Infrastructure Usage:**
- S3 storage for video content, thumbnails, and static assets
- Hosting for GitHub Pages deployments (web-client and branded apps)
- CI/CD infrastructure (self-hosted GitHub Actions runners)
- Asset distribution for mobile/TV app builds

**Platform Philosophy:**
OwnTube enables "many branded apps with few users each" rather than "one app with many users." Each content distributor maintains their own app store presence, review processes, and content responsibility while leveraging shared infrastructure and codebase.

## Additional Documentation

- **`docs/hardware.md`:** Complete hardware specifications for all 4 MinIO servers
- **`docs/github-actions-runners.md`:** Self-hosted GitHub Actions runner installation and verification
- **`docs/hacks-and-troubleshooting.md`:** Hardware quirks and reproducible workarounds
- **`README.md`:** Getting started guide, deployment walkthrough, Auth0 OIDC setup

## Contact & Organization

- **Company:** OwnTube Nordic AB (Swedish org. number: 559517-7196)
- **Location:** Stockholm, Sweden
- **Project contact:** hello@owntube.tv
- **Company website:** [www.owntube.se](https://www.owntube.se)
- **Project website:** [www.owntube.tv](https://www.owntube.tv)
- **GitHub Organization:** [github.com/OwnTube-tv](https://github.com/OwnTube-tv)
