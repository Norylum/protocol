# Norylum Protocol

Contrats d'interface d'API, spécifications de protocoles réseau et schémas canoniques de l'écosystème open-source **Norylum** (serveur média personnel auto-hébergé).

Ce dépôt constitue la **source unique de vérité** (SSOT - *Single Source of Truth*) pour l'ensemble des clients officiels (Desktop, Android, Apple, Web) et des extensions tierces développées pour Norylum.

---

## Sommaire

1. [Architecture & Principes Directeurs](#1-architecture--principes-directeurs)
2. [En-têtes de Contexte & Sécurité Fail-Closed](#2-en-têtes-de-contexte--sécurité-fail-closed)
3. [Spécification OpenAPI 3.0.3 (`openapi.yaml`)](#3-spécification-openapi-303-openapiyaml)
4. [Streaming Direct Play & HTTP Range 206](#4-streaming-direct-play--http-range-206)
5. [Spécification des Plugins (`schemas/plugin-manifest.schema.json`)](#5-spécification-des-plugins-schemasplugin-manifestschemajson)
6. [Instructions de Validation](#6-instructions-de-validation)

---

## 1. Architecture & Principes Directeurs

Conformément à l'**ADR-001** de l'écosystème Norylum, le backend évolue d'une architecture microservices distribuée vers un **Monolithe Modulaire Rust** haute performance et économe en ressources.

Le présent protocole définit :
- Les interfaces REST standardisées exposées par le binaire serveur (`apps/norylum-server`).
- Les mécanismes de contrôle d'accès multi-profils et de protection parentale.
- Le protocole de streaming vidéo *Direct Play* respectant une empreinte mémoire constante en RAM ($O(1)$).
- Le format d'extension et le système de permissions pour les plugins.

---

## 2. En-têtes de Contexte & Sécurité Fail-Closed

Toutes les routes manipulant des données privées (bibliothèque, streaming, progression de lecture) imposent la présence d'en-têtes HTTP de contexte.

### Politique Fail-Closed

Le serveur applique une politique stricte de **refus par défaut** (*Fail-Closed*) :
- Toute requête sans contexte d'authentification valide est immédiatement rejetée avec le code **`401 Unauthorized`**.
- Tout profil enfant tentant d'accéder à un média adulte, non classé *child-safe* ou dépassant la limite d'âge autorisée est immédiatement bloqué avec le code **`403 Forbidden`**.
- Toute tentative d'évasion d'arborescence (Directory Traversal, symlinks non autorisés) est interceptée et bloquée avec le code **`403 Forbidden`**.

### Référentiel des En-têtes

| En-tête | Type | Requis | Description | Exemple |
|---|---|---|---|---|
| `x-account-id` | `string` | **Oui** (sauf `/health`) | Identifiant unique du compte Norylum | `acc_famille` |
| `x-profile-id` | `string` | **Oui** (sauf `/health`) | Identifiant du profil utilisateur actif | `prof_enfant` |
| `x-device-id` | `string` | **Oui** (sauf `/health`) | Identifiant de l'appareil client (TV, mobile, web) | `dev_living_room` |
| `x-is-child` | `boolean` | Optionnel | Active le filtrage strict du contrôle parental (`true`/`false`) | `true` |
| `x-age-rating-max` | `integer` | Optionnel | Âge limite maximal autorisé pour le profil enfant (ex. PEGI / CNC) | `10` |

### Format Uniforme des Erreurs (`ErrorResponse`)

Lorsqu'une requête échoue, le serveur retourne systématiquement un corps JSON structuré :

```json
{
  "error": "Accès refusé au média 'med_9f8e7d6c5b4a3210' : profil enfant non autorisé",
  "code": "FORBIDDEN"
}
```

Codes machine standardisés :
- `UNAUTHORIZED` (`401`)
- `FORBIDDEN` (`403`)
- `NOT_FOUND` (`404`)
- `VALIDATION_ERROR` (`400`)
- `DIRECTORY_TRAVERSAL` (`403`)
- `RANGE_NOT_SATISFIABLE` (`416`)
- `FFPROBE_ERROR` (`500`)
- `IO_ERROR` (`500`)
- `INTERNAL_SERVER_ERROR` (`500`)

---

## 3. Spécification OpenAPI 3.0.3 (`openapi.yaml`)

Le fichier [`openapi.yaml`](openapi.yaml) documente l'ensemble des points d'accès disponibles sur le serveur Norylum :

### Résumé des Endpoints

| Méthode | Route | Rôle | Sécurité / En-têtes |
|---|---|---|---|
| `GET` | `/health` | Vérification de la disponibilité de l'instance | Public |
| `POST` | `/v1/library/scan` | Déclenche l'inspection `ffprobe` et l'indexation de la bibliothèque | Authentifié (`x-account-id`, `x-profile-id`, `x-device-id`) |
| `GET` | `/v1/library/media` | Retourne la liste des médias autorisés pour le profil actif | Authentifié + Filtre parental (`x-is-child`, `x-age-rating-max`) |
| `GET` | `/v1/media/{media_id}` | Streaming vidéo Direct Play avec négociation de plage d'octets | Authentifié + Header standard `Range: bytes=X-Y` |
| `POST` | `/v1/playback/progress` | Enregistre la progression de lecture d'un média (position, durée, complétion) | Authentifié |
| `GET` | `/v1/playback/{media_id}` | Récupère la dernière progression enregistrée pour un média donné | Authentifié |

---

## 4. Streaming Direct Play & HTTP Range 206

Le point d'accès `/v1/media/{media_id}` est spécialement optimisé pour la diffusion multimédia sans transcodage inutile (*Direct Play*).

### Fonctionnement Mémoire Constante $O(1)$

Le serveur ne charge **jamais** un fichier vidéo entier en mémoire vive (RAM). La lecture s'effectue par morceaux bornés de **64 Ko** (`STREAM_BUFFER_SIZE`) via des flux asynchrones Tokio (`ReaderStream` combiné à `AsyncReadExt::take`).

### Scénarios de Requêtes HTTP Range

1. **Diffusion standard (sans Range) :**
   - Requête : `GET /v1/media/med_12345`
   - Réponse : `200 OK`
   - En-têtes : `Accept-Ranges: bytes`, `Content-Length: 42000000`, `Content-Type: video/mp4`
   - Corps : Flux complet du conteneur vidéo.

2. **Lecture progressive / Fast-Forward (avec Range) :**
   - Requête : `GET /v1/media/med_12345` avec `Range: bytes=0-1048575` (premier mégaoctet)
   - Réponse : `206 Partial Content`
   - En-têtes :
     - `Accept-Ranges: bytes`
     - `Content-Range: bytes 0-1048575/42000000`
     - `Content-Length: 1048576`
     - `Content-Type: video/mp4`
   - Corps : Exactement les 1 048 576 octets sollicités.

3. **Plage invalide ou hors limites :**
   - Requête : `Range: bytes=50000000-60000000` (fichier de 42 Mo)
   - Réponse : `416 Range Not Satisfiable`
   - En-têtes : `Content-Range: bytes */42000000`

---

## 5. Spécification des Plugins (`schemas/plugin-manifest.schema.json`)

Le fichier [`schemas/plugin-manifest.schema.json`](schemas/plugin-manifest.schema.json) définit la structure formelle validant les manifestes `plugin.json` des extensions tierces.

### Champs Obligatoires

- **`id`** (`string`) : Identifiant unique immuable en kebab-case (`^[a-z0-9]+(-[a-z0-9]+)*$`).
- **`name`** (`string`) : Nom lisible du plugin.
- **`version`** (`string`) : Version sémantique valide (SemVer 2.0.0).
- **`description`** (`string`) : Résumé concis des fonctionnalités.
- **`entrypoint`** (`string`) : Chemin relatif vers le fichier principal exécutable ou script (`.js`, `.wasm`, binaire).
- **`permissions`** (`array`) : Liste explicite des permissions système requises.

### Système de Permissions (Moindre Privilège)

| Permission | Description |
|---|---|
| `library:read` | Lecture des métadonnées de la bibliothèque multimédia |
| `library:write` | Création ou modification d'éléments dans la bibliothèque |
| `metadata:read` | Consultation des métadonnées enrichies |
| `metadata:write` | Écriture de métadonnées, affiches, résumés et notations |
| `playback:read` | Consultation des états de lecture |
| `playback:write` | Mise à jour de la progression de visionnage |
| `storage:read` | Lecture dans le dossier de données propre au plugin |
| `storage:write` | Écriture dans le dossier de données propre au plugin |
| `network:outbound` | Requêtes HTTP/HTTPS externes (API tierces : TMDB, OpenSubtitles, MusicBrainz) |
| `system:notifications` | Émission de notifications système aux utilisateurs |

### Exemple de Manifeste `plugin.json`

```json
{
  "id": "tmdb-metadata",
  "name": "TheMovieDB Metadata Provider",
  "version": "1.0.0",
  "description": "Enrichit automatiquement les fiches films et séries avec affiches, synopsis et notations TMDB.",
  "entrypoint": "dist/index.js",
  "type": "metadata",
  "permissions": [
    "library:read",
    "metadata:write",
    "network:outbound"
  ],
  "author": {
    "name": "Norylum Community",
    "url": "https://github.com/norylum"
  },
  "license": "GPL-3.0-or-later",
  "homepage": "https://norylum.org/plugins/tmdb-metadata",
  "repository": "https://github.com/norylum/plugin-tmdb-metadata",
  "norylum_version": ">=1.0.0"
}
```

---

## 6. Instructions de Validation

Les schémas et spécifications de ce dépôt sont vérifiables en ligne de commande avec des outils standardisés de l'écosystème Node.js / Python.

### Validation de la Spécification OpenAPI (`openapi.yaml`)

Utilisation du CLI officiel **Redocly** :

```bash
npx --yes @redocly/cli lint openapi.yaml
```

Validation alternative avec **Swagger CLI** :

```bash
npx --yes @apidevtools/swagger-cli validate openapi.yaml
```

Génération d'une documentation interactive HTML :

```bash
npx --yes @redocly/cli build-docs openapi.yaml -o index.html
```

### Validation du Schéma JSON de Plugin (`schemas/plugin-manifest.schema.json`)

Vérification de la validité intrinsèque du métaschéma Draft 7 avec **Ajv CLI** :

```bash
npx --yes ajv-cli compile --spec=draft7 --strict=false -s schemas/plugin-manifest.schema.json
```

Validation d'un manifeste `plugin.json` concret par rapport au schéma :

```bash
npx --yes ajv-cli validate --spec=draft7 --strict=false -s schemas/plugin-manifest.schema.json -d plugin.json
```
