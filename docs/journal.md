# Journal de bord - Administration Système

## Vendredi 2 octobre 2026

*   **Dimensionnement du stockage :** Allocation de 50 Go à la VM pour permettre la création des partitions totalisant plus de 30 Go, tel que défini dans le cahier des charges.
*   **Création et installation des machines virtuelles :** Déploiement de deux serveurs avec les ressources requises, `rhel9-lab` sous RHEL 9 et `ubuntu24-lab` sous Ubuntu Server 24.04 LTS.
*   **Partitionnement LVM et systèmes de fichiers :** Mise en place du schéma de partitionnement manuel exigé avec la création d'une partition classique d'amorçage (`/boot`) et d'un volume physique LVM hébergeant le groupe de volumes `vg_sys`.
*   **Séparation des usages :** Découpage du groupe de volumes en plusieurs volumes logiques spécifiques (`lv_root`, `lv_var`, `lv_log`, `lv_home`, `lv_swap`) afin d'isoler le système, le cache, les journaux applicatifs et les données des comptes utilisateurs. L'espace restant du disque physique a été volontairement gardé non alloué pour constituer une réserve d'extension à chaud.
*   **Configuration de base :** Définition des noms d'hôtes et application des mises à jour complètes des systèmes (validé via `dnf history` sur RHEL et `apt history` sur Ubuntu). 

## Séance du 10 Octobre 2026

### Objectifs de la séance
* Gestion des utilisateurs et élévation de privilèges.
* Mise en place de l'authentification SSH par clé.
* Durcissement de base du service SSH.
* Génération des livrables (preuves de configuration).

### Actions réalisées

**1. Gestion des comptes et privilèges (RHEL 9)**
* Récupération du compte `root` suite à un verrouillage prématuré (`passwd -u root`).
* Création des utilisateurs standards (`toto`, puis `killian`) via `useradd`.
* Définition des mots de passe (`passwd`) pour permettre les connexions.
* Ajout des utilisateurs au groupe administrateur `wheel` (`usermod -aG wheel`).
* Vérification de la délégation de droits via `sudo`.

**2. Configuration des accès SSH**
* Export de la clé publique locale (depuis Windows/PowerShell) vers le serveur RHEL 9.
* Dépannage d'une erreur de permission liée à un compte utilisateur non initialisé (mot de passe manquant).
* Vérification de la connexion distante par clé.

**3. Sécurisation du service SSH (RHEL 9 & Ubuntu 24.04)**
* Modification explicite du fichier de configuration pour interdire la connexion directe en root : `PermitRootLogin no`.
* Maintien de l'authentification par mot de passe globale (`PasswordAuthentication yes`) comme exigé par l'énoncé actuel.
* Validation de la configuration (`sshd -t`) et rechargement des services (`sshd` sur Red Hat, `ssh` sur Ubuntu).
* Vérification du blocage effectif de root via `ssh -o BatchMode=yes root@... true`.

**4. Collecte des preuves (Livrables)**
* Création de l'arborescence requise (`tp01/preuves/`).
* Scripting de la collecte d'informations système (horodatage, hostnamectl, lsblk, dnf/apt history, ip, firewall) redirigée vers `rhel9-lab.txt` et `Ubuntu24.04-lab.txt`.
* Génération des exports de partitionnement (`lsblk-rhel9-lab.txt`, `lsblk-ubuntu24-lab.txt`).
* Résolution d'un conflit de permissions (écriture dans le répertoire `~` du bon utilisateur courant).