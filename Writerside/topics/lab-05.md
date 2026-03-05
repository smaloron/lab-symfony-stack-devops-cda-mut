# Lab 5 — Orchestration avec Docker Compose

## Objectifs du lab

À la fin de ce lab, vous serez capable de :

- Orchestrer plusieurs conteneurs avec Docker Compose
- Configurer les services PostgreSQL, Redis, PHP-FPM, Nginx et React
- Gérer les dépendances, réseaux et volumes entre services
- Valider le bon fonctionnement de l'ensemble de la stack


**Prérequis :**

- Les Dockerfiles du backend (`api/Dockerfile`) et du frontend (`front/Dockerfile`) sont déjà créés (labs précédents)
- Docker et Docker Compose sont installés sur votre machine
- Vous êtes à la racine du projet `yapuka/`

---

## Étape 1 — Comprendre l'architecture cible (5 min)

Avant d'écrire la moindre ligne de configuration, prenons le temps de comprendre ce que nous allons construire.

### 1.1 Les services nécessaires

Notre application Yapuka est composée de **5 services** qui doivent communiquer entre eux :

| Service      | Rôle                                 | Image / Build         | Port exposé |
|--------------|--------------------------------------|-----------------------|-------------|
| **nginx**    | Reverse proxy, point d'entrée unique | `nginx:alpine`        | 8080 → 80   |
| **php**      | Backend Symfony (PHP-FPM)            | Build depuis `api/`   | —           |
| **front**    | Frontend React (serveur de dev Vite) | Build depuis `front/` | 5173        |
| **database** | Base de données PostgreSQL           | `postgres:16-alpine`  | 5432        |
| **redis**    | Cache et pub/sub                     | `redis:7-alpine`      | 6379        |

### 1.2 Flux des requêtes

```
Navigateur → :8080 → Nginx
                       ├── /api/*  → PHP-FPM (:9000)
                       │                ├── PostgreSQL (:5432)
                       │                └── Redis (:6379)
                       └── /*      → Vite (:5173)
```

> **Question pour réfléchir :** Pourquoi utiliser Nginx comme point d'entrée plutôt que d'accéder directement aux
> services PHP et React ?

### 1.3 Exercice — Schéma d'architecture

Sur papier ou dans un outil de diagramme, dessinez :

1. Les 5 services sous forme de boîtes
2. Les flèches de communication entre eux
3. Les ports exposés vers l'hôte (votre machine)
4. Les volumes persistants nécessaires

#### Correction schéma d'architecture {collapsible="true"}

Les communications sont les suivantes :

- **Nginx** → **PHP-FPM** (FastCGI, port 9000) pour les requêtes `/api`
- **Nginx** → **Front** (HTTP, port 5173) pour les requêtes `/`
- **PHP-FPM** → **PostgreSQL** (TCP, port 5432) pour les données
- **PHP-FPM** → **Redis** (TCP, port 6379) pour le cache

Ports exposés vers l'hôte :

- `8080:80` (Nginx — point d'entrée principal)
- `5173:5173` (Vite — accès direct optionnel pour le dev)
- `5432:5432` (PostgreSQL — accès direct pour les outils DB)
- `6379:6379` (Redis — accès direct optionnel)

Volumes persistants :

- `db_data` → `/var/lib/postgresql/data` (données PostgreSQL)
- `redis_data` → `/data` (données Redis)

---

## Étape 2 — Créer le fichier docker-compose.yml (10 min)

Créez un fichier `docker-compose.yml` à la racine du projet. Nous allons le construire service par service.

### 2.1 Structure de base et service PostgreSQL

Commencez par la structure minimale avec le service de base de données.

**À vous de jouer :** créez le fichier `docker-compose.yml` avec :

- Un service `database` utilisant l'image `postgres:16-alpine`
- Les variables d'environnement : nom de la base `yapuka`, utilisateur `yapuka`, mot de passe `yapuka`
- Un volume nommé `db_data` monté sur `/var/lib/postgresql/data`
- Un **healthcheck** utilisant la commande `pg_isready -U yapuka` (intervalle 5s, timeout 5s, 5 retries)
- Le port 5432 exposé

> **Indice :** Les variables d'environnement de l'image officielle PostgreSQL sont `POSTGRES_DB`, `POSTGRES_USER` et
`POSTGRES_PASSWORD`.

#### Correction structure de base et service PostgreSQL {collapsible="true"}

```yaml
services:
  database:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: yapuka
      POSTGRES_USER: yapuka
      POSTGRES_PASSWORD: yapuka
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U yapuka" ]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

### 2.2 Service Redis

Ajoutez le service Redis à la suite.

**À vous de jouer :** ajoutez un service `redis` avec :

- L'image `redis:7-alpine`
- Un volume nommé `redis_data` monté sur `/data`
- Le port 6379 exposé

#### Correction Service Redis {collapsible="true"}

```yaml
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  volumes:
    db_data:
    redis_data:
```

### 2.3 Service Backend (PHP-FPM)

Le backend doit être **construit** à partir du Dockerfile dans `api/` et doit attendre que PostgreSQL soit prêt avant de
démarrer.

**À vous de jouer :** ajoutez un service `php` avec :

- Un `build` pointant vers le contexte `./api` et le Dockerfile `Dockerfile`
- Un **bind mount** du dossier `./api` vers `/var/www/api` (pour le développement)
- Les variables d'environnement : `APP_ENV=dev`, `DATABASE_URL` pointant vers le service `database`, `REDIS_URL`
  pointant vers le service `redis`
- Une dépendance vers `database` avec la condition `service_healthy` et vers `redis` avec `service_started`

> **Indice :** Dans Docker Compose, les services se référencent par leur nom. L'URL PostgreSQL sera de la forme
`postgresql://user:password@nom_service:5432/base`.

#### Correction Service Backend {collapsible="true"}

```yaml
  php:
    build:
      context: ./api
      dockerfile: Dockerfile
    volumes:
      - ./api:/var/www/api
    environment:
      APP_ENV: dev
      DATABASE_URL: "postgresql://yapuka:yapuka@database:5432/yapuka?serverVersion=16"
      REDIS_URL: "redis://redis:6379"
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_started
```

### 2.4 Service Frontend (React / Vite)

**À vous de jouer :** ajoutez un service `front` avec :

- Un `build` pointant vers `./front`
- Un bind mount de `./front` vers `/app`, **mais** en excluant `node_modules` avec un volume anonyme
- La variable d'environnement `VITE_API_URL` pointant vers `http://localhost:8080`
- Le port 5173 exposé

> **Indice :** Pour exclure `node_modules` du bind mount, ajoutez un volume anonyme : `/app/node_modules` (sans chemin
> hôte). Docker utilisera les `node_modules` installés dans l'image plutôt que ceux de votre machine.

#### Correction Service Frontend {collapsible="true"}

```yaml
  front:
    build:
      context: ./front
      dockerfile: Dockerfile
    volumes:
      - ./front:/app
      - /app/node_modules
    environment:
      VITE_API_URL: "http://localhost:8080"
    ports:
      - "5173:5173"
```

### 2.5 Service Nginx (Reverse Proxy)

Le reverse proxy est le point d'entrée unique. Il utilise une image officielle et un fichier de configuration que nous
allons créer.

**À vous de jouer :** ajoutez un service `nginx` avec :

- L'image `nginx:alpine`
- Le port `8080:80`
- Deux volumes en lecture seule (`:ro`) : le fichier de config Nginx et le dossier `api/public`
- Des dépendances vers `php` et `front`

> **Indice :** Le fichier de configuration sera monté depuis `./docker/nginx/default.conf` vers
`/etc/nginx/conf.d/default.conf`. Le dossier `api/public` est monté en `/var/www/api/public` pour servir les assets
> statiques.

#### Correction Service Nginx {collapsible="true"}

```yaml
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./api/public:/var/www/api/public:ro
    depends_on:
      - php
      - front
```

### 2.6 Réseau et assemblage final

Par défaut, Docker Compose crée un réseau `bridge` commun pour tous les services. Pour un projet de formation, un seul
réseau suffit.

**À vous de jouer :** ajoutez un réseau nommé `yapuka` de type `bridge` et connectez-y **tous** les services.

> **Indice :** Ajoutez un bloc `networks:` global, puis ajoutez `networks: - yapuka` dans chaque service.

#### Correction Réseau et assemblage final {collapsible="true"}

Ajoutez à la fin du fichier :

```yaml
networks:
  yapuka:
    driver: bridge
```

Et dans chaque service, ajoutez :

```yaml
    networks:
      - yapuka
```

Le fichier complet est :

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./api/public:/var/www/api/public:ro
    depends_on:
      - php
      - front
    networks:
      - yapuka

  php:
    build:
      context: ./api
      dockerfile: Dockerfile
    volumes:
      - ./api:/var/www/api
    environment:
      APP_ENV: dev
      DATABASE_URL: "postgresql://yapuka:yapuka@database:5432/yapuka?serverVersion=16"
      REDIS_URL: "redis://redis:6379"
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - yapuka

  front:
    build:
      context: ./front
      dockerfile: Dockerfile
    volumes:
      - ./front:/app
      - /app/node_modules
    environment:
      VITE_API_URL: "http://localhost:8080"
    ports:
      - "5173:5173"
    networks:
      - yapuka

  database:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: yapuka
      POSTGRES_USER: yapuka
      POSTGRES_PASSWORD: yapuka
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U yapuka" ]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - yapuka

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - yapuka

volumes:
  db_data:
  redis_data:

networks:
  yapuka:
    driver: bridge
```

---

## Étape 3 — Configurer le Reverse Proxy Nginx (8 min)

Nginx reçoit toutes les requêtes sur le port 80 et les distribue au bon service.

### 3.1 Créer le fichier de configuration

Créez le dossier et le fichier :

```bash
mkdir -p docker/nginx
```

**À vous de jouer :** créez le fichier `docker/nginx/default.conf` qui :

1. Écoute sur le port 80
2. Route les requêtes `/api` vers le backend PHP-FPM via **FastCGI** (port 9000 du service `php`)
3. Route toutes les autres requêtes (`/`) vers le frontend Vite (port 5173 du service `front`)
4. Support les WebSockets pour le Hot Module Replacement de Vite (headers `Upgrade` et `Connection`)

> **Indice FastCGI :** Pour transmettre les requêtes PHP à PHP-FPM, utilisez la directive `fastcgi_pass php:9000;` et
> spécifiez le `SCRIPT_FILENAME` pointant vers `/var/www/api/public/index.php`. N'oubliez pas d'inclure `fastcgi_params`.

> **Indice WebSocket :** Vite utilise les WebSockets pour le rechargement à chaud. Ajoutez dans le bloc `location /` :
> ```
> proxy_http_version 1.1;
> proxy_set_header Upgrade $http_upgrade;
> proxy_set_header Connection "upgrade";
> ```

#### Correction configuration Nginx {collapsible="true"}

```nginx
server {
    listen 80;
    server_name localhost;

    # Taille maximale des requêtes
    client_max_body_size 10M;

    # Backend Symfony — requêtes /api/*
    location ~ ^/api/(.*)$ {
        root /var/www/api/public;

        fastcgi_pass php:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME /var/www/api/public/index.php;
        fastcgi_param DOCUMENT_ROOT /var/www/api/public;

        # Timeout étendu pour SSE
        fastcgi_read_timeout 300;
        fastcgi_buffering off;
    }

    # Frontend React — toutes les autres requêtes
    location / {
        proxy_pass http://front:5173;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Support WebSocket (Vite HMR)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 3.2 Comprendre le routage

Répondez à ces questions pour vérifier votre compréhension :

1. Un utilisateur accède à `http://localhost:8080/api/tasks`. Quel service traite cette requête ?
2. Un utilisateur accède à `http://localhost:8080/login`. Quel service traite cette requête ?
3. Pourquoi utilise-t-on `fastcgi_pass` pour PHP et `proxy_pass` pour React ?

#### Correction routage {collapsible="true"}

1. **PHP-FPM** — l'URL commence par `/api`, donc Nginx la transmet au backend Symfony via FastCGI.
2. **Vite (front)** — l'URL ne commence pas par `/api`, donc Nginx la transmet au serveur de développement React.
3. **FastCGI** est le protocole natif de PHP-FPM (il ne comprend pas HTTP directement). **proxy_pass** est un proxy HTTP
   classique, adapté au serveur Node.js de Vite qui parle HTTP.

---

## Étape 4 — Lancer et valider la stack (7 min)

### 4.1 Démarrage

Lancez l'ensemble des services :

```bash
docker compose up -d
```

Vérifiez que tous les services sont en cours d'exécution :

```bash
docker compose ps
```

> **Attendu :** 5 services avec le statut `Up` ou `running`. Le service `database` devrait indiquer `(healthy)`.

Si un service est en erreur, consultez ses logs :

```bash
docker compose logs -f php
```

### 4.2 Vérifier les connexions

**Testez PostgreSQL :**

```bash
docker compose exec database psql -U yapuka -c "SELECT version();"
```

> **Attendu :** la version de PostgreSQL s'affiche.

**Testez Redis :**

```bash
docker compose exec redis redis-cli ping
```

> **Attendu :** `PONG`

**Testez le backend Symfony :**

```bash
docker compose exec php php bin/console about
```

> **Attendu :** les informations Symfony s'affichent (version, environnement, etc.)

### 4.3 Initialiser la base de données

Exécutez les commandes suivantes pour préparer l'application :

```bash
# Générer les clés JWT (nécessaires pour l'authentification)
docker compose exec php php bin/console lexik:jwt:generate-keypair

# Créer le schéma de la base de données
docker compose exec php php bin/console doctrine:schema:create

# Charger les données de démonstration
docker compose exec php php bin/console doctrine:fixtures:load --no-interaction
```

### 4.4 Test de bout en bout

Ouvrez votre navigateur sur **http://localhost:8080**.

1. La page de connexion React doit s'afficher
2. Connectez-vous avec `demo@yapuka.dev` / `password`
3. Le dashboard avec les tâches doit s'afficher

> Si la page ne charge pas, vérifiez les logs Nginx : `docker compose logs nginx`

### 4.5 Tester la persistence

Vérifiez que les données survivent à un redémarrage :

```bash
# Arrêter tous les services
docker compose down

# Relancer
docker compose up -d

# Vérifier que les données sont toujours là
docker compose exec database psql -U yapuka -c "SELECT count(*) FROM tasks;"
```

> **Attendu :** le nombre de tâches est identique à avant le redémarrage.

**Attention :** `docker compose down -v` (avec le flag `-v`) supprime les volumes et donc les données. Ne l'utilisez que
si vous voulez repartir de zéro.

#### Correction — Troubleshooting des erreurs courantes {collapsible="true"}

| Symptôme                          | Cause probable                             | Solution                                                            |
|-----------------------------------|--------------------------------------------|---------------------------------------------------------------------|
| `php` ne démarre pas              | PostgreSQL pas encore prêt                 | Vérifiez le `depends_on` avec `condition: service_healthy`          |
| `Connection refused` sur `/api`   | `fastcgi_pass` pointe vers le mauvais hôte | Vérifiez que c'est `php:9000` et non `localhost:9000`               |
| Page blanche sur `:8080`          | Vite n'a pas démarré                       | Vérifiez les logs du front : `docker compose logs front`            |
| `node_modules` introuvable        | Volume anonyme mal configuré               | Vérifiez que `/app/node_modules` est bien dans les volumes du front |
| Données perdues après redémarrage | Volumes nommés non définis                 | Vérifiez le bloc `volumes:` global en bas du fichier                |
| Erreur JWT                        | Clés non générées                          | Relancez `lexik:jwt:generate-keypair`                               |

---

## Récapitulatif

### Commandes essentielles

| Commande                           | Description                                        |
|------------------------------------|----------------------------------------------------|
| `docker compose up -d`             | Démarrer tous les services en arrière-plan         |
| `docker compose ps`                | Voir l'état des services                           |
| `docker compose logs -f [service]` | Suivre les logs d'un service                       |
| `docker compose exec php bash`     | Ouvrir un shell dans le conteneur PHP              |
| `docker compose down`              | Arrêter et supprimer les conteneurs                |
| `docker compose down -v`           | Idem + supprimer les volumes (⚠️ perte de données) |
| `docker compose build`             | Reconstruire les images                            |

### Ce que vous avez appris

- **Orchestration** : un seul fichier `docker-compose.yml` décrit toute l'infrastructure
- **Dépendances** : `depends_on` avec `condition` garantit l'ordre de démarrage
- **Healthchecks** : Docker vérifie qu'un service est réellement prêt, pas juste démarré
- **Réseaux** : les services communiquent par leur nom (DNS interne Docker)
- **Volumes** : les données persistent entre les redémarrages grâce aux volumes nommés
- **Reverse proxy** : Nginx distribue les requêtes au bon service selon l'URL