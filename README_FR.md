> [English version](README_EN.md)

# cmdb_collect — Collecte inventaire CMDB

Playbooks Ansible pour collecter l'inventaire de tous les nodes et le pousser vers l'API CMDB.

Deploye sur les bastions (sirius, rigel) sous le nom `cmdb_collect`.

## Architecture d'acces

Deux utilisateurs dedies, zero sudo/doas pour l'automation :

| Utilisateur | Privilege | Cle SSH | Usage |
|---|---|---|---|
| `cmdb-collect` | sans privilege | `~/.ssh/cmdb_collect_ed25519` | OS, paquets, kernel |
| `cmdb-scan` | root (forced command) | `~/.ssh/cmdb_scan_ed25519` | trivy, lshw, docker inspect |

Chaque utilisateur est restreint par SSH forced command — aucun shell interactif possible.
Voir `docs/secure-access.md` pour le detail du modele de securite.

## Pre-requis

- Cles SSH CMDB generees (voir "Setup initial" ci-dessous)
- Users `cmdb-collect` / `cmdb-scan` deployes sur les nodes (via `bootstrap_access.yml`)
- Acces reseau vers l'API CMDB : `http://10.0.1.1:8000` (vega wg1)
- Token CMDB (voir section "Token CMDB" ci-dessous)

## Setup initial

### 1. Generer les cles SSH

```bash
ansible-playbook playbooks/setup_keys.yml
```

Cree `~/.ssh/cmdb_collect_ed25519` et `~/.ssh/cmdb_scan_ed25519` si elles n'existent pas.
Pour les CentOS 5/6 (legacy RSA) : `ansible-playbook playbooks/setup_keys.yml -e generate_rsa=true`

### 2. Deployer les users sur les nodes

```bash
# Bootstrap initial (en root)
ansible-playbook -i inventory/hosts.ini playbooks/bootstrap_access.yml \
  -e 'ansible_ssh_common_args="-o StrictHostKeyChecking=accept-new"'

# Un seul host
ansible-playbook -i inventory/hosts.ini playbooks/bootstrap_access.yml -l vega
```

Ce playbook :
- Cree les users `cmdb-collect` et `cmdb-scan` (compte systeme, password lock)
- Deploie les wrappers SSH forced command (`cmdb-collect.sh`, `cmdb-scan.sh`)
- Configure les `authorized_keys` avec restriction
- Installe `doas` (Linux) ou configure RBAC (AIX) pour le debug interactif humain

### 3. Deployer les tokens API (optionnel, per-node)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/deploy_tokens.yml --ask-vault-pass
```

Cree un token `collect:w` par node, stocke dans `inventory/host_vars/<node>/vault.yml` (chiffre Ansible Vault).

## Token CMDB

Le token est necessaire pour authentifier les appels a l'API. Trois methodes :

**Methode 1 — tokens per-node (recommande) :**

Deployer via `deploy_tokens.yml` (voir ci-dessus). Chaque node a son propre token dans Ansible Vault.

**Methode 2 — fichier defaults (bastions) :**

Editer `roles/collect_inventory/defaults/main.yml` :
```yaml
cmdb_api_token: "votre-token-ici"
```

**Methode 3 — extra-var (ponctuel) :**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -e cmdb_api_token=TOKEN
```

Le token correspond a `COLLECT_TOKEN` dans le `.env` de la stack CMDB sur vega.

## Premier lancement

Si c'est la premiere connexion depuis ce bastion, les host keys SSH ne sont pas encore connus. Accepter les cles d'un coup :

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml \
  -e 'ansible_ssh_common_args="-o StrictHostKeyChecking=accept-new"' \
  -e cmdb_api_token=TOKEN
```

Les cles sont stockees dans `~/.ssh/known_hosts` — les runs suivants n'en ont plus besoin.

## Inventaire

Fichier : `inventory/hosts.ini`

| Groupe | Hosts | Description |
|--------|-------|-------------|
| bare_metal | vega, altair, deneb | Serveurs physiques |
| container_host | vega, altair, deneb, mizar, alcor, castor, pollux | Hosts Docker |
| vm | antares, rigel, sirius, capella, betelgeuse, mizar, alcor, castor, pollux | Machines virtuelles |
| bastion | rigel, sirius | Points d'entree VPN |
| backup_server | capella, mizar, alcor | Serveurs de backup |
| workstation | procyon | Postes de travail (sudo, pas root) |

## Commandes

### Collecte complete (tous les serveurs)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml
```

### Collecte workstations (sudo)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect_workstations.yml --ask-become-pass
```

### Limiter a un host

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l vega
```

### Limiter a un groupe

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l bare_metal
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l container_host
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l vm
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l bastion
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l backup_server
```

### Limiter a plusieurs hosts

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l "vega,altair,deneb"
```

### Exclure un host ou groupe

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l 'all:!betelgeuse'
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l 'all:!workstation:!backup_server'
```

### Passer le token en extra-var

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -e cmdb_api_token=TOKEN
```

### Dry-run (check mode)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml --check
```

### Vérifier la connectivité API CMDB (tous les hosts)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/check_api.yml
```

Teste les deux réseaux (wg1 `10.0.1.1:8000` et wg0 `10.0.0.1:8000`) et échoue
si un host ne peut joindre l'API sur aucun des deux.

### Verbose (debug)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -vvv
```

### Limiter les taches (tags non definis — usage futur)

```bash
ansible-playbook -i inventory/hosts.ini playbooks/collect.yml -l vega -v
```

## Ce qui est collecte

| Donnee | Source | Notes |
|--------|--------|-------|
| OS, version, arch | `ansible_facts` | Automatique via `gather_facts` |
| Kernel | `uname -r` | |
| Hardware (CPU, RAM, disques) | `inxi -SCmDxxx` | Installe automatiquement si absent |
| Paquets | `dpkg-query` / `pacman -Q` / `rpm -qa` | Selon le package manager (apt, pacman, dnf/yum) |
| Containers Docker | `docker inspect` | Networks, ports, labels, digest |
| Exposure | auto-classifie | internet/wireguard/docker_internal/localhost |
| Traefik/Authelia/GeoIP | labels Docker | Detection automatique des middlewares |
| Repositories | auto-seed | Chaque image container cree une entree repository |
| Type de node | groupes Ansible | Tous les groupes matchants (bare_metal, container_host, vm...) |

## Trivy — Scan CVE

Prerequis : Trivy server sur obs_stack (vega `10.0.1.1:4954`).

### Installer Trivy sur les hosts

```bash
ansible-playbook -i inventory/hosts.ini playbooks/install_trivy.yml
```

### Lancer un scan CVE (rootfs + images Docker)

```bash
# Tous les hosts
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml

# Un seul host
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml -l vega

# Un groupe
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml -l container_host

# Verbose
ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml -l vega -v
```

Le playbook :
1. Verifie que trivy est installe
2. Trouve un repertoire cache avec >2GB libre (`/data`, `/srv`, `/var/lib`, fallback `~/.cache/trivy`)
3. Scan rootfs (`trivy rootfs /`) via le server centralise — skip Docker/containerd overlays
4. Sur les `container_host` : scan chaque image Docker (`trivy image <img>`)
5. POST les resultats vers `POST /api/v1/cves/report`

### Collecte des logs Restic backup

```bash
# Tous les hosts
ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml

# Un host
ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml -l mizar

# Changer la fenetre de collecte (defaut 48h)
ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml -e max_age_hours=72
```

Parse les logs dans `/var/log/restic-backup/` (JSON et texte) et POST vers `POST /api/v1/backup/report`.

### Scan misconfigurations (Trivy config)

```bash
# Tous les hosts
ansible-playbook -i inventory/hosts.ini playbooks/scan_misconf.yml

# Un host
ansible-playbook -i inventory/hosts.ini playbooks/scan_misconf.yml -l vega
```

Detecte les problemes de config systeme (SSH, permissions, Docker, etc.) et POST vers `POST /api/v1/misconfigurations/report`.

### Cron recommande

```cron
# Collecte inventaire — toutes les 6h
0 */6 * * * cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/collect.yml
# Scan Trivy CVE — daily 3h
0 3 * * * cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/scan_trivy.yml
# Scan misconfigurations — weekly dimanche 4h
0 4 * * 0 cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/scan_misconf.yml
# Backup logs — daily 5h
0 5 * * * cd /opt/cmdb_collect && ansible-playbook -i inventory/hosts.ini playbooks/collect_backup_logs.yml
```

## Synchronisation vers les bastions

```bash
rsync -a --delete \
  --exclude='.env' \
  --exclude='.vault_pass' \
  --exclude='.claude/' \
  --exclude='.git/' \
  ./ansible/ bastion:/opt/cmdb_collect/
```

## Playbooks reference

| Playbook | User SSH | Role |
|---|---|---|
| `setup_keys.yml` | localhost | Genere les cles SSH cmdb-collect/cmdb-scan |
| `bootstrap_access.yml` | root | Cree les users + deploie forced commands sur les nodes |
| `collect.yml` | cmdb-collect + root | Collecte inventaire (2 passes) |
| `scan_trivy.yml` | root (cmdb-scan) | Scan CVE rootfs + images Docker |
| `scan_misconf.yml` | root (cmdb-scan) | Scan misconfigurations Trivy |
| `collect_backup_logs.yml` | cmdb-collect | Parse et POST logs Restic |
| `check_drift.yml` | cmdb-collect | Detection d'ecarts de configuration |
| `check_api.yml` | — | Teste la connectivite API CMDB depuis chaque host |
| `install_trivy.yml` | root | Installe le binaire Trivy |
| `deploy_tokens.yml` | localhost | Cree un token API par node (Ansible Vault) |
| `rotate_tokens.yml` | localhost | Regenere les tokens (rotation) |
| `revoke_tokens.yml` | localhost | Revoque ou supprime les tokens |
| `collect_workstations.yml` | shdw (sudo) | Collecte workstations |

## Gestion des tokens

```bash
# Deployer des tokens per-node
ansible-playbook playbooks/deploy_tokens.yml --ask-vault-pass

# Avec expiration 24h
ansible-playbook playbooks/deploy_tokens.yml -e token_ttl_hours=24 --ask-vault-pass

# Rotation (invalide l'ancien immediatement)
ansible-playbook playbooks/rotate_tokens.yml --ask-vault-pass

# Revoquer un node compromis
ansible-playbook playbooks/revoke_tokens.yml -e target_node=vega

# Supprimer definitivement
ansible-playbook playbooks/revoke_tokens.yml -e hard_delete=true
```

## API CMDB

- Endpoint : `POST /api/v1/nodes/report`
- Auth : `Authorization: Bearer <COLLECT_TOKEN>`
- Le token est defini dans le `.env` de la stack CMDB sur vega
