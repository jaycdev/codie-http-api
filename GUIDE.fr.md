# Codie 1.6.0 HTTP API — Guide d'intégration bêta

Ce guide s'adresse aux testeurs qui se connectent à un appareil Codie sous 1.6.0 depuis un script, une application de bureau ou une application mobile sur le réseau local. Il décrit le code source de la branche master. Il ne signifie ni que tous les appareils ont reçu cette version, ni que toutes les opérations ont été testées sur appareil.

## Utiliser le paquet documentaire

Décompressez `Codie-HTTP-API-1.6.0.zip` et ouvrez `index.html`. Consultez hors ligne toutes les API publiques, leurs paramètres, structures de requête et de réponse et autorisations. Le nombre d’opérations et de chemins figure dans `manifest.json`. Aucun serveur ni accès à la page Laboratoire de l’appareil n’est nécessaire.

| Fichier | Utilisation |
| --- | --- |
| `index.html` | Référence complète hors ligne, avec recherche et impression depuis le navigateur |
| `guide.zh-CN.html` / `GUIDE.zh-CN.md` | Guide en chinois |
| `guide.ja.html` / `GUIDE.ja.md` | Guide en japonais |
| `guide.en.html` / `GUIDE.en.md` | Guide en anglais |
| `guide.de.html` / `GUIDE.de.md` | Guide en allemand |
| `guide.fr.html` / `GUIDE.fr.md` | Guide en français |
| `openapi.json` | Description OpenAPI 3.1 en anglais par défaut ; les outils API doivent implémenter séparément la signature HMAC |
| `openapi.zh-CN.json` | OpenAPI en chinois ; chemins, champs, énumérations et signature identiques au contrat anglais |
| `examples/local_api.py` | Association et chat ; le mode texte utilise la bibliothèque standard Python 3, les dépendances du micro sont indiquées dans le script |
| `examples/external_tools.py` | Enregistrer et connecter des outils externes, puis renvoyer les résultats des appels |
| `examples/media_session.py` | Sessions temporaires d’images, d’audio et de vidéo |
| `manifest.json` | Version de l'appareil, commit source, date de génération et empreintes SHA-256 |

Les cinq guides sont fournis en HTML et en Markdown. Sélectionnez la langue en haut de `index.html`. La référence complète utilise une navigation en chinois et des descriptions en anglais par défaut. OpenAPI est fourni en anglais dans `openapi.json` et en chinois dans `openapi.zh-CN.json`. Les chemins, noms de champs, valeurs d'énumération et exemples de code restent identiques entre les langues.

Dans OpenAPI, `info.version=1` désigne la version existante de la description d'API. La version logicielle de l'appareil, 1.6.0, figure séparément dans `x-device-version` et le manifeste.

## 1. Préparer l'appareil

1. Vérifiez que l'appareil utilise la version 1.6.0 et qu'un compte utilisateur y est connecté.
2. Reliez l'ordinateur et l'appareil à un réseau local permettant leur communication, et relevez l'adresse IP de l'appareil. L'adresse par défaut est `http://DEVICE_IP:8081` ; utilisez le port réel s'il a été modifié.
3. Réveillez et déverrouillez l'appareil avant une conversation. Les requêtes utilisent l'utilisateur et l'emplacement de personnage actuellement actifs sur l'appareil.
4. Vérifiez la connexion, puis lancez un exemple pour effectuer l'appairage.

```sh
curl http://DEVICE_IP:8081/v1/api/auth/ping
```

Le chemin de base HTTP est `/v1/api`. Exemple d'URL complète : `http://DEVICE_IP:8081/v1/api/data/user/system`. Pour les scripts d'exemple, indiquez uniquement le schéma, l'hôte et le port, sans `/v1/api`.

## 2. Essayer une conversation textuelle

Depuis le répertoire racine du paquet décompressé, exécutez l’exemple Python :

```sh
python examples/local_api.py http://DEVICE_IP:8081 "Bonjour" --credentials credentials.json
```

Au premier lancement, saisissez dans le terminal le code d'appairage affiché sur l'appareil. L'exemple enregistre les identifiants, lit l'emplacement actif, crée un tour de conversation textuel, interroge son état et affiche la réponse du personnage. Les lancements suivants réutilisent le même fichier d'identifiants.

Pour lire la réponse sur l'appareil, ajoutez `--speech device` :

```sh
python examples/local_api.py http://DEVICE_IP:8081 "Bonjour" --credentials credentials.json --speech device --timeout 180
```

Par défaut, `--speech none` désactive la lecture vocale et le délai d'attente est de 120 secondes. Avec `device`, `completed` signifie que la génération et la lecture sur l'appareil sont terminées. Un délai client dépassé n'annule pas le tour côté serveur. Son identifiant est affiché pour permettre de poursuivre les consultations.

Les exemples demandent explicitement des identifiants `session`, valables 12 heures et invalidés au redémarrage du service de l'appareil. Pour les identifiants longue durée existants, ajoutez `--auth-scope permanent` lors du premier appairage. Les identifiants session permettent les API utilisateur publiques ; les API administrateur exigent permanent. Les restrictions de fonctionnalités restent applicables. Les fichiers existants ne sont ni écrasés ni convertis automatiquement : choisissez un nouveau nom lors d'un nouvel appairage. Conservez ces identifiants de manière confidentielle ; ne les joignez ni aux rapports ni au paquet redistribué.

## 3. Implémenter l'appairage

Seules ces cinq opérations HTTP publiques ne nécessitent pas de HMAC :

| Méthode | Chemin avec le préfixe de base | Utilisation |
| --- | --- | --- |
| GET | `/v1/api/auth/ping` | Vérifier la connexion |
| GET | `/v1/api/auth/bootstrap` | Obtenir le `user_id` actuel, l'état de connexion et les langues |
| POST | `/v1/api/auth/pair/request` | Afficher un code ; renvoie `challenge_id` ; demande valable 5 minutes |
| POST | `/v1/api/auth/pair/confirm` | Soumettre le code et obtenir les identifiants |
| POST | `/v1/api/auth/pair/cancel` | Annuler une demande d'appairage |

Corps JSON de confirmation :

```json
{
  "challenge_id": "valeur renvoyée par pair/request",
  "pairing_code": "code affiché sur l'appareil",
  "auth_scope": "session"
}
```

`auth_scope` accepte `session` ou `permanent`. En l'absence de valeur, l'API choisit `permanent` ; les exemples indiquent explicitement `session`. Enregistrez `data.local_secret`, `data.local_session_id`, `data.local_session_expires_at_ms`, `data.auth_scope`, ainsi que `data.user_id` obtenu via bootstrap. Les scripts et applications natives peuvent omettre `Origin/Referer` ; les navigateurs restent soumis aux contrôles d'origine existants de l'appareil.

## 4. Signer chaque requête métier

Concaténez les valeurs dans cet ordre avec `|`, sans saut de ligne final :

```text
METHOD|PATH|TIMESTAMP|NONCE|PAYLOAD_MODE|BODY_HASH|USER_ID
```

```text
signature = hex_lower(HMAC_SHA256(UTF8(local_secret), UTF8(canonical)))
```

| En-tête | Valeur |
| --- | --- |
| `X-Local-Sig` | Signature de 64 caractères hexadécimaux minuscules |
| `X-Local-Timestamp` | Horodatage Unix en secondes, sous forme de chaîne ; écart autorisé de 300 secondes avec l'appareil |
| `X-Local-Nonce` | Nouvelle valeur aléatoire à chaque requête, UUID recommandé ; 1 à 128 caractères, sans barre verticale ni saut de ligne |
| `X-Local-Payload-Mode` | Mode indiqué ci-dessous |
| `X-Local-Body-Hash` | Empreinte indiquée ci-dessous |
| `X-Local-User-Id` | `user_id` actuel renvoyé par bootstrap |
| `X-Local-Session-Id` | Obligatoire si l'appairage a renvoyé un `local_session_id` non vide |

`METHOD` est en majuscules. `PATH` est le chemin après décodage URL, incluant `/v1/api`, sans hôte ni chaîne de requête. Utilisez comme clé les octets UTF-8 de la chaîne originale `local_secret`, sans décodage hexadécimal. Signez exactement les octets envoyés. Ne reformatez pas le JSON après la signature.

| Corps | PAYLOAD_MODE | BODY_HASH |
| --- | --- | --- |
| Aucun corps | `empty` | SHA-256 des octets vides, en hexadécimal minuscule |
| JSON | `json` | SHA-256 des octets JSON réellement envoyés, en hexadécimal minuscule |
| Formulaire de fichiers | `multipart` | Chaîne fixe `UNSIGNED-PAYLOAD` |
| Bloc compressé zstd | `zstd-stream` | SHA-256 des octets compressés réels, en hexadécimal minuscule |
| Bloc compressé gzip | `gzip-stream` | SHA-256 des octets compressés réels, en hexadécimal minuscule |

Empreinte du corps vide : `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`. Les blocs compressés utilisent `application/octet-stream`, uniquement pour les API qui les prennent explicitement en charge. Laissez la bibliothèque HTTP générer le `Content-Type` multipart et sa boundary.

## 5. API de conversation unifiée

| Méthode | Chemin relatif à `/v1/api` | Utilisation |
| --- | --- | --- |
| POST | `/data/user/chat/turns` | Créer un tour de conversation |
| GET | `/data/user/chat/turns/{turn_id}` | Obtenir l'état, la transcription et la réponse |
| POST | `/data/user/chat/turns/{turn_id}/input/complete` | Terminer l'entrée du microphone de l'appareil et poursuivre la génération ; corps vide |
| POST | `/data/user/chat/turns/{turn_id}/cancel` | Annuler l'entrée, la génération et la lecture de ce tour ; corps vide |

Commencez par lire `data.slot_number` via `GET /data/user/slot`. Pour chaque nouveau tour métier, utilisez un nouvel UUID pour `request_id` et remplacez `slot` par l'emplacement actif :

```json
{
  "request_id": "3f743580-bd45-4c24-96c0-61a67ac5f741",
  "slot": 1,
  "input": {"type": "text", "text": "Bonjour"},
  "output": {"speech": "none"}
}
```

| Entrée | Content-Type | Contenu |
| --- | --- | --- |
| `text` | `application/json` | `input.text` obligatoire et non vide après suppression des espaces aux extrémités ; 16000 caractères maximum |
| `device_microphone` | `application/json` | Enregistrement sur l'appareil ; omettre `text` ; terminer avec `input/complete` |
| `image_text` | `multipart/form-data` | `request` contient la configuration en chaîne JSON, avec `input.type=image_text` et `text` facultatif ; champ fichier : `image` |
| `audio_file` | `multipart/form-data` | `request` contient la configuration en chaîne JSON, avec `input.type=audio_file`, sans `text` ; champ fichier : `audio` |

Les fichiers image ou WAV sont limités à 1 MiB. Le WAV doit être PCM16, 16 kHz, mono et durer au maximum 30 secondes. Le champ obligatoire `output.speech` accepte `none`, `device` ou `client`. Les champs inconnus sont refusés.

Le champ facultatif `interrupt` vaut `true` par défaut : si l'appareil est en conversation, parle ou exécute une tâche au démarrage du tour, cette session est d'abord terminée. Avec `false`, rien n'est interrompu : le tour se termine en `failed` avec `error.code=DEVICE_BUSY` (`retryable=true`) et son message utilisateur est retiré de l'historique ; réessayez plus tard.

Le champ facultatif `retention` fixe ce que le tour laisse sur l'appareil et vaut `full` par défaut : il est enregistré dans l'historique et met à jour l'état du dialogue, le résumé, la mémoire à long terme et le journal (tous extraits de l'historique du tour). `chat` l'enregistre dans l'historique sans ces mises à jour. `none` ne laisse rien : ni historique, ni image enregistrée, ni mise à jour ; la réponse figure seulement dans `assistant_text` du tour, avec `user_chat_ids` et `assistant_chat_ids` vides. Quelle que soit la valeur, la réponse reste générée à partir de l'historique et de la mémoire existants.

Le champ facultatif `tools` indique si le modèle peut appeler des outils pendant ce tour et vaut `all` par défaut : le modèle peut appeler les outils de l’appareil et les outils externes disponibles dans la session. Avec `none`, aucun outil n’est proposé au modèle pour ce tour, ce qui convient aux tours qui n’attendent qu’une réponse et ne doivent déclencher ni action de l’appareil ni tâche externe.

`input.type=audio_stream` utilise JSON avec `audio.format=pcm_s16le`, `sample_rate=16000`, `channels=1`, `container=raw`, pendant 30 secondes maximum. Pour une entrée en flux ou `speech=client`, connectez `/v1/ws/chat/turns/{turn_id}/audio` par un GET WebSocket signé dans les 15 secondes et attendez `ready`. La sortie est en MP3, WAV ou Ogg Opus selon les métadonnées. Après `audio.end`, envoyez `audio.ack` avec la séquence finale pour confirmer la réception, sans indiquer la fin de lecture. La déconnexion termine la tâche ; reconnexion et répétition ne sont pas prises en charge. Voir OpenAPI pour les formats des messages.

```sh
python -m pip install websocket-client sounddevice av
python examples/local_api.py http://DEVICE_IP:8081 --mic --seconds 10 --credentials credentials.json --timeout 180
```

Lisez `data.turn_id` dans la réponse d'acceptation et consultez le tour, par exemple une fois par seconde. Les états terminaux sont `completed`, `failed` et `cancelled`. Lisez `data.assistant_text` à la fin ou `data.error` en cas d'échec. `transcript` contient la transcription ; `user_chat_ids` et `assistant_chat_ids` relient les enregistrements d'historique. Un état intermédiaire n'est pas un succès final.

Les autres états sont `queued` (en attente), `starting` (ouverture de session), `receiving_input` (microphone ou flux audio), `thinking` (attente de génération), `responding` (génération ou livraison) et `cancelling` (annulation en cours). `input_complete` termine uniquement l’entrée ; `transcript` peut arriver avant un état terminal. `device_microphone` termine automatiquement l’entrée après 30 secondes au maximum ; `input/complete` permet de terminer plus tôt. `speech=none` exécute toujours le modèle sans lecture vocale ; `retention` détermine si la conversation est conservée. Consultez `/data/user/device/state` avant chaque requête pour vérifier les conditions courantes.

`202` signifie seulement accepté. Pendant la période de conservation, les mêmes identifiants, `request_id` et contenu renvoient le tour original ; un contenu différent produit `409 REQUEST_ID_CONFLICT`. Chaque nouvelle tentative HTTP nécessite tout de même un nouveau nonce de signature. Avec `interrupt=true`, un nouveau tour interrompt l’ancienne session ; la répétition idempotente ne déclenche pas de nouvelle interruption.

Les tours terminés sont conservés 30 minutes, avec une limite de 256 tours en mémoire. Le redémarrage du service efface les tours et les données d'idempotence. Un changement de personnage ou d'utilisateur, des identifiants invalides ou un historique effacé peuvent arrêter les anciens tours ou les rendre inaccessibles. L'annulation concerne uniquement le tour désigné créé avec ces identifiants.

### Synthèse vocale indépendante (TTS)

Faites lire directement par le personnage actuel un texte déjà déterminé par une application externe, par exemple le résultat d’une action Home Assistant. Cette API n’appelle pas de LLM, ne génère pas de réponse du personnage et n’écrit pas dans l’historique des conversations.

Envoyez le JSON à `POST /data/user/tts/utterances`. Les chemins sont relatifs à `/v1/api` ; appliquez la signature HMAC décrite plus haut :

```json
{
  "request_id": "e14eec98-5f67-445f-b605-bcc0b23f8f75",
  "slot": 1,
  "text": "La lumière du salon est éteinte."
}
```

Interrogez `GET /data/user/tts/utterances/{utterance_id}`. Pour annuler : `POST /data/user/tts/utterances/{utterance_id}/cancel`, avec un corps vide.

202 indique seulement l’acceptation ; récupérez `data.utterance_id`. États : `queued`, `synthesizing`, `playing`, `cancelling` ; états terminaux : `completed`, `failed`, `cancelled`. completed exige la fin de la synthèse et de la lecture sur l’appareil. Les erreurs sont dans `data.error`. text doit contenir 1–16000 caractères après suppression des espaces aux extrémités ; corps JSON limité à 128 KiB. Le slot actuel et la voix, la langue et le volume de son personnage sont utilisés. L’appareil doit être éveillé et déverrouillé. Une nouvelle lecture interrompt la session au premier plan ; une nouvelle conversation peut interrompre la lecture. Il n’y a pas de file d’attente.

Une répétition identique avec les mêmes identifiants et request_id renvoie la tâche initiale (200) ; un contenu différent donne 409. L’espace request_id et les 256 entrées de cache sont partagés avec les conversations. Conservation des états terminaux : 30 minutes ; un redémarrage les efface. Des identifiants invalides, un changement d’utilisateur/personnage ou l’effacement de l’historique peuvent arrêter ou invalider les tâches. Durée maximale : 180 secondes. Aucun fichier ni flux audio n’est renvoyé.

```sh
python examples/local_api.py http://DEVICE_IP:8081 "La lumière du salon est éteinte." --credentials credentials.json --tts --timeout 180
```

`playing` nécessite la confirmation du démarrage de la lecture par l’appareil pour le flux courant. Si les animations sont activées, une animation Talking est choisie aléatoirement. L’annulation renvoie 202 si elle est acceptée, 200 si elle est déjà en cours ou terminée, et 409 si la tâche est terminée ou a échoué. Un délai d’attente dépassé côté client n’annule pas automatiquement la tâche sur l’appareil.

## 6. Autres fonctions et confirmation du résultat

Consultez la référence complète hors ligne et OpenAPI pour les chemins, champs et réponses exacts.

| Catégorie | Fonctions |
| --- | --- |
| Accès et appairage | Connexion, appairage, annulation |
| Appareil et état | Informations système, autorisations, état appareil/conversation, langue IA |
| Personnage et interaction proactive | Propriétés, emplacements, import/export de modèles et tenues, chargement, réinitialisation, réglages proactifs |
| Conversation et historique | Tours, consultation/suppression d'historique, images |
| Mémoire et journal | Consultation et modification |
| Alarmes et notes | Gestion des ressources, activation/désactivation des alarmes |
| Musique | Envoi, gestion de fichiers, lecture et état |
| Voix et son | Autorisations, synthèse vocale indépendante, écoute d’un texte fixe, contrôle, clonage vocal |
| Éclairage | Projets, aperçu, compilation, lecture, application, interrupteur général |
| Services IA | Fournisseurs, profils et tests, routage des capacités, prompt système, changement d’architecture/backend, dernière erreur d’exécution |

L'appairage n'ajoute aucune autorisation fonctionnelle. Lisez `capabilities` dans `/data/user/system` ; chaque opération précise ses `x-capabilities`. La page de documentation API du Laboratoire et la configuration des services IA sont accessibles à tous les profils fonctionnels. L'authentification existante par signature après appairage reste applicable.

Une réponse positive peut signifier enregistrement ou mise en file d'attente, sans que l'action soit terminée. Consultez le tour pour une conversation, l'état de tâche/chargement pour l'envoi, l'application ou l'export d'un modèle, l'état de lecture pour la musique et `/data/user/tts_audition/state` pour l'écoute vocale. L'écoute accepte seulement `voice_id` et un `group_key` facultatif. Elle utilise un texte fixe dans la langue IA actuelle, et n'accepte pas de texte libre à prononcer.

Fichier de modèle : 200 MiB maximum. Fichier de clonage vocal : 10 MiB maximum. Envoi groupé de musique : 200 MiB et 64 fichiers maximum par requête. Corps signés ordinaires et blocs compressés : 8 MiB maximum. La taille totale multipart comprend aussi les champs ; consultez les limites de chaque API.

Canal d'état temps réel facultatif : `/v1/ws/events`. Utilisez une négociation GET signée avec les paramètres `sig`, `timestamp`, `nonce`, `payload_mode=empty`, `body_hash`, `user_id` et `session_id` s'il n'est pas vide. Signez `/v1/ws/events` sans sa chaîne de requête. Ce canal transmet des instantanés d'état, pas le texte de conversation généré en flux. Des identifiants invalides ou un changement d'utilisateur, entre autres, ferment la connexion.

Après `ready`, envoyez `{"type":"subscribe","request_id":"sub-1","data":{"events":["chat_history","chat_turn"]}}` et attendez un `result` réussi. `events` remplace tous les abonnements ; un tableau vide les annule. Après reconnexion, réabonnez-vous puis restaurez l’état via HTTP. Le contrat public ne comprend que les fonctionnalités ouvertes dans les versions Release avec le tag RELEASE. Les descriptions OpenAPI sont en anglais par défaut.

## 7. Dépannage et signalement

| Symptôme ou code | Action |
| --- | --- |
| Connexion impossible | Vérifier IP, port et réseau local ; commencer par ping |
| `401 REQUEST_EXPIRED` | Synchroniser les horloges ; générer un nouvel horodatage et une nouvelle signature |
| `401 NONCE_REUSED` | Utiliser un nouveau nonce pour chaque requête, y compris les consultations et nouvelles tentatives |
| `401 LOCAL_SESSION_EXPIRED` | Refaire l'appairage avec un nouveau nom de fichier d'identifiants |
| `401 LOCAL_USER_MISMATCH` | Vérifier l'utilisateur actuel ; récupérer bootstrap et refaire l'appairage |
| `401 LOCAL_SECRET_MISMATCH` / `INVALID_SIGNATURE` | Vérifier le secret original, le chemin signé, les octets du corps et les en-têtes |
| `403` | Vérifier les erreurs et autorisations de l'appareil ; l'appairage ne les contourne pas |
| `409 SLOT_MISMATCH` | Relire l'emplacement actif |
| `409 DEVICE_BLOCKED` / `AMBIENT_MODE_BLOCKED` | Lire `reason` ; réveiller/déverrouiller l'appareil et vérifier si le mode actuel autorise la conversation |
| `413` / `415` | Vérifier taille, Content-Type, types d'entrée/sortie et format WAV |
| `503` | Selon le code, vérifier si le service est prêt ou si la limite de tours est atteinte |

Dans OpenAPI, `code.enum` énumère les erreurs HTTP locales des conversations, TTS, alarmes, notes et enregistrements de clonage. `x-error-codes.http` fournit la liste machine ; `x-error-codes.task` les erreurs asynchrones connues, extensibles par les fournisseurs. Après HTTP 202, lisez `data.error`. Une origine refusée renvoie `403 PAIRING_SOURCE_NOT_ALLOWED`, un conflit d’idempotence `409 REQUEST_ID_CONFLICT` et une note absente `404 MEMO_NOT_FOUND`.

`x-error-codes` couvre aussi les erreurs connues des fournisseurs IA, tâches de modèle, réinitialisations de personnage, mémoires et musique. Les anciens identifiants du journal figurent dans `legacy_error` et dans le champ de réponse `error`. Les erreurs de tâche incluent `NO_SPEECH` ; `DEVICE_INTERRUPTED.reason=vad_start` indique un début de parole détecté. reason reste extensible.

Les structures d'erreur varient selon les fonctions. Vérifiez d'abord le statut HTTP, puis les champs documentés tels que `code`, `message` et `reason`. Ne répétez pas aveuglément les opérations d'écriture.

Dans un signalement, indiquez la version de l'appareil, le commit source de `manifest.json`, l'heure de l'incident et le fuseau de l'appareil, la méthode et le chemin, le statut HTTP, la réponse expurgée, les `request_id` / `turn_id` concernés et le comportement observé. Ne transmettez pas `local_secret`, de fichiers d'identifiants ou d'en-têtes de signature.

Les API d'administration, l'audio de débogage, les journaux diagnostiques et la maintenance interne sont hors de ce contrat public. Ce paquet vérifie le contrat au niveau du code source. Les compilations, appels sur appareil et parcours cloud de bout en bout nécessitent une validation distincte.
