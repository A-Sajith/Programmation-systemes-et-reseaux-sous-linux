# Installation de rhel9-lab et ubuntu24-lab

## 1. Périmètre

* **Machine 1 (RHEL) :** Red Hat Enterprise Linux 9.8 (Plow).
* **Machine 2 (Ubuntu) :** Ubuntu Server 24.04.5 LTS.

## 2. Ressources des machines virtuelles

| Paramètre      | rhel9-lab                        | ubuntu24-lab                     |
|:---------------|:---------------------------------|:---------------------------------|
| **Processeurs**    | 2 vCPU                           | 2 vCPU                           |
| **Mémoire (RAM)**  | 2048 Mio                         | 2048 Mio                         |
| **Disque Système** | 50 Gio (alloués dynamiquement)   | 50 Gio (alloués dynamiquement)   |
| **Réseau**         | NAT + lab-interne (10.0.60.1/24) | NAT + lab-interne (10.0.60.2/24) |

*(Note : Le disque a été provisionné à 50 Gio sur l'hyperviseur pour les deux machines, contrairement aux 25/30 Gio du plan initial, mais l'espace a été géré via LVM pour conserver une réserve proportionnelle).*

## 3. Partitionnement appliqué

**Tableau des partitions obtenues (identique sur RHEL et Ubuntu, aux systèmes de fichiers près) :**

| Volume              | Taille obtenue | Point de montage | Système de fichiers        |
|:--------------------|:---------------|:-----------------|:---------------------------|
| `/dev/sda1` (ou `sda2`) | 1 Gio          | `/boot`            | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `vg_sys-lv_root`      | 20 Gio         | `/`                | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `vg_sys-lv_var`       | 10 Gio         | `/var`             | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `vg_sys-lv_log`       | 5 Gio          | `/var/log`         | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `vg_sys-lv_home`      | 5 Gio          | `/home`            | `xfs` (RHEL) / `ext4` (Ubuntu) |
| `vg_sys-lv_swap`      | 2 Gio *        | `[SWAP]`           | `swap`                       |
| `tmpfs`               | 512 Mio        | `/tmp`             | `tmpfs` (Ubuntu uniquement)  |
| **Espace libre**        | ~ 5 Gio        | *Non monté*        | *Réserve LVM (vg_sys)*       |

*\*Anomalie constatée : Sur RHEL 9, le volume swap a été généré avec une taille de 4 Mio au lieu de 2 Gio suite à une erreur de saisie dans l'interface de partitionnement. À corriger lors d'une prochaine intervention.*

**Justification des choix :**

* **Séparation de /var/log :** Isoler ce volume empêche un crash complet du système en cas de saturation causée par la prolifération de journaux d'erreurs d'une application.
* **Partition /boot classique :** Conservée hors LVM pour simplifier le processus d'amorçage et permettre au chargeur (GRUB) de lire le noyau sans nécessiter le chargement préalable de modules complexes.
* **Taille du Swap (2 Gio) :** Alignée sur la quantité de RAM pour éviter l'intervention de l'OOM Killer en cas de pic de charge, sans pour autant dégrader les performances globales avec un swap trop volumineux.
* **Espace non alloué :** Gardé libre au sein du groupe de volumes `vg_sys` pour conserver la flexibilité de LVM, ce qui permettra d'agrandir des partitions à chaud (comme `/var/log`) ou de réaliser des snapshots ultérieurement.

## 4. Installation

* **Profil :** Installation minimale (sans environnement graphique, basic functionality).
* **Noms d'hôtes :** `rhel9-lab` et `ubuntu24-lab`.
* **Paramètres régionaux :** Locale clavier configurée en `fr-azerty` (Red Hat) / `fr pc105` (Ubuntu).
* **Fuseau horaire et horloge :** Configuré sur `Europe/Paris` avec la synchronisation NTP active (`system clock synchronized: yes`).

## 5. Mises à jour

L'ensemble des paquets a été mis à jour après le déploiement du système de base.

* **RHEL 9 :** `sudo dnf -y update` (Validé par l'historique DNF : `update` appliqué avec succès).
* **Ubuntu 24.04 :** `sudo apt update && sudo apt -y full-upgrade` (Validé par l'historique APT et `unattended-upgrades`).

## 6. Comptes et accès distant

* **Comptes créés :** Administrateurs nominatifs (`toto` et `killian`).
* **Délégation :** Les droits d'administration sont délégués via le groupe `wheel` sur RHEL 9 et le groupe `sudo` sur Ubuntu.
* **Configuration SSH (**`sshd_config`**) :**
  * Interdiction explicite de la connexion directe en administrateur (`PermitRootLogin no`).
  * Maintien temporaire de l'authentification par mot de passe (`PasswordAuthentication yes`) conformément aux instructions.
  * Mise en place de l'authentification par clé SSH (ED25519) sur le compte standard depuis le poste client Windows.

## 7. Pare-feu

L'accès est filtré pour n'autoriser que les flux nécessaires (SSH).

* **Outils :** `firewalld` (RHEL) et `ufw` (Ubuntu).
* **Règles appliquées :** Ouverture exclusive du port `22/tcp` (OpenSSH).
* **Vérification :** Validée via `sudo firewall-cmd --list-all` (RHEL) et `sudo ufw status verbose` (Ubuntu).

## 8. Écarts entre les deux distributions

| Opération               | RHEL 9                | Ubuntu 24.04         | Raison de la différence                                                          |
|:------------------------|:----------------------|:---------------------|:---------------------------------------------------------------------------------|
| **Gestionnaire de paquets** | `dnf`                   | `apt`                  | Outils natifs différents selon la famille de l'OS (Red Hat vs Debian).           |
| **Service Pare-feu**        | `firewalld`             | `ufw`                  | Outils de frontend par défaut (`firewall-cmd` vs `ufw`) interagissant avec le noyau. |
| **Rechargement SSH**        | `systemctl reload sshd` | `systemctl reload ssh` | Nommage différent du démon/service selon la distribution.                        |
| **Groupe d'administration** | `wheel`                 | `sudo`                 | Convention historique différente pour le groupe octroyant les droits root.       |

## 9. Incidents et corrections

Plusieurs incidents mineurs ont été rencontrés et corrigés durant la configuration:

1. **Verrouillage prématuré de root :** Le compte root a été verrouillé (`passwd -l root`) avant de tester les accès du compte utilisateur. *Correction :* Déverrouillage immédiat via `passwd -u root` depuis la session encore ouverte.
2. **Échec d'envoi de la clé SSH (Permission denied) :** Impossible de copier la clé publique via PowerShell vers le compte distant `killian`. *Diagnostic :* Le compte nouvellement créé avec `useradd` n'avait pas encore de mot de passe défini, ce qui bloque l'accès distant par défaut. *Correction :* Initialisation avec `passwd killian` avant de relancer le transfert.
3. **Permissions refusées lors de la collecte des preuves :** Impossibilité d'écrire le fichier `lsblk-rhel9-lab.txt`. *Diagnostic :* Le terminal était positionné dans le répertoire personnel de `toto` avec l'utilisateur `killian`. *Correction :* Déplacement vers le répertoire personnel légitime (`cd ~`) avant la génération du fichier.

## 10. Reconstruction rapide

Séquence minimale post-installation pour configurer l'administration d'un serveur vierge :

**Sur RHEL 9 :**

```bash
sudo dnf -y update
sudo useradd -m -c "Admin System" killian
sudo passwd killian
sudo usermod -aG wheel killian
sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload sshd
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

**Sur Ubuntu 24.04:**

```bash
sudo apt update && sudo apt -y full-upgrade
sudo adduser killian
sudo usermod -aG sudo killian
sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload ssh
sudo ufw allow OpenSSH
sudo ufw enable
```
