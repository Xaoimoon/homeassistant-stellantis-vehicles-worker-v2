# homeassistant-stellantis-vehicles-worker-v2

Service de login distant utilisé par l'intégration Home Assistant
[homeassistant-stellantis-vehicles](https://github.com/andreadegiovine/homeassistant-stellantis-vehicles)
pour automatiser la connexion OAuth Stellantis (Peugeot / Citroën / DS / Opel /
Vauxhall) via un navigateur headless (Playwright + Chromium), et récupérer le
code d'autorisation OAuth sans que l'utilisateur ait à faire l'échange manuel
dans son navigateur.

## Fonctionnement

1. L'intégration Home Assistant envoie `POST /` avec l'URL de login de la
   marque, l'email et le mot de passe du compte.
2. Le worker ouvre un contexte de navigateur headless, remplit le formulaire
   de login Gigya, le soumet, valide l'écran de consentement, puis intercepte
   la redirection contenant le code d'autorisation OAuth.
3. Il renvoie ce code à l'intégration, qui l'échange ensuite elle-même contre
   un access token.

## Déploiement

### Render (par défaut)

Le dépôt contient un [`render.yaml`](render.yaml) (service Docker, plan
gratuit, déploiement auto sur push sur `main`). Il suffit de connecter le
dépôt à un service Render et de définir la variable d'environnement
`WORKER_API_KEY` (voir ci-dessous).

### Docker Compose (local)

Voir [`compose.yaml`](compose.yaml) — build local, port 3000 publié sur
l'hôte. Copier [`.env.example`](.env.example) en `.env` et y renseigner
`WORKER_API_KEY`, puis :

```bash
docker compose up -d
```

### Docker Swarm (self-hosted, sans passer par Render)

Voir [`swarm-stack.yaml`](swarm-stack.yaml) — utilise l'image déjà publiée
par la CI (`ghcr.io/xaoimoon/stellantis:latest`), pas de build. Pas de
Traefik : le port 3000 est publié via la routing mesh Swarm (`mode: ingress`),
donc joignable sur l'IP de n'importe quel nœud du cluster, ce qui convient
pour un appelant en réseau interne (ex. Home Assistant tournant en
`network_mode: host`, qui n'a pas accès à la résolution DNS d'un réseau
overlay de toute façon). Ne pas exposer ce port sur Internet (pas de TLS ici) :
`WORKER_API_KEY` transiterait en clair si le port était accessible depuis
l'extérieur.

```bash
docker stack deploy -c swarm-stack.yaml stellantis-worker
```

## Variables d'environnement

| Variable         | Obligatoire | Description                                                                 |
|------------------|-------------|-------------------------------------------------------------------------------|
| `WORKER_API_KEY` | Oui*        | Secret partagé exigé pour appeler `POST /`. Si absente, l'endpoint refuse tout. |
| `WORKER_API_KEY_FILE` | Non    | Chemin d'un fichier contenant la clé (secret Docker/Swarm, ex. `/run/secrets/stellantis_worker_api_key`). Prioritaire sur `WORKER_API_KEY`. |

\* L'une des deux.

`WORKER_API_KEY` est ce qui empêche n'importe qui connaissant l'URL du worker
d'y poster des identifiants et une URL de son choix (le service ferait alors
naviguer son navigateur headless — et taper les identifiants fournis —
n'importe où : SSRF, relais de credential-stuffing anonyme, etc.). C'est une
chaîne aléatoire que tu choisis toi-même (`openssl rand -hex 32` par
exemple), à définir une seule fois côté serveur.

Elle se vérifie via la **query string** (`?key=...`) et non un header
d'autorisation classique (`Authorization: Bearer ...`), parce que
l'intégration Home Assistant qui appelle ce worker envoie sa requête sans
aucun header custom — son seul point de configuration est l'URL elle-même
(le champ *"Login service URL"*, voir plus bas). Glisser la clé dans la
query string de cette URL permet de l'authentifier sans toucher au code de
l'intégration.

Comportement **fail-secure** : si `WORKER_API_KEY` n'est pas définie côté
serveur, `POST /` refuse toutes les requêtes (401) plutôt que d'accepter
tout le monde par défaut — il faut donc la configurer avant que le service
ne soit utilisable (Render → Environment, ou `.env` pour
`compose.yaml`/`swarm-stack.yaml`).

## Configuration côté Home Assistant

Ce worker est utilisé au moment de l'ajout (ou de la ré-authentification) de
l'intégration **Stellantis Vehicles** dans Home Assistant :

1. **Paramètres → Appareils et services → Ajouter une intégration →
   Stellantis Vehicles**.
2. Choisir la marque (Peugeot / Citroën / DS / Opel / Vauxhall) puis le pays.
3. À l'étape du mode OAuth, laisser la case **"manuel" décochée** pour
   utiliser le login distant (c'est ce mode qui appelle le worker — le mode
   manuel demande de récupérer le code OAuth soi-même dans les outils de
   développement du navigateur, sans passer par ce service).
4. Sur l'écran suivant (étape `oauth_remote`), en plus de l'email et du mot
   de passe du compte Stellantis, se trouve le champ **"Login service URL"**.
   Il est pré-rempli avec l'URL publique par défaut de l'auteur de
   l'intégration (`https://homeassistant-stellantis-vehicles-worker.onrender.com`,
   sans clé) — c'est cette valeur qu'il faut remplacer par l'URL de **ton**
   déploiement pour que l'authentification passe par ce worker et pas par
   celui de l'auteur :

   ```
   https://<ton-worker>/?key=<WORKER_API_KEY>                # Render / instance avec TLS
   http://<ip-d-un-noeud-swarm>:3000/?key=<WORKER_API_KEY>    # déploiement interne (swarm-stack.yaml)
   ```

L'intégration transmet cette URL telle quelle à sa requête `POST` (y compris
la query string), il n'y a donc rien à modifier côté intégration — voir
[stellantis.py:339](https://github.com/andreadegiovine/homeassistant-stellantis-vehicles/blob/develop/custom_components/stellantis_vehicles/stellantis.py#L339)
dans le dépôt de l'intégration.

La valeur saisie est mémorisée dans la configuration de l'entrée
d'intégration : elle sera réutilisée automatiquement à chaque
ré-authentification (`reauth`), pas besoin de la ressaisir à chaque fois.

## API

### `POST /?key=<WORKER_API_KEY>`

Corps JSON :

| Champ           | Obligatoire | Défaut  | Description                                              |
|------------------|-------------|---------|------------------------------------------------------------|
| `url`            | Oui         | —       | URL de login OAuth de la marque (doit être un des hôtes autorisés, voir Sécurité) |
| `email`          | Oui         | —       | Identifiant du compte Stellantis                          |
| `password`       | Oui         | —       | Mot de passe du compte Stellantis                         |
| `timeout_page`   | Non         | 50000   | Timeout de navigation/chargement, en ms (borné à 1000–60000) |
| `timeout_input`  | Non         | 50000   | Timeout d'attente des champs du formulaire, en ms (borné à 1000–60000) |
| `debug`          | Non         | `false` | Active les logs détaillés pour cette requête               |

Réponses :

- **Succès (200)** : `{"code": "<code OAuth capturé>"}`
- **Échec** : `{"message": "<description> [<process_id>]", "code": <statut HTTP>}`

⚠️ Le champ `code` a un sens différent selon le statut : sur un `200` c'est le
code OAuth capturé, sur une erreur c'est le statut HTTP dupliqué dans le
corps. Ne pas confondre les deux côté client.

### `GET /health`

Vérifie que le navigateur headless est toujours connecté et le redémarre au
besoin. Utilisé comme health check par Render.

## Sécurité

- **Authentification** : `POST /` exige `?key=<WORKER_API_KEY>` correspondant
  à la variable d'environnement du même nom (comparaison en temps constant).
  Sans clé configurée côté serveur, tout appel est refusé.
- **Liste blanche de domaines** : `url` doit pointer vers un des hôtes de
  login OAuth Stellantis connus (`idpcvs.peugeot.com`, `idpcvs.citroen.com`,
  `idpcvs.driveds.com`, `idpcvs.opel.com`, `idpcvs.vauxhall.co.uk`), en HTTPS.
  Toute autre URL est rejetée avant même d'ouvrir un navigateur.
- **Timeouts bornés** : `timeout_page`/`timeout_input` sont plafonnés à 60s,
  quelle que soit la valeur envoyée par l'appelant.
- **`/health` non bloquant** : il ne dépend du verrou global que si un
  redémarrage du navigateur est réellement nécessaire, donc il ne reste jamais
  coincé derrière une requête `/` en cours.

Limitations connues :
- Une seule requête `/` est traitée à la fois (verrou global sur l'unique
  instance de navigateur) — les appels concurrents sont mis en file, pas
  parallélisés.
- Si Stellantis ajoute une nouvelle marque (nouvel hôte `oauth_url` non listé
  ci-dessus), le login échouera avec `"URL not allowed"` tant que l'hôte n'est
  pas ajouté dans `main.py`.

## Fichiers

- [`main.py`](main.py) — service FastAPI + Playwright
- [`Dockerfile`](Dockerfile) — image Docker (Python 3.11-slim + Chromium)
- [`render.yaml`](render.yaml) — déploiement Render
- [`requirements.txt`](requirements.txt) — dépendances Python
