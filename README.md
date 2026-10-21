# Virtualization Labs — VMware, Proxmox, Linux et Windows

> Cinq projets de virtualisation menés comme des missions pour une PME : infrastructure de 3 serveurs, plan de reprise par snapshots, labo réseau isolé avec pare-feu nftables, migration P2V et comparatif d'hyperviseurs chiffré.

![VMware](https://img.shields.io/badge/VMware-Workstation%2017-607078) ![Proxmox](https://img.shields.io/badge/Proxmox-VE%209-E57000) ![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04-E95420) ![Windows](https://img.shields.io/badge/Windows-11-0078D6) ![nftables](https://img.shields.io/badge/firewall-nftables-black)

*Projets réalisés dans le cadre du cours Virtualisation (formation E1A, Geneva Institute of Technology). Les scénarios sont reformulés ; aucun énoncé de l'école n'est reproduit.*

| # | Projet | Ce qui est démontré | Dossier |
|---|---|---|---|
| 01 | Infrastructure virtuelle pour une PME | 3 serveurs (SMB Windows, CUPS, LAMP), plan d'adressage, dimensionnement | [`01-pme-virtual-infrastructure`](01-pme-virtual-infrastructure/) |
| 02 | Snapshots et plan de reprise | Incident simulé, restauration chronométrée, **PRA avec RTO/RPO** | [`02-snapshots-disaster-recovery`](02-snapshots-disaster-recovery/) |
| 03 | Réseau virtuel isolé | LAN Segment, **passerelle Linux + pare-feu nftables** (default deny), tests d'isolation | [`03-isolated-lab-nftables`](03-isolated-lab-nftables/) |
| 04 | Migration P2V | Inventaire, migration tar/scp, **intégrité md5**, bascule et gestion d'incident | [`04-p2v-migration`](04-p2v-migration/) |
| 05 | Comparatif d'hyperviseurs | Benchmarks sysbench VMware vs Proxmox, **notation pondérée**, recommandation client | [`05-hypervisor-benchmark`](05-hypervisor-benchmark/) |

## Environnement
- PC portable Windows 11, VMware Workstation 17 Pro (hyperviseur de type 2, utilisé comme maquette)
- Proxmox VE 9 en virtualisation imbriquée (projet 05)
- Invités : Ubuntu 26.04 LTS, Ubuntu Server, Windows 11

## Fil conducteur sécurité
- **Moindre exposition** : labo isolé où seule une passerelle filtrante sort, en DNS/HTTP/HTTPS uniquement (03)
- **Disponibilité** : snapshots, PRA, RTO/RPO, sauvegarde intégrée Proxmox (02, 05)
- **Intégrité** : empreintes md5 avant/après migration, copie chiffrée via SSH (04)
- **Esprit critique** : chaque projet liste ses limites (type 2 ≠ production, snapshot ≠ sauvegarde, etc.)

---
Auteure : **Rrudhani Deaa**
