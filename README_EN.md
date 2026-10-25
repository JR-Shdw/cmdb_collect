> [Version francaise](README_FR.md)

# cmdb_collect — CMDB Inventory Collection

Ansible playbooks to collect the inventory of all nodes and push it to the CMDB API.

Deployed on bastions (sirius, rigel) under the name `cmdb_collect`.

## Access architecture

Two dedicated users, zero sudo/doas for automation:

| User | Privilege | SSH key | Usage |
|---|---|---|---|
| `cmdb-collect` | unprivileged | `~/.ssh/cmdb_collect_ed25519` | OS, packages, kernel |
| `cmdb-scan` | root (forced command) | `~/.ssh/cmdb_scan_ed25519` | trivy, lshw, docker inspect |

Each user is restricted by SSH forced command — no interactive shell possible.
See `docs/secure-access.md` for the full security model.

## Prerequisites

- CMDB SSH keys generated (see "Initial setup" below)
- `cmdb-collect` / `cmdb-scan` users deployed on nodes (via `bootstrap_access.yml`)
- Network access to the CMDB API: `http://10.0.1.1:8000` (vega wg1)
- CMDB token (see "CMDB Token" section below)

## Initial setup

### 1. Generate SSH keys

```bash
ansible-playbook playbooks/setup_keys.yml
```

Creates `~/.ssh/cmdb_collect_ed25519` and `~/.ssh/cmdb_scan_ed25519` if they don't exist.
For CentOS 5/6 (legacy RSA): `ansible-playbook playbooks/setup_keys.yml -e generate_rsa=true`

### 2. Deploy users on nodes

```bash
# Initial bootstrap (as root)
ansible-playbook -i inventory/hosts.ini playbooks/bootstrap_access.yml \
  -e 'ansible_ssh_common_args="-o StrictHostKeyChecking=accept-new"'

# Single host
ansible-playbook -i inventory/hosts.ini playbooks/bootstrap_access.yml -l vega
```

This playbook:
- Creates `cmdb-collect` and `cmdb-scan` users (system account, password locked)
- Deploys SSH forced command wrappers (`cmdb-collect.sh`, `cmdb-scan.sh`)
- Configures `authorized_keys` with restriction
- Installs `doas` (Linux) or configures RBAC (AIX) for interactive human debug

### 3. Deploy API tokens (optional, per-node)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/deploy_tokens.yml --ask-vault-pass
```

Creates a `collect:w` token per node, stored in `inventory/host_vars/<node>/vault.yml` (Ansible Vault encrypted).

## CMDB Token

The token is required to authenticate API calls. Three methods:

**Method 1 — per-node tokens (recommended):**

Deploy via `deploy_tokens.yml` (see above). Each node gets its own token in Ansible Vault.

**Method 2 — defaults file (bastions):**

Edit `roles/collect_inventory/defaults/main.yml`:
```yaml
cmdb_api_token: "your-token-here"
```

**Method 3 — extra-var (one-off):**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -e cmdb_api_token=TOKEN
```

The token corresponds to `COLLECT_TOKEN` in the `.env` of the CMDB stack on vega.

## First run

If this is the first connection from this bastion, SSH host keys are not yet known. Accept the keys in one go:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml \
  -e 'ansible_ssh_common_args="-o StrictHostKeyChecking=accept-new"' \
  -e cmdb_api_token=TOKEN
```

Keys are stored in `~/.ssh/known_hosts` — subsequent runs no longer need this.

## Inventory

File: `inventory/hosts.ini`

| Group | Hosts | Description |
|-------|-------|-------------|
| bare_metal | vega, altair, deneb | Physical servers |
| container_host | vega, altair, deneb, mizar, alcor, castor, pollux | Docker hosts |
| vm | antares, rigel, sirius, capella, betelgeuse, mizar, alcor, castor, pollux | Virtual machines |
| bastion | rigel, sirius | VPN entry points |
| backup_server | capella, mizar, alcor | Backup servers |
| workstation | procyon | Workstations (sudo, not root) |

## Commands

### Full collection (all servers)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml
```

### Workstation collection (sudo)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect_workstations.yml --ask-become-pass
```

### Limit to a single host

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l vega
```

### Limit to a group

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l bare_metal
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l container_host
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l vm
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l bastion
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l backup_server
```

### Limit to multiple hosts

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l "vega,altair,deneb"
```

### Exclude a host or group

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l 'all:!betelgeuse'
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l 'all:!workstation:!backup_server'
```

### Pass the token as extra-var

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -e cmdb_api_token=TOKEN
```

### Dry-run (check mode)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml --check
```

### Check CMDB API connectivity (all hosts)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/check_api.yml
```

Tests both networks (wg1 `10.0.1.1:8000` and wg0 `10.0.0.1:8000`) and fails
if a host cannot reach the API on either of them.

### Verbose (debug)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -vvv
```

### Limit tasks (tags not defined — future use)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l vega -v
```

## What is collected

| Data | Source | Notes |
|------|--------|-------|
| OS, version, arch | `ansible_facts` | Automatic via `gather_facts` |
| Kernel | `uname -r` | |
| Hardware (CPU, RAM, disks) | `inxi -SCmDxxx` | Automatically installed if missing |
| Packages | `dpkg-query` / `pacman -Q` / `rpm -qa` | Depending on the package manager (apt, pacman, dnf/yum) |
| Docker containers | `docker inspect` | Networks, ports, labels, digest |
| Exposure | auto-classified | internet/wireguard/docker_internal/localhost |
| Traefik/Authelia/GeoIP | Docker labels | Automatic middleware detection |
| Repositories | auto-seed | Each container image creates a repository entry |
| Node type | Ansible groups | All matching groups (bare_metal, container_host, vm...) |

## Trivy — CVE Scan

Prerequisite: Trivy server on obs_stack (vega `10.0.1.1:4954`).

### Install Trivy on hosts

```bash
ansible-playbook -i inventory/hosts.ini playbooks/install_trivy.yml
```

### Run a CVE scan (rootfs + Docker images)

```bash
# All hosts
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml

# Single host
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml -l vega

# A group
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml -l container_host

# Verbose
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml -l vega -v
```

The playbook:
1. Checks that trivy is installed
2. Finds a cache directory with >2GB free (`/data`, `/srv`, `/var/lib`, fallback `~/.cache/trivy`)
3. Scans rootfs (`trivy rootfs /`) via the centralized server — skips Docker/containerd overlays
4. On `container_host` nodes: scans each Docker image (`trivy image <img>`)
5. POSTs results to `POST /api/v1/cves/report`

### Collect Restic backup logs

```bash
# All hosts
ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml

# Single host
ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml -l mizar

# Change the collection window (default 48h)
ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml -e max_age_hours=72
```

Parses logs in `/var/log/restic-backup/` (JSON and text) and POSTs to `POST /api/v1/backup/report`.

### Misconfiguration scan (Trivy config)

```bash
# All hosts
ansible-playbook -i inventory/hosts.ini playbooks/scan_misconf.yml

# Single host
ansible-playbook -i inventory/hosts.ini playbooks/scan_misconf.yml -l vega
```

Detects system configuration issues (SSH, permissions, Docker, etc.) and POSTs to `POST /api/v1/misconfigurations/report`.

### Recommended cron

```cron
# Inventory collection — every 6h
0 */6 * * * cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/collect.yml
# Trivy CVE scan — daily at 3am
0 3 * * * cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml
# Misconfiguration scan — weekly Sunday at 4am
0 4 * * 0 cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/scan_misconf.yml
# Backup logs — daily at 5am
0 5 * * * cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml
```

## Sync to bastions

```bash
rsync -a --delete \
  --exclude='.env' \
  --exclude='.vault_pass' \
  --exclude='.claude/' \
  --exclude='.git/' \
  ./ansible/ bastion:/opt/cmdb_collect/
```

## Playbooks reference

| Playbook | SSH user | Role |
|---|---|---|
| `setup_keys.yml` | localhost | Generate cmdb-collect/cmdb-scan SSH keys |
| `bootstrap_access.yml` | root | Create users + deploy forced commands on nodes |
| `collect.yml` | cmdb-collect + root | Inventory collection (2 passes) |
| `scan_trivy.yml` | root (cmdb-scan) | CVE scan rootfs + Docker images |
| `scan_misconf.yml` | root (cmdb-scan) | Trivy misconfiguration scan |
| `collect_backup_logs.yml` | cmdb-collect | Parse and POST Restic logs |
| `check_drift.yml` | cmdb-collect | Configuration drift detection |
| `check_api.yml` | — | Test CMDB API connectivity from each host |
| `install_trivy.yml` | root | Install Trivy binary |
| `deploy_tokens.yml` | localhost | Create one API token per node (Ansible Vault) |
| `rotate_tokens.yml` | localhost | Regenerate tokens (rotation) |
| `revoke_tokens.yml` | localhost | Revoke or delete tokens |
| `collect_workstations.yml` | shdw (sudo) | Workstation collection |

## Token management

```bash
# Deploy per-node tokens
ansible-playbook playbooks/deploy_tokens.yml --ask-vault-pass

# With 24h expiration
ansible-playbook playbooks/deploy_tokens.yml -e token_ttl_hours=24 --ask-vault-pass

# Rotate (invalidates old token immediately)
ansible-playbook playbooks/rotate_tokens.yml --ask-vault-pass

# Revoke a compromised node
ansible-playbook playbooks/revoke_tokens.yml -e target_node=vega

# Hard delete (permanent)
ansible-playbook playbooks/revoke_tokens.yml -e hard_delete=true
```

## CMDB API

- Endpoint: `POST /api/v1/nodes/report`
- Auth: `Authorization: Bearer <COLLECT_TOKEN>`
- The token is defined in the `.env` of the CMDB stack on vega
