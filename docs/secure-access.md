# CMDB — Accès Ansible sécurisé sans sudo

## Problème

`sudo` est un vecteur d'attaque connu (~200k LOC, CVE récurrentes : Baron Samedit CVE-2021-3156,
CVE-2023-22809...). Pour une petite équipe, la surface d'attaque est disproportionnée par rapport
au besoin.

## Solution retenue

**Deux mécanismes, zéro SUID :**

1. **SSH forced commands** — universel, tous OS (Linux + AIX, CentOS 5 à RHEL 9)
2. **doas** (Linux) / **RBAC natif** (AIX) — uniquement pour le debug interactif humain

L'automation Ansible n'utilise **jamais** sudo ni doas. Tout passe par SSH forced commands.

## Architecture

```
                    sirius / altair (contrôleur Ansible)
                    ┌─────────────────────────────────┐
                    │  cmdb_collect_ed25519  (user)    │
                    │  cmdb_scan_ed25519    (root)     │
                    └────────┬────────────┬────────────┘
                             │            │
              SSH forced cmd │            │ SSH forced cmd
              (user level)   │            │ (root level)
                             ▼            ▼
                    ┌──────────────────────────────────┐
                    │         Node cible                │
                    │                                   │
                    │  cmdb-collect                  │
                    │    └─ cmdb-collect.sh             │
                    │       • os-release, kernel        │
                    │       • packages (dpkg/rpm/pacman)│
                    │       • containers (docker group) │
                    │                                   │
                    │  root (forced command only)       │
                    │    └─ cmdb-scan.sh                │
                    │       • trivy rootfs / config     │
                    │       • trivy image <name>        │
                    │       • lshw -json                │
                    │       • docker inspect            │
                    │                                   │
                    │  doas (Linux) / RBAC (AIX)        │
                    │    └─ debug interactif humain     │
                    └──────────────────────────────────┘
```

## Deux utilisateurs, deux niveaux de privilège

### cmdb-collect (sans privilège)

- **Compte système** : pas de mot de passe, shell `/bin/sh`
- **Pas de groupe docker** : le socket Docker = accès root (mount `/`, `--privileged`). Tout Docker passe par `cmdb-scan.sh`.
- **SSH forced command** : seul `cmdb-collect.sh` peut être exécuté
- **Commandes autorisées** : `os-release`, `kernel`, `packages`, `collect`
- **Clé SSH** : `cmdb_collect_ed25519` (ed25519) ou `cmdb_collect_rsa` (RSA 4096 pour CentOS 5/6)

### cmdb-scan (root via forced command)

- **Pas de compte dédié** : la clé est dans `/root/.ssh/authorized_keys`
- **SSH forced command** : seul `cmdb-scan.sh` peut être exécuté
- **Commandes autorisées** : `lshw`, `trivy-rootfs`, `trivy-config`, `trivy-image <nom>`, `docker-images`, `docker-inspect`
- **Sanitization** : les noms d'images sont validés par regex `^[a-zA-Z0-9/_:.\-]+$`
- **Clé SSH** : `cmdb_scan_ed25519` (ed25519) ou `cmdb_scan_rsa` (RSA 4096 pour CentOS 5/6)

## Compatibilité OS

| OS | Version | Clé SSH | `restrict` | doas | Testé |
|---|---|---|---|---|---|
| Arch Linux | current | ed25519 | ✓ | `opendoas` (repos officiels) | ✓ |
| Debian | 11+ | ed25519 | ✓ | `doas` (repos officiels) | ✓ |
| Ubuntu | 22.04+ | ed25519 | ✓ | `doas` (repos officiels) | ✓ |
| RHEL | 8, 9 | ed25519 | ✓ | `opendoas` (EPEL) | ✓ |
| Rocky | 8, 9 | ed25519 | ✓ | `opendoas` (EPEL) | ✓ |
| CentOS | 7 | ed25519 | ✓ | `opendoas` (EPEL) | ✓ |
| CentOS | 5, 6 | RSA 4096 | ✗ (flags explicites) | ✗ (non dispo) | ○ |
| AIX | 7.x | ed25519 | ✓ | ✗ (RBAC natif IBM) | ○ |

## Élévation interactive (debug humain)

### Linux — doas

```
# /etc/doas.conf (déployé par bootstrap_access.yml)
permit nopass cmdb-scan cmd /usr/local/bin/trivy
permit nopass cmdb-scan cmd /usr/bin/trivy
permit nopass cmdb-scan cmd /usr/bin/systemctl args status *
permit nopass cmdb-scan cmd /usr/bin/journalctl args -u *
deny cmdb-collect
```

**Pourquoi doas plutôt que sudo :**
- ~6000 LOC vs ~200 000 LOC (sudo)
- Origine OpenBSD, audité régulièrement
- Config en 5 lignes (pas de sudoers avec ses 500 options)
- Historique CVE quasi-vide vs dizaines pour sudo

### AIX — RBAC natif

```bash
mkrole authorizations=aix.system.config.list cmdb_collect_role
chuser roles=cmdb_collect_role cmdb-collect
```

Intégré à l'OS, maintenu par IBM, zéro install.

## Déploiement

### Prérequis

```bash
# Générer les paires de clés (une seule fois, sur le contrôleur)
ssh-keygen -t ed25519 -f ~/.ssh/cmdb_collect_ed25519 -C "cmdb-collect"
ssh-keygen -t ed25519 -f ~/.ssh/cmdb_scan_ed25519 -C "cmdb-scan"

# Optionnel: clés RSA pour CentOS 5/6
ssh-keygen -t rsa -b 4096 -f ~/.ssh/cmdb_collect_rsa -C "cmdb-collect-legacy"
ssh-keygen -t rsa -b 4096 -f ~/.ssh/cmdb_scan_rsa -C "cmdb-scan-legacy"
```

### Exécution (en root, bootstrap initial)

```bash
cd ~/dev/tools/cmdb/ansible
ansible-playbook -i inventory/hosts.ini playbooks/bootstrap_access.yml
```

### Après bootstrap

Les playbooks de collecte et scan doivent être adaptés pour utiliser les nouveaux utilisateurs :

```ini
# inventory/hosts.ini — après bootstrap
[all:vars]
# Collecte = user cmdb-collect (pas de root)
ansible_user=cmdb-collect
ansible_ssh_private_key_file=~/.ssh/cmdb_collect_ed25519

# Scan = override par playbook (root avec forced command)
```

```yaml
# Dans scan_trivy.yml — override pour root forced command
- name: CMDB — Trivy CVE scan
  hosts: all:!workstation
  vars:
    ansible_user: root
    ansible_ssh_private_key_file: ~/.ssh/cmdb_scan_ed25519
```

## Sécurité

### Modèle de menace

| Scénario | Impact avec cette config |
|---|---|
| Clé `collect` compromise | Lecture inventaire uniquement (packages, containers). Pas de shell, pas de root. |
| Clé `scan` compromise | Exécution trivy/lshw/docker inspect uniquement. Pas de shell interactif. |
| Les deux clés compromises | Aucun accès shell. Lecture inventaire + scans Trivy. Pas d'écriture fichier. |
| Dump du contrôleur Ansible | Clés SSH exposées mais sans shell. Rotate les clés et relancer bootstrap. |

### Rotation des clés

```bash
# 1. Générer nouvelles clés
ssh-keygen -t ed25519 -f ~/.ssh/cmdb_collect_ed25519_new -C "cmdb-collect"

# 2. Redéployer
ansible-playbook -i inventory/hosts.ini playbooks/bootstrap_access.yml \
  -e collect_pubkey="$(cat ~/.ssh/cmdb_collect_ed25519_new.pub)"

# 3. Tester puis remplacer
mv ~/.ssh/cmdb_collect_ed25519_new ~/.ssh/cmdb_collect_ed25519
```

## Tokens API — stratégie

Les SSH forced commands sécurisent le **transport**. Le **token CMDB** sécurise l'**API**.

### Playbooks de gestion des tokens

| Playbook | Rôle |
|---|---|
| `deploy_tokens.yml` | Crée un token `collect:w` par node, stocke dans vault |
| `revoke_tokens.yml` | Révoque (soft) ou supprime (hard) les tokens |
| `rotate_tokens.yml` | Régénère les tokens, met à jour les vault files |

```bash
# Option 1 : tokens permanents par node
ansible-playbook playbooks/deploy_tokens.yml --ask-vault-pass

# Option 2 : tokens avec expiration 24h
ansible-playbook playbooks/deploy_tokens.yml -e token_ttl_hours=24 --ask-vault-pass

# Rotation (invalide l'ancien immédiatement)
ansible-playbook playbooks/rotate_tokens.yml --ask-vault-pass

# Révoquer un node compromis
ansible-playbook playbooks/revoke_tokens.yml -e target_node=vega

# Révoquer et supprimer (irréversible)
ansible-playbook playbooks/revoke_tokens.yml -e hard_delete=true

# Nettoyer les tokens expirés
ansible-playbook playbooks/revoke_tokens.yml -e revoke_expired=true
```

### Structure des tokens

```
inventory/
  host_vars/
    vega/
      vault.yml          ← cmdb_api_token: "cmdb_xxx..." (chiffré Ansible Vault)
    altair/
      vault.yml
    sirius/
      vault.yml
    ...
```

Chaque node a son propre token avec scope `collect:w` uniquement.
Un node compromis = révoquer un seul token, les autres restent valides.

### Modèle de menace tokens

| Scénario | Impact |
|---|---|
| Token node volé | Peut pousser de l'inventaire faux pour CE node. Pas de lecture. Révoquer immédiatement. |
| Vault file déchiffrée | Accès au token d'un node. Sans la clé vault, inutilisable. |
| DB dump volé | Tokens hashés HMAC-SHA512. Sans `TOKEN_HMAC_KEY`, inutilisable. |
| Contrôleur Ansible compromis | Vault files chiffrées. Sans vault password, rien n'est lisible. |

## Options d'évolution — tokens éphémères

### Option 1 : Token par node (implémenté)

Token permanent, scope `collect:w`, un par node. Stocké dans Ansible Vault.
Rotation manuelle via `rotate_tokens.yml` ou planifiée via CMDB scheduler.

```bash
ansible-playbook playbooks/deploy_tokens.yml --ask-vault-pass
```

### Option 2 : Token avec expiration (implémenté)

Le schema CMDB supporte déjà `expires_at`. L'auth refuse les tokens expirés.
Utiliser `-e token_ttl_hours=24` pour créer des tokens à durée limitée.

Rotation automatique possible via le CMDB scheduler :
```json
{
  "name": "rotate-collect-tokens",
  "job_type": "woodpecker_pipeline",
  "cron": "0 0 * * *",
  "config": {
    "repo_name": "shdw/cmdb",
    "branch": "main",
    "variables": {"PLAYBOOK": "rotate_tokens", "TOKEN_TTL": "26"}
  }
}
```

Le TTL de 26h laisse 2h de chevauchement avec la rotation quotidienne.

### Option 2b : Vault externe (évolution future)

Au lieu de stocker les tokens dans Ansible Vault (fichiers), les injecter
depuis un vault centralisé. Trois options évaluées :

| Solution | Self-hosted | Maintenu | Complexité | Secret dynamique |
|---|---|---|---|---|
| **Vaultwarden** | ✓ | ✓ actif | Faible | ✗ (coffre statique) |
| **HashiCorp Vault** | ✓ (OSS) | ✓ actif (BSL 1.1 depuis 2023) | Élevée | ✓ (secrets engine) |
| **Infisical** | ✓ (OSS) | ✓ actif (MIT) | Moyenne | ✓ (rotation auto) |

**Vaultwarden** : bon pour stocker des secrets statiques (mots de passe, tokens),
mais pas conçu pour injecter des secrets dans des workflows automation.
C'est un gestionnaire de mots de passe (Bitwarden-compatible), pas un secrets engine.

**HashiCorp Vault** : le standard, mais la licence BSL 1.1 (2023) interdit les usages
concurrents. Pour un usage interne c'est OK. La complexité est élevée (unsealing,
HA, backup du storage backend). Ansible a un plugin natif `hashi_vault`.
```yaml
# Exemple avec HashiCorp Vault
- name: Get CMDB token from Vault
  set_fact:
    cmdb_api_token: "{{ lookup('hashi_vault', 'secret=cmdb/data/{{ inventory_hostname }}:token') }}"
```

**Infisical** : alternative open-source (MIT), API REST simple, rotation automatique
intégrée, CLI légère. Plus simple que HashiCorp Vault pour un solo operator.
```yaml
# Exemple avec Infisical
- name: Get CMDB token from Infisical
  ansible.builtin.uri:
    url: "https://vault.example.com/api/v3/secrets/raw/CMDB_TOKEN_{{ inventory_hostname | upper }}"
    headers:
      Authorization: "Bearer {{ infisical_token }}"
  register: secret
```

**Recommandation** : Infisical si tu veux un vault centralisé sans la complexité
de HashiCorp. Sinon, l'option 2 (tokens avec expiration + rotation scheduler)
couvre 90% du besoin sans dépendance supplémentaire.

### Option 3 : SSH certificates courte durée

Orthogonal aux tokens API — sécurise le transport SSH, pas l'API CMDB.
Combinable avec les options 1 ou 2.

```bash
# Sur le contrôleur, avant chaque ansible-playbook run :
ssh-keygen -s ~/.ssh/ca_key \
  -I "cmdb-collect-$(date +%s)" \
  -V +4h \
  -n cmdb-collect \
  ~/.ssh/cmdb_collect_ed25519.pub
```

Nécessite `TrustedUserCAKeys /etc/ssh/ca.pub` dans `sshd_config` sur chaque node.
Le certificat expire après 4h. Combiné avec un token CMDB à expiration,
les deux canaux (SSH + API) sont éphémères.

### Résumé recommandé

```
Phase 1 (maintenant)  : Option 1 — token par node + Ansible Vault
Phase 2 (rotation)    : Option 2 — token_ttl_hours + scheduler CMDB
Phase 3 (si besoin)   : Option 2b — Infisical ou HashiCorp Vault
Phase 3 (transport)   : Option 3 — SSH certificates (indépendant)
