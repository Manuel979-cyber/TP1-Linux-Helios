# TP1 — Configuration avancée du système Linux (Hélios Logistique)

**Module** : Administration d'un Linux Server · BTC2 — **Auteur** : Beka Kamewe Manuel — **Date** : 02/11/2026
**Environnement** : VirtualBox 7.2.4 sous Windows 10 · Rocky Linux 9.8 (noyau 5.14.0-687.10.1.el9_8)

Toutes les valeurs ci-dessous viennent des sorties réelles rangées dans `TP1/captures/` (le nom du fichier est indiqué entre parenthèses).

---

## 0. Socle (README §3 — reconstitué, le README n'était pas fourni)

| Élément | srv-core | srv-edge |
|---|---|---|
| CPU / RAM / disque | 2 vCPU · 2048 Mo · 20 Go (LVM) | idem |
| Carte 1 `enp0s3` | NAT, DHCP (10.0.2.15), **route par défaut** | idem |
| Carte 2 `enp0s8` | Réseau privé hôte, **192.168.56.10/24** | **192.168.56.11/24** |
| Carte 3 `enp0s9` | `helios-lan`, promiscuité allow-all → bond0 | `helios-lan` → lien « metier » + VLAN 42 |
| Carte 4 `enp0s10` | `helios-lan`, promiscuité allow-all → bond0 | désactivée |
| Nom d'hôte | srv-core.helios.lan | srv-edge.helios.lan |
| SELinux | Enforcing | Enforcing |
| Snapshots | `00-socle-propre` (après installation) puis `01-fin-TP1` (fin de séance, VM éteinte) | idem |

- Installation **automatisée par kickstart** (`kickstart/srv-*-ks.cfg`) : français, clavier fr, fuseau Europe/Paris, paquets `@^minimal-environment`, compte `etudiant` (groupe wheel), connexion SSH de root **par clé uniquement** (Rocky 9 interdit le mot de passe root en SSH).
- Le kickstart est fourni sur un mini-CD nommé **`OEMDRV`** : Anaconda cherche automatiquement un `ks.cfg` sur un volume portant ce nom, sans paramètre de démarrage.
- Les VM tournent en **headless** et sont pilotées depuis Windows par `ssh`/`scp` sur le réseau privé hôte.

---

## Mission 1 — Services, processus et séquence de démarrage

### 1.1 Temps de démarrage avant / après (`m1-1-boot-avant.txt`, `m1-1-boot-apres.txt`, `boot-avant.svg`, `boot-apres.svg`)

| | Avant | Après |
|---|---|---|
| `systemd-analyze` (total) | **2 min 55,3 s** (kernel 4,8 s + initrd 12,5 s + userspace 2 min 38 s) | **36,1 s** (2,1 + 9,3 + 24,7 s) |
| `multi-user.target` atteint après | 1 min 22,2 s | 14,4 s |
| 1re ligne de `blame` | `kdump.service` 1 min 16 s | périphériques ttyS ~13,8 s |
| 2e ligne de `blame` | `NetworkManager-wait-online.service` 1 min 0,2 s | — (masqué) |
| Chaîne critique | … → `firewalld.service` +8,3 s → `network-pre.target` → … → `network-online.target` @1 min 21 s | `NetworkManager` → `network-online.target` @13,5 s |

**Unités désactivées et justification :**
- **`NetworkManager-wait-online.service` → masqué** (`systemctl mask` = lien vers `/dev/null`, il ne peut plus être démarré même comme dépendance). Il bloquait `network-online.target` jusqu'à 1 min parce que les cartes `enp0s9`/`enp0s10` attendaient un DHCP inexistant sur `helios-lan`. Le service de collecte n'a pas besoin d'attendre que *toutes* les cartes soient configurées.
- **Contrepartie de ce masquage :** `network-online.target` est alors atteint sans que le réseau soit réellement configuré. Or `helios-collect.service` déclare `Wants=`/`After=network-online.target` justement pour démarrer après le réseau. Ici le collecteur n'écrit que des fichiers locaux, donc ce n'est pas gênant. En revanche, un service qui doit joindre un serveur distant au démarrage pourrait échouer : il faudrait alors plutôt limiter l'attente aux cartes utiles (par exemple `connection.wait-device-timeout`, ou retirer les profils DHCP inutiles) au lieu de masquer le service.
- **`firewalld.service` → `disable --now`** (arrêté et retiré du démarrage). Il était sur la chaîne critique (+8,3 s). C'est **temporaire** : il sera réactivé au TP3 (un serveur en production garde son pare-feu).

**Honnêteté sur la mesure :** le boot « avant » contient aussi `kdump.service` (1 min 16 s). Ce service construit son initramfs de secours quand il est absent, c'est-à-dire lors des tout premiers démarrages après l'installation ; au boot « après », il n'apparaît plus dans le top 15. Une partie du gain (2 min 55 s → 36 s) vient donc très probablement de ce coût ponctuel et non de nos réglages. Le gain qui vient réellement de nos réglages : environ 60 s pour wait-online et 8 s pour firewalld.

### 1.2 Service de collecte (`m1-2-service.txt`, `configs/helios-collect.service`)

Le collecteur `/usr/local/sbin/helios-collect.sh` écrit toutes les N secondes `date,charge,MemAvailable` dans `/var/lib/helios/telemetry.csv`. Il tourne sous le compte système `helios` (`useradd --system --shell /sbin/nologin`), qui n'a pas de shell de connexion.

Rôle de chaque directive :

| Directive | Rôle |
|---|---|
| `After=` / `Wants=network-online.target` | Démarre après le réseau et le demande, sans en dépendre strictement |
| `Type=simple` | Le processus lancé par ExecStart **est** le service (boucle infinie) |
| `User=` / `Group=helios` | Moindre privilège : pas de root |
| `Environment=HELIOS_INTERVAL=15` | Variable lue par le script (`${HELIOS_INTERVAL:-15}`) |
| `Restart=on-failure` + `RestartSec=5s` | Relance automatique 5 s après un arrêt anormal (signal, code ≠ 0) |
| `TimeoutStopSec=20s` | Au-delà de 20 s, systemd force l'arrêt (SIGKILL) |
| `NoNewPrivileges=yes` | Interdit tout gain de privilège (setuid, capabilities) |
| `PrivateTmp=yes` | `/tmp` privé, isolé des autres services |
| `ProtectSystem=strict` | Tout le système de fichiers est en lecture seule… |
| `ReadWritePaths=/var/lib/helios /var/log/helios` | …sauf ces deux dossiers |
| `ProtectHome=yes` | `/home`, `/root`, `/run/user` invisibles |
| `ProtectKernelTunables=yes` | `/proc/sys`, `/sys` en lecture seule |
| `ProtectControlGroups=yes` | `/sys/fs/cgroup` en lecture seule |
| `RestrictSUIDSGID=yes` | Interdit de créer des fichiers setuid/setgid |
| `CapabilityBoundingSet=` (vide) | Retire **toutes** les capabilities Linux |
| `WantedBy=multi-user.target` | `enable` crée le lien dans `multi-user.target.wants/` |

**Scores `systemd-analyze security`** (plus le score est bas, mieux le service est isolé) :

| Étape | Score d'exposition | Capture |
|---|---|---|
| Pour information : unit sans aucune directive de protection | 9.2 UNSAFE | `m1-2-security-avant.txt` |
| **Avant** : unit de l'énoncé, telle que fournie | **5.3 MEDIUM** | `m1-2-security-unit-enonce.txt` |
| **Après** : unit + drop-in `hardening.conf` | **1.1 OK** | `m1-2-security-hardening.txt` |

**Durcissement** par un second drop-in `/etc/systemd/system/helios-collect.service.d/hardening.conf` (`configs/hardening.conf`). Comme `override.conf`, il ne touche pas au fichier d'origine.

| Directive | Rôle |
|---|---|
| `PrivateDevices=yes` | `/dev` privé : aucun accès aux périphériques physiques |
| `PrivateUsers=yes` | Espace de noms utilisateur isolé |
| `ProtectKernelModules=yes` | Interdit de charger ou décharger des modules noyau |
| `ProtectKernelLogs=yes` | Pas d'accès aux journaux du noyau (`dmesg`, `/proc/kmsg`) |
| `ProtectClock=yes` | Interdit de changer l'horloge système |
| `ProtectProc=invisible` | Les processus des autres utilisateurs sont invisibles dans `/proc` |
| `LockPersonality=yes` | Bloque le changement de « personnalité » d'exécution (émulation d'ABI) |
| `MemoryDenyWriteExecute=yes` | Interdit la mémoire à la fois inscriptible et exécutable (anti-injection de code) |
| `RestrictRealtime=yes` | Pas d'ordonnancement temps réel (évite de monopoliser le CPU) |
| `RestrictNamespaces=yes` | Interdit de créer de nouveaux espaces de noms |
| `RestrictAddressFamilies=AF_UNIX` | Seulement des sockets locales. C'est valable ici car le collecteur n'ouvre aucune connexion réseau, mais ça casserait un service réseau |
| `SystemCallFilter=@system-service` | N'autorise que les appels système d'un service classique |
| `SystemCallErrorNumber=EPERM` | Un appel système interdit renvoie « permission refusée » au lieu de tuer le processus |
| `SystemCallArchitectures=native` | Seulement les appels système de l'architecture native (x86_64) |
| `UMask=0077` | Les fichiers créés ne sont lisibles que par `helios` |

**Vérification fonctionnelle :** après durcissement, le service est `active` et les lignes restent complètes (`2026-11-02T16:08:25+01:00,0.00,1414544`).

**Piège à éviter :** ne pas ajouter `ProcSubset=pid`. Cette directive masquerait `/proc/loadavg` et `/proc/meminfo` : le collecteur tournerait encore, mais écrirait des champs vides. Le durcissement se fait service par service, en vérifiant le résultat, jamais par copier-coller.

### 1.3 Redémarrage automatique et drop-in (`m1-3-redemarrage-dropin.txt`, `configs/override.conf`)

- **`kill -9` du PID 2028** : 8 s plus tard, le service est `active (running)` avec un **nouveau PID 2043** et `NRestarts=1`. `Restart=on-failure` a joué, car SIGKILL compte comme un échec.
- **`on-failure` contre `always`** (`m1-3-restart-on-failure.txt`) :

| Action | Résultat observé |
|---|---|
| `kill -9` (PID 18857) | relancé : `ActiveState=active`, nouveau PID 18882, `NRestarts=1` |
| `systemctl stop` | **pas** relancé : `ActiveState=inactive`, `MainPID=0`, `NRestarts` reste à 1 |

  Un arrêt demandé par l'administrateur n'est pas un échec, donc `on-failure` respecte le `stop`. Avec `Restart=always`, le service serait relancé après n'importe quelle sortie, même propre (code 0). Un `systemctl stop` reste par contre toujours respecté, quel que soit le réglage.
- **Drop-in** `/etc/systemd/system/helios-collect.service.d/override.conf` : c'est le fichier que `systemctl edit` crée. Ici il a été écrit directement, puis `systemctl daemon-reload`, parce que l'éditeur interactif ne peut pas être piloté par script. Le fichier d'origine n'est **jamais modifié** et les réglages locaux survivent à une mise à jour de l'unit.

| Directive du drop-in | Valeur observée (`systemctl show`) |
|---|---|
| `Environment=HELIOS_INTERVAL=5` | collecte toutes les 5 s |
| `CPUQuota=20%` | `CPUQuotaPerSecUSec=200ms` (20 % d'un CPU) |
| `MemoryMax=96M` | `MemoryMax=100663296` (= 96 × 1024²) |
| `IOWeight=50` | poids d'E/S réduit (défaut 100) |
| `Nice=5` | `Nice=5` (priorité CPU plus basse) |

- `systemd-cgls` montre le processus seul dans son cgroup `/system.slice/helios-collect.service`.
- `systemd-cgtop -b` (`-b` = mode batch, obligatoire hors terminal interactif) mesure environ 684 Ko de mémoire et environ 6 % de CPU sur un échantillon.

### 1.4 Cible métier et priorités (`m1-4-target-priorites.txt`, `configs/helios.target`)

- `helios.target` : `Requires=multi-user.target` + `After=`. `AllowIsolate=no` empêche un `systemctl isolate helios.target`. Un isolate arrête toutes les unités qui ne sont pas dépendances de la cible, y compris **`sshd`** : on perdrait l'accès distant au serveur.
- `systemctl add-wants helios.target helios-collect.service` crée le lien `helios.target.wants/helios-collect.service`.
- `systemctl list-dependencies helios.target` affiche bien `helios-collect.service` en tête.
- `stop` puis `start` de la cible : ensuite `helios.target` et `helios-collect.service` sont **active**.

| Commande | Effet observé sur `sha256sum` (PID 2194) |
|---|---|
| `nice -n 19` au lancement | `NI=19` (priorité la plus basse) |
| `renice -n 5` | « priorité précédente 19, nouvelle priorité 5 », `NI=5 PRI=14` |
| `ionice -c 2 -n 7` | `best-effort : priorité 7` (classe 2 = best-effort, 7 = la plus basse) |
| `chrt -p` | `SCHED_OTHER`, priorité 0 (ordonnancement normal, pas temps réel) |

**Livrables M1** : `captures/m1-1-*`, `boot-avant.svg`, `boot-apres.svg`, `configs/helios-collect.service`, `configs/override.conf`, `configs/helios.target`, `configs/hardening.conf`, `captures/m1-2-security-*`, `captures/m1-2-durcissement.txt`, `captures/m1-3-restart-on-failure.txt`.

---

## Mission 2 — Réseau avancé : bond, VLAN, routage par règles

### 2.1 Bond active-backup (`m2-1-bond.txt`, `configs/bond0.txt`)

- `bond0` en mode `active-backup`, avec `miimon=100` (vérifie le lien toutes les 100 ms), `primary=enp0s9` (lien préféré) et `fail_over_mac=active`. Avec cette dernière option, le bond prend l'adresse MAC de la carte active : c'est utile sous VirtualBox, d'où aussi la promiscuité allow-all.
- Adresse `10.42.0.10/24`. `ipv4.never-default yes` : cette carte ne fournit jamais la route par défaut (celle-ci reste sur `enp0s3`).
- Les ports `bond0-p1` (enp0s9) et `bond0-p2` (enp0s10) sont des profils `ethernet` avec `master bond0`. Le mot-clé `master` est toujours accepté par NetworkManager sur Rocky 9, mais l'écriture recommandée depuis NM 1.46 est `controller bond0` / `port-type bond`. Les deux donnent le même résultat.
- Avec `miimon=100`, une panne de lien est détectée en moins de 100 ms.
- **Ne jamais mettre `enp0s8` (administration) ni `enp0s3` (NAT) dans le bond qu'on teste** : en cas d'erreur, on perdrait l'accès à la VM.
- Les profils créés automatiquement par l'installateur sur `enp0s9`/`enp0s10` ont été supprimés d'abord, pour éviter deux profils concurrents sur la même carte.
- srv-edge : profil `metier` sur `enp0s9`, en `10.42.0.11/24`.

### 2.2 Bascule (`m2-2-*`)

| Moment | `Currently Active Slave` | enp0s9 | enp0s10 |
|---|---|---|---|
| Avant coupure | **enp0s9** | up | up |
| Pendant `VBoxManage controlvm srv-core setlinkstate3 off` | **enp0s10** | **down** (Link Failure Count 1) | up |
| Après `setlinkstate3 on` | enp0s9 (retour au primaire) | up | up |

- **Ping 10.42.0.11 pendant la coupure : 25 transmis, 24 reçus, 1 perdu (icmp_seq=7)**. Le critère est 2 au maximum, il est respecté.
- Journal noyau : `bond0: (slave enp0s9): link status definitely down, disabling slave` puis `(slave enp0s10): making interface the new active one`.
- Le mode **802.3ad (LACP)** n'est pas réalisable sous VirtualBox, qui n'émule pas de commutateur LACP. Il est seulement décrit, comme indiqué dans le README §5.

### 2.3 VLAN 802.1Q « transport » id 42 (`m2-3-vlan.txt`, `m2-3-tcpdump-vlan42.txt`)

- srv-core : `bond0.42` en `10.42.42.10/24`. `ip -d link` affiche `vlan protocol 802.1Q id 42`.
- srv-edge : `enp0s9.42` en `10.42.42.11/24`.
- **Preuve du marquage** avec `tcpdump -nei bond0 'vlan 42'` : chaque trame porte `ethertype 802.1Q (0x8100) … vlan 42, p 0`. On voit la requête ARP, la réponse ARP, puis les paquets ICMP echo request/reply entre 10.42.42.10 et 10.42.42.11. Bilan : 5 paquets capturés, 0 perdu.

### 2.4 Table de routage secondaire (`m2-4-routage.txt`)

- **Incohérence de l'énoncé** : sur Rocky 9, `/etc/iproute2/rt_tables` n'existe plus, le fichier par défaut est dans `/usr/share/iproute2/`. On copie d'abord le fichier par défaut dans `/etc/iproute2/` (sinon le `echo >>` créerait un fichier sans les tables `main`/`local`), puis on ajoute `142 transport`.
- `ipv4.routes "0.0.0.0/0 10.42.42.11 table=142"` ajoute une route par défaut **dans la table 142 seulement**.
- `ipv4.routing-rules "priority 100 from 10.42.42.10/32 table 142"` : tout paquet dont la source est 10.42.42.10 consulte la table 142.

| Commande | Résultat |
|---|---|
| `ip rule show` | `100: from 10.42.42.10 lookup transport proto static` |
| `ip route show table transport` | `default via 10.42.42.11 dev bond0.42` |
| `ip route get 8.8.8.8 from 10.42.42.10` | `via 10.42.42.11 dev bond0.42 table transport` ✅ |
| `ip route get 8.8.8.8` | `via 10.0.2.2 dev enp0s3` ✅ |

### 2.5 Persistance après `systemctl reboot` (`m2-5-apres-reboot.txt`, `srv-edge/m2-edge-apres-reboot.txt`)

Après redémarrage des deux VM, tout est revenu sans intervention, parce que les profils NetworkManager sont enregistrés sur disque :
- bond0 avec ses 2 ports `MII Status: up` et enp0s9 actif ;
- bond0.42, la règle 100 et la table transport ;
- les deux `ip route get` donnent le même résultat qu'avant le reboot ;
- ping 10.42.0.11 et 10.42.42.11 : 0 % de perte.

**Livrables M2** : `configs/bond0.txt`, `captures/m2-2-bond-avant-coupure.txt` / `-pendant-coupure.txt` + `m2-2-ping-bascule.txt`, `m2-3-tcpdump-vlan42.txt`, `m2-4-routage.txt`.

---

## Mission 3 — Automatisation : script robuste et timer systemd

### 3.1 Script `helios-archive.sh` (`configs/helios-archive.sh`, `m3-1-script-codes-retour.txt`)

Les cinq règles d'un script de production, et comment le script les applique :

| Règle | Implémentation |
|---|---|
| Mode strict | `set -euo pipefail` (arrêt sur erreur, variable non définie, échec dans un pipe) + `IFS=$'\n\t'` |
| Verrou exclusif | `exec 9>"$LOCK"` + `flock -n 9` (non bloquant) → sinon `exit 2` |
| Nettoyage sur sortie | `trap cleanup EXIT` : supprime le dossier temporaire et journalise le code retour. `trap … INT TERM` : arrêt propre sur signal |
| Journalisation journald | `logger -t helios-archive -p daemon.<niveau>` → lisible avec `journalctl -t helios-archive` |
| Codes retour explicites | 0 OK · 1 erreur interne · 2 verrou déjà pris · 3 prérequis manquant |

**Les trois codes retour observés :**

| Test | Code obtenu | Message dans le journal |
|---|---|---|
| Exécution normale | **0** | `archive …/telemetry-20261102-135532.csv.gz (41 lignes)` puis `fin d execution rc=0` |
| Verrou déjà pris par un autre processus | **2** | `une autre instance tourne deja` puis `rc=2` |
| Source déplacée dans `/tmp/t.csv` | **3** | `source illisible : /var/lib/helios/telemetry.csv` puis `rc=3` |

**Incohérence de l'énoncé** : avec le drop-in (`HELIOS_INTERVAL=5`), le collecteur **recrée** `telemetry.csv` en moins de 5 s après le `mv`, et le test du code 3 peut alors renvoyer 0. Le collecteur a donc été arrêté pendant ce test (`systemctl stop`/`start helios-collect.service`).

### 3.2 Timer systemd (`configs/helios-archive.{service,timer}`, `m3-2-timer-cron.txt`)

- **Service** `Type=oneshot` : il s'exécute puis se termine.
  - `ConditionPathExists=` : si la source manque, le service est sauté sans être compté comme un échec.
  - `Nice=10` et `IOSchedulingClass=idle` : il passe après tout le reste, en CPU comme en disque.
- **Timer** :
  - `OnCalendar=*-*-* 02:30:00` : tous les jours à 2 h 30.
  - `AccuracySec=1min` : précision de déclenchement d'une minute.
  - `RandomizedDelaySec=120` : décalage aléatoire de 0 à 2 min.
  - `Persistent=yes` : rattrapage si la machine était éteinte à l'heure prévue.
- `systemd-analyze calendar` donne les 3 prochaines échéances : 03/11, 04/11 et 05/11 à 02:30 CET.
- `systemctl list-timers` affiche `NEXT Tue 2026-11-03 02:30:11 CET`. Lors de la vérification finale, il affichait `02:31:49` : c'est l'effet de `RandomizedDelaySec`.
- Lancement manuel `systemctl start helios-archive.service` : `Result=success`, `ExecMainStatus=0`.

**Tableau comparatif cron / timer (observations) :**

| Critère | `cron` (`/etc/cron.d/helios-legacy`) | `systemd timer` |
|---|---|---|
| Rattrapage après VM éteinte | Aucun : une exécution manquée est perdue (il faudrait anacron, paquet `cronie-anacron`, qui ne travaille qu'à la granularité du jour) | **Observé** : échéance manquée à 16:10 pendant que la VM était éteinte, archivage lancé automatiquement à 16:14:23, après le redémarrage de 16:13 (voir démonstration ci-dessous) |
| Journalisation et corrélation | `journalctl -u crond` ne montre que le démarrage de crond. La sortie du job est jetée (`MAILTO=""`), on ne voit pas de code retour | `journalctl -u helios-archive.service` regroupe `Starting…`, les messages du script, `status=0/SUCCESS` et `Deactivated successfully`, le tout rattaché à l'unité |
| Expression de dépendances | Aucune | `ConditionPathExists=`, `Unit=`, et possibilité d'`After=`/`Requires=` |
| Limitation de ressources | Rien par défaut (il faudrait préfixer `nice`/`ionice` dans la ligne) | `Nice=10`, `IOSchedulingClass=idle`, et tout le contrôle cgroup (`CPUQuota`, `MemoryMax`…) |
| Dispersion de la charge | Seulement `RANDOM_DELAY`, global à crond (« RANDOM_DELAY will be scaled… » dans le journal) | `RandomizedDelaySec=120` pour ce seul timer : NEXT observé à 02:30:11 puis 02:31:49 |
| Prochaine exécution | Pas de commande native : il faut lire la ligne `30 2 * * *` | `systemctl list-timers` (colonnes NEXT/LEFT/LAST) et `systemd-analyze calendar` |

**Démonstration de `Persistent=yes`** (`m3-4-persistent-rattrapage.txt`) : pour ne pas attendre 02:30, l'échéance a été déplacée temporairement par un drop-in du timer (`OnCalendar=` vide pour effacer, puis `OnCalendar=*-*-* 16:10:00`).

| Heure | Événement |
|---|---|
| 16:08:44 | Drop-in posé, `NEXT` = 16:10:07. Le stamp date encore de 13:55:55 |
| 16:08:46 | `systemctl poweroff` : la VM est éteinte **avant** l'échéance |
| 16:10 | Échéance manquée, VM éteinte |
| 16:13:18 | VM rallumée |
| 16:13:41 | `Started Declenche l archivage…` (le timer démarre au boot) |
| 16:14:23 | **`helios-archive.service` exécuté** (le délai vient de `RandomizedDelaySec`) : archive de 1591 lignes, `rc=0`. Le stamp passe à 16:14:22, `LAST` = 16:14:22 |

Le drop-in a ensuite été supprimé : `NEXT` est revenu à 02:30:03. Avec cron, l'exécution de 16:10 aurait simplement été perdue.

### Bonus — unité `path` (`configs/helios-inbox.path`, `m3-3-bonus-path.txt`)

- `PathChanged=/srv/helios/inbox` déclenche `helios-archive.service` à chaque modification du dossier.
- `touch /srv/helios/inbox/test.csv` a provoqué une archive (`telemetry-20261102-135605.csv.gz`) dans la seconde, avec `rc=0`.
- On utilise **`PathChanged`** plutôt que `PathExistsGlob` parce que le script ne vide pas l'inbox. Avec `PathExistsGlob`, la condition resterait vraie et le service serait relancé en boucle jusqu'à `unit-start-limit-hit`.

---

## Critères de réussite

| Critère | Résultat |
|---|---|
| `helios-collect.service` enabled et survit à `kill -9` | ✅ (PID 11821 → 12118) |
| `MemoryMax=100663296` sans modifier l'unit d'origine | ✅ (réglage uniquement dans `override.conf`) |
| `list-dependencies helios.target` montre `helios-collect.service` | ✅ |
| bond0 : 2 ports `MII Status: up` + un `Active Slave` | ✅ |
| Coupure de la carte 3 : ≤ 2 requêtes ICMP perdues | ✅ (1 perdue) |
| `ip route get 8.8.8.8 from 10.42.42.10` → bond0.42 ; `ip route get 8.8.8.8` → enp0s3 | ✅ |
| Configuration réseau persistante après reboot | ✅ |
| Script : 2 si instance en cours, 3 si source absente | ✅ |
| `list-timers` : NEXT cohérent avec 02:30 | ✅ (02:31:49, dans la fenêtre de 2 min) |

---

## Écarts et incohérences relevés

1. **README §3 absent** : le socle (adresses 192.168.56.10/.11, noms `*.helios.lan`, compte `etudiant`) a été reconstitué. À confronter au README officiel.
2. **Installation automatique de VirtualBox (`VBoxManage unattended`)** : elle échoue avec l'ISO Rocky 9.8 (« ISOLINUX : Image checksum error »). Elle a été remplacée par un kickstart sur un volume `OEMDRV`.
3. **`/etc/iproute2/rt_tables`** n'existe pas par défaut sur Rocky 9. Il a été créé à partir de `/usr/share/iproute2/rt_tables` avant d'ajouter `142 transport`.
4. **Test du code retour 3** : le collecteur à 5 s recrée la source. Il a été arrêté pendant le test.
5. **Score de sécurité** : l'unit fournie par l'énoncé contient déjà les premières protections (5.3 MEDIUM). « Durcissez » a été traité par un second drop-in `hardening.conf` (1.1 OK), sans modifier l'unit d'origine.
6. **Gain au démarrage** : il est en partie dû à `kdump`, qui génère son initramfs au premier boot (voir 1.1).
7. **`systemctl edit`** (interactif) a été remplacé par la création directe du drop-in + `daemon-reload`. Le résultat est identique.
8. **`firewalld` désactivé** : à réactiver au TP3. Avec firewalld actif, le ping vers 10.42.0.11 pourrait être rejeté sur `helios-lan`. C'est aussi pour ça que l'énoncé le coupe dès la mission 1.1.
9. **Snapshot de fin de séance** : `01-fin-TP1` pris sur srv-core et srv-edge, VM éteintes proprement. Il servira de point de départ pour le TP5.
