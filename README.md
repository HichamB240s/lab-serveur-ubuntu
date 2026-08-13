[procedure-serveur-ubuntu.md](https://github.com/user-attachments/files/31010393/procedure-serveur-ubuntu.md)
# Mise en service d'un serveur Ubuntu : réseau, comptes, SSH et partage Samba

**Auteur** : Hicham Benkaddouche — [LinkedIn](https://linkedin.com/in/hicham-benkaddouche)
**Date** : août 2026
**Contexte** : laboratoire personnel, en complément de la formation Technicien Systèmes, Réseaux et Sécurité (Institut Aston)

---

## Objectif

Mettre en service un serveur Ubuntu depuis une installation vierge jusqu'à un partage de fichiers accessible depuis un poste Windows, en ligne de commande uniquement.

Le fil conducteur est l'interopérabilité : un serveur Linux qui rend service à des postes Windows, avec une gestion des droits par groupe.

---

## Environnement

| Élément | Valeur |
|---|---|
| Hyperviseur | VMware Workstation |
| Système invité | Ubuntu 26.04 LTS |
| Mode réseau | NAT |
| Interface | `ens33` |
| Adresse cible | 192.168.203.50/24 |
| Passerelle | 192.168.203.2 |
| Poste client | Windows 11 (machine hôte) |

---

## Étape 1 — Passer d'une adresse DHCP à une adresse fixe

### Pourquoi

Un serveur doit être joignable à une adresse stable. Une adresse distribuée par DHCP peut changer au redémarrage, ce qui rendrait le partage et l'accès SSH inaccessibles sans reconfiguration côté client.

### État initial

```bash
ip a
ip r
resolvectl status | head -20
```

Relever : le nom de l'interface, l'adresse actuelle, la passerelle (`default via`) et les serveurs DNS.

### Choix de l'adresse

VMware distribue ses baux DHCP à partir de `.128`. On choisit une adresse **en dessous de cette plage** pour éviter tout conflit : `192.168.203.50`.

### Configuration

```bash
sudo nmcli con mod netplan-ens33 \
  ipv4.addresses 192.168.203.50/24 \
  ipv4.gateway 192.168.203.2 \
  ipv4.dns "192.168.203.2 1.1.1.1" \
  ipv4.method manual
```

| Option | Rôle |
|---|---|
| `ipv4.addresses` | Adresse et masque. Le `/24` indique que les trois premiers octets identifient le réseau. |
| `ipv4.gateway` | Sortie vers l'extérieur. |
| `ipv4.dns` | Résolveur principal, plus un résolveur public en secours. |
| `ipv4.method manual` | **Bascule du DHCP au statique.** Sans cette ligne, les précédentes sont ignorées. |

### Application et vérification

```bash
sudo nmcli con up netplan-ens33
ip a
ping -c 3 ubuntu.com
```

Le `ping` valide deux choses d'un coup : la route sort par la passerelle, et la résolution DNS fonctionne.

> **Résultat attendu** : une seule adresse IPv4 sur l'interface, en `valid_lft forever`.
---

## Étape 2 — Utilisateurs, groupes et droits

### Pourquoi

Le partage sera autorisé à un **groupe**, pas à des comptes nommés. Ajouter un collaborateur revient alors à l'ajouter au groupe, sans toucher à la configuration du partage.

### Création

```bash
sudo adduser tech1
sudo addgroup techs
sudo usermod -aG techs tech1
id tech1
```

> `adduser` est interactif : le lancer seul et répondre aux questions. Ubuntu refuse les mots de passe présents dans un dictionnaire et exige au minimum 8 caractères.

### Dossier partagé et permissions

```bash
sudo mkdir /srv/partage
sudo chown root:techs /srv/partage
sudo chmod 770 /srv/partage
ls -ld /srv/partage
```

Lecture du `770` :

| Chiffre | Cible | Droits |
|---|---|---|
| 7 | propriétaire (`root`) | lecture, écriture, exécution |
| 7 | groupe (`techs`) | lecture, écriture, exécution |
| 0 | autres | aucun |

Sur un répertoire, le bit d'exécution autorise à **entrer** dedans.

> **Résultat attendu** : `drwxrwx--- root techs /srv/partage`

---

## Étape 3 — Accès distant par SSH

### Installation

```bash
sudo apt update
sudo apt install openssh-server
systemctl status ssh
ss -tlnp | grep :22
```

### Point de vigilance : l'activation par socket

Sur les versions récentes d'Ubuntu, `systemctl status ssh` peut afficher `inactive (dead)` **alors que le service fonctionne**. La ligne `TriggeredBy: ssh.socket` en est la raison.

Le mécanisme : systemd écoute lui-même sur le port 22 et ne démarre le démon `sshd` qu'à la première connexion entrante. Cela évite de maintenir un processus en mémoire sur une machine peu sollicitée.

**Ne pas lancer `systemctl enable --now ssh`** dans ce cas : deux mécanismes entreraient en concurrence sur le même port.

La vérification fiable est `ss -tlnp | grep :22`, qui montre un écouteur actif.

### Pare-feu

```bash
sudo ufw status
```

Sur une installation par défaut, `ufw` est présent mais inactif. **En production il serait actif**, et il faudrait alors ouvrir explicitement le service :

```bash
sudo ufw allow ssh
```

Oublier le pare-feu est une cause classique de « le service ne répond pas » alors qu'il tourne correctement.

### Test depuis Windows

Le client OpenSSH est intégré à Windows, aucune installation nécessaire.

```powershell
ssh tech1@192.168.203.50
```

À la première connexion, le client affiche l'empreinte de la clé du serveur et demande confirmation. Cette empreinte est mémorisée : toute modification ultérieure déclenchera une alerte, ce qui protège contre l'usurpation du serveur.

---

## Étape 4 — Partage de fichiers Samba

### Installation et compte Samba

```bash
sudo apt install samba
sudo smbpasswd -a tech1
```

> Samba maintient sa **propre base de mots de passe**, distincte de celle du système. Le compte Unix `tech1` doit donc recevoir un mot de passe Samba pour pouvoir s'authentifier en SMB.

### Configuration

Sauvegarder avant toute modification :

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
sudo nano /etc/samba/smb.conf
```

Ajouter en fin de fichier :

```ini
[partage]
   path = /srv/partage
   browseable = yes
   read only = no
   valid users = @techs
   force group = techs
   create mask = 0660
   directory mask = 0770
```

| Directive | Rôle |
|---|---|
| `valid users = @techs` | Autorise le **groupe** entier plutôt que des comptes nommés. |
| `force group` | Tout fichier créé appartient au groupe, quel que soit son auteur. |
| `create mask` / `directory mask` | Garantissent que les fichiers restent accessibles au groupe. |

Sans `force group` ni les masques, les fichiers créés depuis Windows porteraient des droits par défaut et le partage deviendrait inutilisable à plusieurs.

### Validation

```bash
testparm
sudo systemctl restart smbd
systemctl status smbd
```

`testparm` contrôle la syntaxe **avant** le redémarrage du service — réflexe à conserver sur tout fichier de configuration.

### Test depuis Windows

Dans la barre d'adresse de l'Explorateur :

```
\\192.168.203.50\partage
```

S'authentifier avec `tech1` et son mot de passe Samba, puis créer un fichier.

### Vérification côté serveur

```bash
sudo ls -l /srv/partage
```

> **Résultat attendu** : `-rw-rw----+ 1 tech1 techs ... test-depuis-windows.txt`

Deux observations :

- Le `ls` sans `sudo` échoue avec « Permission refusée » depuis un compte hors du groupe `techs`. C'est la preuve que le `770` remplit son rôle.
- Le `+` en fin de permissions signale une **ACL POSIX étendue**, posée automatiquement par Samba. Elle se consulte avec `getfacl /srv/partage`.

---

## Incidents rencontrés

### 1. Deux adresses IPv4 simultanées après le passage en statique

**Symptôme** — après configuration de l'adresse fixe, `ip a` affiche toujours l'ancienne adresse DHCP, marquée `secondary dynamic`, avec un bail qui se renouvelle.

**Cause** — netplan lit les fichiers de `/etc/netplan/` par ordre alphabétique et **fusionne** leurs clés au lieu de les remplacer :

- `00-installer-config.yaml`, écrit à l'installation, déclare `dhcp4: true`
- `90-NM-<uuid>.yaml`, généré par `nmcli`, ajoute l'adresse statique

Aucun des deux ne contredit l'autre : le second n'écrit nulle part `dhcp4: false`. Les deux configurations sont donc appliquées.

**Correction**

```bash
sudo cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.bak
sudo nano /etc/netplan/00-installer-config.yaml
```

Passer `dhcp4` et `dhcp6` à `false`. Le YAML étant sensible à l'indentation, ne modifier que la valeur.

### 2. Perte totale du réseau après `netplan apply`

**Symptôme** — après `sudo netplan apply`, l'interface n'a plus aucune adresse. Message : `Failed to reload network settings: Unit dbus-org.freedesktop.network1.service not found` et `systemd-networkd is not running`.

**Cause** — sur cette installation, le fichier `01-network-manager-all.yaml` désigne **NetworkManager** comme moteur (`renderer: NetworkManager`). `netplan apply` tente malgré tout de recharger via `systemd-networkd`, absent.

**Correction**

```bash
sudo nmcli con reload
sudo nmcli con up netplan-ens33
```

**À retenir** — la configuration réseau a ici deux pilotes : netplan écrit les fichiers, NetworkManager les applique. Sur ce type d'installation, la commande d'application est `nmcli`, pas `netplan apply`.

### 3. Avertissement `gateway4 has been deprecated`

Sans effet sur le fonctionnement. La syntaxe recommandée est désormais un bloc `routes:` avec `to: default`.

---

## Récapitulatif des vérifications

| Étape | Commande | Attendu |
|---|---|---|
| Adressage | `ip a` | une seule IPv4, `valid_lft forever` |
| Routage et DNS | `ping -c 3 ubuntu.com` | 0 % de perte |
| Groupe | `id tech1` | appartenance à `techs` |
| Permissions | `ls -ld /srv/partage` | `drwxrwx--- root techs` |
| SSH | `ss -tlnp \| grep :22` | écouteur actif |
| SSH depuis Windows | `ssh tech1@192.168.203.50` | invite du serveur |
| Samba | `testparm` | configuration validée |
| Partage | `sudo ls -l /srv/partage` | fichier en `tech1 techs`, droits `660` |

---

## Ce que cette mise en service met en œuvre

- Adressage IP statique, passerelle et résolution DNS
- Gestion des comptes, des groupes et des permissions POSIX
- Administration de services avec systemd, et compréhension de l'activation par socket
- Accès distant sécurisé par SSH
- Partage de fichiers interopérable entre Linux et Windows
- Diagnostic d'un conflit de configuration réseau et retour à un état fonctionnel
