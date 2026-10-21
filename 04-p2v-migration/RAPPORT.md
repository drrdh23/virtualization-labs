# Projet 4 — Migration d'un serveur physique vers une VM (P2V)

| | |
| --- | --- |
| **Cours** | Virtualisation — Mettre en place et exploiter une plateforme de virtualisation |
| **Classe** | E1A |
| **Niveau** | Avancé |
| **Durée** | 1 semaine (PPE) |
| **Auteur** | Rrudhani Deaa |
| **Date** | 07/10/2026 |

---

## <a id="sommaire"></a>Sommaire

1. [Contexte et objectifs](#s1)
2. [Le P2V : principe et choix de la méthode](#s2)
3. [Scénario et architecture](#s3)
    - [3.1 Les deux serveurs](#s3-1)
    - [3.2 Schéma](#s3-2)
    - [3.3 Chronologie du TP](#s3-3)
4. [Serveur source SRV.PHY — préparation](#s4)
    - [4.1 La VM qui joue le serveur physique](#s4-1)
    - [4.2 Nom et adresse IP fixe](#s4-2)
    - [4.3 Installation des services](#s4-3)
    - [4.4 Création du site de l'entreprise](#s4-4)
5. [Inventaire du serveur source](#s5)
6. [Mesures avant migration](#s6)
7. [Serveur cible SRV-VM — construction](#s7)
    - [7.1 Création de la VM](#s7-1)
    - [7.2 Nom et adresse IP temporaire](#s7-2)
    - [7.3 Installation des services](#s7-3)
    - [7.4 Vérification du pare-feu et des ports](#s7-4)
8. [Migration des données](#s8)
    - [8.1 Création et envoi de l'archive](#s8-1)
    - [8.2 Restauration sur la VM cible](#s8-2)
    - [8.3 Vérification de l'intégrité](#s8-3)
9. [La bascule](#s9)
    - [9.1 Procédure prévue](#s9-1)
    - [9.2 Incident : conflit d'adresse IP](#s9-2)
    - [9.3 Rétablissement et durée d'interruption](#s9-3)
10. [Vérifications après migration](#s10)
11. [Comparatif avant / après](#s11)
12. [Problèmes rencontrés et solutions](#s12)
13. [Limites et risques d'une migration P2V](#s13)
    - [13.1 Risques constatés pendant ce TP](#s13-1)
    - [13.2 Risques d'un vrai P2V](#s13-2)
    - [13.3 Limites de la simulation](#s13-3)
14. [Bilan et conclusion](#s14)
    - [14.1 Objectifs atteints](#s14-1)
    - [14.2 Ce que ce TP m'a appris](#s14-2)
    - [14.3 À quoi a servi ce TP](#s14-3)
    - [14.4 Conclusion](#s14-4)

---

## <a id="s1"></a>1. Contexte et objectifs

Une entreprise possède un ancien serveur physique. Elle veut le virtualiser pour prolonger sa durée de vie logicielle et simplifier sa maintenance : une VM se sauvegarde, se déplace et se restaure beaucoup plus facilement qu'une machine physique.

**Objectifs pédagogiques :**

- Comprendre le principe d'une migration physique-vers-virtuel (P2V).
- Réaliser (ou simuler) une migration P2V.
- Valider le fonctionnement identique après migration.
- Documenter les limites et risques d'une migration P2V.

**Résultat :** le serveur web intranet de l'entreprise a été migré vers une nouvelle VM. Après la bascule, le site répond à la même adresse (192.168.209.50), avec le même nom de serveur et des données strictement identiques, prouvées par empreintes md5.

**Environnement utilisé :**

| Élément | Choix |
| --- | --- |
| Hyperviseur | VMware Workstation |
| Réseau | NAT VMware (192.168.209.0/24, passerelle 192.168.209.2) |
| Serveur source (« physique ») | SRV.PHY — Ubuntu 26.04.1 LTS, clone lié de UBU11 |
| Serveur cible (VM) | SRV-VM — Ubuntu 26.04.1 LTS, clone lié de UBU11 |
| Service migré | Serveur web Apache 2.4.66 (intranet) |
| Outils de migration | tar, scp (SSH), md5sum |



---

## <a id="s2"></a>2. Le P2V : principe et choix de la méthode

**Le P2V (Physical to Virtual)** consiste à transformer un serveur physique en machine virtuelle. Le service rendu doit rester identique : même rôle, mêmes données, même adresse pour les utilisateurs.

Il existe trois grandes méthodes :

| Méthode | Principe | Exemples d'outils | Avantage | Inconvénient |
| --- | --- | --- | --- | --- |
| **À chaud** | Copie du disque pendant que le serveur tourne | VMware vCenter Converter | Peu d'interruption | Données modifiées pendant la copie, pilotes du vieux matériel copiés |
| **À froid** | Le serveur est arrêté, on démarre sur une clé et on copie le disque | Clonezilla | Copie parfaitement cohérente | Serveur arrêté pendant toute la copie |
| **Reconstruction** | On crée une VM neuve, on réinstalle les services et on copie les données | Méthode manuelle | VM propre, sans pilotes ni traces du vieux matériel | Plus long, risque d'oublier un réglage |

**Méthode choisie : la reconstruction.** La consigne l'autorise (« reconstruire équivalent en VM »). Elle produit une VM propre, et elle oblige à bien connaître le serveur source, ce qui est le cœur d'une migration réussie.



---

## <a id="s3"></a>3. Scénario et architecture

### <a id="s3-1"></a>3.1 Les deux serveurs

Le « serveur physique » est simulé par une VM, SRV.PHY. Elle joue le rôle du vieux serveur de l'entreprise.

| | SRV.PHY (source) | SRV-VM (cible) |
| --- | --- | --- |
| Rôle | Serveur web intranet | Le même, après migration |
| Nom Ubuntu | srv-web | srv-cible pendant la migration, puis srv-web |
| Adresse IP | 192.168.209.50 (fixe) | 192.168.209.51 pendant la migration, puis 192.168.209.50 |
| État final | Éteint (conservé pour un retour arrière) | En production |

### <a id="s3-2"></a>3.2 Schéma

```
   AVANT LA BASCULE                         APRÈS LA BASCULE

   Utilisateurs (PC)                        Utilisateurs (PC)
          |                                        |
   NAT VMware 192.168.209.0/24             NAT VMware 192.168.209.0/24
      |                  |                         |
 +------------+   +--------------+          +--------------+
 |  SRV.PHY   |   |   SRV-VM     |          |   SRV-VM     |
 |  srv-web   |-->|  srv-cible   |          |   srv-web    |
 |   .50      |scp|    .51       |          |    .50       |
 | (en prod)  |   | (en prépa)   |          |  (en prod)   |
 +------------+   +--------------+          +--------------+
                                             SRV.PHY éteint
```

### <a id="s3-3"></a>3.3 Chronologie du TP

| Heure | Étape |
| --- | --- |
| 15:41 – 15:43 | Création de la VM source SRV.PHY (clone lié) |
| 15:48 | Nom, IP fixe et installation des services sur SRV.PHY |
| 15:51 | Création du site de l'entreprise |
| 15:56 – 16:00 | Inventaire et mesures avant migration |
| 16:18 – 16:39 | Construction de la VM cible SRV-VM |
| 17:07 | Envoi des données de SRV.PHY vers SRV-VM |
| 17:15 – 17:19 | Restauration et vérification des empreintes |
| 17:20 | Bascule : arrêt de SRV.PHY, reprise de l'identité par SRV-VM |
| 17:21 | Site indisponible (incident de conflit d'IP) |
| 18:21 | Site de nouveau en ligne, servi par SRV-VM |
| 18:24 – 18:25 | Mesures après migration |



---

## <a id="s4"></a>4. Serveur source SRV.PHY — préparation

### <a id="s4-1"></a>4.1 La VM qui joue le serveur physique

La VM **SRV.PHY** est un clone lié de UBU11 (la VM du Projet 1).

1. UBU11 éteinte, clic droit → **Manage** → **Clone**.
2. Choix de **An existing snapshot** : **Base clones**.
3. Choix de **Create a linked clone**, puis nom : **SRV.PHY**.

![Capture d'écran 2026-10-07 154134](https://hackmd.io/_uploads/SkeUM8xEiGl.png)
![Capture d'écran 2026-10-07 154225](https://hackmd.io/_uploads/HJUzUl4jfx.png)




**Pourquoi un clone lié :** il se crée en quelques secondes, prend peu de place et ne modifie pas la VM d'origine. **Pourquoi partir d'un snapshot :** un clone lié a besoin d'un état figé de la VM d'origine, sur lequel il s'appuie.

La VM obtenue a 2 Go de RAM, 2 processeurs, un disque de 50 Go et une seule carte réseau en **NAT**.

![Capture d'écran 2026-10-07 154339](https://hackmd.io/_uploads/BkrzUxNize.png)

### <a id="s4-2"></a>4.2 Nom et adresse IP fixe

```bash
sudo hostnamectl set-hostname srv-web
sudo sed -i 's/srv-app/srv-web/g' /etc/hosts
sudo mkdir -p /root/netplan-backup
sudo mv /etc/netplan/*.yaml /root/netplan-backup/
echo 'network: {version: 2, renderer: NetworkManager, ethernets: {ens33: {dhcp4: false, addresses: [192.168.209.50/24], routes: [{to: default, via: 192.168.209.2}], nameservers: {addresses: [192.168.209.2]}}}}' | sudo tee /etc/netplan/01-srv.yaml
sudo chmod 600 /etc/netplan/01-srv.yaml
sudo netplan apply
ping -c 3 google.com
```

| Commande | Rôle |
| --- | --- |
| `hostnamectl set-hostname srv-web` | Donne au serveur son nom. Le clone portait encore le nom de la VM d'origine (srv-app). |
| `sed -i 's/srv-app/srv-web/g' /etc/hosts` | Remplace l'ancien nom dans `/etc/hosts`, sinon sudo affiche « unable to resolve host ». |
| `mkdir` + `mv` | Met de côté les anciennes configs réseau, pour qu'elles n'entrent pas en conflit avec la nouvelle. |
| `echo '…' \| sudo tee` | Écrit la nouvelle config réseau. Elle tient sur une ligne pour éviter les erreurs d'indentation de netplan. |
| `chmod 600` | Rend le fichier lisible seulement par root (netplan l'exige). |
| `netplan apply` | Applique la config immédiatement. |
| `ping -c 3 google.com` | Vérifie l'accès à Internet et le DNS, nécessaires pour installer les paquets. |

**Pourquoi une IP fixe :** un serveur doit toujours être à la même adresse, sinon les utilisateurs ne le retrouvent plus. **192.168.209.2** est la passerelle du NAT VMware : elle donne accès à Internet et sert aussi de DNS.

*Remarque : sur les captures, le prompt affiche encore `srv-app`. C'est normal : le nouveau nom n'apparaît qu'à l'ouverture d'un nouveau terminal.*

![Capture d'écran 2026-10-07 154828](https://hackmd.io/_uploads/r1Z_8gEjMx.png)


### <a id="s4-3"></a>4.3 Installation des services

```bash
sudo apt update
sudo apt install -y apache2 openssh-server
sudo systemctl enable --now apache2 ssh
```

| Commande | Rôle |
| --- | --- |
| `apt update` | Met à jour la liste des paquets disponibles. |
| `apt install -y apache2 openssh-server` | Installe le serveur web (le service à migrer) et le serveur SSH (pour copier les données plus tard). |
| `systemctl enable --now` | Démarre les services tout de suite et à chaque démarrage. |

**Résultat :** Apache était **déjà installé** (version 2.4.66), hérité de la VM du Projet 1. Seuls **openssh-server** et ses dépendances ont été installés, et **openssh-client** a été mis à jour en version **1:10.2p1-2ubuntu3.7**.

![Capture d'écran 2026-10-07 155052](https://hackmd.io/_uploads/r1dkweNofg.png)

![Capture d'écran 2026-10-07 155100](https://hackmd.io/_uploads/S1F-DlVife.png)






### <a id="s4-4"></a>4.4 Création du site de l'entreprise

```bash
echo '<h1>Intranet de l entreprise</h1><p>Serveur SRV-WEB - documents internes</p>' | sudo tee /var/www/html/index.html
sudo mkdir -p /var/www/html/documents
echo 'Procedure de sauvegarde v1' | sudo tee /var/www/html/documents/procedure.txt
echo 'Liste du materiel 2026' | sudo tee /var/www/html/documents/materiel.txt
```

- `/var/www/html` est le dossier où Apache cherche les pages du site.
- `index.html` est la page d'accueil, affichée quand on ouvre l'adresse du serveur.
- Le dossier `documents` simule les fichiers internes de l'entreprise : ce sont les **données à migrer**.

![Capture d'écran 2026-10-07 155154](https://hackmd.io/_uploads/BykDPe4iMe.png)

**Test :** le site s'affiche à l'adresse **http://192.168.209.50**.

![Capture d'écran 2026-10-07 155341](https://hackmd.io/_uploads/BJIcDlVsGe.png)


---

## <a id="s5"></a>5. Inventaire du serveur source

L'inventaire est l'étape **la plus importante** d'une migration par reconstruction : tout ce qui n'est pas relevé ici risque d'être oublié sur la nouvelle VM.

```bash
hostnamectl
cat /etc/os-release
lscpu | grep -E 'Model name|^CPU\(s\)'
free -h
df -h /
ip a
ip route
systemctl status apache2 --no-pager
apache2 -v
ss -tlnp
ls -lR /var/www/html
```

| Commande | Ce qu'elle montre |
| --- | --- |
| `hostnamectl` | Nom du serveur, système, noyau, type de machine |
| `cat /etc/os-release` | Version exacte d'Ubuntu |
| `lscpu` | Processeur. *La commande n'a rien affiché : le système est en français, la ligne s'appelle « Nom de modèle » et non « Model name ».* |
| `free -h` | Mémoire vive (`-h` = lisible, en Go/Mo) |
| `df -h /` | Espace disque utilisé et libre |
| `ip a` | Adresse IP et **adresse MAC** de la carte réseau |
| `ip route` | Passerelle par défaut |
| `systemctl status apache2` | État du service web |
| `apache2 -v` | Version d'Apache |
| `ss -tlnp` | Ports TCP en écoute (`-t` TCP, `-l` en écoute, `-n` numéros, `-p` processus) |
| `ls -lR /var/www/html` | Tous les fichiers du site, avec tailles et dates |


![Capture d'écran 2026-10-07 155659](https://hackmd.io/_uploads/ByUFqx4jGx.png)
![Capture d'écran 2026-10-07 155906](https://hackmd.io/_uploads/ry739eVizg.png)
![Capture d'écran 2026-10-07 155919](https://hackmd.io/_uploads/BJmhqg4oze.png)





**Fiche d'inventaire de SRV.PHY :**

| Élément | Valeur |
| --- | --- |
| Nom | srv-web |
| Système | Ubuntu 26.04.1 LTS (Resolute Raccoon), noyau Linux 7.0.0-38-generic |
| Mémoire | 1,6 Gi (2 Go alloués dans VMware) |
| Disque | 49 Go, dont 11 Go utilisés (23 %) |
| Adresse IP | 192.168.209.50/24 |
| Adresse MAC | 00:0c:29:30:83:90 |
| Passerelle | 192.168.209.2 |
| Service web | Apache 2.4.66, actif |
| Ports en écoute | 22 (SSH), 80 (web), 3306 (MySQL, local), 631 (impression, local), 53 (DNS local) |
| Fichiers du site | index.html, info.php, documents/materiel.txt, documents/procedure.txt |

**Ce que l'inventaire révèle :** en plus du site, le serveur contient **info.php** (un fichier PHP du Projet 1) et un service **MySQL** sur le port 3306. Ces éléments n'étaient pas prévus au départ : c'est exactement ce qu'un inventaire doit faire apparaître.


---

## <a id="s6"></a>6. Mesures avant migration

```bash
md5sum /var/www/html/index.html /var/www/html/documents/*
systemd-analyze
```

- `md5sum` calcule l'**empreinte** de chaque fichier : un code unique de 32 caractères. Si un seul caractère du fichier change, l'empreinte change complètement. Comparer les empreintes avant et après permet de **prouver** que les données sont identiques.
- `systemd-analyze` mesure le **temps de démarrage** du serveur : **31,342 s**.

![Capture d'écran 2026-10-07 160001](https://hackmd.io/_uploads/Sy7GieNoGg.png)

**Temps de réponse du site**, mesuré depuis le PC hôte (PowerShell) :

```powershell
curl.exe -s -o NUL -w "%{time_total}`n" http://192.168.209.50
```

- `-s` : mode silencieux.
- `-o NUL` : la page téléchargée est jetée (on ne veut que le temps).
- `-w "%{time_total}"` : affiche le temps total de la requête, en secondes.

| Mesure | Temps |
| --- | --- |
| 1 | 0,089 s |
| 2 | 0,020 s |
| 3 | 0,021 s |

La première requête est plus lente : la connexion s'établit pour la première fois. Les suivantes profitent des caches.
![Capture d'écran 2026-10-07 160050](https://hackmd.io/_uploads/H1uQieEoMe.png)




---

## <a id="s7"></a>7. Serveur cible SRV-VM — construction

### <a id="s7-1"></a>7.1 Création de la VM

SRV-VM est un nouveau clone lié de UBU11, créé de la même façon que SRV.PHY (section 4.1), à partir du snapshot **Base clones**, avec une carte réseau en **NAT**.

### <a id="s7-2"></a>7.2 Nom et adresse IP temporaire

```bash
sudo hostnamectl set-hostname srv-cible
sudo sed -i 's/srv-app/srv-cible/g' /etc/hosts
sudo mkdir -p /root/netplan-backup
sudo mv /etc/netplan/*.yaml /root/netplan-backup/
echo 'network: {version: 2, renderer: NetworkManager, ethernets: {ens33: {dhcp4: false, addresses: [192.168.209.51/24], routes: [{to: default, via: 192.168.209.2}], nameservers: {addresses: [192.168.209.2]}}}}' | sudo tee /etc/netplan/01-srv.yaml
sudo chmod 600 /etc/netplan/01-srv.yaml
sudo netplan apply
```

Les commandes sont les mêmes que pour SRV.PHY (section 4.2). Deux choses changent :

- **Le nom temporaire srv-cible** permet de distinguer les deux serveurs pendant la migration.
- **L'IP temporaire 192.168.209.51** est obligatoire : les deux serveurs tournent en même temps pendant la préparation. Deux machines avec la même IP créeraient un **conflit d'adresse**.

![Capture d'écran 2026-10-07 161852](https://hackmd.io/_uploads/rJpRheEife.png)

![Capture d'écran 2026-10-07 163211](https://hackmd.io/_uploads/r1SxTlNsfe.png)

### <a id="s7-3"></a>7.3 Installation des services

```bash
sudo apt install -y apache2 openssh-server
sudo systemctl enable --now apache2 ssh
apache2 -v
```

- Apache était déjà présent (hérité du clone) : **version 2.4.66**, identique à SRV.PHY.
- openssh-server a été installé en version **1:10.2p1-2ubuntu3.6**.
- `apache2 -v` vérifie que la version d'Apache est la même que sur la source : une version différente pourrait changer le comportement du site.

**Différence repérée :** sur SRV.PHY, `apt update` avait été lancé avant l'installation, donc OpenSSH est en version **3.7**. Sur SRV-VM, `apt update` n'a pas été lancé, donc la version installée est la **3.6**, plus ancienne. Les deux serveurs n'ont donc pas exactement les mêmes versions de paquets (voir section 13).

![Capture d'écran 2026-10-07 163221](https://hackmd.io/_uploads/HJJzpxVoMg.png)


### <a id="s7-4"></a>7.4 Vérification du pare-feu et des ports

```bash
sudo ufw status
ss -tlnp | grep -E ':22|:80'
```

- `ufw status` affiche l'état du pare-feu d'Ubuntu. S'il était actif, il pourrait bloquer la copie (port 22) ou le site (port 80). **Résultat : inactif.**
- `ss -tlnp | grep` filtre les ports 22 et 80 : les deux sont en écoute (LISTEN).


![Capture d'écran 2026-10-07 163915](https://hackmd.io/_uploads/ry68pgNizx.png)




---

## <a id="s8"></a>8. Migration des données

### <a id="s8-1"></a>8.1 Création et envoi de l'archive

Sur **SRV.PHY** :

```bash
sudo tar czf /tmp/migration.tar.gz /var/www/html
scp /tmp/migration.tar.gz user@192.168.209.51:/tmp/
```

| Commande | Rôle |
| --- | --- |
| `tar czf` | Crée une archive compressée (`c` créer, `z` compresser, `f` nom du fichier) de tout le site, en gardant les droits des fichiers. |
| `scp` | Copie l'archive vers SRV-VM à travers le réseau, de façon chiffrée grâce à SSH. |

Le message « Suppression de « / » au début des noms des membres » est normal : tar enregistre des chemins relatifs, pour pouvoir restaurer les fichiers n'importe où. À la première connexion, SSH demande de confirmer l'identité de SRV-VM (empreinte de sa clé) : on répond `yes`. Le transfert atteint **100 %**.

!![Capture d'écran 2026-10-07 170719](https://hackmd.io/_uploads/H1OjplNsGx.png)


### <a id="s8-2"></a>8.2 Restauration sur la VM cible

Sur **SRV-VM** :

```bash
sudo tar xzf /tmp/migration.tar.gz -C /
ls -lR /var/www/html
```

- `tar xzf` extrait l'archive (`x` extraire).
- `-C /` extrait à partir de la racine : les fichiers retrouvent **exactement** leur emplacement d'origine, `/var/www/html`.
- `ls -lR` montre le résultat : le dossier `documents` est arrivé, et `index.html` fait maintenant 77 octets (la page de l'entreprise a remplacé la page par défaut d'Apache, qui faisait 10 672 octets).
![Capture d'écran 2026-10-07 171503](https://hackmd.io/_uploads/H1T6plVsfg.png)


### <a id="s8-3"></a>8.3 Vérification de l'intégrité

La même commande `md5sum` est lancée sur les deux serveurs :

```bash
md5sum /var/www/html/info.php
md5sum /var/www/html/index.html /var/www/html/documents/*
```
![Capture d'écran 2026-10-07 171917](https://hackmd.io/_uploads/B1Vb0e4szg.png)
![Capture d'écran 2026-10-07 171831](https://hackmd.io/_uploads/HJEWClEjzl.png)


| Fichier | SRV.PHY | SRV-VM | Identique |
| --- | --- | --- | --- |
| index.html | d859e741299b935dbe340c7ca8cfb3aa | d859e741299b935dbe340c7ca8cfb3aa | Oui |
| materiel.txt | baa8980eac65df82a795073481ddd6ab | baa8980eac65df82a795073481ddd6ab | Oui |
| procedure.txt | 1a4a8da58a7a5bbd738fdff1ed9f0405 | 1a4a8da58a7a5bbd738fdff1ed9f0405 | Oui |
| info.php | 48101bbdd897877cc62b8704a293a436 | 48101bbdd897877cc62b8704a293a436 | Oui |

**Conclusion :** les quatre empreintes sont identiques. Les données ont été migrées **sans aucune perte ni modification**.


---

## <a id="s9"></a>9. La bascule

### <a id="s9-1"></a>9.1 Procédure prévue

La bascule est le moment où la VM **remplace** l'ancien serveur. Elle prend son nom et son adresse IP, pour que la migration soit **invisible** pour les utilisateurs.

1. Prise d'un snapshot de SRV-VM nommé **« Avant bascule »**, pour pouvoir revenir en arrière.
2. Arrêt de SRV.PHY (Power → Shut Down Guest). **Le service est coupé.**
3. Sur SRV-VM, reprise de l'identité de l'ancien serveur :

```bash
sudo hostnamectl set-hostname srv-web
sudo sed -i 's/srv-cible/srv-web/g' /etc/hosts
sudo sed -i 's/192.168.209.51/192.168.209.50/' /etc/netplan/01-srv.yaml
sudo netplan apply
ip a
```

- Les deux premières commandes redonnent le nom **srv-web**.
- `sed -i 's/…51/…50/'` remplace l'IP temporaire par l'IP définitive dans le fichier netplan.

**Pourquoi reprendre le même nom et la même IP :** les utilisateurs et les autres machines connaissent le serveur par son adresse. Si elle changeait, il faudrait reconfigurer tous les postes.

### <a id="s9-2"></a>9.2 Incident : conflit d'adresse IP

Juste après les commandes de bascule (17:20), `ip a` montre que la carte ens33 n'a **aucune adresse IPv4** : il n'y a pas de ligne `inet` sous ens33.

![Capture d'écran 2026-10-07 172020](https://hackmd.io/_uploads/S1SBReNsMx.png)


À **17:21**, le site est inaccessible depuis le navigateur.

![Capture d'écran 2026-10-07 172158](https://hackmd.io/_uploads/r1V80x4szl.png)

**Diagnostic** sur SRV-VM :

```bash
sudo cat /etc/netplan/01-srv.yaml
ip a
systemctl status apache2 --no-pager
```

- `sudo cat` montre que le fichier contenait **bien** l'adresse 192.168.209.50 : la commande `sed` avait fonctionné. *(Sans `sudo`, la lecture est refusée, à cause du `chmod 600`.)*
- `ip a` confirme l'absence d'adresse IPv4 sur ens33.
- `systemctl status` montre qu'Apache tournait normalement.

**Cause :** au moment du `netplan apply`, SRV.PHY n'était pas encore complètement éteinte et utilisait toujours l'adresse 192.168.209.50. Ubuntu a détecté le **conflit d'adresse** et a refusé de prendre l'IP.

**Correction :** après avoir vérifié dans VMware que SRV.PHY était bien éteinte, la config a été réécrite et réappliquée :

```bash
echo 'network: {version: 2, renderer: NetworkManager, ethernets: {ens33: {dhcp4: false, addresses: [192.168.209.50/24], routes: [{to: default, via: 192.168.209.2}], nameservers: {addresses: [192.168.209.2]}}}}' | sudo tee /etc/netplan/01-srv.yaml
sudo chmod 600 /etc/netplan/01-srv.yaml
sudo netplan apply
ip a
```

Cette fois, `ip a` affiche `inet 192.168.209.50/24 … ens33`. L'adresse est en place.




### <a id="s9-3"></a>9.3 Rétablissement et durée d'interruption

À **18:21**, le site « Intranet de l'entreprise » s'affiche de nouveau sur **http://192.168.209.50**, alors que SRV.PHY est éteinte. C'est donc bien **la VM** qui sert le site.

![Capture d'écran 2026-10-07 182131](https://hackmd.io/_uploads/SJ-qRl4jMe.png)

| Moment | Heure |
| --- | --- |
| Site inaccessible | 17:21 |
| Site rétabli | 18:21 |
| **Interruption mesurée** | **environ 1 heure** |

**Analyse :** une bascule normale (changer le nom et l'IP) prend **1 à 2 minutes**. L'heure d'interruption s'explique par l'incident : il a fallu comprendre que la VM n'avait pas d'adresse, trouver la cause (le conflit d'IP), puis corriger. En entreprise, ce scénario montre l'importance d'une **procédure de bascule écrite** et vérifiée à l'avance, et d'un **plan de retour arrière** (ici, le snapshot « Avant bascule » et SRV.PHY conservée).


---

## <a id="s10"></a>10. Vérifications après migration

Les commandes de l'inventaire ont été relancées sur SRV-VM, devenue srv-web :

```bash
hostnamectl
cat /etc/os-release
ip a
ip route
systemctl status apache2 --no-pager
apache2 -v
php -v
ss -tlnp
systemd-analyze
```

Résultats principaux :

```
Static hostname: srv-web
Operating System: Ubuntu 26.04.1 LTS
Kernel: Linux 7.0.0-38-generic
link/ether 00:0c:29:54:1c:5f
inet 192.168.209.50/24 … ens33
default via 192.168.209.2 dev ens33 proto static
Active: active (running) — Total requests: 2
Server version: Apache/2.4.66 (Ubuntu)
PHP 8.5.4 (cli)
Ports en écoute : 22, 53, 80, 631, 3306
Startup finished in … = 22.063s
```

- **Total requests: 2** dans l'état d'Apache confirme que la VM a bien reçu les visites du navigateur.
- `php -v` confirme que PHP 8.5.4 est présent, nécessaire au fichier info.php.

**Vérification de la base de données MySQL** (port 3306, repéré à l'inventaire) :

```bash
sudo mysql -e "SHOW DATABASES;"
```

Résultat : seulement les 4 bases système (`information_schema`, `mysql`, `performance_schema`, `sys`), créées automatiquement par chaque installation de MySQL. Il n'y avait **aucune base de données d'entreprise**, donc aucune donnée n'a été perdue en ne migrant que `/var/www/html`.

![Capture d'écran 2026-10-07 182548](https://hackmd.io/_uploads/Hkol1-EiGx.png)


**Temps de réponse du site après migration**, mesuré depuis le PC hôte avec la même commande qu'avant :

| Mesure | Temps |
| --- | --- |
| 1 | 0,021 s |
| 2 | 0,008 s |
| 3 | 0,009 s |

Cette mesure prouve aussi que le site est **accessible depuis le PC d'un utilisateur**, pas seulement depuis la VM elle-même.

![Capture d'écran 2026-10-07 182402](https://hackmd.io/_uploads/SkOzyWNoGl.png)



---

## <a id="s11"></a>11. Comparatif avant / après

| Critère | Avant (SRV.PHY) | Après (SRV-VM) | Identique ? |
| --- | --- | --- | --- |
| Nom du serveur | srv-web | srv-web | Oui |
| Adresse IP | 192.168.209.50/24 | 192.168.209.50/24 | Oui |
| Passerelle | 192.168.209.2 | 192.168.209.2 | Oui |
| **Adresse MAC** | **00:0c:29:30:83:90** | **00:0c:29:54:1c:5f** | **Non (normal)** |
| Système | Ubuntu 26.04.1 LTS | Ubuntu 26.04.1 LTS | Oui |
| Noyau | Linux 7.0.0-38-generic | Linux 7.0.0-38-generic | Oui |
| Version d'Apache | 2.4.66 | 2.4.66 | Oui |
| Version d'OpenSSH | 1:10.2p1-2ubuntu3.7 | 1:10.2p1-2ubuntu3.6 | **Non** |
| Service apache2 | active (running) | active (running) | Oui |
| Ports en écoute | 22, 53, 80, 631, 3306 | 22, 53, 80, 631, 3306 | Oui |
| Fichiers du site | 4 fichiers | 4 fichiers | Oui |
| Empreintes md5 | voir 8.3 | voir 8.3 | Oui |
| Bases MySQL | bases système | bases système | Oui |
| Temps de démarrage | 31,342 s | 22,063 s | Proche |
| Temps de réponse du site | 0,020 à 0,089 s | 0,008 à 0,021 s | Comparable |
| Durée d'interruption | — | environ 1 h (incident) | |

**Lecture du tableau :**

- **Le service est identique** : même nom, même adresse, même logiciel, mêmes ports, mêmes données. Pour un utilisateur, rien n'a changé.
- **L'adresse MAC est différente**, c'est normal : c'est une nouvelle carte réseau virtuelle. C'est aussi la preuve que c'est bien une autre machine qui répond.
- **OpenSSH n'est pas à la même version**, car `apt update` n'a pas été lancé sur la cible (voir section 13).
- **Le temps de démarrage varie** (31 s contre 22 s). Cet écart n'est pas significatif : il dépend de la charge du PC hôte au moment du démarrage, et les deux VM ont la même configuration.
- **Le temps de réponse reste très court** (quelques centièmes de seconde) et même un peu plus rapide après. La différence n'est pas significative non plus, mais elle montre que la migration n'a pas dégradé les performances.



---

## <a id="s12"></a>12. Problèmes rencontrés et solutions

| Serveur | Problème | Cause | Solution |
| --- | --- | --- | --- |
| SRV.PHY | Le prompt affichait encore `srv-app` après le renommage | Le nom n'est pris en compte qu'à l'ouverture d'un nouveau terminal | Fermeture et réouverture du terminal |
| SRV.PHY | `lscpu \| grep 'Model name'` n'affichait rien | Le système est en français : la ligne s'appelle « Nom de modèle » | Relevé non bloquant pour la migration |
| SRV-VM | Première restauration : « open impossible: Aucun fichier ou dossier de ce nom » | L'archive n'avait pas encore été envoyée depuis SRV.PHY | Envoi relancé avec `scp`, transfert à 100 %, puis restauration réussie |
| SRV-VM | `cat /etc/netplan/01-srv.yaml` : « Permission denied » | Le fichier est en `chmod 600`, lisible seulement par root | Lecture avec `sudo cat` |
| SRV-VM | Après la bascule, aucune adresse IPv4 sur ens33 | Conflit d'IP : SRV.PHY utilisait encore 192.168.209.50 | Arrêt complet de SRV.PHY vérifié, config réappliquée |
| SRV-VM | `curl` : commande introuvable | curl n'est pas installé par défaut sur Ubuntu Desktop | Test du site avec le navigateur et avec `curl.exe` depuis le PC |



---

## <a id="s13"></a>13. Limites et risques d'une migration P2V

### <a id="s13-1"></a>13.1 Risques constatés pendant ce TP

| Risque | Ce qui s'est passé | Bonne pratique |
| --- | --- | --- |
| **Service oublié** | L'inventaire a montré MySQL (port 3306) et info.php, qui n'étaient pas prévus au départ | Faire un inventaire complet (`ss -tlnp`, liste des services) **avant** de migrer. Exporter les bases avec `mysqldump` si elles contiennent des données |
| **Conflit d'adresse IP** | La VM n'a pas pu prendre l'IP de l'ancien serveur, encore allumé | Éteindre complètement l'ancien serveur, et le vérifier, avant de donner son IP au nouveau |
| **Versions de paquets différentes** | OpenSSH en 3.7 sur la source, en 3.6 sur la cible | Lancer `apt update` et aligner les versions sur la cible, puis les comparer à l'inventaire |
| **Interruption de service** | Environ 1 heure au lieu de quelques minutes | Préparer une procédure de bascule écrite, la tester avant, et choisir un moment de faible activité |
| **Adresse MAC différente** | La MAC a changé avec la nouvelle carte | Mettre à jour les réservations DHCP ou les règles liées à la MAC |

### <a id="s13-2"></a>13.2 Risques d'un vrai P2V (non rencontrés dans la simulation)

| Risque | Explication |
| --- | --- |
| **Pilotes** | Un vrai serveur a des pilotes pour son matériel (carte RAID, carte réseau). Ce matériel n'existe pas dans une VM : le système peut refuser de démarrer (écran bleu sous Windows). |
| **Licences** | Windows et certains logiciels sont liés au matériel. Après migration, ils peuvent demander une réactivation. Certaines licences interdisent même la virtualisation. |
| **Performances** | La VM partage le processeur, la mémoire et le disque avec d'autres VM. Un serveur très sollicité peut devenir plus lent. |
| **Données modifiées pendant la copie** | Si un utilisateur modifie un fichier après la copie, la modification est perdue. Il faut figer le service (ou faire une dernière copie) juste avant la bascule. |
| **Matériel spécifique** | Une clé de licence USB, un port série ou une carte spéciale peuvent être difficiles, voire impossibles, à rendre accessibles à une VM. |

### <a id="s13-3"></a>13.3 Limites de la simulation

- **Le « serveur physique » est déjà une VM** : `hostnamectl` affiche `Virtualization: vmware` sur les deux serveurs. Les problèmes de pilotes et de matériel ne pouvaient donc pas se produire.
- **Les deux serveurs sont des clones de la même VM** : ils partagent le même identifiant de machine (`Machine ID: 37f80e8e…`) et avaient déjà Apache, PHP et MySQL. Une vraie reconstruction partirait d'un système neuf, avec plus de choses à réinstaller et à configurer.
- **Le volume de données est très faible** (quelques fichiers) : sur un vrai serveur, la copie peut prendre des heures et doit être planifiée.



---

## <a id="s14"></a>14. Bilan et conclusion

### <a id="s14-1"></a>14.1 Objectifs atteints

| Objectif / livrable | Réalisation | Statut |
| --- | --- | --- |
| Identifier le rôle et la configuration du serveur | Inventaire complet de SRV.PHY (section 5) | Atteint |
| Choisir une méthode de migration | Reconstruction, justifiée en section 2 | Atteint |
| Réaliser la migration | SRV-VM construite, données copiées, bascule faite | Atteint |
| VM migrée fonctionnelle | SRV-VM sert le site sur 192.168.209.50 | Atteint |
| Comparer avant / après | Tableau comparatif (section 11) et empreintes md5 | Atteint |
| Identifier les problèmes | Sections 12 et 13 | Atteint |
| Rapport avec limites et risques | Ce document | Atteint |

### <a id="s14-2"></a>14.2 Ce que ce TP m'a appris

- **Une migration commence par l'inventaire.** C'est en listant les ports ouverts que j'ai découvert MySQL et info.php. Sans inventaire, un service peut disparaître sans que personne ne s'en rende compte avant qu'un utilisateur se plaigne.
- **Il faut prouver que les données sont identiques, pas seulement le supposer.** Les empreintes md5 donnent une preuve objective, fichier par fichier.
- **La bascule est le moment le plus risqué.** Une seule erreur d'ordre (ancien serveur pas encore éteint) a transformé 2 minutes de coupure en 1 heure.
- **Un serveur se reconnaît à son nom et à son IP, pas à son matériel.** En reprenant les deux, la nouvelle VM remplace l'ancien serveur sans que les utilisateurs aient quoi que ce soit à changer.
- **Il faut toujours garder une porte de sortie** : le snapshot « Avant bascule » et l'ancien serveur éteint, mais pas supprimé, permettent de revenir en arrière en cas de problème.
- **Les commandes Linux de diagnostic** (`ip a`, `ss -tlnp`, `systemctl status`, `md5sum`, `tar`, `scp`) sont les outils de base pour comprendre et vérifier un serveur.

### <a id="s14-3"></a>14.3 À quoi a servi ce TP

Ce TP m'a fait vivre une migration de bout en bout, comme elle se passe en entreprise : connaître le serveur, préparer la nouvelle machine, transférer les données, basculer, puis vérifier. J'ai compris que la partie technique (copier des fichiers) est la plus simple. La vraie difficulté est l'**organisation** : ne rien oublier, prouver que tout est identique, limiter la coupure pour les utilisateurs et pouvoir revenir en arrière.

L'incident du conflit d'IP a été la partie la plus formatrice : il montre qu'une migration réussie ne dépend pas seulement de savoir taper des commandes, mais de **suivre une procédure dans le bon ordre** et de savoir **diagnostiquer** quand quelque chose ne marche pas.

### <a id="s14-4"></a>14.4 Conclusion

La migration du serveur web intranet vers une VM est réussie. Le service est identique pour les utilisateurs : même adresse, même nom, même site, mêmes données prouvées par empreintes, et des temps de réponse aussi bons qu'avant. L'ancien serveur est conservé éteint pour un éventuel retour arrière. L'entreprise peut désormais profiter des avantages de la virtualisation : snapshots, sauvegardes simplifiées et indépendance vis-à-vis d'un matériel vieillissant.

