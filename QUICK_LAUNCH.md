# CMDB Collect — Quick Launch

## Setup initial (une seule fois)

```bash
# 1. Generer les cles SSH (sur le controleur)
ansible-playbook playbooks/setup_keys.yml

# 2. Deployer les users cmdb-collect/cmdb-scan sur tous les nodes (en root)
ansible-playbook -i inventory/hosts.ini playbooks/bootstrap_access.yml \
  -e 'ansible_ssh_common_args="-o StrictHostKeyChecking=accept-new"'

# 3. Installer Trivy sur les nodes
ansible-playbook -i inventory/hosts.ini playbooks/install_trivy.yml

# 4. (Optionnel) Deployer des tokens API per-node
ansible-playbook -i inventory/hosts.ini playbooks/deploy_tokens.yml --ask-vault-pass
```

## Collecte courante

```bash
# Inventaire complet
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -e cmdb_api_token=TOKEN

# Scan CVE (rootfs + images Docker)
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml -e cmdb_api_token=TOKEN

# Scan misconfigurations
ansible-playbook -i inventory/hosts.ini playbooks/scan_misconf.yml -e cmdb_api_token=TOKEN

# Logs backup Restic
ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml -e cmdb_api_token=TOKEN

# Drift detection
ansible-playbook -i inventory/hosts.ini playbooks/check_drift.yml -e cmdb_api_token=TOKEN

# Workstations (sudo)
ansible-playbook -i inventory/hosts.ini playbooks/collect_workstations.yml --ask-become-pass
```

## Diagnostics

```bash
# Tester la connectivite API depuis tous les hosts
ansible-playbook -i inventory/hosts.ini playbooks/check_api.yml

# Limiter a un host
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l vega -e cmdb_api_token=TOKEN

# Verbose
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -v -e cmdb_api_token=TOKEN
```

## Tokens

```bash
# Rotation
ansible-playbook playbooks/rotate_tokens.yml --ask-vault-pass

# Revoquer un node
ansible-playbook playbooks/revoke_tokens.yml -e target_node=vega

# Supprimer definitivement
ansible-playbook playbooks/revoke_tokens.yml -e hard_delete=true
```
