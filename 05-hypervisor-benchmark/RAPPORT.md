# Projet 5 — Comparatif d'hyperviseurs pour un cahier des charges client

**Cours :** Virtualisation — Mettre en place et exploiter une plateforme de virtualisation
**Classe :** E1A
**Auteur :** Rrudhani Deaa
**Durée :** 1 semaine (PPE)

---

## Sommaire

[TOC]



---

## 1. Contexte et scénario client

Un client hésite entre plusieurs solutions de virtualisation (Proxmox VE, Hyper-V, VMware, VirtualBox) pour son infrastructure. Il demande une recommandation argumentée.


| Élément | Valeur |
| --- | --- |
| Taille | PME de 15 à 30 salariés, 1 site |
| Serveurs à virtualiser | 3 à 5 VM (fichiers, impression, applicatif, AD) |
| Budget | Limité, pas d'abonnement logiciel élevé |
| Compétences internes | 1 technicien polyvalent (bases Windows et Linux) |
| Matériel | 1 serveur physique, évolution possible vers 2 |

**Méthode :** 4 solutions comparées sur documentation, 2 solutions testées en pratique avec exactement la même VM :
- **VMware Workstation** (hyperviseur de type 2, installé sur Windows)
- **Proxmox VE** (hyperviseur de type 1, installé dans une VM VMware = virtualisation imbriquée)

---

## 2. Critères de comparaison

Chaque solution est notée de 1 (faible) à 5 (excellent) sur 5 critères pondérés.

| Critère | Ce qui est évalué | Poids |
| --- | --- | --- |
| Coût / licences | Prix de la licence, abonnement, support payant | 30 % |
| Facilité d'administration | Interface, prise en main, documentation | 20 % |
| Fonctions professionnelles | Snapshots, sauvegarde, cluster, haute disponibilité | 20 % |
| Performances | Démarrage, CPU, disque, RAM consommée | 15 % |
| Écosystème / pérennité | Communauté, support, avenir de l'éditeur | 15 % |

Le coût pèse le plus lourd car le client a un budget limité.

---

## 3. Matériel et logiciels utilisés

| Élément | Détail |
| --- | --- |
| PC hôte | HP Envy x360 |
| OS hôte | Windows 11 |
| Hyperviseur 1 | VMware Workstation Pro (version : 17.5.2) |
| Hyperviseur 2 | Proxmox VE 9.2.2 (noyau 7.0.2-6-pve), installé dans une VM VMware |
| OS invité | Ubuntu Server 24.04 LTS (même ISO pour les deux VM) |

**Configuration identique des 2 VM de test :**

| Paramètre | Valeur |
| --- | --- |
| Nom | `TEST-VMWARE` / `TEST-PROXMOX` |
| vCPU | 2 |
| RAM | 4 Go (4096 Mo) |
| Disque | 25 Go |
| Utilisateur | `admin` |

**Schéma du test :**

```
PC Windows (HP Envy)
├── VMware Workstation
│   ├── VM TEST-VMWARE (Ubuntu Server)        ← Test n°1
│   └── VM PROXMOX (Proxmox VE)
│       └── VM TEST-PROXMOX (Ubuntu Server)   ← Test n°2
```

---

## 4. Préparation du PC hôte

### 4.1 Noter les caractéristiques du PC

1. Appuie sur `Windows + R`, tape `msinfo32`, puis `Entrée`.
2. Repère : **Processeur**, **Mémoire physique installée (RAM)**, **Nom du système d'exploitation**.

![image](https://hackmd.io/_uploads/ByooJnvjGe.png)

### 4.2 Vérifier que Hyper-V est désactivé

Si Hyper-V est actif, VMware tourne plus lentement et **Proxmox ne pourra pas faire tourner de VM** (la virtualisation imbriquée ne marchera pas).

1. `Windows + R`, tape `optionalfeatures`, puis `Entrée`.
2. Vérifie que ces cases sont **décochées** :
   - Hyper-V
   - Plateforme d'ordinateur virtuel
   - Plateforme d'hyperviseur Windows
3. Si tu as dû décocher quelque chose : clique **OK** et **redémarre** le PC.

![Capture d'écran 2026-10-10 143300](https://hackmd.io/_uploads/BJCfG2DsMg.png)
Dans mon cas mon pc est sur windows famille donc l'hyper-V n'existe pas donc par conséquent il est déjà désacrivé

### 4.3 Télécharger les ISO

1. **Ubuntu Server :** va sur `https://ubuntu.com/download/server`, télécharge **Ubuntu Server 24.04 LTS**.
2. **Proxmox VE :** va sur `https://www.proxmox.com/en/downloads`, télécharge la dernière **Proxmox VE ISO Installer**.
3. Range les deux fichiers `.iso` dans `C:\ISO\`.

### 4.4 Noter la version de VMware

Ouvre VMware Workstation, menu **Aide > À propos**.

![image](https://hackmd.io/_uploads/r1Mm42Difg.png)


> **Conseil :** pendant tous les tests, ferme les autres programmes (navigateur sauf pour Proxmox, Teams, etc.) et branche le PC sur secteur. Une seule VM de test allumée à la fois.

---

## 5. Test n°1 : VMware Workstation

### 5.1 Créer la VM

1. Ouvre VMware Workstation.
2. **Fichier > Nouvelle machine virtuelle**.
3. Choisis **Personnalisée (avancée)** → **Suivant**.![image](https://hackmd.io/_uploads/HJ8vS2wofg.png)

4. Compatibilité : laisse par défaut → **Suivant**. ![image](https://hackmd.io/_uploads/SJDurhPsMx.png)

5. Choisis **J'installerai le système d'exploitation plus tard** → **Suivant**.
   *(On évite l'« installation facile » de VMware pour que l'installation soit identique à celle faite sur Proxmox.)* ![image](https://hackmd.io/_uploads/BkKjB3wiMl.png)

6. Système : **Linux**, version **Ubuntu 64 bits** → **Suivant**. ![image](https://hackmd.io/_uploads/H1dhBnDszx.png)

7. Nom : `TEST-VMWARE` → **Suivant**.
8. Processeurs : **1 processeur, 2 cœurs** (total 2) → **Suivant**.![image](https://hackmd.io/_uploads/HJKyU2Djzx.png)

9. Mémoire : **4096 Mo** → **Suivant**.![image](https://hackmd.io/_uploads/S1qxI3Pjfg.png)

10. Réseau : **NAT** → **Suivant**.![image](https://hackmd.io/_uploads/BJGZLhwoGl.png)

11. Contrôleur et type de disque : par défaut → **Suivant**.![image](https://hackmd.io/_uploads/BJMzUnvjzl.png)

12. **Créer un nouveau disque virtuel** → taille **25 Go**, ne coche pas « Allouer tout l'espace maintenant », choisis **Stocker le disque en un seul fichier** → **Suivant**.![image](https://hackmd.io/_uploads/HkmNL3PsGl.png)
![image](https://hackmd.io/_uploads/H1kUUnwiMg.png)

13. **Terminer**.![image](https://hackmd.io/_uploads/H1kUUnwiMg.png)

14. Clique sur **Modifier les paramètres de la machine virtuelle** → **CD/DVD** → **Utiliser un fichier image ISO** → sélectionne l'ISO Ubuntu Server → **OK**.
![image](https://hackmd.io/_uploads/Hkm_LhDszg.png)

![image](https://hackmd.io/_uploads/Sk4jInDjGe.png)


### 5.2 Installer Ubuntu Server (chronométré)

> ⏱️ **Lance le chronomètre** au moment où tu cliques sur **Démarrer cette machine virtuelle**. Arrête-le quand l'écran de connexion `login:` apparaît après le redémarrage.

1. Démarre la VM.
2. Choisis **Try or Install Ubuntu Server**.
3. Langue : **Français**.![image](https://hackmd.io/_uploads/HkfVv2Poze.png)

4. Si proposé, **Continuer sans mettre à jour** l'installeur.
5. Clavier : **Switzerland** / **French(switzerland** → **Terminé**.![image](https://hackmd.io/_uploads/HkgwDhwoMx.png)

6. Type d'installation : **Ubuntu Server** (pas « minimized ») → **Terminé**.![image](https://hackmd.io/_uploads/Bkzav3PoGx.png)

7. Réseau : laisse l'adresse obtenue en DHCP → **Terminé**.
8. Proxy : laisse vide → **Terminé**.
9. Miroir : laisse par défaut, attends le test → **Terminé**.
10. Stockage : **Utiliser un disque entier** → **Terminé** → **Terminé** → **Continuer**.
11. Profil :
    - Votre nom : `admin`
    - Nom du serveur : `test-vmware`
    - Nom d'utilisateur : `admin`
    - Mot de passe : `[ton mot de passe]`
12. Ubuntu Pro : **Passer pour l'instant** → **Continuer**.
13. SSH : ne coche rien → **Terminé**.
14. Snaps : ne coche rien → **Terminé**.
15. Attends la fin, puis **Redémarrer maintenant**. Si on te demande de retirer le support, appuie sur `Entrée`.
16. Quand `login:` apparaît : **arrête le chronomètre**.

![image](https://hackmd.io/_uploads/rJ1_a2wjzl.png)

**Temps d'installation VMware :** `20:42:15 min`

### 5.3 Préparer la VM pour les mesures

Connecte-toi avec `admin`, puis tape :

```bash
sudo apt update
sudo apt install sysbench -y
```
sysbench fileio --file-total-size=1G prepare
sysbench fileio --file-total-size=1G --file-test-mode=rndrw --time=30 run
sysbench fileio --file-total-size=1G cleanup

![image](https://hackmd.io/_uploads/rk34AhPiMg.png)

![image](https://hackmd.io/_uploads/Sk9wChvsfg.png)

![image](https://hackmd.io/_uploads/BJ65RhvsMl.png)

![Capture d'écran 2026-10-10 153406](https://hackmd.io/_uploads/BJoGkpwjfl.png)
![image](https://hackmd.io/_uploads/rkKVkaviGe.png)
![image](https://hackmd.io/_uploads/SJrtJawozx.png)

Fais maintenant les mesures de la **partie 7** sur cette VM, puis **éteins-la** avant de passer à Proxmox :

```bash
sudo poweroff
```

---

## 6. Test n°2 : Proxmox VE

### 6.1 Créer la VM qui va contenir Proxmox

1. Dans VMware : **Fichier > Nouvelle machine virtuelle** → **Personnalisée (avancée)** → **Suivant**.
2. Compatibilité : par défaut → **Suivant**.
3. **J'installerai le système d'exploitation plus tard** → **Suivant**.
4. Système : **Linux**, version **Debian** (la version 64 bits la plus récente de la liste) → **Suivant**.
5. Nom : `PROXMOX` → **Suivant**.
6. Processeurs : **1 processeur, 4 cœurs** (si ton PC n'a que 4 cœurs au total, mets **2**) → **Suivant**.![image](https://hackmd.io/_uploads/HkCQeaDizx.png)

7. Mémoire : **6144 Mo** (6 Go) minimum, **8192 Mo** si ton PC a 16 Go ou plus → **Suivant**.![image](https://hackmd.io/_uploads/S1vEgpDofl.png)

   *(Proxmox a besoin d'environ 1 à 2 Go pour lui, plus les 4 Go de la VM de test.)*
8. Réseau : **NAT** → **Suivant**.![image](https://hackmd.io/_uploads/B1WXlavjzl.png)

9. Contrôleur et disque : par défaut → **Suivant**.
10. Nouveau disque virtuel : **64 Go**, un seul fichier → **Suivant** → **Terminer**.![image](https://hackmd.io/_uploads/SyVdgpvjzl.png)

11. **Modifier les paramètres de la machine virtuelle** :
    - **CD/DVD** → **Utiliser un fichier image ISO** → sélectionne l'ISO **Proxmox**.
    - **Processeurs** → **coche « Virtualiser ![image](https://hackmd.io/_uploads/HkO3xavoMx.png)
Intel VT-x/EPT ou AMD-V/RVI »**. **Étape obligatoire**, sinon Proxmox ne pourra pas lancer de VM.![image](https://hackmd.io/_uploads/By0pepwozg.png)

12. **OK**.



> Si VMware affiche « Virtualized Intel VT-x/EPT is not supported on this platform » au démarrage : Hyper-V est encore actif, refais la partie 4.2.

### 6.2 Installer Proxmox VE

1. Démarre la VM `PROXMOX`.
2. Choisis **Install Proxmox VE (Graphical)**.
3. Si un message « No support for KVM virtualization detected » apparaît : la case VT-x de l'étape 6.1 n'est pas cochée. Éteins la VM, corrige, recommence.
4. Licence : **I agree**.
5. Disque cible : laisse le disque de 64 Go → **Next**.
6. Localisation :
   - Country : **Swizterland**
   - Time zone : **Europe/Zurich**
   - Keyboard Layout : **French (Switzerlnd**
   → **Next**.
7. Mot de passe `root` : `[ton mot de passe]`, e-mail : `[ton e-mail]` → **Next**.
8. Réseau : laisse l'interface et l'adresse IP proposées.
   - Hostname (FQDN) : `pve.lab.local`
   - **Note l'adresse IP affichée** (exemple : `192.168.209.10/24)
   → **Next**.
9. Résumé → **Install**.
10. À la fin, la VM redémarre. L'écran affiche l'adresse de l'interface web, du type `https://192.168.xxx.xxx:8006`.



### 6.3 Se connecter à l'interface web

1. Sur Windows, ouvre ton navigateur et va sur `https://19.168.100.145:8006`.
2. Message « Votre connexion n'est pas privée » : clique **Paramètres avancés** → **Continuer vers le site**. (Normal : le certificat est auto-signé.)
3. Connexion :
   - Nom d'utilisateur : `root`
   - Mot de passe : celui de l'étape 6.2
   - Domaine : **Linux PAM standard authentication**
   - Langue : **Français**
4. Message « Aucun abonnement valide » : clique **OK**. (Normal : Proxmox est gratuit, l'abonnement sert seulement au support.)

![image](https://hackmd.io/_uploads/Skl6P6wofg.png)


### 6.4 Envoyer l'ISO Ubuntu dans Proxmox

1. À gauche, déplie **pve** → clique sur **local (pve)**.
2. Clique **Images ISO** → **Téléverser**.
3. **Sélectionner un fichier** → choisis l'ISO Ubuntu Server → **Téléverser**.
4. Attends « TASK OK », puis ferme la fenêtre.

![image](https://hackmd.io/_uploads/SkVxKTPjGl.png)

![image](https://hackmd.io/_uploads/Sk7bYpwjMe.png)

### 6.5 Créer la VM de test dans Proxmox

1. En haut à droite, clique **Créer une VM**.
2. **Général** : Nom : `TEST-PROXMOX` → **Suivant**.
3. **Système d'exploitation** : Stockage **local**, Image ISO : l'ISO Ubuntu → **Suivant**.
4. **Système** : laisse par défaut → **Suivant**.
5. **Disques** : Taille **25** Gio → **Suivant**.
6. **Processeur** : Cœurs **2**, Type **host** → **Suivant**.
7. **Mémoire** : **4096** Mio → **Suivant**.
8. **Réseau** : laisse par défaut (`vmbr0`, VirtIO) → **Suivant**.
9. **Confirmation** : ne coche pas « Démarrer après la création » → **Terminer**.



### 6.6 Installer Ubuntu Server (chronométré)

> **Lance le chronomètre** au moment où tu cliques sur **Démarrer**. Arrête-le quand `login:` apparaît après le redémarrage.

1. Sélectionne la VM `TEST-PROXMOX` → **Démarrer** → onglet **Console**.
2. Refais **exactement les étapes 2 à 16 de la partie 5.2**, avec un seul changement :
   - Nom du serveur : `test-proxmox`

> Si après le redémarrage la VM relance l'installeur : éteins la VM, va dans **Matériel** → **Lecteur CD/DVD** → **Éditer** → **Ne pas utiliser de média**, puis redémarre.
> Si la VM n'a pas de réseau pendant l'installation : continue sans réseau, tu pourras le vérifier après avec `ip a`.

![image](https://hackmd.io/_uploads/r1z8TCvoMe.png)

![image](https://hackmd.io/_uploads/BJkKaCDofg.png)


**Temps d'installation Proxmox :** ` 1heure 08:17:78min`

### 6.7 Préparer la VM pour les mesures

```bash
sudo apt update
sudo apt install sysbench -y
```
### Informations de base de la VM

Avant les tests de performance, on relève les caractéristiques de la VM pour vérifier que les deux VM (VMware et Proxmox) ont bien la même configuration.

| Commande | À quoi ça sert |
|---|---|
| `systemd-analyze` | Affiche le **temps de démarrage** de la VM (noyau + système). Permet de comparer la rapidité de démarrage entre les deux hyperviseurs. |
| `free -h` | Affiche la **mémoire RAM** totale, utilisée et libre, en Go/Mo (`-h` = lisible par un humain). |
| `df -h` | Affiche l'**espace disque** total, utilisé et disponible sur chaque partition. |
| `nproc` | Affiche le **nombre de processeurs (cœurs)** que la VM voit. |
| `ip a` | Affiche les **cartes réseau** de la VM et leur **adresse IP**. |

```bash
systemd-analyze
free -h
df -h
nproc
ip a
```


> Si les valeurs de RAM, de disque ou de processeurs sont différentes entre les deux VM, la comparaison des performances n'est pas équitable.
![image](https://hackmd.io/_uploads/SkN0TRvjGx.png)
![image](https://hackmd.io/_uploads/HyuOACDsGl.png)

### Tests de performance avec sysbench

Les mêmes tests sont lancés sur les deux VM, avec exactement les mêmes options, pour pouvoir comparer les résultats.

#### Test processeur (CPU)

Le processeur calcule des nombres premiers pendant 30 secondes.

```bash
sysbench cpu --time=30 run
```

**Valeur à relever :** `events per second` (plus c'est élevé, plus le processeur est rapide).

#### Test mémoire (RAM)

On écrit 2 Go de données en mémoire et on mesure la vitesse.

```bash
sysbench memory --memory-total-size=2G run
```

**Valeur à relever :** la vitesse en `MiB/sec` (plus c'est élevé, mieux c'est).



#### Test disque

On crée 1 Go de fichiers de test, on fait des lectures/écritures aléatoires pendant 30 secondes, puis on supprime les fichiers.

| Commande | À quoi ça sert |
|---|---|
| `prepare` | **Crée** les fichiers de test (1 Go). |
| `run` | **Lance le test** de lecture/écriture aléatoire (`rndrw`) pendant 30 s. |
| `cleanup` | **Supprime** les fichiers de test pour libérer l'espace. |

```bash
sysbench fileio --file-total-size=1G prepare
sysbench fileio --file-total-size=1G --file-test-mode=rndrw --time=30 run
sysbench fileio --file-total-size=1G cleanup
```

**Valeurs à relever :** `reads/s`, `writes/s` et le débit en `MiB/s` (lecture et écriture).


> Sur Proxmox, la VM tourne **sans KVM** (virtualisation imbriquée impossible sur le PC hôte Windows 11 Famille) : les résultats seront nettement plus faibles que sur VMware. Ce n'est pas une limite de Proxmox lui-même, mais de l'environnement de test.
![image](https://hackmd.io/_uploads/H10TRCwsGg.png)
![image](https://hackmd.io/_uploads/BkhxkJdofx.png)
![image](https://hackmd.io/_uploads/ryC4yyuife.png)
![image](https://hackmd.io/_uploads/S1fpJydjGe.png)
![image](https://hackmd.io/_uploads/HkVJe1OjMe.png)
![image](https://hackmd.io/_uploads/H1K-gJdiGx.png)

---
## 7. Mesures de performance

> **Méthode :** chaque mesure a été faite **2 fois** sur chaque VM, et on garde la **moyenne**.
> Les deux VM ont la même configuration : Ubuntu Server 26.04.1 LTS, **2 processeurs**, **4 Go de RAM**.
> Les tests ont d'abord été faits sur **TEST-VMWARE**, puis sur **TEST-PROXMOX**, avec exactement les mêmes commandes.

---

### 7.A Tests sur TEST-VMWARE (VMware Workstation)

#### 7.A.1 Temps de démarrage

La VM est redémarrée entre chaque essai, car `systemd-analyze` mesure uniquement le dernier démarrage.

```bash
sudo reboot
systemd-analyze
```

On relève la valeur après `=` à la fin de la ligne.

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 28,238 s | 17,311 s | **19,99 s** |

![image](https://hackmd.io/_uploads/B1t0u4OjGl.png)


#### 7.A.2 Performance CPU

```bash
sysbench cpu --threads=2 --time=30 run
```

Le processeur calcule des nombres premiers pendant 30 secondes avec 2 threads. On relève **events per second** (plus c'est élevé, mieux c'est).

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 8 125,93 | 8 184,88 | **8 155,41 événements/s** |



#### 7.A.3 Vitesse d'écriture disque

```bash
dd if=/dev/zero of=test bs=1M count=1024 oflag=direct
rm test
```

On écrit un fichier de 1 Go directement sur le disque (`oflag=direct` évite le cache), puis on le supprime. On relève la vitesse en **MB/s**.

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 185 MB/s | 204 MB/s | **194,5 MB/s** |



#### 7.A.4 RAM consommée

**Problème rencontré :** dans le Gestionnaire des tâches Windows, `vmware-vmx.exe` n'affichait que **12,9 Mo**, même avec les colonnes « Plage de travail partagée / privée ». VMware réserve la mémoire de la VM via son pilote, en dehors du processus : cette valeur n'est pas la vraie consommation.

**Méthode retenue :** mesure depuis l'intérieur de la VM, après 2 minutes sans activité (même méthode utilisée ensuite sur Proxmox) :

```bash
free -m
```

On relève la colonne **utilisé** sur la ligne **Mem**.

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 524 Mo | 527 Mo | **525,5 Mo** |


![image](https://hackmd.io/_uploads/ByM8F4_jfx.png)

#### 7.A.5 Snapshot et restauration

**a) Fichier témoin**

```bash
echo "fichier important" > temoin.txt
ls
```

**b) Snapshot** ⏱️ du clic jusqu'à la disparition de la barre de progression :
**VM > Instantané > Prendre un instantané**, nom `avant_incident` (essai 2 : `avant_incident2`).


**c) Incident simulé**

```bash
rm temoin.txt
ls
```

**d) Restauration** ⏱️ du clic sur **Oui** jusqu'à ce que la VM soit utilisable :
**VM > Instantané > Revenir à l'instantané** → **Oui**.

**e) Vérification**

```bash
ls
cat temoin.txt
```

Le fichier est revenu et affiche `fichier important`. ✅

| Mesure | Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- | --- |
| Snapshot | 27 s | 13,15 s | **20,08 s** |
| Restauration | 50 s | 19,06 s | **34,53 s** |


---

### 7.B Tests sur TEST-PROXMOX (Proxmox VE)

> Rappel : cette VM tourne **sans KVM** (processeur émulé `x86-64-v2-AES`), car la virtualisation imbriquée AMD-V n'était pas disponible sur le PC hôte. Les résultats sont donc beaucoup plus faibles qu'avec Proxmox installé sur un vrai serveur.

#### 7.B.1 Temps de démarrage

```bash
sudo reboot
systemd-analyze
```

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 156,927 s (2 min 37) | 258,869 s (4 min 19) | **207,90 s** (≈ 3 min 28) |

![Capture d'écran 2026-10-10 234134](https://hackmd.io/_uploads/B1h7cVujfl.png)![Capture d'écran 2026-10-10 234950](https://hackmd.io/_uploads/B1luhVuoGg.png)



#### 7.B.2 Performance CPU

```bash
sysbench cpu --threads=2 --time=30 run
```

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 383,25 | 395,45 | **389,35 événements/s** |

![Capture d'écran 2026-10-10 234134](https://hackmd.io/_uploads/B1h7cVujfl.png)
![Capture d'écran 2026-10-10 235412](https://hackmd.io/_uploads/SyPJpEdife.png)



#### 7.B.3 Vitesse d'écriture disque

```bash
dd if=/dev/zero of=test bs=1M count=1024 oflag=direct
rm test
```

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 122 MB/s | 135 MB/s | **128,5 MB/s** |

![Capture d'écran 2026-10-10 234134](https://hackmd.io/_uploads/B1h7cVujfl.png)
![Capture d'écran 2026-10-10 235412](https://hackmd.io/_uploads/By7ToVdoGg.png)
#### 7.B.4 RAM consommée

Même méthode que sur VMware, après 2 minutes sans activité :

```bash
free -m
```

| Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- |
| 445 Mo | 447 Mo | **446 Mo** |

![Capture d'écran 2026-10-10 234134](https://hackmd.io/_uploads/rJDlsVuizx.png)
![Capture d'écran 2026-10-10 235412](https://hackmd.io/_uploads/By7ToVdoGg.png)

#### 7.B.5 Snapshot et restauration

**a) Fichier témoin**

```bash
echo "fichier important" > temoin.txt
ls
```

**b) Snapshot** du clic jusqu'à « TASK OK » :
VM 100 → **Instantanés** → **Prendre un instantané**, nom `avant_incident` (essai 2 : `avant_incident2`), **sans** inclure la RAM.

![Capture d'écran 2026-10-10 235712](https://hackmd.io/_uploads/rkNMsE_iMl.png)
![Capture d'écran 2026-10-10 235704](https://hackmd.io/_uploads/r1NzsNOjzg.png)


**c) Incident simulé**

```bash
rm temoin.txt
ls
```

**d) Restauration** du clic sur **Oui** jusqu'à ce que la VM soit utilisable :
**Instantanés** → sélection du snapshot → **Rollback** → **Oui**. Le snapshot ne contenant pas la RAM, la VM est éteinte après le rollback : **Démarrer** → **Console** → reconnexion. Ce redémarrage est inclus dans le temps mesuré.

**e) Vérification**

```bash
ls
cat temoin.txt
```

Le fichier est revenu et affiche `fichier important`. 

| Mesure | Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- | --- |
| Snapshot | 27,85 s | 16,05 s | **21,95 s** |
| Restauration | 133 s (2 min 13) | 73,26 s | **103,13 s** |

![image](https://hackmd.io/_uploads/HyLBjVdsGe.png)


#### 7.B.6 Bonus : sauvegarde intégrée

VM 100 → **Sauvegarde** → **Sauvegarder maintenant** → Stockage `local`, Mode `Snapshot`, Compression `ZSTD`.

- Durée : **43 secondes** (`Finished Backup of VM 100 (00:00:43)`)
- Disque de 25 Go → archive de **1,61 Go** : 87 % du disque était vide et a été ignoré, le reste a été compressé.

Cette fonction n'existe pas dans VMware Workstation.

![Capture d'écran 2026-10-11 000535](https://hackmd.io/_uploads/ryoPoV_izl.png)

---

## 8. Résultats des mesures

### VMware Workstation

| Mesure | Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- | --- |
| Installation Ubuntu (min) | — | — | **20,7** |
| Démarrage (s) | 22,672 | 17,311 | **19,99** |
| CPU (événements/s) | 8 125,93 | 8 184,88 | **8 155,41** |
| Écriture disque (MB/s) | 185 | 204 | **194,5** |
| RAM utilisée (Mo) | 524 | 527 | **525,5** |
| Snapshot (s) | 27 | 13,15 | **20,08** |
| Restauration (s) | 50 | 19,06 | **34,53** |

### Proxmox VE

| Mesure | Essai 1 | Essai 2 | Moyenne |
| --- | --- | --- | --- |
| Installation Ubuntu (min) | — | — | **68** |
| Démarrage (s) | 156,927 | 258,869 | **207,90** |
| CPU (événements/s) | 383,25 | 395,45 | **389,35** |
| Écriture disque (MB/s) | 122 | 135 | **128,5** |
| RAM utilisée (Mo) | 445 | 447 | **446** |
| Snapshot (s) | 27,85 | 16,05 | **21,95** |
| Restauration (s) | 133 | 73,26 | **103,13** |

### Synthèse

| Mesure | VMware Workstation | Proxmox VE | Meilleur |
| --- | --- | --- | --- |
| Installation (min) | 20,7 | 68 | VMware |
| Démarrage (s) | 19,99 | 207,90 | VMware |
| CPU (événements/s) | 8 155,41 | 389,35 | VMware |
| Écriture disque (MB/s) | 194,5 | 128,5 | VMware |
| RAM utilisée (Mo) | 525,5 | 446 | Proxmox |
| Snapshot + restauration (s) | 54,61 | 125,08 | VMware |
| Sauvegarde intégrée | Non | Oui (43 s) | Proxmox |
| Interface | Logiciel Windows | Navigateur web | Proxmox |

 Pour le temps et la RAM : **plus petit = meilleur**. Pour le CPU et le disque : **plus grand = meilleur**.

 Proxmox tourne ici à l'intérieur d'une VM VMware, sur un PC hôte en **Windows 11 Famille** avec un processeur **AMD**. VMware n'a pas pu transmettre la virtualisation matérielle (AMD-V) à la VM Proxmox. La VM `TEST-PROXMOX` a donc tourné **sans KVM**, avec un processeur entièrement émulé (`x86-64-v2-AES`). C'est ce qui explique les écarts très importants : ces résultats ne reflètent pas les performances de Proxmox installé sur un vrai serveur.

**Observations :**
VMware est nettement plus rapide dans ce test : environ **21 fois plus** de calculs CPU par seconde, un démarrage **10 fois plus rapide** et une écriture disque environ **50 % plus rapide**. Ces écarts viennent surtout de l'absence de KVM côté Proxmox, et pas de Proxmox lui-même. En revanche, Proxmox consomme **environ 80 Mo de RAM en moins** et ses snapshots sont aussi rapides que ceux de VMware (≈ 22 s contre ≈ 20 s). Son interface web est pratique et sa **sauvegarde intégrée** (43 s, archive de 1,61 Go) est un vrai avantage. L'installation et la mise en réseau de Proxmox ont demandé plus de configuration (adresse IP, virtualisation imbriquée).

---

## 9. Tableau comparatif des hyperviseurs

| | Proxmox VE | Hyper-V | VMware ESXi / vSphere | VirtualBox |
| --- | --- | --- | --- | --- |
| Type | 1 (Debian + KVM) | 1 | 1 (ESXi), 2 (Workstation) | 2 |
| Licence | Open source, gratuit ; support payant optionnel | Inclus dans Windows Server et Windows 10/11 Pro | Abonnement Broadcom ; ESXi gratuit limité | Open source, gratuit ; Extension Pack payant en entreprise |
| Administration | Interface web | Gestionnaire Hyper-V, Windows Admin Center | vSphere Client, vCenter (payant) | Interface graphique simple |
| Snapshots | Oui | Oui (points de contrôle) | Oui | Oui |
| Sauvegarde | Intégrée | Outils Windows ou tiers | Outils tiers, pas d'API en version gratuite | Export manuel |
| Cluster / haute dispo | Oui, gratuit | Oui, avec Windows Server | Oui, versions payantes | Non |
| Cible | PME, budget serré | Parc Microsoft | Grandes entreprises | Tests, formation |
| Testé dans ce projet | ✅ |  (documentation, non disponible sur Windows 11 Famille) | ✅ (Workstation) | ❌ (documentation) |

**Point d'actualité VMware :** après le rachat de VMware par Broadcom, la version gratuite d'ESXi a été supprimée en février 2024, puis remise à disposition avec ESXi 8.0 Update 3e. Cette version gratuite n'a pas de support, ne peut pas être gérée par vCenter et n'a pas d'API de sauvegarde.

---

## 10. Notation pondérée

| Solution | Coût (30 %) | Admin (20 %) | Fonctions (20 %) | Perf. (15 %) | Écosyst. (15 %) | Note /5 |
| --- | --- | --- | --- | --- | --- | --- |
| **Proxmox VE** | 5 | 4 | 5 | 4 | 4 | **4,5** |
| Hyper-V | 3 | 4 | 4 | 4 | 4 | 3,7 |
| VMware vSphere | 1 | 4 | 5 | 5 | 3 | 3,3 |
| VirtualBox | 5 | 5 | 1 | 2 | 2 | 3,3 (éliminé) |

**Calcul :** note finale = somme de (note × poids).
Exemple Proxmox : 5×0,30 + 4×0,20 + 5×0,20 + 4×0,15 + 4×0,15 = **4,5**

**Justification des notes après les tests :**
- **Performances Proxmox (4/5) :** les mesures de la partie 8 sont faibles, mais elles ont été obtenues sans KVM, dans une configuration imbriquée qui n'existe pas en production. Sur un serveur physique, Proxmox utilise KVM, qui offre des performances proches du matériel. La note est donc basée sur ce fonctionnement normal, et pas sur ce test limité.
- **Administration Proxmox (4/5) :** l'interface web est claire et accessible depuis n'importe quel poste, mais la configuration réseau a demandé des commandes Linux.

VirtualBox est éliminé malgré sa note : c'est un hyperviseur de type 2, fait pour un poste de travail et pas pour héberger des serveurs en production.

---

## 11. Recommandation argumentée

**Je recommande Proxmox VE, installé directement sur le serveur physique du client.**

**Pourquoi Proxmox :**

- **Coût :** aucune licence à payer pour l'hyperviseur. Un abonnement de support peut être pris plus tard si le client veut une assistance.
- **Fonctions pro incluses :** snapshots, sauvegarde et cluster sont gratuits. Mes tests l'ont montré : la sauvegarde complète d'une VM a pris **43 secondes** et a produit une archive de seulement **1,61 Go** pour un disque de 25 Go. Avec un deuxième serveur, la haute disponibilité devient possible sans surcoût.
- **Administration simple :** tout se fait depuis un navigateur web, depuis n'importe quel poste, ce qui convient à un technicien seul.
- **Sobriété :** dans mes tests, la VM Proxmox consommait environ **80 Mo de RAM en moins** que la VM VMware.
- **Pérennité :** solution open source très utilisée, avec une grande communauté. Beaucoup d'entreprises s'y tournent depuis les changements de licence VMware.

**Pourquoi pas les autres :**

- **VMware :** très performant, mes tests le confirment, mais l'abonnement Broadcom est trop cher pour cette taille d'entreprise, et la version gratuite d'ESXi n'a ni support ni sauvegarde au niveau de l'hôte. VMware Workstation, testé ici, est un outil de poste de travail, pas une solution serveur.
- **Hyper-V :** bon choix si le client utilise déjà Windows Server et Active Directory partout, car la licence est alors déjà payée. Sinon, il faut acheter Windows Server.
- **VirtualBox :** pratique pour tester, mais hyperviseur de type 2, donc pas adapté à la production.

**Points de vigilance :**
- Proxmox repose sur Linux : le technicien devra se former aux bases de l'administration Linux (j'ai moi-même dû corriger la configuration réseau en ligne de commande).
- Le serveur devra avoir la **virtualisation matérielle activée** (Intel VT-x ou AMD-V/SVM dans le BIOS), sinon les performances seront aussi faibles que dans mon test.

---

## 12. Limites du test et difficultés rencontrées

**Limites :**
- Proxmox a été testé **en virtualisation imbriquée** (dans une VM VMware) et **sans KVM** : ses performances sont très inférieures à ce qu'elles seraient sur un vrai serveur.
- Tous les tests ont été faits sur un PC portable sous Windows 11 Famille, pas sur un serveur.
- Mesures faites sur **2 essais** au lieu de 3, par manque de temps.
- Les temps de snapshot et de restauration ont été chronométrés à la main, avec une petite marge d'erreur.
- Hyper-V et VirtualBox sont comparés sur documentation uniquement.
- Une seule VM testée à la fois : pas de mesure avec plusieurs VM en charge.

**Difficultés rencontrées :**
- **Adresse IP incohérente :** l'adresse de Proxmox (`192.168.100.145`) ne correspondait pas au réseau NAT de VMware (`192.168.209.0/24`, passerelle `192.168.209.2`). L'interface web était inaccessible. Corrigé en changeant l'adresse en `192.168.209.10` dans `/etc/network/interfaces` et `/etc/hosts`.
- **Virtualisation imbriquée impossible :** VMware affichait « Virtualized AMD-V/RVI is not supported on this platform ». La désactivation de l'intégrité de la mémoire et de l'hyperviseur Windows (`bcdedit /set hypervisorlaunchtype off`) n'a pas suffi. L'option a dû être décochée pour que la VM Proxmox démarre.![Capture d'écran 2026-10-10 162645](https://hackmd.io/_uploads/HJ5ZMgYsMl.png)

- **VM sans KVM :** dans Proxmox, l'option « Virtualisation matérielle KVM » a été désactivée, et le type de processeur changé de `host` à `x86-64-v2-AES` (le type `host` nécessite KVM).
- **Mesure de la RAM :** le Gestionnaire des tâches ne montrait pas la vraie mémoire de la VM VMware. Méthode remplacée par `free -m` dans les deux VM.

**Amélioration possible :** installer Proxmox directement sur un mini PC dédié pour mesurer ses vraies performances avant la mise en production.

---

## 13. Liste des captures d'écran

| N° | Nom du fichier | Contenu |
| --- | --- | --- |
| 01 | `01_infos_pc_hote.png` | Caractéristiques du PC hôte |
| 02 | `02_hyperv_desactive.png` | Fonctionnalités Windows désactivées (Hyper-V absent sur Windows 11 Famille) |
| 03 | `03_vmware_version.png` | Version de VMware Workstation |
| 04 | `04_vmware_config_vm.png` | Configuration VM VMware |
| 05 | `05_vmware_installation.png` | Installation Ubuntu sur VMware |
| 06 | `06_vmware_login.png` | VM VMware démarrée |
| 07 | `07_vmware_sysbench_installe.png` | sysbench installé (VMware) |
| 08 | `08_proxmox_config_vm.png` | VM Proxmox (virtualisation AMD-V décochée) |
| 09 | `09_proxmox_installation.png` | Résumé de l'installation Proxmox |
| 10 | `10_proxmox_console.png` | Adresse de l'interface web (`192.168.209.10:8006`) |
| 11 | `11_proxmox_interface_web.png` | Interface web Proxmox |
| 12 | `12_proxmox_iso.png` | ISO Ubuntu envoyée dans Proxmox |
| 13 | `13_proxmox_config_vm.png` | Configuration VM TEST-PROXMOX (2 cœurs, 4 Go, KVM désactivé, x86-64-v2-AES) |
| 14 | `14_proxmox_ubuntu_installation.png` | Installation Ubuntu sur Proxmox |
| 15 | `15_proxmox_ubuntu_login.png` | VM Proxmox démarrée |
| 16 | `16_proxmox_sysbench_installe.png` | sysbench installé (Proxmox) |
| 17 | `17_vmware_boot.png` | Temps de démarrage — VMware |
| 18 | `18_vmware_cpu.png` | Test CPU — VMware |
| 19 | `19_vmware_disque.png` | Test disque — VMware |
| 20 | `20_vmware_ram.png` / `20_vmware_ram_gestionnaire.png` | RAM utilisée (`free -m`) + Gestionnaire des tâches — VMware |
| 21 | `21_vmware_snapshot.png` | Snapshot — VMware |
| 22 | `22_vmware_restauration_ok.png` | Fichier témoin restauré — VMware |
| 17 | `17_proxmox_boot.png` | Temps de démarrage — Proxmox |
| 18 | `18_proxmox_cpu.png` | Test CPU — Proxmox |
| 19 | `19_proxmox_disque.png` | Test disque — Proxmox |
| 20 | `20_proxmox_ram.png` | RAM utilisée (`free -m`) — Proxmox |
| 21 | `21_proxmox_snapshot.png` | Snapshot — Proxmox |
| 22 | `22_proxmox_restauration_ok.png` | Fichier témoin restauré — Proxmox |
| 23 | `23_proxmox_sauvegarde.png` | Sauvegarde Proxmox terminée (TASK OK) |