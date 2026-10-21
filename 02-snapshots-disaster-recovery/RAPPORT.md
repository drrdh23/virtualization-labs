---
title: Projet 2 — Snapshots et plan de reprise d'une VM critique
tags: Virtualisation, E1A, PPE, VMware, Snapshots, PRA
---

# Projet 2 — Snapshots et plan de reprise d'une VM critique

| | |
|---|---|
| **Cours** | Virtualisation — Mettre en place et exploiter une plateforme de virtualisation |
| **Classe** | E1A |
| **Niveau** | Intermédiaire |
| **Durée** | 1 semaine (PPE) |
| **Auteur** | Rrudhani Deaa|
| **Date** | 06/10/2026 |

[TOC]

## 1. Contexte

Le serveur applicatif virtualisé de l'entreprise doit pouvoir être **restauré rapidement** en cas de mauvaise mise à jour ou de panne.
Ce projet met en place des **snapshots** (instantanés) sur une VM de test, simule un incident, restaure la VM et mesure le temps de reprise.

### Objectifs

- Comprendre le mécanisme des snapshots.
- Mettre en place une politique de snapshots réguliers.
- Simuler un incident et restaurer la VM à partir d'un snapshot.
- Mesurer le temps de restauration.
- Documenter un plan de reprise simple (RTO / RPO).

## 2. Qu'est-ce qu'un snapshot ?

Un **snapshot** enregistre l'état d'une VM à un instant précis :
- l'état du **disque** (fichiers, système, applications) ;
- l'état de la **mémoire** (si la VM est allumée) : la VM repart exactement là où elle était ;
- la **configuration** de la VM.

Après le snapshot, VMware écrit les modifications dans un **fichier delta** séparé. Restaurer (« Go To »), c'est jeter ce delta et revenir à l'état enregistré.


**Un snapshot n'est pas une sauvegarde.** Il est stocké sur le même disque que la VM : si le disque physique tombe en panne, la VM et ses snapshots sont perdus ensemble.


## 3. Environnement de test

| Élément | Valeur |
|---|---|
| Hyperviseur | VMware Workstation (type 2) |
| Machine hôte | PC portable personnel (Windows) |
| VM de test | `SRV-APP-TEST` (clone complet de la VM Windows 11) |
| Système invité | Windows 11 x64 |
| Application | Serveur web **IIS** (page `index.html`) |
| Réseau | NAT VMware — `192.168.209.0/24` |
| Adresse IP de la VM | `192.168.209.22` |
| Passerelle | `192.168.209.2` |
| URL de l'application | `http://192.168.209.22` |
![image](https://hackmd.io/_uploads/BysTEufoMl.png)


## 4. Réalisation

### Étape 1 — Création de la VM et installation de l'application

#### Clonage de la VM

Dans VMware : clic droit sur la VM Windows 11 → **Manage → Clone → Full clone**, nom `SRV-APP-TEST`, dans un dossier séparé.

**À quoi ça sert :** créer une copie indépendante de la VM pour faire les tests sans risquer d'abîmer la VM d'origine.

![image](https://hackmd.io/_uploads/r1-QBOfsze.png)
![Capture d'écran 2026-10-06 151645](https://hackmd.io/_uploads/Hy2EBdzife.png)


#### Installation d'IIS

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerRole -All
```

**À quoi ça sert :** installer le serveur web **IIS** de Windows. C'est l'application qu'on va protéger avec les snapshots.
![image](https://hackmd.io/_uploads/ryRcruGjzl.png)

![Capture d'écran 2026-10-06 143709](https://hackmd.io/_uploads/SJ8YHOGjfg.png)



#### Création de la page web

```powershell
Set-Content C:\inetpub\wwwroot\index.html "<h1>Application SRV-APP - Version 1.0</h1>"
```

**À quoi ça sert :** créer la page d'accueil du site avec le texte « Version 1.0 ». Ce texte permet de voir quelle version de l'application est en ligne.

![Capture d'écran 2026-10-06 143921](https://hackmd.io/_uploads/B1XCSOMjGx.png)

#### Ouverture du pare-feu

```powershell
Enable-NetFirewallRule -DisplayGroup "Services World Wide Web (HTTP)"
```
![Capture d'écran 2026-10-06 144034](https://hackmd.io/_uploads/B14xU_fjzx.png)

**À quoi ça sert :** autoriser le trafic web (HTTP, port 80) dans le pare-feu de la VM, pour que le PC hôte puisse accéder au site.

```powershell
Enable-NetFirewallRule -DisplayGroup "World Wide Web Services (HTTP)"
```

**À quoi ça sert :** même commande avec le nom anglais. Elle renvoie une erreur parce que la VM est en français. Ce n'est pas grave : la version française avait déjà fonctionné.

![Capture d'écran 2026-10-06 144034](https://hackmd.io/_uploads/B14xU_fjzx.png)

#### Récupération de l'adresse IP

```powershell
ipconfig
```

**À quoi ça sert :** afficher la configuration réseau de la VM, pour connaître son adresse IP (`192.168.209.22`) et accéder au site depuis le PC.

![image](https://hackmd.io/_uploads/SkI7UdfiMl.png)


#### Test de l'application

Depuis le navigateur du PC hôte : `http://192.168.209.22`

**À quoi ça sert :** vérifier que l'application fonctionne avant de prendre le snapshot.
![Capture d'écran 2026-10-06 145342](https://hackmd.io/_uploads/B1NrU_fjGx.png)


### Étape 2 — Snapshot et politique de snapshots

#### Prise du snapshot

Dans VMware : **VM → Snapshot → Take Snapshot**, nom `AVANT-MAJ-v1.0`.

**À quoi ça sert :** enregistrer l'état complet de la VM pendant que tout fonctionne. C'est le point de retour en cas de problème.

![Capture d'écran 2026-10-06 144508](https://hackmd.io/_uploads/S1wDIOGjfx.png)

#### Snapshots automatiques

Dans VMware : **VM → Settings → Options → AutoProtect**, intervalle *Daily*, 3 snapshots maximum.

**À quoi ça sert :** prendre un snapshot automatiquement chaque jour et garder seulement les 3 derniers.

![Capture d'écran 2026-10-06 144609](https://hackmd.io/_uploads/Bk9dUuGiGl.png)


### Étape 3 — Simulation d'une mise à jour

```powershell
Set-Content C:\inetpub\wwwroot\index.html "<h1>Version 2.0 - mise a jour</h1>"
```

**À quoi ça sert :** remplacer la page par « Version 2.0 » pour simuler une mise à jour faite après le snapshot.

![image](https://hackmd.io/_uploads/rkHMPuGjzg.png)
![Capture d'écran 2026-10-06 152614](https://hackmd.io/_uploads/HJ14vdMsGx.png)


### Étape 4 — Simulation de l'incident

#### Suppression du fichier

```powershell
Remove-Item C:\inetpub\wwwroot\index.html
```

**À quoi ça sert :** supprimer la page web pour simuler une perte de fichier.

#### Arrêt du service web

```powershell
Stop-Service W3SVC
```

**À quoi ça sert :** arrêter le service IIS (`W3SVC`) pour simuler une panne de l'application : le site ne répond plus. Le message jaune « Attente de l'arrêt du service » est normal.

#### Désactivation du service

```powershell
Set-Service W3SVC -StartupType Disabled
```

**À quoi ça sert :** empêcher le service de redémarrer tout seul. La panne reste en place, même après un redémarrage de la VM.

![Capture d'écran 2026-10-06 145747](https://hackmd.io/_uploads/BkWww_Mozg.png)


#### Vérification de la panne

Depuis le navigateur du PC hôte : `http://192.168.209.22`

**À quoi ça sert :** constater que le site est inaccessible.

![Capture d'écran 2026-10-06 150100](https://hackmd.io/_uploads/r1adPdMjGl.png)


![Capture d'écran 2026-10-06 145154](https://hackmd.io/_uploads/ByZ6cOfize.png)



### Étape 5 — Restauration et mesure du temps

#### Boucle de surveillance (sur le PC hôte)

```powershell
while ($true) { try { Invoke-WebRequest http://192.168.209.22 -UseBasicParsing -TimeoutSec 1 | Out-Null; "$(Get-Date -f HH:mm:ss) OK" } catch { "$(Get-Date -f HH:mm:ss) KO" }; Start-Sleep 1 }
```

**À quoi ça sert :** tester le site chaque seconde et afficher l'heure avec **OK** (site en ligne) ou **KO** (site en panne). Elle sert de chronomètre pour mesurer le temps de restauration. On l'arrête avec **Ctrl + C**.

:::warning
La boucle doit tourner **sur le PC hôte**, pas dans la VM : en restaurant le snapshot, la VM revient à son ancienne heure, donc la mesure serait fausse.
:::

#### Restauration du snapshot

Dans VMware : **VM → Snapshot → Snapshot Manager** → sélection de `AVANT-MAJ-v1.0` → **Go To** → **Yes**.

**À quoi ça sert :** ramener la VM à l'état du snapshot. La page est récupérée et le service IIS redémarre, sans aucune commande.

![Capture d'écran 2026-10-06 145423](https://hackmd.io/_uploads/HJzMo_GoGx.png)

![Capture d'écran 2026-10-06 145342](https://hackmd.io/_uploads/SJEDOdMjfl.png)


#### Mesure du temps

**À quoi ça sert :** relever l'heure du clic sur « Yes » et l'heure du premier **OK** dans la boucle. La différence donne le temps de restauration.


#### Vérification

Depuis le navigateur du PC hôte : `http://192.168.209.22`

**À quoi ça sert :** confirmer que l'application est revenue à la « Version 1.0 ».

![Capture d'écran 2026-10-06 145342](https://hackmd.io/_uploads/r1U7FdGjMe.png)


## 5. Résultats des mesures

### 5.1 Durée d'interruption du service

La mesure va du **premier KO** (le site tombe en panne) au **premier OK** (le site est de nouveau en ligne). Elle couvre toute l'interruption du service : la panne, le temps d'intervention et la restauration du snapshot. Deux essais ont été réalisés.

| Essai | 1er KO (début de la panne) | 1er OK (retour du service) | Durée d'interruption |
|:---:|:---:|:---:|:---:|
| 1 | 14:51:35 | 14:54:15 | **2 min 40 s** (160 s) |
| 2 | 15:00:27 | 15:01:41 | **1 min 14 s** (74 s) |
| **Moyenne** | | | **≈ 1 min 57 s** (117 s) |

:::info
**Calcul :** durée = heure du 1er OK − heure du 1er KO.
Moyenne = (160 + 74) ÷ 2 = **117 s ≈ 1 min 57 s**.
:::

### 5.2 Temps de restauration du snapshot

Le retour au snapshot (du clic « Yes » dans le Snapshot Manager jusqu'au retour du service) prend **17 secondes**.

| Mesure | Durée | Ce que ça représente |
|---|---|---|
| Restauration du snapshot | **17 s** | Temps de travail de VMware pour revenir au snapshot |
| Interruption totale (moyenne) | **≈ 1 min 57 s** | Panne + intervention humaine + restauration |

### 5.3 Analyse

- La restauration a **réussi aux 2 essais** : l'application revient à la « Version 1.0 », le fichier supprimé est récupéré et le service IIS est de nouveau actif, sans réinstallation.
- La restauration du snapshot ne prend que **17 secondes**. La plus grande partie de l'interruption vient du **temps d'intervention humaine** : voir la panne, ouvrir le Snapshot Manager, choisir le snapshot.
- Le 2ᵉ essai est **plus de deux fois plus rapide** que le 1er (74 s contre 160 s) : avec l'habitude de la procédure, l'intervention est plus rapide. Une procédure écrite et testée régulièrement réduit donc la durée des pannes.

## 6. Plan de reprise d'activité (PRA simplifié)

### 6.1 Périmètre

| | |
|---|---|
| **Système concerné** | VM `SRV-APP` — serveur applicatif web (IIS sur Windows 11) |
| **Criticité** | Haute : application utilisée par les salariés au quotidien |
| **Incidents couverts** | Mauvaise mise à jour, fichier supprimé, panne applicative, mauvaise configuration |
| **Hors périmètre** | Panne du disque ou du PC hôte (voir 6.6) |

### 6.2 Objectifs RTO / RPO

| Indicateur | Définition | Valeur |
|---|---|---|
| **RPO** | Perte de données maximale acceptable | **24 h** (1 snapshot automatique par jour) ; **0** pour une mise à jour (snapshot pris juste avant) |
| **RTO mesuré** | Temps technique de restauration | **`34s`** (moyenne des 2 essais) |
| **RTO cible** | Temps maximal pour relancer le service (détection + décision + restauration + vérification) | **15 min** |

### 6.3 Politique de snapshots

- **1 snapshot automatique par jour** (AutoProtect), **3 conservés au maximum**.
- **1 snapshot manuel obligatoire avant chaque mise à jour** ou modification importante.
- Nommage : `AVANT-<ACTION>-<VERSION>` (ex. `AVANT-MAJ-v1.0`), avec la date dans la description.
- Un snapshot manuel est **supprimé après 7 jours** si la mise à jour est validée. Trop de snapshots ralentit la VM et remplit le disque.

### 6.4 Procédure de reprise

1. **Détecter** : l'application ne répond plus (signalement d'un utilisateur ou test de `http://192.168.209.22`).
2. **Alerter** le responsable informatique.
3. **Décider** : choisir le dernier snapshot **sain** dans le Snapshot Manager.
4. **Restaurer** : **VM → Snapshot → Snapshot Manager → Go To → Yes**.
5. **Vérifier** : la page s'affiche et le service IIS (`W3SVC`) est démarré.
6. **Informer** les utilisateurs du retour du service.
7. **Tracer** l'incident : date, cause, snapshot utilisé, durée d'interruption, données perdues.

### 6.5 Rôles

| Rôle | Responsabilité |
|---|---|
| Responsable informatique | Décide de la restauration et choisit le snapshot |
| Technicien / administrateur | Exécute la restauration et vérifie l'application |
| Utilisateurs | Signalent la panne et confirment le retour du service |

### 6.6 Limites et améliorations

- **Un snapshot n'est pas une sauvegarde** : il est sur le même disque que la VM. Il faut aussi une **copie externe** régulière de la VM (disque externe, NAS) pour couvrir une panne matérielle.
- Les données créées **entre le snapshot et l'incident sont perdues** lors de la restauration.
- Un nombre élevé de snapshots **dégrade les performances** de la VM.
- Le PRA doit être **testé régulièrement**, par exemple une fois par trimestre.

## 7. Conclusion

Le snapshot permet de **revenir en quelques secondes** à un état stable après une mauvaise mise à jour ou une panne applicative, sans réinstallation. La restauration a réussi à chaque essai, avec un temps moyen de **`__ s`**.
Avec une politique de snapshots réguliers et une sauvegarde externe, c'est une solution simple et efficace pour remettre rapidement en service un serveur critique.