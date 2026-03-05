# Lab 3 — Dockerisation production de l'API Symfony

> **Prérequis** : le projet Yapuka fonctionne en mode développement via `docker compose up -d`. Vous avez une
> connaissance basique de Docker (images, conteneurs, volumes).

## Contexte

L'application Yapuka dispose déjà d'un `Dockerfile` de développement (avec Xdebug, volumes montés, etc.). En production,
les exigences sont différentes : image légère, pas d'outils de debug, assets compilés, utilisateur non-root, et surtout
un **multi-stage build** qui sépare la construction des dépendances de l'image finale.

Dans ce lab, vous allez créer un Dockerfile de production optimisé et une configuration Nginx durcie, puis valider le
tout avec un build local.

> **Multi-stage build** : technique Docker qui utilise plusieurs `FROM` dans un seul Dockerfile. Chaque stage peut
> copier des fichiers du précédent avec `COPY --from=`. Le résultat final ne contient que le dernier stage → image
> beaucoup plus légère.

## Partie 1 — Analyse des dépendances (5 min)

### 1.1 Lister les besoins de l'application

Avant d'écrire un Dockerfile, il faut savoir exactement ce dont l'application a besoin.

**À faire** : ouvrez un shell dans le conteneur PHP actuel et répondez aux questions suivantes.

```bash
docker compose exec php sh
```

1. **Extensions PHP requises** : listez les extensions chargées avec `php -m`. Identifiez celles qui sont spécifiques à
   Yapuka (pas les extensions standard). Indice : regardez ce que le `Dockerfile` de dev installe.

2. **Version PHP** : quelle version de PHP est utilisée ? (`php -v`)

3. **Dépendances Composer** : combien de packages sont installés ? (`composer show | wc -l`). En production, combien y
   en aurait-il sans les `require-dev` ?

4. **Variables d'environnement** : ouvrez `.env` et listez les variables indispensables en production (base de données,
   JWT, Redis, CORS).

5. **Fichiers publics** : quel dossier doit être servi par Nginx ? (`ls public/`)

#### Correction 1.1 {collapsible="true"}

1. **Extensions spécifiques** : `pdo_pgsql` (PostgreSQL), `intl` (internationalisation), `opcache` (cache bytecode),
   `redis` (cache applicatif). En dev, `xdebug` est aussi installé — **à ne pas inclure en production**.

2. **Version** : PHP 8.4 (image `php:8.4-fpm-alpine`).

3. **Dépendances** : en dev il y a ~80+ packages. En production (`composer install --no-dev`), il y en a moins car les
   packages de test (PHPUnit, Behat, Foundry, etc.) sont exclus.

4. **Variables essentielles en production** :

| Variable                            | Exemple production                           |
|-------------------------------------|----------------------------------------------|
| `APP_ENV`                           | `prod`                                       |
| `APP_SECRET`                        | Un vrai secret aléatoire                     |
| `DATABASE_URL`                      | `postgresql://user:pass@db-host:5432/yapuka` |
| `JWT_SECRET_KEY` / `JWT_PUBLIC_KEY` | Chemins vers les clés RSA                    |
| `JWT_PASSPHRASE`                    | Passphrase de la clé privée                  |
| `REDIS_URL`                         | `redis://redis-host:6379`                    |
| `CORS_ALLOW_ORIGIN`                 | `^https://mondomaine\.com$`                  |

5. **Fichiers publics** : `public/index.php` est le seul point d'entrée. Le dossier `public/` peut aussi contenir des
   assets statiques (CSS, JS, images) si l'application en génère — ici c'est une API pure donc il n'y a que `index.php`.

### 1.2 Créer le .dockerignore

Le `.dockerignore` empêche Docker de copier des fichiers inutiles dans le contexte de build, ce qui accélère le build et
réduit la taille de l'image.

**À faire** : créez un fichier `api/.dockerignore` qui exclut :

- Les dossiers de cache et logs Symfony (`var/`)
- Les dépendances (`vendor/`) — elles seront installées dans le build
- Les clés JWT (`config/jwt/`)
- Les fichiers Git, IDE, Docker et CI
- Les tests et fixtures
- Les fichiers `.env.local` et `.env.test`

#### Correction 1.2 {collapsible="true"}

```docker
# api/.dockerignore

# Dépendances (installées dans le build)
/vendor/

# Cache et logs Symfony
/var/

# Clés JWT (montées en secret, pas dans l'image)
/config/jwt/

# Tests (pas en production)
/tests/
/features/
/phpunit.xml.dist
/behat.yml

# Fixtures et fichiers de dev
/.env.local
/.env.test

# Fichiers HTTP de test PHPStorm
/http/

# Git et IDE
.git/
.gitignore
.idea/
.vscode/

# Docker (éviter la récursion)
Dockerfile*
docker-compose*
```

## Partie 2 — Dockerfile multi-stage (15 min)

### 2.1 Stage 1 : image de base

Le premier stage installe les extensions PHP et les dépendances système. Il sert de fondation aux stages suivants.

**À faire** : créez un fichier `api/Dockerfile.prod` et écrivez le premier stage.

**Consignes** :

- Partez de `php:8.4-fpm-alpine` (Alpine = image minimale)
- Nommez ce stage `base` avec `AS base`
- Installez les dépendances système nécessaires pour compiler les extensions PHP : `icu-dev` (pour intl), `libpq-dev` (
  pour pdo_pgsql), `linux-headers` (pour certaines compilations)
- Installez les extensions PHP avec `docker-php-ext-install` : `pdo_pgsql`, `intl`, `opcache`
- Installez l'extension `redis` via PECL (`pecl install redis && docker-php-ext-enable redis`)
- **Ne pas installer Xdebug** (production uniquement)
- Nettoyez les caches APK pour réduire la taille (`rm -rf /var/cache/apk/*`)

> **Astuce Alpine** : les paquets de développement (`*-dev`) ne sont nécessaires que pour la compilation. Vous pouvez
> les marquer comme dépendances virtuelles avec `apk add --virtual .build-deps` puis les supprimer après compilation avec
`apk del .build-deps`.

#### Correction 2.1 {collapsible="true"}

```docker
# =============================================================================
# Stage 1 : Base — Extensions PHP et dépendances système
# =============================================================================
FROM php:8.4-fpm-alpine AS base

# Dépendances système permanentes (runtime)
RUN apk add --no-cache \
    icu-libs \
    libpq \
    && \
    # Dépendances temporaires (compilation uniquement)
    apk add --no-cache --virtual .build-deps \
    icu-dev \
    libpq-dev \
    linux-headers \
    && \
    # Extensions PHP
    docker-php-ext-install -j$(nproc) \
    pdo_pgsql \
    intl \
    opcache \
    && \
    # Extension Redis via PECL
    pecl install redis && docker-php-ext-enable redis \
    && \
    # Nettoyage des dépendances de compilation
    apk del .build-deps \
    && rm -rf /var/cache/apk/* /tmp/*
```

> **Pourquoi une seule instruction `RUN`** : chaque `RUN` crée un layer Docker. En chaînant les commandes avec `&&`, on
> crée un seul layer qui inclut l'installation ET le nettoyage → image plus légère.

### 2.2 Stage 2 : installation des dépendances Composer

Le deuxième stage installe les dépendances PHP via Composer. On le sépare pour bénéficier du cache Docker : tant que
`composer.json` et `composer.lock` ne changent pas, ce stage est mis en cache.

**À faire** : ajoutez un deuxième stage dans le même Dockerfile.

**Consignes** :

- Partez du stage `base` avec `FROM base AS deps`
- Copiez le binaire Composer depuis l'image officielle : `COPY --from=composer:2 /usr/bin/composer /usr/bin/composer`
- Définissez le workdir : `/var/www/api`
- Copiez **uniquement** `composer.json` et `composer.lock` (pas tout le code — pour optimiser le cache)
- Lancez `composer install` avec les flags de production : `--no-dev`, `--no-scripts`, `--no-interaction`,
  `--optimize-autoloader`

> **Pourquoi copier seulement les fichiers Composer d'abord ?** Docker met en cache chaque layer. Si vous copiez tout
> le code puis lancez `composer install`, un changement dans n'importe quel fichier PHP invalide le cache. En copiant
> d'abord les fichiers Composer seuls, le `composer install` n'est relancé que si les dépendances changent.

#### Correction 2.2 {collapsible="true"}

```docker
# =============================================================================
# Stage 2 : Deps — Installation des dépendances Composer
# =============================================================================
FROM base AS deps

# Copier Composer depuis l'image officielle
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/api

# Copier uniquement les fichiers de dépendances (optimisation du cache Docker)
COPY composer.json composer.lock ./

# Installer les dépendances de production uniquement
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-interaction \
    --optimize-autoloader \
    --no-progress
```

### 2.3 Stage 3 : image de production

Le dernier stage assemble l'image finale : le code applicatif + les dépendances installées au stage précédent + la
configuration PHP/OPcache + un utilisateur non-root.

**À faire** : ajoutez le stage final.

**Consignes** :

1. Partez du stage `base` avec `FROM base AS production`
2. Créez un fichier de configuration OPcache optimisé pour la production. Les paramètres importants :
    - `opcache.enable=1`
    - `opcache.memory_consumption=128`
    - `opcache.max_accelerated_files=10000`
    - `opcache.validate_timestamps=0` (ne pas vérifier les fichiers modifiés — on ne modifie jamais en production)
3. Définissez le workdir `/var/www/api`
4. Copiez le dossier `vendor/` depuis le stage `deps`
5. Copiez tout le code applicatif (le `.dockerignore` exclura les fichiers inutiles)
6. Créez les dossiers `var/cache` et `var/log` et donnez-les à l'utilisateur `www-data`
7. Définissez les variables d'environnement par défaut : `APP_ENV=prod`, `APP_DEBUG=0`
8. Passez à l'utilisateur `www-data` (non-root)
9. Exposez le port 9000 (PHP-FPM)

> **Sécurité** : ne jamais tourner en `root` en production. L'utilisateur `www-data` existe déjà dans l'image
> PHP-FPM.

#### Correction 2.3 {collapsible="true"}

```docker
# =============================================================================
# Stage 3 : Production — Image finale optimisée
# =============================================================================
FROM base AS production

# Configuration OPcache pour la production
RUN echo '\
opcache.enable=1\n\
opcache.memory_consumption=128\n\
opcache.interned_strings_buffer=16\n\
opcache.max_accelerated_files=10000\n\
opcache.validate_timestamps=0\n\
opcache.save_comments=1\n\
opcache.fast_shutdown=1\n\
' > /usr/local/etc/php/conf.d/opcache-prod.ini

WORKDIR /var/www/api

# Copier les vendors depuis le stage deps (déjà optimisés, sans --dev)
COPY --from=deps /var/www/api/vendor ./vendor

# Copier le code applicatif (le .dockerignore exclut les fichiers inutiles)
COPY . .

# Créer les dossiers nécessaires et fixer les permissions
RUN mkdir -p var/cache var/log \
    && chown -R www-data:www-data var/

# Variables d'environnement de production
ENV APP_ENV=prod
ENV APP_DEBUG=0

# Passer en utilisateur non-root
USER www-data

# Port PHP-FPM
EXPOSE 9000

CMD ["php-fpm"]
```

**Dockerfile complet** (`api/Dockerfile.prod`) — les 3 stages assemblés :

```docker
# =============================================================================
# Dockerfile de production — Yapuka API
# =============================================================================
# Multi-stage build :
#   1. base       → Extensions PHP, dépendances système
#   2. deps       → Composer install (cache optimisé)
#   3. production → Image finale légère, user non-root
# =============================================================================

# --- Stage 1 : Base ---
FROM php:8.4-fpm-alpine AS base

RUN apk add --no-cache icu-libs libpq \
    && apk add --no-cache --virtual .build-deps \
       icu-dev libpq-dev linux-headers \
    && docker-php-ext-install -j$(nproc) pdo_pgsql intl opcache \
    && pecl install redis && docker-php-ext-enable redis \
    && apk del .build-deps \
    && rm -rf /var/cache/apk/* /tmp/*

# --- Stage 2 : Dépendances Composer ---
FROM base AS deps

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
WORKDIR /var/www/api
COPY composer.json composer.lock ./
RUN composer install \
    --no-dev --no-scripts --no-interaction \
    --optimize-autoloader --no-progress

# --- Stage 3 : Production ---
FROM base AS production

RUN echo '\
opcache.enable=1\n\
opcache.memory_consumption=128\n\
opcache.interned_strings_buffer=16\n\
opcache.max_accelerated_files=10000\n\
opcache.validate_timestamps=0\n\
opcache.save_comments=1\n\
opcache.fast_shutdown=1\n\
' > /usr/local/etc/php/conf.d/opcache-prod.ini

WORKDIR /var/www/api

COPY --from=deps /var/www/api/vendor ./vendor
COPY . .

RUN mkdir -p var/cache var/log \
    && chown -R www-data:www-data var/

ENV APP_ENV=prod
ENV APP_DEBUG=0

USER www-data
EXPOSE 9000
CMD ["php-fpm"]
```

## Partie 3 — Configuration Nginx production (5 min)

### 3.1 Créer la configuration Nginx

En production, Nginx sert les fichiers statiques et passe les requêtes PHP à PHP-FPM. La configuration de dev fonctionne
mais manque de durcissement (headers de sécurité, compression, cache des assets).

**À faire** : créez un fichier `docker/nginx/prod.conf` avec les éléments suivants :

1. **Server block** écoutant sur le port 80
2. **Root** pointant vers `/var/www/api/public`
3. **Location /** avec `try_files $uri /index.php$is_args$args` (toutes les routes Symfony passent par `index.php`)
4. **Location ~ \.php$** qui passe les requêtes à `php:9000` via FastCGI, avec le bon `SCRIPT_FILENAME`
5. **Headers de sécurité** :
    - `X-Content-Type-Options: nosniff`
    - `X-Frame-Options: DENY`
    - `X-XSS-Protection: 1; mode=block`
6. **Compression gzip** activée pour `application/json` et `text/plain`
7. **Désactiver l'accès** aux fichiers `.env`, `.git`, et aux dossiers sensibles
8. **SSE** : désactiver le buffering pour le pattern `/api/notifications/stream`

> Le nom d'hôte `php` sera résolu par Docker Compose si les deux conteneurs sont sur le même réseau.

#### Correction 3.1 {collapsible="true"}

```nginx
# docker/nginx/prod.conf
# =============================================================================
# Configuration Nginx de production pour l'API Symfony
# =============================================================================

server {
    listen 80;
    server_name _;
    root /var/www/api/public;

    # --- Compression ---
    gzip on;
    gzip_types application/json text/plain application/xml;
    gzip_min_length 256;

    # --- Headers de sécurité ---
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # --- Bloquer l'accès aux fichiers sensibles ---
    location ~ /\.(env|git|htaccess) {
        deny all;
        return 404;
    }

    # --- SSE : désactiver le buffering ---
    location = /api/notifications/stream {
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME $document_root/index.php;
        include fastcgi_params;

        fastcgi_buffering off;
        proxy_buffering off;
        fastcgi_read_timeout 3600s;
        fastcgi_param HTTP_X_ACCEL_BUFFERING no;
    }

    # --- Toutes les routes passent par index.php ---
    location / {
        try_files $uri /index.php$is_args$args;
    }

    # --- Passer les requêtes PHP à PHP-FPM ---
    location ~ ^/index\.php(/|$) {
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # Cacher le header X-Powered-By
        fastcgi_hide_header X-Powered-By;

        # En production, bloquer l'accès direct à index.php dans l'URL
        internal;
    }

    # --- Bloquer l'accès à tout autre fichier .php ---
    location ~ \.php$ {
        return 404;
    }

    # --- Logs ---
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
}
```

**Points clés par rapport à la config de dev** :

- `internal` sur le location PHP → empêche l'accès direct à `index.php` dans l'URL
- Blocage des `.env`, `.git` et autres fichiers sensibles
- Headers de sécurité ajoutés
- Compression gzip pour réduire la bande passante
- Pas de proxy vers Vite (le frontend est servi séparément en production)

## Partie 4 — Build et validation (5 min)

### 4.1 Builder l'image

**À faire** :

1. Buildez l'image de production depuis le dossier `api/` :

```bash
docker build -f Dockerfile.prod -t yapuka-api:prod .
```

2. Vérifiez la taille de l'image :

```bash
docker images yapuka-api:prod
```

3. Comparez avec l'image de dev :

```bash
docker images | grep yapuka
```

> L'image de production devrait être **significativement plus petite** que l'image de dev (pas de Xdebug, pas de
> packages `--dev`, pas de tests).

#### Correction 4.1 {collapsible="true"}

```bash
$ docker build -f Dockerfile.prod -t yapuka-api:prod .
[+] Building 45.2s (16/16) FINISHED
 => [base] FROM php:8.4-fpm-alpine
 => [base] RUN apk add ...
 => [deps] COPY composer.json composer.lock
 => [deps] RUN composer install --no-dev ...
 => [production] COPY --from=deps vendor
 => [production] COPY . .
 => exporting to image

$ docker images yapuka-api
REPOSITORY    TAG    SIZE
yapuka-api    prod   ~120MB
```

La taille varie selon les dépendances, mais une image Alpine avec les extensions PHP typiques fait entre 100 et 150 MB.
L'image de dev avec Xdebug et les packages de test peut faire 200+ MB.

**Si le build échoue** :

| Erreur                            | Solution                                                              |
|-----------------------------------|-----------------------------------------------------------------------|
| `composer.lock not found`         | Assurez-vous que `composer.lock` est commité (pas dans `.gitignore`)  |
| `failed to fetch icu-dev`         | Ajoutez `--repository=http://dl-cdn.alpinelinux.org/alpine/edge/main` |
| `pecl install redis` timeout      | Ajoutez `--retry 3` ou vérifiez votre connexion réseau                |
| `COPY . .` copie trop de fichiers | Vérifiez votre `.dockerignore`                                        |

### 4.2 Inspecter l'image

**À faire** : utilisez ces commandes pour analyser votre image.

```bash
# Voir l'historique des layers (taille de chaque instruction)
docker history yapuka-api:prod

# Vérifier que l'utilisateur est bien www-data
docker run --rm yapuka-api:prod whoami

# Vérifier les extensions PHP installées
docker run --rm yapuka-api:prod php -m

# Vérifier qu'Xdebug n'est PAS installé
docker run --rm yapuka-api:prod php -m | grep xdebug
```

#### Correction 4.2 {collapsible="true"}

```bash
$ docker run --rm yapuka-api:prod whoami
www-data

$ docker run --rm yapuka-api:prod php -m | grep -i xdebug
(aucune sortie = Xdebug n'est pas installé ✅)

$ docker run --rm yapuka-api:prod php -m | grep -E "pdo_pgsql|intl|opcache|redis"
intl
opcache
pdo_pgsql
redis
```

`docker history` montre chaque layer avec sa taille. Les plus gros layers sont typiquement l'installation des extensions
PHP et le `composer install`. Si un layer est anormalement gros, c'est peut-être qu'un nettoyage manque dans le même
`RUN`.

### 4.3 Tester le conteneur

**À faire** : lancez le conteneur avec les variables d'environnement minimales et vérifiez qu'il démarre.

```bash
docker run --rm -d \
  --name yapuka-api-test \
  -e DATABASE_URL="postgresql://yapuka:yapuka@host.docker.internal:5432/yapuka" \
  -e REDIS_URL="redis://host.docker.internal:6379" \
  -e JWT_PASSPHRASE="yapuka_jwt_passphrase" \
  -e APP_SECRET="test-secret-change-me" \
  -e CORS_ALLOW_ORIGIN="^https?://localhost(:[0-9]+)?$" \
  yapuka-api:prod

# Vérifier que le conteneur tourne
docker ps | grep yapuka-api-test

# Vérifier les logs PHP-FPM
docker logs yapuka-api-test

# Arrêter et nettoyer
docker stop yapuka-api-test
```

> Ce test ne permet pas de faire de requêtes HTTP car il n'y a pas de Nginx devant. C'est normal — l'objectif est de
> vérifier que PHP-FPM démarre sans erreur. Le prochain lab (Docker Compose production) assemblera Nginx + PHP.

#### Correction 4.3 {collapsible="true"}

```bash
$ docker run --rm -d --name yapuka-api-test \
  -e DATABASE_URL="postgresql://yapuka:yapuka@host.docker.internal:5432/yapuka" \
  -e REDIS_URL="redis://host.docker.internal:6379" \
  -e JWT_PASSPHRASE="yapuka_jwt_passphrase" \
  -e APP_SECRET="test-secret-change-me" \
  -e CORS_ALLOW_ORIGIN="^https?://localhost(:[0-9]+)?$" \
  yapuka-api:prod

$ docker logs yapuka-api-test
[05-Feb-2026 14:00:00] NOTICE: fpm is running, pid 1
[05-Feb-2026 14:00:00] NOTICE: ready to handle connections
```

Si vous voyez `ready to handle connections`, PHP-FPM a démarré correctement.

**Erreurs possibles** :

| Log                                 | Cause                                     | Solution                                                                                                               |
|-------------------------------------|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| `Permission denied on var/cache`    | Le dossier n'a pas les bonnes permissions | Vérifiez le `chown -R www-data:www-data var/` dans le Dockerfile                                                       |
| `APP_ENV=prod requires APP_DEBUG=0` | Variable manquante                        | Déjà défini dans le Dockerfile via `ENV`, ne devrait pas arriver                                                       |
| `Unable to load class App\Kernel`   | Autoloader cassé                          | Le `composer install --optimize-autoloader` a peut-être échoué. Vérifiez que `vendor/autoload.php` existe dans l'image |

## Critères de validation

| # | Critère                                                                                                         | Points  |
|---|-----------------------------------------------------------------------------------------------------------------|---------|
| 1 | Le `.dockerignore` exclut les fichiers de dev, tests, et secrets                                                | /2      |
| 2 | Le Dockerfile utilise un multi-stage build avec 3 stages nommés                                                 | /3      |
| 3 | Le stage `base` installe les 4 extensions (pdo_pgsql, intl, opcache, redis) et nettoie les dépendances de build | /3      |
| 4 | Le stage `deps` copie d'abord `composer.json` + `.lock` puis installe en `--no-dev`                             | /2      |
| 5 | Le stage `production` utilise un user non-root (`www-data`), configure OPcache, et expose le port 9000          | /3      |
| 6 | Xdebug n'est **pas** dans l'image de production                                                                 | /2      |
| 7 | La config Nginx inclut les headers de sécurité, le gzip, et le blocage des fichiers sensibles                   | /3      |
| 8 | `docker build` réussit et `docker run` affiche "ready to handle connections"                                    | /2      |
|   | **Total**                                                                                                       | **/20** |

## Pour aller plus loin

Si vous avez terminé en avance :

- **Healthcheck** : ajoutez une instruction `HEALTHCHECK` dans le Dockerfile qui vérifie que PHP-FPM répond (via
  `php-fpm-healthcheck` ou un script custom `php -r "echo 'ok';"`).
- **Docker Compose production** : créez un `docker-compose.prod.yml` qui assemble Nginx + PHP + PostgreSQL + Redis en
  utilisant vos images de production.
- **Scan de vulnérabilités** : lancez `docker scout cves yapuka-api:prod` (ou Trivy) pour détecter les CVE dans l'image.
- **Build args** : paramétrez la version PHP avec un `ARG PHP_VERSION=8.4` pour faciliter les mises à jour.