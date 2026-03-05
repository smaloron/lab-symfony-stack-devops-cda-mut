# 5. Orchestration avec Docker Compose

## De l'immeuble à la ville : pourquoi orchestrer ?

Vous savez déjà construire un conteneur Docker : un Dockerfile, une
image, et le tour est joué. C'est comme savoir construire un immeuble.

Mais une application réelle ne tient jamais dans un seul bâtiment.
Yapuka, l'application que vous allez orchestrer dans ce lab, nécessite
cinq services qui doivent coexister, communiquer, et démarrer dans le
bon ordre. Gérer tout cela à la main, avec cinq commandes `docker run`
distinctes, c'est comme construire une ville en posant chaque brique
à la main, sans plan d'urbanisme.

**Docker Compose** est ce plan d'urbanisme. Un seul fichier YAML décrit
l'ensemble de l'infrastructure : les bâtiments (services), les routes
(réseaux), les entrepôts (volumes) et les règles de circulation
(dépendances). Une seule commande suffit ensuite pour tout faire
démarrer.

```mermaid
graph TD
    subgraph "Sans Docker Compose"
        A[docker run postgres] --> B[docker run redis]
        B --> C[docker run php]
        C --> D[docker run react]
        D --> E[docker run nginx]
        E --> F[Tout brancher manuellement...]
    end

    subgraph "Avec Docker Compose"
        G["docker compose up -d"] --> H[Tout démarre dans le bon ordre]
    end
```

---

## L'anatomie d'un fichier docker-compose.yml

Un fichier `docker-compose.yml` est organisé en trois grandes sections,
comme les plans d'une ville :

- **`services`** : les bâtiments et leurs habitants
- **`volumes`** : les entrepôts de données persistantes
- **`networks`** : les routes entre les bâtiments

```yaml
services:
  mon-service:
    image: nginx:alpine   # Le "plan" du bâtiment
    ports:
      - "8080:80"         # La porte d'entrée publique
    volumes:
      - mes-donnees:/data # L'entrepôt attribué
    networks:
      - mon-reseau        # La route sur laquelle il donne

volumes:
  mes-donnees:            # Déclaration de l'entrepôt

networks:
  mon-reseau:             # Déclaration de la route
    driver: bridge
```

### Image ou build ?

Pour chaque service, Docker Compose a besoin de savoir d'où vient
l'image. Il y a deux cas :

**Image toute prête** (depuis Docker Hub) : on indique directement son
nom. C'est comme commander un meuble en kit : quelqu'un d'autre l'a
déjà fabriqué.

```yaml
database:
  image: postgres:16-alpine
```

**Image à construire** (depuis un Dockerfile local) : on indique le
chemin du contexte de build. C'est comme construire le meuble
soi-même depuis ses propres plans.

```yaml
php:
  build:
    context: ./api
    dockerfile: Dockerfile
```

---

## Les réseaux : comment les services se parlent

### Le DNS interne de Docker

Imaginez un immeuble de bureaux où chaque entreprise a son propre
interphone. Vous n'avez pas besoin de connaître l'adresse IP de
l'entreprise voisine — il vous suffit de composer son nom sur
l'annuaire.

Dans Docker Compose, chaque service enregistre automatiquement son
**nom de service** comme nom DNS sur le réseau interne. Le service
`php` peut donc joindre le service `database` en utilisant simplement
l'hôte `database` — Docker se charge de la résolution d'adresse.

```mermaid
graph LR
    subgraph "Réseau Docker 'yapuka'"
        PHP["php<br />(172.20.0.3)"]
        DB["database<br />(172.20.0.2)"]
        RD["redis<br />(172.20.0.4)"]
    end
    PHP -->|"postgresql://database:5432"| DB
    PHP -->|"redis://redis:6379"| RD
```

C'est pourquoi, dans la variable d'environnement `DATABASE_URL` du
service PHP, on écrit `@database:5432` et non `@localhost:5432` : on
s'adresse au service par son nom, pas par une adresse locale.

### Le réseau de type bridge

Un réseau `bridge` est comme un quartier privé avec sa propre voirie.
Les services qui y sont connectés peuvent se parler librement entre
eux, mais ils sont isolés du reste. Pour qu'un service soit accessible
depuis votre machine (l'extérieur du quartier), il faut ouvrir une
porte explicitement.

```mermaid
graph TB
    subgraph "Votre machine (hôte)"
        Browser["Navigateur<br />:8080"]
        PGAdmin["pgAdmin<br />:5432"]
    end

    subgraph "Réseau bridge 'yapuka' (interne)"
        Nginx["nginx<br />:80"]
        PHP["php<br />:9000"]
        Front["front<br />:5173"]
        DB["database<br />:5432"]
        Redis["redis<br />:6379"]
    end

    Browser -->|"port mappé 8080→80"| Nginx
    PGAdmin -->|"port mappé 5432→5432"| DB
    Nginx -->|interne uniquement| PHP
    Nginx -->|interne uniquement| Front
    PHP -->|interne uniquement| DB
    PHP -->|interne uniquement| Redis
```

La notation `"8080:80"` dans la section `ports` signifie :
"ouvrir la porte 8080 de ma machine et la relier à la porte 80
du conteneur". Sans cette déclaration, le port reste privé au réseau
interne.

---

## Les volumes : la mémoire persistante

### Le problème de l'amnésie

Un conteneur est, par nature, éphémère : quand il s'arrête, tout ce
qu'il a écrit dans son système de fichiers interne disparaît. C'est
comme un tableau blanc qui s'efface à chaque fin de journée.

Pour garder des données entre les redémarrages — les lignes de votre
base PostgreSQL, par exemple — il faut les stocker en dehors du
conteneur, dans un **volume**.

### Deux types de volumes

**Le volume nommé** : Docker gère lui-même l'emplacement du stockage.
C'est comme louer un box de stockage dans un entrepôt géré par Docker :
vous n'avez pas à vous soucier de l'adresse exacte, Docker s'en charge.
Il suffit de déclarer le volume et de l'associer à un chemin dans le
conteneur.

```yaml
database:
  volumes:
    - db_data:/var/lib/postgresql/data  # entrepôt:chemin_interne

volumes:
  db_data:  # déclaration du volume nommé
```

**Le bind mount** : vous mappez directement un dossier de votre machine
vers un chemin dans le conteneur. C'est comme partager votre bureau
avec le conteneur : toute modification de votre côté est immédiatement
visible à l'intérieur, et vice versa. C'est idéal pour le
développement.

```yaml
php:
  volumes:
    - ./api:/var/www/api  # dossier_hôte:chemin_interne
```

```mermaid
graph LR
    subgraph "Votre machine"
        HostDir["./api/<br />(vos fichiers sources)"]
        DockerStore["Stockage Docker<br />(géré automatiquement)"]
    end

    subgraph "Conteneur php"
        ContPath["/var/www/api"]
    end

    subgraph "Conteneur database"
        DBPath["/var/lib/postgresql/data"]
    end

    HostDir <-->|"bind mount<br />(synchronisé en temps réel)"| ContPath
    DockerStore <-->|"volume nommé<br />(persistant)"| DBPath
```

### Le volume anonyme : masquer node_modules

Dans le lab, le service frontend utilise une astuce :

```yaml
front:
  volumes:
    - ./front:/app          # bind mount du code source
    - /app/node_modules     # volume anonyme : masque le dossier hôte
```

Sans la deuxième ligne, le dossier `node_modules` de votre machine
(qui peut contenir des binaires incompatibles) écraserait celui installé
dans l'image par le `Dockerfile`. Le volume anonyme `/app/node_modules`
indique à Docker : "pour ce chemin précis, utilise un stockage interne
plutôt que le bind mount parent". Le conteneur utilise ses propres
dépendances, compilées pour son système d'exploitation.

---

## Les dépendances et les healthchecks

### Le problème de l'ordre de démarrage

Imaginez une cuisine : le serveur ne peut pas prendre les commandes
si le cuisinier n'est pas encore arrivé, et le cuisinier ne peut pas
cuisiner si le livreur n'a pas encore apporté les ingrédients.

Dans notre stack, le service PHP a besoin que PostgreSQL soit prêt
avant de démarrer. Si PHP essaie de se connecter à la base de données
avant qu'elle soit initialisée, il échoue.

### depends_on : l'ordre de démarrage

La directive `depends_on` indique qu'un service doit attendre qu'un
autre soit dans un certain état avant de démarrer.

```yaml
php:
  depends_on:
    database:
      condition: service_healthy  # attendre que database soit "healthy"
    redis:
      condition: service_started  # attendre simplement que redis soit démarré
```

Il existe trois conditions :

- `service_started` : le conteneur a été lancé (mais pas forcément prêt)
- `service_healthy` : le healthcheck du conteneur a retourné "OK"
- `service_completed_successfully` : le conteneur s'est terminé avec
  le code 0 (utile pour les scripts d'initialisation)

### Les healthchecks : "es-tu vraiment prêt ?"

Un conteneur peut être "démarré" sans être "prêt". PostgreSQL, par
exemple, prend quelques secondes pour initialiser ses fichiers de données
après le lancement du processus. Pendant ce temps, toute tentative de
connexion échouera.

Un **healthcheck** est une commande que Docker exécute régulièrement
à l'intérieur du conteneur pour vérifier qu'il fonctionne correctement.
C'est comme un agent de sécurité qui frappe à la porte toutes les
5 secondes et demande "service en ordre ?". Tant que la réponse n'est
pas "oui", le service reste en état `starting` plutôt que `healthy`.

```yaml
database:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U yapuka"]
    interval: 5s    # frappe toutes les 5 secondes
    timeout: 5s     # attend 5s max la réponse
    retries: 5      # abandonne après 5 échecs consécutifs
```

```mermaid
sequenceDiagram
    participant DC as Docker Compose
    participant DB as database
    participant PHP as php

    DC->>DB: Démarrer le conteneur
    DB-->>DC: Conteneur lancé (state: starting)

    loop Toutes les 5 secondes
        DC->>DB: pg_isready -U yapuka ?
        DB-->>DC: Pas encore prêt...
    end

    DC->>DB: pg_isready -U yapuka ?
    DB-->>DC: Prêt ! (state: healthy)

    DC->>PHP: Démarrer le conteneur
    Note over PHP: depends_on database: service_healthy
```

---

## Nginx : le chef d'orchestre des requêtes

### Qu'est-ce qu'un reverse proxy ?

Imaginez un grand hôtel avec une seule réception. Peu importe ce que
vous demandez — réserver une table au restaurant, appeler le service
d'étage, réserver la salle de conférence — vous passez toujours par
la même réception. Celle-ci écoute votre demande, identifie le bon
service interne, et le contacte en votre nom.

Nginx joue ce rôle dans notre stack. Tous les appels entrants arrivent
sur le port 8080. Nginx analyse l'URL et décide vers quel service
interne acheminer la requête.

```mermaid
graph TD
    Client["Navigateur<br />localhost:8080"]
    Nginx["Nginx<br />(Reverse Proxy)<br />:80"]

    Client -->|"Toutes les requêtes"| Nginx

    Nginx -->|"URL commence par /api/*<br />FastCGI → :9000"| PHP["PHP-FPM<br />(Symfony)"]
    Nginx -->|"Toutes les autres URLs<br />HTTP → :5173"| Front["Vite<br />(React)"]

    PHP --> DB["PostgreSQL<br />:5432"]
    PHP --> Redis["Redis<br />:6379"]
```

### FastCGI vs proxy HTTP : deux langues différentes

Nginx peut parler à des services aval de deux façons, selon leur nature.

**`proxy_pass` (HTTP)** : Nginx transmet la requête HTTP telle quelle
à un autre serveur HTTP. C'est comme passer un message dans la même
langue. Vite (Node.js) parle HTTP nativement, donc on utilise
`proxy_pass`.

**`fastcgi_pass` (FastCGI)** : PHP-FPM ne comprend pas HTTP directement.
Il parle FastCGI, un protocole binaire optimisé pour exécuter des
scripts. Nginx doit "traduire" la requête HTTP en appel FastCGI. C'est
comme avoir un interprète entre vous et quelqu'un qui parle une autre
langue.

```mermaid
sequenceDiagram
    participant C as Navigateur
    participant N as Nginx
    participant V as Vite (front)
    participant P as PHP-FPM

    C->>N: GET /login (HTTP)
    N->>V: proxy_pass http://front:5173 (HTTP)
    V-->>N: 200 OK + HTML
    N-->>C: 200 OK + HTML

    C->>N: GET /api/tasks (HTTP)
    N->>P: fastcgi_pass php:9000 (FastCGI)
    Note over N,P: Nginx traduit HTTP en FastCGI
    P-->>N: Réponse FastCGI
    N-->>C: 200 OK + JSON
```

### Les WebSockets pour le rechargement à chaud

Vite (le serveur de développement React) utilise les WebSockets pour
notifier le navigateur dès qu'un fichier source change, afin de
recharger la page automatiquement. C'est le Hot Module Replacement (HMR).

Un WebSocket est une connexion HTTP qui "monte en grade" : elle commence
comme une requête HTTP ordinaire, puis demande à passer en mode
bidirectionnel persistant (comme passer d'un courrier postal à un
appel téléphonique). Pour que Nginx laisse passer cette "montée en
grade", il faut lui transmettre les en-têtes correspondants :

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

---

## Les variables d'environnement

Les variables d'environnement permettent de configurer un conteneur
sans modifier son image. C'est comme les paramètres d'une recette :
la recette (l'image) reste la même, mais vous pouvez ajuster la
quantité de sel (la configuration) à chaque fois que vous la préparez.

Dans Docker Compose, on les déclare sous la clé `environment` :

```yaml
php:
  environment:
    APP_ENV: dev
    DATABASE_URL: "postgresql://yapuka:yapuka@database:5432/yapuka"
    REDIS_URL: "redis://redis:6379"
```

Remarquez que les URLs de connexion utilisent les **noms de services**
(`database`, `redis`) comme hôtes — pas `localhost`. C'est le DNS
interne Docker qui résout ces noms vers les bonnes adresses IP.

---

## Vue d'ensemble de la stack Yapuka

Voici l'architecture complète que vous allez mettre en place dans
le lab :

```mermaid
graph TB
    subgraph "Hôte (votre machine)"
        Browser["Navigateur"]
        DevTools["Outils DB<br />(pgAdmin, etc.)"]
    end

    subgraph "Réseau Docker 'yapuka'"
        subgraph "Entrée"
            Nginx["nginx<br />image: nginx:alpine<br />port hôte: 8080"]
        end

        subgraph "Application"
            PHP["php<br />build: ./api<br />PHP-FPM + Symfony"]
            Front["front<br />build: ./front<br />React + Vite<br />port hôte: 5173"]
        end

        subgraph "Données"
            DB["database<br />image: postgres:16-alpine<br />port hôte: 5432"]
            Redis["redis<br />image: redis:7-alpine<br />port hôte: 6379"]
        end
    end

    subgraph "Volumes"
        V1["db_data"]
        V2["redis_data"]
        V3["./api (bind mount)"]
        V4["./front (bind mount)"]
    end

    Browser -->|":8080"| Nginx
    DevTools -->|":5432"| DB

    Nginx -->|"FastCGI :9000<br />/api/*"| PHP
    Nginx -->|"HTTP :5173<br />/*"| Front

    PHP -->|":5432"| DB
    PHP -->|":6379"| Redis

    DB --- V1
    Redis --- V2
    PHP --- V3
    Front --- V4
```

### Ordre de démarrage garanti

```mermaid
graph LR
    DB["database<br />healthcheck:<br />pg_isready"]
    Redis["redis"]
    PHP["php<br />(depends_on database:healthy<br />redis:started)"]
    Front["front"]
    Nginx["nginx<br />(depends_on php<br />front)"]

    DB -->|"service_healthy"| PHP
    Redis -->|"service_started"| PHP
    PHP -->|"démarré"| Nginx
    Front -->|"démarré"| Nginx
```

---

## Ce qu'il faut retenir avant de commencer

Avant de vous lancer dans le lab, assurez-vous d'avoir bien compris
ces quatre points essentiels :

**1. Les services se référencent par leur nom**
Dans toutes les URLs de connexion (`DATABASE_URL`, `REDIS_URL`,
`fastcgi_pass`, `proxy_pass`), on utilise le nom du service défini
dans le `docker-compose.yml`, pas `localhost`.

**2. La différence entre volume nommé et bind mount**
Le volume nommé conserve les données entre les redémarrages (base de
données). Le bind mount synchronise un dossier hôte avec le conteneur
(code source en développement).

**3. `depends_on` avec `condition: service_healthy` nécessite un
healthcheck**
Un service ne peut attendre qu'un autre soit `healthy` que si ce
dernier a déclaré un healthcheck. Sans healthcheck, seule la condition
`service_started` est utilisable.

**4. FastCGI pour PHP-FPM, proxy HTTP pour tout le reste**
PHP-FPM ne parle pas HTTP. Nginx doit utiliser `fastcgi_pass` et
fournir la variable `SCRIPT_FILENAME` pour que PHP sache quel fichier
exécuter.

---

## Ressources de référence

- [Documentation officielle Docker Compose](https://docs.docker.com/compose/)
- [Référence du fichier compose.yml](https://docs.docker.com/compose/compose-file/)
- [Documentation Nginx — module FastCGI](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)
- [Documentation image officielle PostgreSQL](https://hub.docker.com/_/postgres)
- [Documentation image officielle Redis](https://hub.docker.com/_/redis)