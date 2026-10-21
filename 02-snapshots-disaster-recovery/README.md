# 02 — Snapshots et plan de reprise d'une VM critique

## Objectif
Pouvoir remettre en service rapidement un serveur applicatif après une mauvaise mise à jour ou une panne : snapshots, incident simulé, restauration chronométrée et **plan de reprise d'activité (PRA)** avec objectifs RTO/RPO.

## Environnement
| Élément | Valeur |
|---|---|
| Hyperviseur | VMware Workstation |
| VM | `SRV-APP-TEST` (clone complet Windows 11) |
| Application | Serveur web IIS, page « Version 1.0 » |
| Réseau | NAT VMware 192.168.209.0/24 |

## Scénario
1. Installation d'IIS et d'une page versionnée (PowerShell), ouverture du port 80.
2. Snapshot `AVANT-MAJ-v1.0` + **AutoProtect** quotidien (3 snapshots conservés).
3. Mise à jour simulée (« Version 2.0 »).
4. **Incident** : suppression de la page, arrêt et désactivation du service `W3SVC`.
5. Restauration via le Snapshot Manager, mesurée par une boucle de supervision lancée **depuis l'hôte** :

```powershell
while ($true) {
  try { Invoke-WebRequest http://192.168.209.22 -UseBasicParsing -TimeoutSec 1 | Out-Null; "$(Get-Date -f HH:mm:ss) OK" }
  catch { "$(Get-Date -f HH:mm:ss) KO" }
  Start-Sleep 1
}
```
*(la mesure ne peut pas tourner dans la VM : la restauration ramène aussi son horloge en arrière)*

## Résultats
| Mesure | Valeur |
|---|---|
| Restauration du snapshot (travail de VMware) | **17 s** |
| Interruption totale, essai 1 → essai 2 | 160 s → **74 s** |
| Interruption moyenne | ≈ 1 min 57 s |

L'essentiel de l'interruption vient de l'**intervention humaine**, pas de la technique : une procédure écrite et répétée divise le temps par deux.

## PRA simplifié
| Indicateur | Valeur |
|---|---|
| RPO | 24 h (snapshot quotidien) ; 0 pour une mise à jour (snapshot juste avant) |
| RTO cible | 15 min (détection + décision + restauration + vérification) |

Politique : snapshot obligatoire avant chaque changement, nommage `AVANT-<ACTION>-<VERSION>`, purge après 7 jours, procédure en 7 étapes (détecter → alerter → décider → restaurer → vérifier → informer → tracer), rôles définis, test trimestriel.

## Limite clé
**Un snapshot n'est pas une sauvegarde** : il vit sur le même disque que la VM. Une copie externe (NAS, disque externe) reste indispensable contre une panne matérielle ou un ransomware.

## Compétences
Continuité d'activité (PRA, RTO/RPO) · gestion des snapshots · supervision scriptée en PowerShell · analyse de mesures
