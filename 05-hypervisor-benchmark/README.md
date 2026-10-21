# 05 — Comparatif d'hyperviseurs pour un client PME

## Objectif
Conseiller une PME (15–30 salariés, budget limité, un technicien polyvalent) sur le choix entre **Proxmox VE, Hyper-V, VMware et VirtualBox**, en combinant étude documentaire et **tests mesurés** sur deux solutions.

## Méthode
- Même VM de test sur les deux plateformes : Ubuntu Server, 2 vCPU, 4 Go, 25 Go.
- **VMware Workstation** (type 2) vs **Proxmox VE 9** (type 1, ici en virtualisation imbriquée **sans KVM** — limite assumée de l'environnement).
- Mesures doublées : démarrage (`systemd-analyze`), CPU (`sysbench`), écriture disque (`dd oflag=direct`), RAM au repos (`free -m`), snapshot/restauration chronométrés, sauvegarde intégrée.

## Résultats (moyennes)
| Mesure | VMware Workstation | Proxmox VE (sans KVM) |
|---|---|---|
| Démarrage | **20 s** | 208 s |
| CPU (événements/s) | **8 155** | 389 |
| Écriture disque | **194,5 MB/s** | 128,5 MB/s |
| RAM au repos | 525 Mo | **446 Mo** |
| Snapshot + restauration | **55 s** | 125 s |
| Sauvegarde intégrée | ✗ | ✅ 43 s, 25 Go → 1,61 Go (ZSTD) |

Les performances Proxmox sont pénalisées par l'absence de KVM en imbriqué : elles ne reflètent pas un déploiement sur serveur physique.

## Notation pondérée
| Solution | Coût 30 % | Admin 20 % | Fonctions 20 % | Perf. 15 % | Écosystème 15 % | **Note /5** |
|---|---|---|---|---|---|---|
| **Proxmox VE** | 5 | 4 | 5 | 4 | 4 | **4,5** |
| Hyper-V | 3 | 4 | 4 | 4 | 4 | 3,7 |
| VMware vSphere | 1 | 4 | 5 | 5 | 3 | 3,3 |
| VirtualBox | 5 | 5 | 1 | 2 | 2 | 3,3 (éliminé : type 2) |

## Recommandation
**Proxmox VE sur le serveur physique du client** : pas de licence, snapshots/sauvegarde/cluster inclus, administration web, solution pérenne et largement adoptée depuis les changements de licence VMware chez Broadcom.
Vigilance : montée en compétence Linux du technicien, virtualisation matérielle (VT-x / AMD-V) à activer dans le BIOS.

## Difficultés résolues
IP Proxmox hors du réseau NAT (corrigée dans `/etc/network/interfaces`) · type de CPU `host` incompatible sans KVM (passage à `x86-64-v2-AES`) · mesure RAM faussée côté hôte (remplacée par une mesure dans l'invité).

## Compétences
Benchmark et protocole de mesure · analyse multicritère pondérée · veille technologique (licences VMware/Broadcom) · conseil client argumenté
