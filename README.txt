Fonctionnement :

Ecrire reboot dans reboot.txt pour forcer le redémarrage
Après redémarrage forcé, le vider pour réinitialiser le tag empêchant un bootloop.
La vérification a lieu toutes les 5 minutes.


Le code est le suivant :


#!/bin/bash

URL="https://raw.githubusercontent.com/bacomino/reboot/refs/heads/main/reboot.txt"
TMP_DIR="/volume2/TMP"
LOCK_FILE="$TMP_DIR/github_reboot.lock"
DEBUG_LOG="/volume2/TMP/debug_github.log"

# 1. Création du dossier si nécessaire
mkdir -p "$TMP_DIR"

# 2. Récupération des données (nettoyées des sauts de ligne)
CONTENT=$(curl -s "$URL" | tr -d '\r' | tr -d '\n' | tr -d ' ')
ETAG=$(curl -sI "$URL" | grep -i '^etag:' | tr -d '\r' | awk '{print $2}')

echo "$(date '+%Y-%m-%d %H:%M:%S') | Lu: '$CONTENT' | ETag: '$ETAG'" >> "$DEBUG_LOG"

# ==========================================
# PHASE 1 : INITIALISATION SILENCIEUSE
# ==========================================
# Si le fichier lock n'existe pas, on le crée avec l'ETag actuel et on quitte le script.
# AUCUN redémarrage ne sera déclenché lors de ce premier passage.
if [ ! -f "$LOCK_FILE" ]; then
    echo "$ETAG" > "$LOCK_FILE"
    echo "=> INITIALISATION : Création du verrou. Aucun redémarrage." >> "$DEBUG_LOG"
    exit 0
fi

# ==========================================
# PHASE 2 : VERIFICATION ET REDEMARRAGE
# ==========================================
# Le script ne passe ici que si le fichier lock existait déjà (2ème passage et suivants).
if [ "$CONTENT" == "reboot" ]; then
    
    # Si l'ETag sur GitHub est différent de celui stocké localement
    if [ "$(cat "$LOCK_FILE")" != "$ETAG" ]; then
        
        # Mise à jour du verrou avec le nouvel ETag
        echo "$ETAG" > "$LOCK_FILE"
        echo "=> NOUVEL ETAG DETECTE : REDEMARRAGE DECLENCHE" >> "$DEBUG_LOG"
        
        reboot
    else
        echo "=> ETag identique. Pas de reboot." >> "$DEBUG_LOG"
    fi
else
    echo "=> Mot 'reboot' absent. Pas d'action." > "$DEBUG_LOG"
fi
