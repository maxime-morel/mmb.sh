---
title: "De VeraCrypt à Cryptomator : vers un stockage chiffré dans le cloud"
date: 2026-03-07
lastmod: 2026-03-08
draft: false

slug: "cryptomator-stockage-chiffre-cloud"
aliases:
  - "/posts/2026-03-07-setup-cryptomator/"

description: "Pourquoi j’ai remplacé VeraCrypt par Cryptomator pour garder un stockage cloud pratique tout en chiffrant localement mes fichiers sous Linux."
summary: "Retour d’expérience sur un stockage cloud avec chiffrement côté client avec Cryptomator, avec une mise en place détaillée sous Linux et Kubuntu."

author:
  - "Maxime Morel-Bailly"

tags:
  - "cloud"
  - "cryptographie"
  - "linux"
  - "privacy"
  - "cryptomator"
  - "rclone"
  - "kubuntu"

keywords:
  - "cryptomator"
  - "veracrypt"
  - "google drive"
  - "stockage chiffré"
  - "chiffrement côté client"
  - "rclone bisync"
---

Le mot **« chiffré »** dans le cloud rassure. Pourtant, dans la plupart des cas, vos documents restent techniquement lisibles par l’hébergeur.

Lorsque vous stockez vos fichiers sur Google Drive ou sur un service similaire, **la clé de déchiffrement n’est généralement pas entre vos mains**, mais entre celles du fournisseur qui stocke vos données.

Résultat : si ce service doit analyser ou accéder à vos fichiers dans le cadre de son fonctionnement, de ses traitements internes ou de ses conditions d’utilisation, il en a la capacité technique. Et en cas d’accès frauduleux à votre compte, un attaquant peut lui aussi accéder directement à l’ensemble de vos documents.

À partir de là, il y a plusieurs solutions :

1. accepter ce fonctionnement et faire confiance au service utilisé
2. choisir une solution cloud spécialisée, souvent payante, comme [Proton](https://proton.me/)
3. se risquer à maintenir sa propre solution locale ou à domicile (ce que j'ai envisager)
4. ou alors… **allier les avantages du stockage cloud avec la sécurité d’un chiffrement local**

C’est précisément cette quatrième approche que j’ai testée, en suivant la logique suivante :

```
       [ Mes documents ]
              |
              v
    [ Chiffrement local ]
    avant envoi au cloud
              |
              v
[ Stockage et synchronisation ]
     Google Drive / S3
              |
              v
  [ Accès multi-appareils ]
 ````

Ce n'est pas une solution de sauvegarde, mais une solution de stockage et d'accessibilité.

## Le besoin de départ

J’utilisais jusqu’ici des volumes **VeraCrypt** locaux pour stocker mes documents sensibles.

Côté sécurité / chiffrement, rien à redire. En revanche, côté usage et stockage, **les limites étaient contraignantes** :

- le conteneur devenait vite volumineux, ce qui rendait la sauvegarde peu pratique et difficile à automatiser
- l’accès était limité aux machines disposant de VeraCrypt
- la consultation depuis mobile était compliquée, surtout avec des volumes de plusieurs Gb à transférer.

Je cherchais donc une **alternative capable de conserver un bon niveau de confidentialité, sans perdre la praticité du cloud**. Et la praticité du cloud, pour moi, c'est surtout l'accessibilité.

Je souhaitais également éviter les solutions par abonnement ou fortement orientées SaaS, comme Proton (malgré tous ses avantages), tout en restant sur un outil maintenu, fiable, sécurisé et demandant peu de maintenance.

L’une des problématiques que je cherchais à résoudre était donc de **permettre une sauvegarde dans le cloud**, tout en pouvant **consulter mes données depuis mon smartphone.**

## L’approche retenue

L’idée est simple : **les fichiers sont chiffrés localement avant d’être envoyés dans le cloud**. Le fournisseur cloud n’a alors plus accès au contenu lisible des fichiers.

Dans mon cas, j’ai retenu le montage suivant :

- **Google Drive** comme stockage cloud distant, car j’utilisais déjà cet environnement (et toujours utilisé pour synchroniser mes photos), même si cette approche peut fonctionner avec d’autres fournisseurs, y compris compatibles S3 ;
- **[Cryptomator](https://cryptomator.org/)** pour le chiffrement côté client, compatible avec Windows, macOS, Linux et mobile ;
- **rclone** pour la synchronisation sous Kubuntu ; sur Windows ou macOS, l’application Google Drive peut être utilisée plus simplement ;
- l’application mobile **Cryptomator** pour l’accès ponctuel depuis smartphone.

Ce modèle permet de garder les avantages du cloud : accessibilité, souplesse, multi-appareils, tout en évitant de confier le contenu lisible des documents au service d’hébergement.

## Les autres solutions envisagées

Avant d’arriver à ce montage, j’ai exploré plusieurs pistes. Rester sur **VeraCrypt** était l’option la plus évidente. Le problème n’était pas le chiffrement lui-même, mais le format conteneur : pratique pour de l’archive ou du stockage local, beaucoup moins pour de la synchronisation régulière et un accès mobile.

J’ai aussi regardé les **solutions spécialisées** comme Proton. Elles sont intéressantes, mais impliquent un abonnement et une dépendance plus forte à un service unique, ce que je voulais éviter. L'avantage de Cryptomator est que je peux changer sans contrainte le stockage distant.

L’**auto-hébergement** faisait également partie des options possibles, tel que mettre en place un NAS maison avec une solution sécurisé par design, mais cela nécessite un investissement initial relativement important ainsi que le temps de mise en place que je n'avais pas.

C’est pour cela que j’ai retenu une approche intermédiaire : garder la praticité du cloud, mais ajouter moi-même la couche de chiffrement côté client.

## Pourquoi Cryptomator ?

![Interface Cryptomator](/images/cryptomator/cryptomator-interface.png)

Pour commencer, [Cryptomator](https://cryptomator.org/) est open source et propose une solution gratuite sur desktop, sans obligation de passer sur un modèle SaaS. C’était l’une de mes contraintes dans le choix de la solution. L’application mobile, elle, est payante, autour de 30 euros, mais avec un achat unique et non un abonnement.

Il remplace avantageusement le modèle du gros conteneur chiffré dans mon cas d'utilisation. Au lieu d’avoir un seul gros fichier VeraCrypt, on travaille avec un **coffre chiffré fichier par fichier**, bien plus adapté à la synchronisation.

Cryptomator a aussi un autre avantage important : il est disponible sur **Windows, macOS, Linux, Android et iOS** et est nativement compatible avec Google Drive sur mobile (cela ne veut pas dire que l’expérience est identique à l'application Google Drive).

En pratique, Cryptomator apporte plusieurs bénéfices :
- l’accès mobile reste possible
- le stockage distant ne voit que des fichiers déjà chiffrés
- on évite les limites de sauvegarde liées à un conteneur unique

Leur page [for individuals](https://cryptomator.org/for-individuals/) décrit assez clairement les fonctionnalités principales de la solution : **chiffrement côté client avant l’envoi**, compatibilité avec les services cloud existants, support **Windows / macOS / Linux / Android / iOS**, simplicité d’utilisation, code **open source**, et chiffrement pensé pour un usage quotidien sans connaissances techniques particulières. La FAQ rappelle aussi que les fichiers sont chiffrés individuellement, que les noms de fichiers sont protégés, et que l’outil n’est pas limité à un fournisseur cloud unique.

## Description du setup

### Sous Linux

Sur Linux, et en particulier sous mon système Kubuntu, il n’existe pas, à ma connaissance, de client Google Drive officiel équivalent à celui disponible sur Windows ou macOS. Il faut donc ajouter une couche supplémentaire pour la synchronisation.

J’ai choisi **rclone**, car c’est un outil robuste, flexible et bien adapté à une logique de synchronisation maîtrisée.

Le workflow retenu est le suivant :

1. travailler localement dans le coffre Cryptomator
2. synchroniser le répertoire chiffré vers Google Drive avec `rclone`
3. puis automatiser cette synchronisation avec un timer `systemd --user`.

### Sous Windows ou macOS

Sous Windows ou macOS, l’approche peut être un peu plus simple.

Il est possible d’utiliser :

- **Cryptomator** pour gérer le coffre ;
- l’application **Google Drive** officielle pour synchroniser le répertoire du coffre vers le cloud.

Le principe reste cependant le même : le coffre est chiffré **avant** la synchronisation. Le fournisseur cloud ne reçoit donc que des fichiers déjà chiffrés.

### Sur mobile

C’est un point important, parce que c’était justement l’une des limites de VeraCrypt dans mon cas.

J'ai cependant vu le prix de l'application mobile (29.99 euros), et j’ai été réticent de faire cet achat sans pouvoir tester l'application. Mais les avis étaient rassurants et j'avais déjà validé le fonctionnement local.

Avec Cryptomator, il est possible d’ouvrir le coffre directement depuis l’application mobile, connectée au service cloud. Cela permet d’accéder à ses fichiers depuis le téléphone de manière assez simple.

En revanche, il faut rester prudent avec les usages mobiles en écriture. Pour une consultation ponctuelle, le modèle est très pratique. Pour modifier régulièrement les fichiers depuis plusieurs appareils, je pense qu’il faut procéder avec prudence.

Je trouve l'application mobile assez simple et intuitive. Je n'ai qu'à sélectionner mon compte Google Drive (avec le dossier contenant la clé Cryptomator) et j'ai alors directement pu accéder à mes documents :

{{< two-images
    left="/images/cryptomator/cryptomator-stockage-mobile.png"
    altleft="Choix de la source de fichier cryptomator"
    right="/images/cryptomator/cryptomator-mobile.png"
    altright="Interface Cryptomator mobile"
>}}

J’avais également peur de devoir saisir mon mot de passe à chaque ouverture de l’application, mais les paramètres sont complets et ils ont intégré la biométrie, ce qui rend l’usage bien plus confortable au quotidien.

![Interface Cryptomator mobile](/images/cryptomator/cryptomator-settings.png)

## Pourquoi je préfère ce modèle sous Linux

Sur Linux, et en particulier sous Kubuntu, j’ai préféré **éviter un montage cloud distant comme base de travail**. Le modèle retenu me semble plus prévisible:
1. le coffre chiffré reste sur un vrai système de fichiers local
2. le point de travail principal reste local
3. le cloud sert de stockage synchronisé
4. la logique de sync reste explicite et maîtrisée

Cette approche est plus propre qu’un montage distant permanent, surtout avec un coffre Cryptomator contenant beaucoup de petits fichiers. Il y a aussi un point important: **la source de vérité reste locale**. Le cloud n’est pas mon point de travail principal.

---

## Mise en place détaillée sous Linux / Kubuntu

Cette partie correspond à ma mise en place actuelle sur **Kubuntu / KDE Plasma** avec :

- un **seul** coffre Cryptomator
- un stockage distant **Google Drive**
- `rclone bisync` pour la synchronisation
- une orchestration par scripts Bash + `systemd --user`

Si vous gérez plusieurs coffres, il faudra dupliquer ou paramétrer les scripts.

### Ce qui est spécifique à Kubuntu / Plasma

Voici les points à distinguer :

- la logique `systemd --user` fonctionne sur beaucoup de distributions Linux modernes, pas uniquement Kubuntu
- le chemin de montage par défaut du volume déchiffré Cryptomator (`~/.local/share/Cryptomator/mnt/...`) est le comportement standard Linux de Cryptomator, pas une particularité Kubuntu
- la détection de retour d’activité via **état lock/unlock de l’écran** repose ici sur **KDE / Plasma via D-Bus**. C’est le point le plus spécifique à l’environnement de bureau
- sous GNOME ou un autre environnement, il faudra probablement adapter la fonction `get_lock_state()` ou désactiver ce déclencheur.

### 1. Pré-requis

#### Paquets à installer

Sous Kubuntu, j’ai installé les outils suivants :

```bash
sudo apt update
sudo apt install fuse3 inotify-tools
```

`fuse3` est requis pour le montage FUSE de Cryptomator sous Linux. `inotify-tools` sert au watcher local qui détecte les changements dans le coffre chiffré.

#### Cryptomator

Installez Cryptomator desktop comme d’habitude sur votre machine. L’important ici n’est pas le mode d’installation, mais le fait d’obtenir un coffre local fonctionnel et montable.

#### rclone

Sur ma Kubuntu, le paquet `apt` était trop ancien pour un usage de `bisync`. J’ai donc installé une version plus récente de `rclone` depuis la distribution officielle du projet. Vérifiez la version :

```bash
rclone version
```

Pour ce montage, partez au minimum sur une version récente de `rclone` avec un `bisync` stable.

### 2. Créer l’accès Google Drive dans rclone

#### Étapes côté Google Cloud

Dans les grandes lignes :

1. créer ou sélectionner un projet dans Google Cloud
2. activer **Google Drive API**
3. configurer l’**OAuth consent screen**
4. créer un **OAuth Client ID** de type **Desktop app**
5. récupérer le **client ID** et le **client secret**
6. les injecter ensuite dans `rclone config` (voir ci-dessous)

#### Configuration de base dans rclone

Lancez ensuite :

```bash
rclone config
```

Créer un remote Google Drive, par exemple nommé `gdrive`, puis :

- choisir le backend `drive`
- renseigner le `client_id` et le `client_secret`
- choisir le scope adapté à votre besoin
- autoriser l’accès dans le navigateur
- terminer la configuration

Dans un cas simple, on obtient ensuite un remote du type :
```bash
gdrive:
```

Le fichier de configuration de `rclone` se trouve en général ici :
```text
~/.config/rclone/rclone.conf
```

### 3. Créer le coffre Cryptomator local

Créez le coffre dans un répertoire **local**, pas directement dans un montage cloud.

Exemple :

```text
/home/<utilisateur>/cryptomator/Documents
```

Après création, on obtient un répertoire chiffré contenant notamment :

- `vault.cryptomator`
- `masterkey.cryptomator`
- l’arborescence `d/`
- parfois `c/` selon les opérations de contrôle internes.

Le volume déchiffré, une fois ouvert, est monté côté Linux par défaut sous :

```text
~/.local/share/Cryptomator/mnt/<NomDuCoffre>
```

Dans mon cas :

```text
/home/max/.local/share/Cryptomator/mnt/Documents
```

### 4. Premier envoi vers Google Drive

Avant de passer à `bisync`, j’ai fait un premier envoi simple du coffre chiffré vers Google Drive.

```bash
rclone sync /home/<utilisateur>/cryptomator/Documents gdrive:Cryptomator/Documents -P
```

À adapter :
- `/home/<utilisateur>/cryptomator/Documents`
- `gdrive:Cryptomator/Documents`

### 5. Initialiser `bisync`

Une fois ce premier envoi terminé, je passe sur `rclone bisync` pour la suite.

Première étape : faire un `dry-run`.

```bash
rclone bisync /home/<utilisateur>/cryptomator/Documents gdrive:Cryptomator/Documents \
  --resync \
  --resync-mode path1 \
  --check-access \
  --check-filename vault.cryptomator \
  --compare size,modtime \
  --create-empty-src-dirs \
  --dry-run -vv
```

Si le résultat est cohérent, lancer la vraie initialisation :

```bash
rclone bisync /home/<utilisateur>/cryptomator/Documents gdrive:Cryptomator/Documents \
  --resync \
  --resync-mode path1 \
  --check-access \
  --check-filename vault.cryptomator \
  --compare size,modtime \
  --create-empty-src-dirs -vv
```

Ensuite, en routine, **ne plus utiliser `--resync`** sauf cas particulier.

### 6. Répertoire d’état local

Créer l’arborescence locale utilisée par les scripts :

```bash
mkdir -p ~/.local/bin
mkdir -p ~/.local/state/cryptomator-sync
mkdir -p ~/.config/systemd/user
```

### 7. Les chemins à adapter

Dans les scripts ci-dessous, adaptez au minimum ces variables :

```bash
LOCAL="/home/<utilisateur>/cryptomator/Documents"
REMOTE="gdrive:Cryptomator/Documents"
MOUNTPOINT_PATH="/home/<utilisateur>/.local/share/Cryptomator/mnt/Documents"
```

Si vous gardez un autre nom de coffre, un autre utilisateur ou un autre remote, il faut modifier ces trois chemins.

---

## Les scripts

### Script 1 : exécution réelle de la sync

Fichier :

```text
~/.local/bin/cryptomator-bisync-run.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# À adapter
LOCAL="/home/<utilisateur>/cryptomator/Documents"
REMOTE="gdrive:Cryptomator/Documents"

STATE_DIR="${XDG_STATE_HOME:-$HOME/.local/state}/cryptomator-sync"
LOG_FILE="$STATE_DIR/bisync.log"

RUN_LOCK="$STATE_DIR/run.lock"
PENDING_FILE="$STATE_DIR/pending"
LAST_SYNC_SUCCESS="$STATE_DIR/last_sync_success"
LAST_SYNC_ATTEMPT="$STATE_DIR/last_sync_attempt"
LAST_SYNC_REASON="$STATE_DIR/last_sync_reason"
LAST_RUN_EXIT="$STATE_DIR/last_run_exit"
CHANGE_DEBOUNCE_UNTIL="$STATE_DIR/change_debounce_until"
SYNC_RUNNING_FILE="$STATE_DIR/sync_running"
NEXT_SYNC_REASON="$STATE_DIR/next_sync_reason"

mkdir -p "$STATE_DIR"

exec 9>"$RUN_LOCK"
if ! flock -n 9; then
  exit 0
fi

touch "$SYNC_RUNNING_FILE"
trap 'rm -f "$SYNC_RUNNING_FILE"' EXIT

# Une sync démarre : l'ancien debounce n'a plus de sens.
rm -f "$CHANGE_DEBOUNCE_UNTIL"

if [ -f "$NEXT_SYNC_REASON" ]; then
  reason="$(cat "$NEXT_SYNC_REASON")"
  rm -f "$NEXT_SYNC_REASON"
else
  reason="${1:-manual}"
fi

timestamp() {
  date +%s
}

log() {
  printf '[%s] %s\n' "$(date --iso-8601=seconds)" "$*" >> "$LOG_FILE"
}

notify() {
  command -v notify-send >/dev/null 2>&1 || return 0
  notify-send "$1" "$2"
}

rotate_log() {
  [ -f "$LOG_FILE" ] || return 0
  local max_size=$((10 * 1024 * 1024))
  local size
  size=$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)
  if [ "$size" -ge "$max_size" ]; then
    mv -f "$LOG_FILE" "${LOG_FILE}.1"
  fi
}

rotate_log

echo "$(timestamp)" > "$LAST_SYNC_ATTEMPT"
echo "$reason" > "$LAST_SYNC_REASON"

log "START reason=$reason"

rc=0
if rclone bisync "$LOCAL" "$REMOTE" \
  --check-access \
  --check-filename vault.cryptomator \
  --compare size,modtime \
  --resilient \
  --recover \
  --max-lock 2m \
  --conflict-resolve newer \
  --create-empty-src-dirs \
  --log-level INFO \
  --stats 0 \
  >> "$LOG_FILE" 2>&1
then
  echo "$(timestamp)" > "$LAST_SYNC_SUCCESS"
  echo "0" > "$LAST_RUN_EXIT"
  log "OK reason=$reason"
else
  rc=$?
  echo "$rc" > "$LAST_RUN_EXIT"
  log "ERROR rc=$rc reason=$reason"
  notify "Cryptomator sync failed" "rclone bisync failed (rc=$rc). Check $LOG_FILE"
fi

if [ -f "$PENDING_FILE" ]; then
  log "PENDING detected after sync, keeping flag for agent"
fi

exit "$rc"
```

Rendre le script exécutable :

```bash
chmod +x ~/.local/bin/cryptomator-bisync-run.sh
```

### Script 2 : agent de décision

Fichier :

```text
~/.local/bin/cryptomator-sync-agent.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# À adapter
LOCAL="/home/<utilisateur>/cryptomator/Documents"
MOUNTPOINT_PATH="/home/<utilisateur>/.local/share/Cryptomator/mnt/Documents"

STATE_DIR="${XDG_STATE_HOME:-$HOME/.local/state}/cryptomator-sync"
LOG_FILE="$STATE_DIR/agent.log"

AGENT_LOCK="$STATE_DIR/agent.lock"
LAST_SYNC_SUCCESS="$STATE_DIR/last_sync_success"
LAST_LOCAL_CHANGE="$STATE_DIR/last_local_change"
LAST_VAULT_OPEN_STATE="$STATE_DIR/last_vault_open_state"
LAST_LOCK_STATE="$STATE_DIR/last_lock_state"
LAST_LOCK_SINCE="$STATE_DIR/last_lock_since"
CHANGE_DEBOUNCE_UNTIL="$STATE_DIR/change_debounce_until"
PENDING_FILE="$STATE_DIR/pending"
SYNC_RUNNING_FILE="$STATE_DIR/sync_running"
NEXT_SYNC_REASON="$STATE_DIR/next_sync_reason"

mkdir -p "$STATE_DIR"

exec 8>"$AGENT_LOCK"
if ! flock -n 8; then
  exit 0
fi

timestamp() {
  date +%s
}

read_ts() {
  local file="$1"
  if [ -f "$file" ]; then
    cat "$file"
  else
    echo 0
  fi
}

write_ts() {
  local file="$1"
  local value="$2"
  echo "$value" > "$file"
}

log() {
  printf '[%s] %s\n' "$(date --iso-8601=seconds)" "$*" >> "$LOG_FILE"
}

rotate_log() {
  [ -f "$LOG_FILE" ] || return 0
  local max_size=$((5 * 1024 * 1024))
  local size
  size=$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)
  if [ "$size" -ge "$max_size" ]; then
    mv -f "$LOG_FILE" "${LOG_FILE}.1"
  fi
}

rotate_log

now="$(timestamp)"
last_sync_success="$(read_ts "$LAST_SYNC_SUCCESS")"
last_local_change="$(read_ts "$LAST_LOCAL_CHANGE")"
debounce_until="$(read_ts "$CHANGE_DEBOUNCE_UNTIL")"
last_vault_open_state="$(read_ts "$LAST_VAULT_OPEN_STATE")"
last_lock_state="$(read_ts "$LAST_LOCK_STATE")"
last_lock_since="$(read_ts "$LAST_LOCK_SINCE")"

sync_running=0
if [ -f "$SYNC_RUNNING_FILE" ]; then
  sync_running=1
fi

run_sync() {
  local reason="$1"
  if [ "$sync_running" -eq 1 ]; then
    touch "$PENDING_FILE"
    log "sync already running -> mark pending, reason=$reason"
    exit 0
  fi
  log "launch sync, reason=$reason"
  echo "$reason" > "$NEXT_SYNC_REASON"
  systemctl --user start cryptomator-bisync.service --no-block
  exit 0
}

age_since_last_sync=$(( now - last_sync_success ))

# Retour d'activité : ici on utilise l'état lock/unlock de Plasma.
# Si cela ne fonctionne pas sur votre session, commentez ce bloc
# ou adaptez la fonction get_lock_state().
get_lock_state() {
  if command -v qdbus >/dev/null 2>&1; then
    qdbus org.freedesktop.ScreenSaver /ScreenSaver org.freedesktop.ScreenSaver.GetActive 2>/dev/null && return 0
    qdbus org.kde.screensaver /ScreenSaver org.freedesktop.ScreenSaver.GetActive 2>/dev/null && return 0
  fi
  return 1
}

lock_state_raw=""
if lock_state_raw="$(get_lock_state)"; then
  :
else
  lock_state_raw=""
fi

is_locked=0
case "$lock_state_raw" in
  true|True|1) is_locked=1 ;;
  false|False|0) is_locked=0 ;;
  *) is_locked=0 ;;
esac

if [ "$is_locked" -eq 1 ]; then
  if [ "$last_lock_state" -ne 1 ]; then
    write_ts "$LAST_LOCK_STATE" 1
    write_ts "$LAST_LOCK_SINCE" "$now"
    log "state -> screen locked"
  fi
else
  if [ "$last_lock_state" -eq 1 ]; then
    write_ts "$LAST_LOCK_STATE" 0
    locked_duration=$(( now - last_lock_since ))
    log "state -> screen unlocked after ${locked_duration}s"

    if [ "$locked_duration" -ge 1800 ] && [ "$age_since_last_sync" -gt 600 ]; then
      run_sync "return-activity"
    fi
  else
    write_ts "$LAST_LOCK_STATE" 0
  fi
fi

# Détection d'ouverture du vault déchiffré.
vault_open=0
if mountpoint -q "$MOUNTPOINT_PATH" 2>/dev/null; then
  vault_open=1
fi

if [ "$vault_open" -eq 1 ] && [ "$last_vault_open_state" -ne 1 ]; then
  write_ts "$LAST_VAULT_OPEN_STATE" 1
  log "vault opened"
  if [ "$age_since_last_sync" -gt 300 ]; then
    run_sync "vault-open"
  fi
elif [ "$vault_open" -eq 0 ] && [ "$last_vault_open_state" -ne 0 ]; then
  write_ts "$LAST_VAULT_OPEN_STATE" 0
  log "vault closed"
fi

# Rerun après une modif arrivée pendant la sync précédente.
if [ -f "$PENDING_FILE" ] && [ "$sync_running" -eq 0 ]; then
  if [ "$age_since_last_sync" -gt 120 ]; then
    rm -f "$PENDING_FILE"
    rm -f "$LAST_LOCAL_CHANGE" "$CHANGE_DEBOUNCE_UNTIL"
    log "pending rerun requested after previous sync; clearing local-change debounce state"
    run_sync "pending-rerun"
  fi
fi

# Sync différée après 5 minutes de calme.
if [ "$last_local_change" -gt 0 ] && [ "$debounce_until" -gt 0 ]; then
  if [ "$now" -ge "$debounce_until" ]; then
    if [ "$age_since_last_sync" -gt 300 ]; then
      rm -f "$CHANGE_DEBOUNCE_UNTIL" "$LAST_LOCAL_CHANGE"
      run_sync "local-change"
    fi
  fi
fi

# Filet périodique.
if [ "$age_since_last_sync" -gt 7200 ]; then
  run_sync "periodic-2h"
fi

log "no action"
exit 0
```

Rendre le script exécutable :

```bash
chmod +x ~/.local/bin/cryptomator-sync-agent.sh
```

### Script 3 : watcher local du coffre chiffré

Fichier :

```text
~/.local/bin/cryptomator-vault-watch.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# À adapter
LOCAL="/home/<utilisateur>/cryptomator/Documents"

STATE_DIR="${XDG_STATE_HOME:-$HOME/.local/state}/cryptomator-sync"
LOG_FILE="$STATE_DIR/watch.log"

LAST_LOCAL_CHANGE="$STATE_DIR/last_local_change"
CHANGE_DEBOUNCE_UNTIL="$STATE_DIR/change_debounce_until"
PENDING_FILE="$STATE_DIR/pending"
WATCHER_PID_FILE="$STATE_DIR/watcher.pid"
SYNC_RUNNING_FILE="$STATE_DIR/sync_running"

mkdir -p "$STATE_DIR"
echo $$ > "$WATCHER_PID_FILE"

timestamp() {
  date +%s
}

log() {
  printf '[%s] %s\n' "$(date --iso-8601=seconds)" "$*" >> "$LOG_FILE"
}

if ! command -v inotifywait >/dev/null 2>&1; then
  log "ERROR inotifywait not found. Install inotify-tools."
  exit 1
fi

log "watcher started on $LOCAL"

inotifywait -m -r \
  -e close_write,create,delete,move \
  --format '%w%f %e' \
  "$LOCAL" | while read -r path evt; do

    base="$(basename "$path")"

    case "$base" in
      .DS_Store|Thumbs.db)
        continue
        ;;
    esac

    # Ignorer le bruit technique de Cryptomator à l'ouverture du vault.
    if [[ "$path" == "$LOCAL/c" ]] || [[ "$path" == "$LOCAL/c/write-access"* ]]; then
      log "ignored cryptomator access-check path=$path evt=$evt"
      continue
    fi

    # Si une sync est en cours, on ne programme pas de local-change.
    # On demande juste un rerun à la fin.
    if [ -f "$SYNC_RUNNING_FILE" ]; then
      touch "$PENDING_FILE"
      log "change during sync path=$path evt=$evt -> pending only"
      continue
    fi

    now="$(timestamp)"
    echo "$now" > "$LAST_LOCAL_CHANGE"
    echo $((now + 300)) > "$CHANGE_DEBOUNCE_UNTIL"

    log "change detected path=$path evt=$evt debounce_until=$((now + 300))"
  done
```

Rendre le script exécutable :

```bash
chmod +x ~/.local/bin/cryptomator-vault-watch.sh
```

---

## Les unités systemd utilisateur

### Service de sync

Fichier :

```text
~/.config/systemd/user/cryptomator-bisync.service
```

```ini
[Unit]
Description=Cryptomator bisync to Google Drive
After=network-online.target

[Service]
Type=oneshot
ExecStart=%h/.local/bin/cryptomator-bisync-run.sh
```

### Service de l’agent de décision

Fichier :

```text
~/.config/systemd/user/cryptomator-sync-agent.service
```

```ini
[Unit]
Description=Cryptomator sync decision agent

[Service]
Type=oneshot
ExecStart=%h/.local/bin/cryptomator-sync-agent.sh
```

### Timer de l’agent

Fichier :

```text
~/.config/systemd/user/cryptomator-sync-agent.timer
```

```ini
[Unit]
Description=Run Cryptomator sync agent every minute

[Timer]
OnBootSec=1min
OnUnitActiveSec=1min
AccuracySec=10s
Persistent=true

[Install]
WantedBy=timers.target
```

### Service du watcher

Fichier :

```text
~/.config/systemd/user/cryptomator-vault-watch.service
```

```ini
[Unit]
Description=Watch Cryptomator vault for local changes

[Service]
Type=simple
ExecStart=%h/.local/bin/cryptomator-vault-watch.sh
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

### Activation

```bash
systemctl --user daemon-reload
systemctl --user enable --now cryptomator-vault-watch.service
systemctl --user enable --now cryptomator-sync-agent.timer
```

### Déclenchement manuel

Ajoutez par exemple dans `~/.bashrc` :

```bash
alias sync-cryptomator='systemctl --user start cryptomator-bisync.service'
```

Rechargez ensuite votre shell :

```bash
source ~/.bashrc
```

---

## Comment ça fonctionne en pratique

Le comportement retenu est volontairement adapté à mon cas d'usage, avec peu de modification attendu côté mobile et des délais de synchronisation qui me semblent cohérents :

- **sync filet périodique** toutes les 2 heures
- **sync immédiate** à l’ouverture du vault si la dernière sync a plus de 5 minutes
- **sync immédiate** au retour d’activité si l’écran est resté verrouillé au moins 30 minutes et si la dernière sync a plus de 10 minutes
- **sync différée** après 5 minutes de calme si un fichier du coffre chiffré change
- **rerun unique** si une modification arrive pendant une sync déjà en cours.

Ce n’est pas du temps réel. C’est un compromis entre lisibilité, maîtrise et coût en appels API.

---

## Debug et journaux

### Logs applicatifs

Les journaux principaux sont ici :

```text
~/.local/state/cryptomator-sync/bisync.log
~/.local/state/cryptomator-sync/agent.log
~/.local/state/cryptomator-sync/watch.log
```

### Commandes utiles

```bash
tail -f ~/.local/state/cryptomator-sync/bisync.log
tail -f ~/.local/state/cryptomator-sync/agent.log
tail -f ~/.local/state/cryptomator-sync/watch.log
```

```bash
journalctl --user -u cryptomator-vault-watch.service -f
journalctl --user -u cryptomator-sync-agent.service -f
```

```bash
systemctl --user status cryptomator-vault-watch.service
systemctl --user status cryptomator-sync-agent.timer
systemctl --user list-timers
```

---

## Les limites de cette approche

Cette solution est intéressante, mais elle n’est pas parfaite.

**1. Ce n’est pas une synchronisation temps réel native sous Linux**

Le comportement dépend de la manière dont on a construit le workflow. Sous Linux, avec `rclone`, on est sur une logique de synchronisation périodique ou manuelle.

**2. Il faut rester discipliné sur les modifications simultanées**

Je n'ai pas poussé mes tests sur ce point, mais si un coffre est modifié à plusieurs endroits en même temps, par exemple sur le PC principal et sur mobile, on peut introduire des conflits ou des comportements imprévisibles.

Ce n’est pas forcément un problème si l’usage mobile reste ponctuel, mais il faut l’avoir en tête.

**3. La confiance ne disparaît pas totalement**

Le fournisseur cloud ne voit plus le contenu lisible des fichiers, mais il voit encore :
- qu’il existe des fichiers
- leur volume approximatif
- leur fréquence de modification
- certaines métadonnées de stockage

On améliore donc fortement la confidentialité du contenu, mais on ne devient pas invisible pour autant.

**4. La sauvegarde reste un sujet à part entière**

Une synchronisation cloud n’est pas un backup au sens strict. Si un fichier est supprimé ou corrompu localement, cette erreur peut aussi être répliquée.
Il faut donc compléter le dispositif par une vraie stratégie de sauvegarde indépendante. Je ne me suis pas encore penché sérieusement sur ce sujet. Pour le moment, mon ancien volume VeraCrypt local me sert encore de sauvegarde complémentaire.

**5. Une dépendance existe aux évolutions logicielles**

Cette approche reste dépendante de l’évolution de plusieurs outils : Cryptomator, rclone, l’application mobile, ainsi que du service cloud utilisé.

Comme toujours, cela implique un certain niveau de dépendance technique dans le temps. Pour autant, je n’ai constaté à ce stade aucun problème particulier de fiabilité. Et le fait que les briques principales soient **open source** réduit en partie le risque : le fonctionnement est plus transparent et la dépendance à un acteur unique est plus limitée.

**6. Cette version gère un seul coffre**

Les scripts ci-dessus partent du principe qu’il n’y a **qu’un seul volume chiffré Cryptomator**. Pour plusieurs coffres, il faudra dupliquer les scripts/services ou les paramétrer proprement.

# Mon avis après test

Ce montage n’est pas magique, mais je le trouve très cohérent pour obtenir un bon compromis autour d’un stockage cloud plus sécurisé.

Il permet :
- de conserver le confort d’un stockage cloud
- d’avoir une vraie couche de chiffrement côté client
- d’éviter les limites pratiques d’un gros volume VeraCrypt
- de rester sur une solution relativement simple et peu coûteuse
- et de retrouver un accès mobile raisonnable aux documents sensibles.

VeraCrypt reste excellent pour de l’archive, du stockage statique ou des volumes hors ligne. En revanche, je n’avais pas trouvé avec VeraCrypt un modèle vraiment pratique pour répondre à mes besoins de sauvegarde cloud et d’accès mobile.

# Conclusion

Si vous stockez aujourd’hui des documents sensibles dans un cloud classique en vous fiant uniquement au mot **« chiffré »**, il vaut probablement la peine de regarder de plus près qui détient réellement les clés.

Dans mon cas, ajouter une couche de chiffrement côté client m’a permis de conserver la praticité du cloud, sans lui abandonner l’accès direct au contenu de mes fichiers.

Ce n’est pas une solution parfaite. Elle demande un peu de méthode, un peu de discipline, et une bonne compréhension de ce qu’est réellement une synchronisation. Mais pour mon besoin, c’est aujourd’hui l’un des meilleurs compromis entre **confidentialité, simplicité et coût**.

Ce retour d’expérience vaut surtout pour mon besoin précis et dans la limite des recherches que j’ai menées jusqu’à présent.
