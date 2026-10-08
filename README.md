# Stack Docker Compose : Flask, PostgreSQL, Redis, Nginx et registre privé

Projet de pratique Docker : une application Python minimale servie derrière un reverse proxy, avec base de données, cache et un registre d'images privé protégé par mot de passe.

## Architecture

    Client -> proxy (Nginx, port 8080) -> app (Flask) -> db (PostgreSQL)
                                                      -> cache (Redis)
    registry (registry:2, port 5000) : stocke l'image de l'app

| Service | Image | Réseaux | Rôle |
|---|---|---|---|
| proxy | nginx:alpine | frontend | seul point d'entrée public (8080) |
| app | localhost:5000/todo-app:1.0 | frontend, backend | application Flask |
| db | postgres:16-alpine | backend | données persistantes (volume db_data) |
| cache | redis:7-alpine | backend | cache |
| registry | registry:2 | par défaut | registre privé avec authentification htpasswd (volume registry_data) |

Les réseaux `frontend` et `backend` isolent la base et le cache : le proxy ne peut pas les atteindre, seule l'app fait le pont.

## Prérequis

- Docker Engine et Docker Compose v2 (`docker compose`)
- Un système Linux

## Démarrage à froid (première installation)

Le registre est vide sur une installation neuve : il faut donc y publier l'image de l'app avant de lancer la stack complète.

1. Créer le fichier d'environnement et choisir un mot de passe de base de données :

        cp .env.example .env
        nano .env

2. Créer l'utilisateur du registre (remplacer MOT_DE_PASSE) :

        mkdir auth
        docker run --rm --entrypoint htpasswd httpd:2 -Bbn registryuser 'MOT_DE_PASSE' > auth/htpasswd

3. Démarrer uniquement le registre et s'y connecter :

        docker compose up -d registry
        docker login localhost:5000 -u registryuser

4. Construire et publier l'image de l'app :

        docker build -t localhost:5000/todo-app:1.0 ./app
        docker push localhost:5000/todo-app:1.0

5. Lancer toute la stack :

        docker compose up -d

## Utilisation

- Application : http://localhost:8080 (affiche "Hello depuis Docker !")
- État des services : `docker compose ps` (app, db et cache doivent être `healthy`)
- Logs d'un service : `docker compose logs <service>`
- Arrêt en conservant les données : `docker compose down`
- Arrêt avec suppression des données : `docker compose down -v`
- Contenu du registre : `curl -u registryuser:MOT_DE_PASSE http://localhost:5000/v2/_catalog`

## Robustesse

- `restart: unless-stopped` : un service qui plante est relancé automatiquement.
- Healthchecks sur app, db et cache ; l'app attend que db et cache soient prêts, le proxy attend l'app.
- Les données PostgreSQL survivent à `docker compose down` grâce au volume nommé `db_data`.
- Le proxy re-résout l'adresse de l'app à chaque requête (directive `resolver`), ce qui lui permet de suivre l'app quand elle est recréée.

## Sécurité

- Aucun secret dans le dépôt : `.env` et `auth/` sont dans `.gitignore`. Seul `.env.example` est versionné.
- Le registre exige une authentification pour lire et pousser des images.
- Limite connue : le registre est servi en HTTP, ce qui n'est accepté par Docker que pour `localhost`. Pour l'ouvrir à d'autres machines, il faut ajouter TLS.
