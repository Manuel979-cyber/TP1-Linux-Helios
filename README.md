# TP1 — Configuration avancée du système Linux

**Module** : Administration d'un Linux Server (BTC2) · IPSSI Lille
**Étudiant** : Beka Kamewe Manuel
**Environnement** : VirtualBox 7.2 · 2 VM Rocky Linux 9.8 (`srv-core`, `srv-edge`)

Contexte : reprise en main du serveur de l'entreprise fictive *Hélios Logistique*. Les scripts lancés à la main et via crontab sont remplacés par des services systemd supervisés. Le lien métier est redondé par un bond, le trafic « transport » est isolé dans un VLAN, et l'archivage passe par un timer systemd.

## Contenu du dépôt

| Dossier / fichier | Contenu |
|---|---|
| [`TP1/RAPPORT.md`](TP1/RAPPORT.md) | Rapport complet : démarche, mesures avant/après, explication des directives, tableau cron / timer, écarts rencontrés |
| [`TP1/configs/`](TP1/configs/) | Fichiers de configuration livrés : units systemd, drop-ins, script d'archivage, profil NetworkManager `bond0` |
| [`TP1/captures/`](TP1/captures/) | Sorties des commandes et graphiques de démarrage (`boot-avant.svg`, `boot-apres.svg`) |

## Résultats principaux

| Mission | Résultat |
|---|---|
| 1 — Services et démarrage | Boot de 2 min 55 s à 36 s · service `helios-collect` supervisé et relancé après `kill -9` · score `systemd-analyze security` de 5.3 MEDIUM à **1.1 OK** |
| 2 — Réseau avancé | Bond `active-backup` : **1 ping perdu sur 25** pendant la bascule · VLAN 42 vérifié par `tcpdump` · routage par règles, persistant après reboot |
| 3 — Automatisation | Script avec codes retour **0 / 2 / 3** vérifiés · timer quotidien à 02:30 · rattrapage `Persistent=yes` démontré |
| Bonus | Unité `path` qui déclenche l'archivage au dépôt d'un fichier |

## Plan réseau

| Carte | srv-core | srv-edge |
|---|---|---|
| `enp0s3` (NAT) | 10.0.2.15, route par défaut | 10.0.2.15, route par défaut |
| `enp0s8` (privé hôte) | 192.168.56.10/24 | 192.168.56.11/24 |
| `bond0` / `enp0s9` | 10.42.0.10/24 | 10.42.0.11/24 |
| VLAN 42 « transport » | `bond0.42` 10.42.42.10/24 | `enp0s9.42` 10.42.42.11/24 |
