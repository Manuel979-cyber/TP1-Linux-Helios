# TP1 — Configuration avancée du système Linux

**Auteur** : Beka Kamewe Manuel — **Date** : 02/11/2026 — **Environnement** : Rocky Linux 9.8, srv-core / srv-edge sous VirtualBox

## 1. Temps de démarrage avant / après

| | Avant | Après |
|---|---|---|
| `systemd-analyze` | **2 min 55,3 s** | **36,1 s** |
| `multi-user.target` atteint après | 1 min 22,2 s | 14,4 s |

Graphiques : `captures/boot-avant.svg` et `captures/boot-apres.svg`.

**Unités désactivées :**
- **`NetworkManager-wait-online.service` (masqué)** : il bloquait le démarrage jusqu'à 1 min, parce que `enp0s9`/`enp0s10` attendaient un DHCP qui n'existe pas sur `helios-lan`. Le collecteur n'écrit que des fichiers locaux, il n'a pas besoin d'attendre toutes les cartes.
- **`firewalld.service` (désactivé)** : il ajoutait 8,3 s sur la chaîne critique. C'est temporaire, il est réactivé au TP3.

Une partie du gain vient aussi de `kdump.service` (1 min 16 s au premier démarrage, le temps de générer son initramfs), qui ne coûte plus rien ensuite.

## 2. Score `systemd-analyze security` de helios-collect.service

| | Score |
|---|---|
| Avant (unit de l'énoncé) | **5.3 MEDIUM** |
| Après durcissement | **1.1 OK** |

Le durcissement est fait dans un drop-in, sans toucher à l'unit d'origine : `PrivateDevices`, `ProtectKernelModules`, `ProtectKernelLogs`, `ProtectClock`, `RestrictNamespaces`, `RestrictAddressFamilies=AF_UNIX`, `SystemCallFilter=@system-service`, `MemoryDenyWriteExecute`, `CapabilityBoundingSet=` vide. Le service fonctionne toujours après (lignes du CSV complètes).

## 3. Les trois codes retour de helios-archive.sh

| Test | Code | Message dans le journal |
|---|---|---|
| Exécution normale | **0** | `archive …/telemetry-….csv.gz (41 lignes)` |
| Verrou déjà pris (`flock`) | **2** | `une autre instance tourne deja` |
| Fichier source absent | **3** | `source illisible : /var/lib/helios/telemetry.csv` |

Pour le test du code 3, le collecteur a été arrêté : sinon il recrée le fichier en moins de 5 s.

## 4. Tableau cron vs timer

| Critère | cron | timer systemd |
|---|---|---|
| Rattrapage si la VM était éteinte | Non, l'exécution est perdue | Oui avec `Persistent=yes` : échéance de 16:10 manquée, archive lancée au redémarrage à 16:14 |
| Journalisation | Sortie du job perdue, pas de code retour visible | `journalctl -u helios-archive.service` : messages, code retour, durée |
| Dépendances | Aucune | `ConditionPathExists=`, `After=`, `Requires=` |
| Limitation de ressources | Rien par défaut | `Nice=10`, `IOSchedulingClass=idle`, cgroups |
| Étalement de la charge | `RANDOM_DELAY` global | `RandomizedDelaySec=120` pour ce timer |
| Voir la prochaine exécution | Lire la ligne crontab | `systemctl list-timers` |
