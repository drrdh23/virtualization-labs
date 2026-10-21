# Projet 3 — Réseau virtuel isolé (labo de test)

| | |
| --- | --- |
| **Cours** | Virtualisation — Mettre en place et exploiter une plateforme de virtualisation |
| **Classe** | E1A |
| **Niveau** | Intermédiaire |
| **Durée** | 1 semaine (PPE) |
| **Auteur** | Rrudhani Deaa |
| **Date** | 07/10/2026 |

---

## Sommaire

1. Contexte et objectifs
2. Architecture
    - 2.1 Choix du type de réseau
    - 2.2 Schéma réseau
    - 2.3 Plan d'adressage
3. VM 1 — GW-LAB (passerelle)
    - 3.1 Création de la VM (clone lié)
    - 3.2 Ajout de la deuxième carte réseau
    - 3.3 Création du LAN Segment LAN-ISOLE
    - 3.4 Installation du pare-feu
    - 3.5 Configuration réseau (netplan)
    - 3.6 Vérification de l'accès Internet
    - 3.7 Renommage de la machine
    - 3.8 Activation du routage
    - 3.9 Règles de filtrage (nftables)
    - 3.10 Activation et vérification
4. VM 2 — LAB-UBU (poste labo Ubuntu)
    - 4.1 Création de la VM (clone lié)
    - 4.2 Carte réseau dans VMware
    - 4.3 Renommage et configuration réseau
    - 4.4 Vérification
5. VM 3 — LAB-WIN (poste labo Windows 11)
    - 5.1 Réutilisation de la VM du Projet 2
    - 5.2 Carte réseau dans VMware
    - 5.3 Configuration réseau (PowerShell)
    - 5.4 Vérification
6. Règles d'isolation appliquées
7. Tests d'isolation
    - 7.1 Test A : GW-LAB éteinte
    - 7.2 Test B : GW-LAB allumée
8. Problèmes rencontrés et solutions
9. Bilan du TP

---

## 1. Contexte et objectifs

Un formateur veut disposer d'un labo virtuel isolé du reste du réseau. Les étudiants doivent pouvoir y tester des configurations risquées sans danger pour le réseau de l'école ni pour Internet.

**Objectifs pédagogiques :**

- Créer un réseau virtuel interne isolé d'Internet et du réseau physique.
- Connecter plusieurs VM à ce réseau isolé.
- Mettre en place une VM passerelle pour un accès contrôlé.
- Documenter les règles d'isolation appliquées.

**Résultat :** le labo est opérationnel. Trois VM communiquent sur un réseau virtuel isolé, et seule une passerelle filtrante peut ouvrir un accès contrôlé vers Internet.

**Environnement utilisé :**

| Élément | Choix |
| --- | --- |
| Hyperviseur | VMware Workstation |
| Réseau isolé | LAN Segment nommé LAN-ISOLE |
| VM 1 — passerelle | GW-LAB — Ubuntu Desktop (clone lié de UBU11) |
| VM 2 — poste labo | LAB-UBU — Ubuntu Desktop (clone lié de UBU11) |
| VM 3 — poste labo | LAB-WIN — Windows 11 (VM du Projet 2, nom Windows : PC01) |
| Pare-feu de la passerelle | nftables |

---

## 2. Architecture

### 2.1 Choix du type de réseau

VMware ne propose pas d'option appelée « réseau interne ». Plusieurs types de réseau ont été comparés :

| Type VMware | Accès Internet | Accès PC hôte | Accès réseau physique | Adapté au labo ? |
| --- | --- | --- | --- | --- |
| Bridged | Oui | Oui | Oui | Non |
| NAT | Oui | Oui | Indirect | Non |
| Host-only | Non | Oui | Non | Non (l'hôte reste joignable) |
| **LAN Segment** | **Non** | **Non** | **Non** | **Oui** |

**Choix retenu : le LAN Segment.** C'est un switch virtuel qui relie uniquement les VM qui y sont branchées. Il n'a pas de DHCP, pas de lien avec le PC hôte et pas de sortie vers Internet. C'est l'équivalent VMware d'un réseau interne privé.

### 2.2 Schéma réseau

```
                      Internet
                         |
          NAT VMware (192.168.209.0/24)
                         |
             ens33 : 192.168.209.150 (DHCP)
             +------------------------+
             |    GW-LAB (Ubuntu)     |
             |  routage + nftables    |
             +------------------------+
                ens37 : 10.10.10.1
                         |
     ========= LAN Segment LAN-ISOLE =========
                  10.10.10.0/24
              |                       |
     +-----------------+     +--------------------+
     | LAB-UBU (Ubuntu)|     | LAB-WIN (Win 11)   |
     |  10.10.10.11    |     |  10.10.10.12       |
     +-----------------+     +--------------------+
```

![Capture d'écran 2026-10-07 140418](https://hackmd.io/_uploads/ByWFonmjfx.png)

### 2.3 Plan d'adressage

| VM | Système | Carte | Réseau VMware | Adresse IP | Passerelle | DNS |
| --- | --- | --- | --- | --- | --- | --- |
| GW-LAB | Ubuntu | ens33 | NAT | 192.168.209.150/24 (DHCP) | 192.168.209.2 | DHCP |
| GW-LAB | Ubuntu | ens37 | LAN-ISOLE | 10.10.10.1/24 | — | — |
| LAB-UBU | Ubuntu | ens33 | LAN-ISOLE | 10.10.10.11/24 | 10.10.10.1 | 1.1.1.1 |
| LAB-WIN (PC01) | Windows 11 | Ethernet0 | LAN-ISOLE | 10.10.10.12/24 | 10.10.10.1 | 1.1.1.1 |

---

## 3. VM 1 — GW-LAB (passerelle)

GW-LAB relie le labo à Internet : elle route les paquets entre ses deux cartes et filtre tout ce qui passe. C'est la seule VM qui a accès à l'extérieur.

### 3.1 Création de la VM (clone lié)

1. VM Ubuntu d'origine **UBU11** éteinte (nom Ubuntu : srv-app, utilisée au Projet 1).
2. Prise d'un snapshot nommé **Base clones** (VM → Snapshot → Take Snapshot).
3. Clic droit sur UBU11 → **Manage** → **Clone**.
4. Choix de **An existing snapshot** (Base clones), puis **Create a linked clone**.
5. Nom du clone : **GW-LAB**.

**Pourquoi un clone lié :** il se crée en quelques secondes, prend peu de place sur le disque et ne modifie pas la VM d'origine.

**Contrainte :** un clone lié dépend de sa VM d'origine et du snapshot Base clones. UBU11 et ce snapshot ne doivent jamais être supprimés, sinon le clone ne démarre plus.

![Capture d'écran 2026-10-07 141229](https://hackmd.io/_uploads/rkHjjnXozx.png)


### 3.2 Ajout de la deuxième carte réseau

La première carte (Network Adapter) reste en **NAT** pour la sortie vers Internet. Une deuxième carte est ajoutée avec **Add… → Network Adapter → Finish**.




### 3.3 Création du LAN Segment LAN-ISOLE

1. Sélectionner la nouvelle carte (**Network Adapter 2**) et cocher **LAN segment**.
2. Cliquer sur **LAN Segments…** : la fenêtre Global LAN Segments s'ouvre, vide.
3. Cliquer sur **Add**, nommer le segment **LAN-ISOLE**, valider avec **OK**.
4. Choisir **LAN-ISOLE** dans la liste déroulante.

Le LAN Segment est créé une seule fois. Il est ensuite réutilisé par les deux autres VM.

![Capture d'écran 2026-10-07 141549](https://hackmd.io/_uploads/ryi2onXoMl.png)

**Cartes de GW-LAB :**

| Carte VMware | Réglage | Nom dans Ubuntu | Rôle |
| --- | --- | --- | --- |
| Network Adapter | NAT | ens33 | Sortie vers Internet |
| Network Adapter 2 | LAN Segment LAN-ISOLE | ens37 | Lien avec le labo |

![Capture d'écran 2026-10-07 125928](https://hackmd.io/_uploads/rJPM3hQozx.png)
![Capture d'écran 2026-10-07 125901](https://hackmd.io/_uploads/ryvGnnmofl.png)
![Capture d'écran 2026-10-07 125833](https://hackmd.io/_uploads/HJvGn3moGe.png)


### 3.4 Installation du pare-feu

```bash
sudo apt update && sudo apt install -y nftables
```

- `sudo` exécute la commande avec les droits administrateur.
- `apt update` met à jour la liste des paquets disponibles.
- `apt install -y nftables` installe le pare-feu nftables. `-y` répond « oui » automatiquement.

**Résultat :** `apt update` a affiché des erreurs « Erreur temporaire de résolution », car le clone gardait l'ancienne config réseau de srv-app (voir section 8). nftables était de toute façon déjà installé dans sa version la plus récente (1.1.6-1).

### 3.5 Configuration réseau (netplan)

```bash
ip a
sudo mkdir -p /root/netplan-backup
sudo mv /etc/netplan/*.yaml /root/netplan-backup/
echo 'network: {version: 2, renderer: NetworkManager, ethernets: {ens33: {dhcp4: true}, ens37: {dhcp4: false, addresses: [10.10.10.1/24]}}}' | sudo tee /etc/netplan/01-lab.yaml
sudo chmod 600 /etc/netplan/01-lab.yaml
sudo netplan apply
```

- `ip a` affiche les cartes réseau, leur nom et leurs adresses IP.
- `mkdir -p /root/netplan-backup` crée un dossier de sauvegarde. `-p` évite une erreur si le dossier existe déjà.
- `mv /etc/netplan/*.yaml /root/netplan-backup/` déplace les anciennes configs réseau. Netplan lit tous les fichiers `.yaml` du dossier : les anciens fichiers entreraient en conflit avec la nouvelle config.
- `echo '…' | sudo tee /etc/netplan/01-lab.yaml` écrit la nouvelle config dans le fichier. `tee` permet d'écrire avec les droits sudo.
- `chmod 600` limite la lecture du fichier à root. Netplan affiche un avertissement si le fichier est lisible par tous.
- `netplan apply` applique la configuration immédiatement.

**Contenu de la config :**

| Paramètre | Valeur | Rôle |
| --- | --- | --- |
| renderer | NetworkManager | Gestionnaire réseau d'Ubuntu Desktop |
| ens33 | dhcp4: true | Reçoit une IP du NAT VMware, sortie vers Internet |
| ens37 | 10.10.10.1/24, sans DHCP | Adresse fixe de la passerelle côté labo |

La config est écrite sur une seule ligne (format YAML « en accolades ») pour éviter les erreurs d'indentation au copier-coller.

![Capture d'écran 2026-10-07 132001](https://hackmd.io/_uploads/Hk_nn3mozg.png)

### 3.6 Vérification de l'accès Internet

```bash
ip a
ping -c 3 8.8.8.8
ping -c 3 google.com
```

- `ip a` confirme les deux adresses : **ens33 = 192.168.209.150/24** (donnée par le NAT) et **ens37 = 10.10.10.1/24**.
- `ping 8.8.8.8` prouve que la passerelle sort sur Internet.
- `ping google.com` prouve que la résolution DNS fonctionne.

`-c 3` limite l'envoi à 3 paquets.
![Capture d'écran 2026-10-07 131046](https://hackmd.io/_uploads/SySARh7ifg.png)

### 3.7 Renommage de la machine

```bash
sudo hostnamectl set-hostname gw-lab
sudo sed -i 's/srv-app/gw-lab/g' /etc/hosts
hostnamectl
```

- `hostnamectl set-hostname gw-lab` change le nom de la machine. Le clone portait encore le nom de la VM d'origine (srv-app).
- `sed -i 's/srv-app/gw-lab/g' /etc/hosts` remplace l'ancien nom par le nouveau dans le fichier `/etc/hosts`. Sans cela, sudo affiche l'avertissement « unable to resolve host ».
- `hostnamectl` affiche le nom actuel pour vérifier (ligne **Static hostname**).

Le nouveau nom n'apparaît dans le prompt qu'après la réouverture du terminal.



### 3.8 Activation du routage

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```

- Par défaut, Linux ne fait pas passer les paquets d'une carte à l'autre. Le paramètre `net.ipv4.ip_forward=1` active ce routage : GW-LAB devient un routeur.
- Le fichier `/etc/sysctl.d/99-forward.conf` rend le réglage permanent, même après un redémarrage.
- `sysctl --system` recharge tous les réglages. On voit la ligne « Applique /etc/sysctl.d/99-forward.conf », et la dernière ligne confirme `net.ipv4.ip_forward = 1`.



### 3.9 Règles de filtrage (nftables)

Les règles sont écrites dans `/etc/nftables.conf` avec la commande `sudo tee /etc/nftables.conf > /dev/null << 'EOF' … EOF`. Cette syntaxe écrit tout le bloc dans le fichier d'un coup.

```
#!/usr/sbin/nft -f
flush ruleset

table inet filtre {
  chain input {
    type filter hook input priority 0; policy drop;
    iif "lo" accept
    ct state established,related accept
    iifname "ens37" ip protocol icmp accept
    iifname "ens37" tcp dport 22 accept
  }
  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
    iifname "ens37" ip daddr { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 } drop
    iifname "ens37" oifname "ens33" udp dport 53 accept
    iifname "ens37" oifname "ens33" tcp dport { 80, 443 } accept
  }
  chain output {
    type filter hook output priority 0; policy accept;
  }
}

table ip nat {
  chain postrouting {
    type nat hook postrouting priority 100;
    oifname "ens33" masquerade
  }
}
```

**Explication des parties :**

- `flush ruleset` efface toutes les anciennes règles avant de charger les nouvelles.
- `table inet filtre` regroupe les règles de filtrage (IPv4 et IPv6).
- `chain input` filtre le trafic destiné à la passerelle elle-même. Par défaut, tout est refusé (`policy drop`).
- `chain forward` filtre le trafic qui traverse la passerelle, du labo vers Internet. Par défaut, tout est refusé.
- `chain output` laisse sortir le trafic émis par la passerelle elle-même (`policy accept`).
- `table ip nat` avec `masquerade` remplace l'IP source des VM du labo (10.10.10.x) par celle de la passerelle sur ens33. Sans cela, les réponses d'Internet ne sauraient pas revenir vers le labo.

Le détail de chaque règle se trouve dans la section 6.
![Capture d'écran 2026-10-07 131327](https://hackmd.io/_uploads/BJFjJ6miMe.png)


### 3.10 Activation et vérification

```bash
sudo systemctl enable --now nftables
sudo systemctl restart nftables
sudo nft list ruleset
ip route
```

- `systemctl enable --now nftables` démarre le service et le lance automatiquement à chaque démarrage (« Created symlink »).
- `systemctl restart nftables` recharge le fichier de règles.
- `nft list ruleset` affiche les règles actives. Le résultat correspond exactement au fichier.
- `ip route` affiche la table de routage : la route par défaut passe par **192.168.209.2** (NAT VMware) sur ens33, et le réseau **10.10.10.0/24** est joint directement par ens37.



![Capture d'écran 2026-10-07 132351](https://hackmd.io/_uploads/Bk9Vxamsfx.png)


**Résultat :** GW-LAB est opérationnelle.

---

## 4. VM 2 — LAB-UBU (poste labo Ubuntu)

LAB-UBU est un poste du labo. Il n'a qu'une carte sur LAN-ISOLE et passe par GW-LAB pour tout accès extérieur.

### 4.1 Création de la VM (clone lié)

Même procédure que GW-LAB (section 3.1), à partir du même snapshot **Base clones** de UBU11. Nom du clone : **LAB-UBU**.

### 4.2 Carte réseau dans VMware

| Carte VMware | Réglage | Nom dans Ubuntu | Rôle |
| --- | --- | --- | --- |
| Network Adapter | LAN Segment LAN-ISOLE | ens33 | Lien avec le labo |

Une seule carte : aucune sortie directe vers Internet, l'hôte ou le réseau physique.

![Capture d'écran 2026-10-07 141611](https://hackmd.io/_uploads/Hk0UeT7iGg.png)

### 4.3 Renommage et configuration réseau

```bash
sudo hostnamectl set-hostname lab-ubu
sudo sed -i 's/srv-app/lab-ubu/g' /etc/hosts
sudo mkdir -p /root/netplan-backup
sudo mv /etc/netplan/*.yaml /root/netplan-backup/
echo 'network: {version: 2, renderer: NetworkManager, ethernets: {ens33: {dhcp4: false, addresses: [10.10.10.11/24], routes: [{to: default, via: 10.10.10.1}], nameservers: {addresses: [1.1.1.1]}}}}' | sudo tee /etc/netplan/01-lab.yaml
sudo chmod 600 /etc/netplan/01-lab.yaml
sudo netplan apply
ip a
```

Les commandes sont les mêmes que pour GW-LAB (sections 3.5 et 3.7). Ce qui change, c'est le contenu de la config :

| Paramètre | Valeur | Rôle |
| --- | --- | --- |
| dhcp4 | false | Pas de DHCP sur LAN-ISOLE, donc IP fixe obligatoire |
| addresses | 10.10.10.11/24 | Adresse du poste dans le labo |
| routes → default | via 10.10.10.1 | Tout trafic hors labo est envoyé à GW-LAB |
| nameservers | 1.1.1.1 | Serveur DNS public, joignable seulement à travers GW-LAB (port 53) |

`ip a` confirme l'adresse **10.10.10.11/24** sur ens33, seule carte de la VM.

![Capture d'écran 2026-10-07 132001](https://hackmd.io/_uploads/r13sgamjfl.png)


### 4.4 Vérification

```bash
ip route
ping -c 3 10.10.10.1     # depuis LAB-UBU vers GW-LAB
ping -c 3 10.10.10.11    # depuis GW-LAB vers LAB-UBU
```

- `ip route` doit afficher la ligne `default via 10.10.10.1`.
- Les deux pings répondent avec **0 % de perte** : LAB-UBU et GW-LAB communiquent dans les deux sens.
![image](https://hackmd.io/_uploads/H1ZQbaQozg.png)

![Capture d'écran 2026-10-07 132534](https://hackmd.io/_uploads/BksjlTQszg.png)
![Capture d'écran 2026-10-07 132559](https://hackmd.io/_uploads/ByP_bamsMe.png)




**Résultat :** LAB-UBU est opérationnelle et reliée à la passerelle.

---

## 5. VM 3 — LAB-WIN (poste labo Windows 11)

LAB-WIN est le deuxième poste du labo. Comme LAB-UBU, il n'a qu'une carte sur LAN-ISOLE et passe par GW-LAB.

### 5.1 Réutilisation de la VM du Projet 2

La VM Windows 11 du Projet 2 (déjà rendu) a été réutilisée comme LAB-WIN. Son nom Windows est **PC01**.

### 5.2 Carte réseau dans VMware

Au départ, la VM avait **deux cartes réseau** : une en NAT et une sur LAN-ISOLE.

![Capture d'écran 2026-10-07 132734](https://hackmd.io/_uploads/HyRs-T7sMg.png)


Une carte a été supprimée (**Remove**), et la carte restante a été branchée sur LAN-ISOLE.

**Pourquoi une seule carte :** la carte NAT offrait une sortie directe vers Internet. Elle aurait contourné l'isolation et la passerelle.

| Carte VMware | Réglage | Rôle |
| --- | --- | --- |
| Network Adapter | LAN Segment LAN-ISOLE | Lien avec le labo |

![Capture d'écran 2026-10-07 141633](https://hackmd.io/_uploads/rJmxzaXsfg.png)

### 5.3 Configuration réseau (PowerShell)

Les commandes sont tapées dans **PowerShell lancé en administrateur** (le prompt commence par `PS`).

```powershell
Get-NetAdapter
$nic = (Get-NetAdapter | Select-Object -First 1).Name
$nic
Set-NetIPInterface -InterfaceAlias $nic -Dhcp Disabled
New-NetIPAddress -InterfaceAlias $nic -IPAddress 10.10.10.12 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias $nic -ServerAddresses 1.1.1.1
New-NetFirewallRule -DisplayName "ICMP labo" -Protocol ICMPv4 -IcmpType 8 -Action Allow
```

| Commande | Ce qu'elle fait | Pourquoi |
| --- | --- | --- |
| `Get-NetAdapter` | Liste les cartes réseau | Vérifier qu'il n'en reste qu'une (Ethernet0) |
| `$nic = …` + `$nic` | Met le nom de la carte dans une variable et l'affiche | Évite de taper le nom à chaque commande |
| `Set-NetIPInterface -Dhcp Disabled` | Désactive le DHCP | Il n'y a pas de DHCP sur LAN-ISOLE |
| `New-NetIPAddress` | Donne l'IP 10.10.10.12/24 et la passerelle 10.10.10.1 | Placer le poste dans le labo, sortie via GW-LAB |
| `Set-DnsClientServerAddress` | Définit le DNS 1.1.1.1 | Résoudre les noms à travers la passerelle |
| `New-NetFirewallRule` (ICMP type 8) | Autorise les pings entrants | Windows bloque le ping par défaut, les tests seraient faussés |

`-PrefixLength 24` correspond au masque 255.255.255.0. L'ICMP type 8 est la « demande d'écho », c'est-à-dire le ping.

![Capture d'écran 2026-10-07 133612](https://hackmd.io/_uploads/BJxXfTmoze.png)

![Capture d'écran 2026-10-07 133825](https://hackmd.io/_uploads/r1NIf6Xofg.png)


**Nettoyage de l'ancienne adresse :** le premier `ipconfig` affichait encore l'adresse **192.168.209.22** et la passerelle **192.168.209.2**, restes de l'ancienne carte NAT. Elles ont été supprimées :

```powershell
Remove-NetIPAddress -InterfaceAlias $nic -IPAddress 192.168.209.22 -Confirm:$false
Remove-NetRoute -InterfaceAlias $nic -NextHop 192.168.209.2 -Confirm:$false
ipconfig
```

- `Remove-NetIPAddress` supprime l'ancienne adresse IP de la carte.
- `Remove-NetRoute` supprime l'ancienne route par défaut vers le NAT VMware.
- `-Confirm:$false` évite la demande de confirmation.


### 5.4 Vérification

```powershell
ping 10.10.10.1
ping 10.10.10.11
```

![Capture d'écran 2026-10-07 133838](https://hackmd.io/_uploads/ryh1QTQifg.png)

Les pings vers GW-LAB (10.10.10.1) et LAB-UBU (10.10.10.11) répondent avec **0 % de perte**.



**Résultat :** LAB-WIN est opérationnelle et communique avec les deux autres VM.

---

## 6. Règles d'isolation appliquées

Tout est interdit par défaut. Seuls le DNS et le web (ports 53, 80 et 443) peuvent sortir du labo vers Internet, et uniquement à travers GW-LAB.

### 6.1 Isolation par l'architecture (VMware)

| Règle | Moyen | Effet |
| --- | --- | --- |
| Les postes n'ont qu'une carte, sur LAN-ISOLE | Réglages VMware | Aucune sortie directe vers Internet, l'hôte ou le réseau physique |
| LAN-ISOLE est un LAN Segment | Réglages VMware | Pas de DHCP, pas de lien avec l'hôte ni avec le NAT |
| Seule GW-LAB a une carte NAT | Réglages VMware | La passerelle est l'unique point de sortie |
| GW-LAB éteinte = labo totalement coupé | Conséquence des règles ci-dessus | Isolation complète, prouvée par le Test A |

### 6.2 Filtrage sur GW-LAB (nftables)

| Chaîne | Règle | Décision | Pourquoi |
| --- | --- | --- | --- |
| input | policy drop | Refusé | Par défaut, personne ne parle à la passerelle |
| input | `iif "lo"` | Autorisé | Communication interne de la machine, nécessaire au système |
| input | `ct state established,related` | Autorisé | Laisse revenir les réponses aux connexions ouvertes par la passerelle |
| input | ICMP depuis ens37 | Autorisé | Le labo peut pinger sa passerelle pour les tests |
| input | TCP 22 depuis ens37 | Autorisé | SSH depuis le labo pour administrer la passerelle |
| forward | policy drop | Refusé | Par défaut, rien ne traverse la passerelle |
| forward | `ct state established,related` | Autorisé | Laisse revenir les réponses d'Internet vers le labo |
| forward | vers 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | Refusé | Bloque tous les réseaux privés : réseau physique, box, PC hôte |
| forward | UDP 53 de ens37 vers ens33 | Autorisé | Résolution DNS |
| forward | TCP 80 et 443 de ens37 vers ens33 | Autorisé | Navigation web HTTP et HTTPS |
| output | policy accept | Autorisé | La passerelle peut faire ses propres mises à jour |
| nat postrouting | masquerade sur ens33 | Traduction | Les VM sortent avec l'IP de GW-LAB, les réponses reviennent |

**L'ordre compte :** la règle qui bloque les réseaux privés est placée avant les autorisations DNS et web. Une requête web vers 192.168.1.1 est donc refusée, même sur le port 80.

**Conséquence :** un ping vers Internet (8.8.8.8) est refusé, car l'ICMP ne fait pas partie des autorisations de la chaîne forward.

---

## 7. Tests d'isolation

*L'adresse 192.168.1.1 correspond à la box (ligne « Passerelle par défaut » de `ipconfig` sur le PC hôte).*

### 7.1 Test A : GW-LAB éteinte (preuve d'isolation)

GW-LAB a été éteinte via **Power → Shut Down Guest** dans VMware.

![Capture d'écran 2026-10-07 133935](https://hackmd.io/_uploads/r1kY8pmjzl.png)


**Depuis LAB-UBU :**

| Commande | Attendu | Obtenu |
| --- | --- | --- |
| `ping -c 3 10.10.10.12` | Répond | Répond (TTL 128 = Windows) |
| `ping -c 3 8.8.8.8` | Échec | Destination Host Unreachable, 100 % de perte |
| `ping -c 3 192.168.1.1` | Échec | Destination Host Unreachable, 100 % de perte |

Le message « Destination Host Unreachable » vient de LAB-UBU elle-même (10.10.10.11) : elle ne trouve plus sa passerelle sur le réseau, donc elle ne peut envoyer ses paquets nulle part.

![Capture d'écran 2026-10-07 134315](https://hackmd.io/_uploads/HyXK_T7izx.png)


**Depuis LAB-WIN :**

| Commande | Attendu | Obtenu |
| --- | --- | --- |
| `ping 10.10.10.11` | Répond | Répond |
| `ping 8.8.8.8` | Échec | Impossible de joindre l'hôte de destination |

Attention à la lecture : Windows affiche « reçus = 1, perte 0 % » pour le ping vers 8.8.8.8. Ce n'est pas une vraie réponse d'Internet : c'est le message d'erreur « Impossible de joindre l'hôte de destination », envoyé par LAB-WIN elle-même (10.10.10.12), que Windows compte comme reçu.

![Capture d'écran 2026-10-07 134414](https://hackmd.io/_uploads/H1-nOpmiGx.png)


**Conclusion du Test A :** sans passerelle, le labo est totalement coupé d'Internet et du réseau physique, mais les VM du labo continuent de communiquer entre elles.

### 7.2 Test B : GW-LAB allumée (accès contrôlé)

GW-LAB a été rallumée avant les tests.

![Capture d'écran 2026-10-07 134603](https://hackmd.io/_uploads/rylgFT7szl.png)


**Depuis LAB-UBU :**

| Commande | Attendu | Obtenu |
| --- | --- | --- |
| `wget -qO- https://example.com \| head -5` | Affiche du HTML | Page « Example Domain » affichée |
| `ping -c 3 8.8.8.8` | Bloqué | 0 reçu, 100 % de perte |
| `ping -c 3 192.168.1.1` | Bloqué | 0 reçu, 100 % de perte |

- `wget -qO-` télécharge la page et l'affiche dans le terminal, `head -5` garde les 5 premières lignes. Ce test prouve que le DNS et le HTTPS passent.
- Les pings échouent sans message d'erreur : la passerelle jette les paquets (`drop`) sans prévenir. C'est différent du Test A, où LAB-UBU affichait « Destination Host Unreachable » parce qu'elle ne trouvait pas de passerelle.

![Capture d'écran 2026-10-07 134702](https://hackmd.io/_uploads/ByOzFpmsGg.png)


**Depuis LAB-WIN :**

| Commande | Attendu | Obtenu |
| --- | --- | --- |
| `Test-NetConnection example.com -Port 443` | TcpTestSucceeded : True | True (source 10.10.10.12) |
| `ping 8.8.8.8` | Bloqué | Délai d'attente dépassé, 100 % de perte |

- `Test-NetConnection -Port 443` vérifie qu'une connexion HTTPS s'ouvre depuis Windows. Le résultat montre que le nom example.com a été résolu (DNS) et que la connexion part bien de 10.10.10.12.
- Le ping vers 8.8.8.8 donne « Délai d'attente de la demande dépassé » : la passerelle a jeté le paquet.

![Capture d'écran 2026-10-07 134727](https://hackmd.io/_uploads/ryvStpmoMe.png)
![Capture d'écran 2026-10-07 134721](https://hackmd.io/_uploads/H1DSY6Xsfe.png)


![Capture d'écran 2026-10-07 134742](https://hackmd.io/_uploads/H1QDt6mifl.png)


**Conclusion du Test B :** seul le trafic autorisé (DNS et web) sort du labo. Le ping vers Internet et tout accès au réseau physique restent bloqués.

---

## 8. Problèmes rencontrés et solutions

| VM | Problème | Cause | Solution |
| --- | --- | --- | --- |
| GW-LAB | `apt update` : « Erreur temporaire de résolution » | Le clone gardait l'ancienne config réseau de srv-app (Projet 1) | Remplacement de la config netplan, avec DHCP sur la carte NAT |
| GW-LAB | Risque de config réseau ignorée | Les VM sont des Ubuntu Desktop, gérées par NetworkManager | Ajout de `renderer: NetworkManager` dans netplan |
| GW-LAB | `netplan apply` : « expected mapping (check indentation) » | Les espaces en début de ligne ont disparu au copier-coller, et il y avait un espace en trop dans « network : » | Config réécrite sur une seule ligne (format YAML en accolades), sans indentation |
| GW-LAB | Le nom de la machine ne changeait pas | Le prompt du terminal ne se met à jour qu'à l'ouverture d'un nouveau terminal | Fermeture et réouverture du terminal, mise à jour de `/etc/hosts` |
| LAB-UBU | `Set-NetIPInterface` : commande introuvable | Commande PowerShell tapée dans le terminal Ubuntu au lieu de la VM Windows | Commandes PowerShell lancées dans LAB-WIN, en administrateur |
| LAB-WIN | Deux cartes réseau sur la VM | Carte NAT héritée du Projet 2 | Suppression d'une carte, la carte restante branchée sur LAN-ISOLE |
| LAB-WIN | Ancienne IP 192.168.209.22 encore présente | Adresse et route restées de l'ancienne carte NAT | Suppression avec `Remove-NetIPAddress` et `Remove-NetRoute` |
| LAB-WIN | `Rename-Computer` : « Le domaine spécifié n'existe pas » | PC01 est membre d'un domaine, et son serveur est injoignable depuis le labo isolé | Nom PC01 conservé. Cette erreur confirme aussi que l'isolation fonctionne |



![Capture d'écran 2026-10-07 133838](https://hackmd.io/_uploads/r1vcYaXoMg.png)

---

## 9. Bilan du TP

### 9.1 Objectifs atteints

| Objectif | Réalisation | Statut |
| --- | --- | --- |
| Créer un réseau virtuel interne isolé | LAN Segment LAN-ISOLE dans VMware | Atteint |
| Connecter au moins 2 VM | LAB-UBU et LAB-WIN branchées sur LAN-ISOLE | Atteint |
| Vérifier l'absence d'accès Internet et réseau physique | Test A : tous les accès externes échouent | Atteint |
| Ajouter une VM passerelle avec filtrage | GW-LAB : routage + nftables + NAT | Atteint |
| Tester la connectivité entre les VM | Pings entre les 3 VM : 0 % de perte | Atteint |
| Documenter le schéma et les règles | Sections 2 et 6 de ce rapport | Atteint |

### 9.2 Ce que ce TP m'a appris

- **Les types de réseaux VMware** n'offrent pas le même niveau d'isolation. Seul le LAN Segment coupe à la fois Internet, l'hôte et le réseau physique.
- **Une VM Linux peut devenir un routeur** : il suffit de deux cartes réseau et d'activer `ip_forward`.
- **Un pare-feu se construit en « tout interdit, sauf… »** : la politique `drop` par défaut, puis des autorisations précises. L'ordre des règles est important.
- **Le NAT (masquerade)** est indispensable pour que les réponses d'Internet reviennent vers un réseau privé.
- **Netplan est sensible à l'indentation** : le format sur une seule ligne évite les erreurs de copier-coller.
- **Windows bloque le ping par défaut** : il faut une règle de pare-feu pour faire des tests fiables.
- **Il faut bien lire les résultats d'un ping** : sous Windows, un message d'erreur est compté comme « reçu », et « Destination Host Unreachable » n'a pas la même cause qu'un délai dépassé.
- **Une VM réutilisée garde des traces** de sa config précédente (cartes, adresses IP, domaine) : il faut tout vérifier.
- **Les clones liés** permettent de créer rapidement plusieurs VM à partir d'une seule, à condition de garder la VM d'origine.

### 9.3 Limites et améliorations possibles

- **Pas de DHCP dans le labo** : chaque nouvelle VM demande une IP fixe. Un serveur DHCP sur GW-LAB (par exemple dnsmasq) simplifierait l'ajout de postes.
- **Pas de journalisation** : les paquets bloqués ne sont pas enregistrés. Une règle `log` dans nftables permettrait au formateur de voir les tentatives de sortie.
- **SSH ouvert depuis tout le labo** : il pourrait être limité à une seule IP (celle du formateur).
- **Remise à zéro du labo** : un snapshot propre de chaque VM permettrait au formateur de revenir à l'état initial après chaque séance d'étudiants.

### 9.4 Conclusion

Le labo répond au besoin du formateur. Les étudiants peuvent tester des configurations risquées dans un réseau totalement isolé. Quand un accès Internet est nécessaire, il passe uniquement par GW-LAB, qui n'autorise que le DNS et le web et bloque tout accès au réseau physique. Éteindre la passerelle suffit à couper complètement le labo du reste du monde.