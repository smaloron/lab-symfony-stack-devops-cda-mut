# 3. Dockerisation production d'une API Symfony

## Le problème : votre application marche en dev, et en prod ?

Vous avez réussi à faire tourner l'API Yapuka sur votre machine grâce
à Docker. Parfait. Mais imaginez maintenant que vous déployez cette
même image sur un serveur de production. Votre image embarque Xdebug,
des outils de profiling, des dizaines de packages de test, des
fichiers de configuration de développement... Résultat : une image
lourde, lente à démarrer, exposant des outils qui n'ont rien à faire
en production, et potentiellement vulnérable.

Ce lab vous demande de construire une image Docker "propre" pour la
production. Pour y arriver, vous avez besoin de comprendre trois
choses : comment Docker construit les images couche par couche,
comment le multi-stage build permet de séparer "construire" de
"livrer", et comment Nginx et PHP-FPM collaborent pour servir votre
API.

---

## 1. Les images Docker : une pile de calques

### L'analogie du mille-feuille

Imaginez une image Docker comme un mille-feuille. Chaque instruction
de votre `Dockerfile` (`FROM`, `RUN`, `COPY`, `ENV`...) ajoute une
nouvelle couche de pâte feuilletée par-dessus les précédentes. Une
fois posée, une couche ne change plus. Si vous modifiez une couche du
milieu, toutes celles au-dessus doivent être reconstruites.

Docker appelle ces couches des **layers**. Chaque layer est mis en
cache. Si rien n'a changé depuis le dernier build, Docker réutilise le
cache et saute l'instruction. C'est pourquoi l'ordre des instructions
dans un `Dockerfile` est crucial : on place en haut ce qui change
rarement, en bas ce qui change souvent.

```mermaid
graph TB
    A["FROM php:8.4-fpm-alpine<br/>(image de base)"]
    B["RUN apk add ... <br/>(dépendances système)"]
    C["COPY composer.json<br/>(fichiers Composer)"]
    D["RUN composer install<br/>(dépendances PHP)"]
    E["COPY . .<br/>(code applicatif)"]
    F["USER www-data<br/>(sécurité)"]
    A --> B --> C --> D --> E --> F
    style A fill: #dbeafe, stroke: #3b82f6
    style B fill: #dbeafe, stroke: #3b82f6
    style C fill: #fef9c3, stroke: #eab308
    style D fill: #fef9c3, stroke: #eab308
    style E fill: #fce7f3, stroke: #ec4899
    style F fill: #dcfce7, stroke: #22c55e
```

> Les layers bleus changent rarement (image de base, extensions PHP).
> Les layers jaunes changent quand les dépendances évoluent.
> Le layer rose change à chaque modification du code — et invalide
> tout ce qui suit.

### Ce qui se passe quand on invalide le cache

Si vous écrivez votre `Dockerfile` dans le mauvais ordre — par exemple
en copiant tout le code *avant* de lancer `composer install` — alors
chaque modification d'un fichier PHP force Docker à relancer
`composer install` intégralement. Sur une API avec 80+ dépendances,
cela peut prendre plusieurs minutes à chaque build.

La règle d'or : **du plus stable au plus volatile**.

---

## 2. Le multi-stage build : le camion de déménagement

### L'analogie du chantier de construction

Quand on construit une maison, on a besoin d'une grue, d'une bétonnière,
d'échafaudages — des outils lourds et encombrants. Mais une fois la
maison livrée, le propriétaire n'en a plus besoin. Il emménage dans
une maison propre, sans les outils du chantier.

Le multi-stage build fonctionne exactement comme cela. Vous définissez
plusieurs "chantiers" (stages) dans un seul `Dockerfile`. Chaque stage
peut utiliser des outils lourds (Composer, gcc pour compiler des
extensions...). Seul le dernier stage est livré comme image finale —
sans les outils de construction.

```mermaid
graph LR
    subgraph "Stage 1 : base"
        A["php:8.4-fpm-alpine"]
        B["Extensions PHP<br/>pdo_pgsql, intl,<br/>opcache, redis"]
        A --> B
    end

    subgraph "Stage 2 : deps"
        C["Hérite de base"]
        D["+ Composer 2"]
        E["composer install<br/>--no-dev<br/>--optimize-autoloader"]
        C --> D --> E
    end

    subgraph "Stage 3 : production"
        F["Hérite de base"]
        G["COPY vendor/<br/>depuis deps"]
        H["COPY code<br/>applicatif"]
        I["USER www-data<br/>EXPOSE 9000"]
        F --> G --> H --> I
    end

    subgraph "Image finale livrée"
        J["php:8.4-fpm-alpine<br/>+ extensions<br/>+ vendor/ prod<br/>+ code<br/>~120 MB"]
    end

    B --> C
    E -- " COPY --from=deps " --> G
    I --> J
    style J fill: #dcfce7, stroke: #22c55e
```

Ce qui n'entre **pas** dans l'image finale :

- Composer lui-même (inutile en production)
- Les dépendances de développement (`phpunit`, `behat`, `foundry`...)
- Xdebug
- Les fichiers de test, les fixtures, les clés JWT

Le résultat : une image de ~120 MB au lieu de 200+ MB, et surtout une
image sans outils superflus qui pourraient être exploités.

### La syntaxe clé

```docker
# Nommer un stage avec AS
FROM php:8.4-fpm-alpine AS base

# Créer un autre stage qui hérite du précédent
FROM base AS deps

# Copier des fichiers d'un stage vers un autre
COPY --from=deps /var/www/api/vendor ./vendor

# Copier depuis une image externe (Composer officiel)
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
```

---

## 3. Le .dockerignore : le tri avant le déménagement

Avant qu'un `docker build` commence, Docker rassemble tous les fichiers
du dossier courant dans ce qu'on appelle le **contexte de build** et
les envoie au daemon Docker. Si ce dossier contient `vendor/` (80 MB
de dépendances PHP), les logs, les caches Symfony, les clés JWT et
le dossier `.git`, Docker les envoie tous — pour rien.

Le `.dockerignore` fonctionne exactement comme `.gitignore` : il liste
les fichiers et dossiers à exclure du contexte de build.

```
# Analogie : vous déménagez dans un nouvel appartement.
# Vous n'emportez pas les poubelles, les vieux cartons,
# ni les outils du propriétaire précédent.
```

| Exclure               | Pourquoi                                          |
|-----------------------|---------------------------------------------------|
| `vendor/`             | Sera installé dans le stage `deps`                |
| `var/`                | Cache et logs générés à l'exécution, pas au build |
| `config/jwt/`         | Les clés sont des secrets, jamais dans une image  |
| `tests/`, `features/` | Le code de test n'a pas sa place en production    |
| `.env.local`          | Contient des secrets locaux                       |
| `.git/`               | Inutile et volumineux                             |

---

## 4. OPcache : la traduction mise en cache

### L'analogie de l'interprète

Imaginez que PHP soit un interprète qui traduit vos fichiers `.php`
(écrits en langage humain) en instructions compréhensibles par le
processeur, à chaque requête. C'est lent : lire le fichier, analyser
la syntaxe, compiler en bytecode, exécuter.

OPcache, c'est l'interprète qui garde ses traductions dans un carnet.
La première requête ? Il traduit et note. Les suivantes ? Il relit son
carnet. Le gain de performance est considérable.

En production, un paramètre est particulièrement important :

```ini
opcache.validate_timestamps=0
```

En développement, OPcache vérifie si les fichiers ont changé entre
chaque requête (utile quand vous modifiez du code). En production, le
code ne change jamais pendant que l'application tourne — donc cette
vérification est inutile et coûteuse. On la désactive.

```mermaid
sequenceDiagram
    participant C as Client HTTP
    participant FPM as PHP-FPM
    participant OC as OPcache
    participant FS as Système de fichiers
    C ->> FPM: GET /api/users
    FPM ->> OC: Bytecode de index.php en cache ?

    alt Cache froid (première requête)
        OC -->> FPM: Non
        FPM ->> FS: Lire index.php
        FS -->> FPM: Code source
        FPM ->> OC: Stocker le bytecode
        FPM ->> FPM: Exécuter
    else Cache chaud (requêtes suivantes)
        OC -->> FPM: Oui - bytecode prêt
        FPM ->> FPM: Exécuter directement
    end

    FPM -->> C: Réponse JSON
```

---

## 5. PHP-FPM et Nginx : deux acteurs, un rôle chacun

### L'analogie du restaurant

Un restaurant a deux équipes bien distinctes : la salle (les serveurs
qui accueillent, prennent les commandes, servent les plats) et la
cuisine (les cuisiniers qui préparent). Le client ne parle qu'aux
serveurs. Les serveurs transmettent les commandes à la cuisine et
rapportent les plats.

Dans votre stack de production, c'est la même organisation :

- **Nginx** = la salle. Il reçoit les requêtes HTTP, sert les fichiers
  statiques (images, CSS, JS) directement et sans effort, et transmet
  les requêtes PHP à la cuisine.
- **PHP-FPM** = la cuisine. Il ne parle pas HTTP. Il reçoit des
  instructions via le protocole **FastCGI** et renvoie le résultat à
  Nginx.

```mermaid
graph LR
    Client["Navigateur / Client API"]

    subgraph "Conteneur Nginx :80"
        N["Nginx"]
        N1{"Fichier\nstatique ?"}
        N2["Servir\ndirectement"]
    end

    subgraph "Conteneur PHP :9000"
        FPM["PHP-FPM"]
        S["Symfony\nindex.php"]
    end

    Client -->|" HTTP :80 "| N
    N --> N1
    N1 -->|" Oui (CSS, JS...) "| N2
    N1 -->|" Non (route API) "| FPM
    FPM --> S
    S -->|" Réponse "| FPM
    FPM -->|" FastCGI "| N
    N -->|" HTTP "| Client
```

### Le protocole FastCGI

Nginx communique avec PHP-FPM via **FastCGI**, un protocole binaire
interne, sur le port 9000. Dans la configuration Nginx, la directive
clé est :

```nginx
fastcgi_pass php:9000;
```

Où `php` est le nom du conteneur PHP-FPM résolu par Docker. La
directive `SCRIPT_FILENAME` indique à PHP quel fichier exécuter :

```nginx
fastcgi_param SCRIPT_FILENAME
    $document_root$fastcgi_script_name;
```

Pour une API Symfony, toutes les routes passent par un seul fichier
d'entrée (`public/index.php`). Nginx est configuré pour renvoyer
toute URL inconnue vers ce fichier :

```nginx
location / {
    try_files $uri /index.php$is_args$args;
}
```

Traduction : "essaie de servir le fichier tel quel, sinon passe à
`index.php` avec les paramètres d'URL".

---

## 6. L'utilisateur non-root : le principe du moindre privilège

### L'analogie du badge d'accès

Dans une entreprise, un stagiaire n'a pas accès à la salle des
serveurs. Un développeur n'a pas les droits d'administrateur système.
Chacun dispose exactement des droits nécessaires à son travail, pas
plus. Si quelqu'un vole le badge du stagiaire, les dégâts potentiels
sont limités.

En production, un conteneur qui tourne en `root` est une porte ouverte.
Si votre application est compromise, l'attaquant dispose des droits
maximum sur le conteneur. L'utilisateur `www-data` est un utilisateur
standard pré-existant dans l'image PHP qui n'a accès qu'aux fichiers
de l'application.

```docker
# Dans le Dockerfile, après avoir installé tout ce qui nécessite root
USER www-data
```

Une seule règle : les dossiers que l'application doit écrire (cache,
logs) doivent appartenir à `www-data` *avant* de basculer
d'utilisateur.

```docker
RUN mkdir -p var/cache var/log \
    && chown -R www-data:www-data var/
USER www-data
```

---

## 7. Vue d'ensemble : ce que vous allez construire

```mermaid
graph TB
    subgraph "Votre machine - docker build"
        DF["Dockerfile.prod"]
        DI[".dockerignore"]

        subgraph "Stage base"
            B1["php:8.4-fpm-alpine"]
            B2["pdo_pgsql + intl\nopcache + redis"]
        end

        subgraph "Stage deps"
            D1["Hérite de base"]
            D2["composer install\n--no-dev"]
        end

        subgraph "Stage production"
            P1["Hérite de base"]
            P2["vendor/ (depuis deps)"]
            P3["Code applicatif"]
            P4["OPcache config"]
            P5["USER www-data"]
            P6["EXPOSE 9000"]
        end
    end

    subgraph "Image finale : yapuka-api:prod"
        IMG["~120 MB\nSans Xdebug\nSans tests\nSans Composer"]
    end

    subgraph "Config Nginx"
        NGINX["docker/nginx/prod.conf\nHeaders sécurité\nGzip\nFastCGI vers :9000"]
    end

    DF --> B1 --> B2
    B2 --> D1 --> D2
    D2 --> P1
    P1 --> P2 --> P3 --> P4 --> P5 --> P6
    P6 --> IMG
    DI -.->|" exclut vendor/,\nvar/, tests... "| DF
    IMG --> NGINX
```

---

## Ce dont vous avez besoin pour le lab

Voici un résumé des concepts et instructions indispensables.

### Instructions Dockerfile essentielles

| Instruction         | Rôle                                             |
|---------------------|--------------------------------------------------|
| `FROM image AS nom` | Démarre un stage nommé                           |
| `RUN cmd1 && cmd2`  | Exécute des commandes dans un seul layer         |
| `COPY src dst`      | Copie des fichiers depuis le contexte de build   |
| `COPY --from=stage` | Copie depuis un autre stage ou une image externe |
| `WORKDIR /chemin`   | Définit le répertoire de travail courant         |
| `ENV VAR=valeur`    | Définit une variable d'environnement             |
| `USER nom`          | Change l'utilisateur courant                     |
| `EXPOSE port`       | Documente le port exposé par le conteneur        |

### Commandes Alpine utiles

```bash
# Installer un paquet
apk add --no-cache nom-du-paquet

# Installer des paquets temporaires (compilation)
apk add --no-cache --virtual .build-deps gcc musl-dev

# Supprimer les paquets temporaires après compilation
apk del .build-deps
```

### Optimisation Composer pour la production

```bash
composer install \
    --no-dev \               # Pas de require-dev
    --no-scripts \           # Pas de scripts post-install
    --no-interaction \       # Mode non-interactif
    --optimize-autoloader \  # Classmap pour performances
    --no-progress            # Pas de barre de progression
```

### Variables d'environnement indispensables en production

```
APP_ENV=prod
APP_DEBUG=0
APP_SECRET=<secret-aleatoire-fort>
DATABASE_URL=postgresql://user:pass@host:5432/db
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
JWT_PASSPHRASE=<passphrase>
REDIS_URL=redis://host:6379
CORS_ALLOW_ORIGIN=^https://mondomaine\.com$
```

### Structure des fichiers à créer

```
api/
├── .dockerignore          # Partie 1.2
└── Dockerfile.prod        # Parties 2.1, 2.2, 2.3
docker/
└── nginx/
    └── prod.conf          # Partie 3.1
```