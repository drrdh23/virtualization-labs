# 04 — Migration d'un serveur « physique » vers une VM (P2V)

## Objectif
Migrer le serveur web intranet d'une entreprise vers une machine virtuelle, de façon **invisible pour les utilisateurs** (même nom, même IP, mêmes données), en documentant les risques d'un P2V.

## Méthode choisie : reconstruction
| Méthode | Avantage | Inconvénient |
|---|---|---|
| À chaud (vCenter Converter) | Peu d'interruption | Données qui bougent pendant la copie, pilotes du vieux matériel |
| À froid (Clonezilla) | Copie cohérente | Serveur arrêté pendant toute la copie |
| **Reconstruction** ✅ | VM propre, connaissance fine du serveur | Plus long, risque d'oubli → **inventaire rigoureux** |

## Déroulé
| Phase | Actions |
|---|---|
| Source `SRV.PHY` | Ubuntu 26.04, Apache, site + dossier de documents internes |
| **Inventaire** | `hostnamectl`, `free`, `df`, `ip a` (IP + MAC), `ss -tlnp`, `ls -lR` → découverte de `info.php` et d'un **MySQL** non prévu |
| Mesures avant | empreintes `md5sum`, `systemd-analyze` (31 s), temps de réponse `curl -w %{time_total}` |
| Cible `SRV-VM` | Clone neuf, IP **temporaire** .51 pour éviter un conflit, Apache même version, vérif. `ufw` et ports |
| Migration | `tar czf` → `scp` (chiffré SSH) → `tar xzf -C /` |
| **Intégrité** | 4/4 empreintes md5 identiques source/cible |
| Bascule | Snapshot « Avant bascule », arrêt de la source, reprise du nom `srv-web` et de l'IP .50 |

## Incident pendant la bascule (et ce qu'il enseigne)
Après la bascule, la VM n'avait **aucune IPv4** : la source n'était pas encore complètement éteinte et l'adresse .50 était toujours prise → **détection de conflit d'adresse**. Diagnostic (`ip a`, contenu netplan, état d'Apache), puis réapplication une fois la source arrêtée.

- Bascule normale : 1 à 2 min · bascule réelle avec incident : ~1 h
- Leçon : **procédure de bascule écrite et vérifiée**, contrôle explicite de l'arrêt de la source, **plan de retour arrière** (snapshot + source conservée éteinte).

## Risques d'un P2V documentés
Écarts de versions de paquets entre source et cible (OpenSSH 3.7 vs 3.6 constaté) · services oubliés hors inventaire · pilotes et matériel spécifiques · licences liées au matériel · interruption de service.

## Compétences
Gestion du changement et bascule · inventaire système Linux · transfert sécurisé (SSH/scp) · **contrôle d'intégrité** · diagnostic d'incident réseau · analyse de risques
