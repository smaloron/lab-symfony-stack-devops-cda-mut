# Lab 8 — Déploiement sur Render (Backend) et Vercel (Frontend)

> **Objectif** : Déployer l'application Yapuka en production avec le backend Symfony sur Render et le frontend React sur Vercel, puis automatiser le tout via GitHub Actions.

---

## 8.1 Préparation des Comptes et Outils

### Objectifs {id="objectifs_1"}
- Créer les comptes sur Render et Vercel
- Installer les CLI nécessaires
- Connecter les comptes à GitHub

### Tâches {id="t-ches_1"}
- [ ] Créer un compte Render (https://render.com)
- [ ] Créer un compte Vercel (https://vercel.com)
- [ ] Installer Vercel CLI
- [ ] Connecter Render à votre dépôt GitHub
- [ ] Connecter Vercel à votre dépôt GitHub
- [ ] Comprendre les concepts clés (Render : Web Services, Databases ; Vercel : Deployments, Edge Network)
- [ ] Documenter la configuration

### Installation des CLI
```bash
# Vercel CLI
npm install -g vercel

# Vérification
vercel --version
vercel login

# Render n'a pas de CLI officielle — tout se fait via le dashboard ou l'API
```

### Livrables {id="livrables_1"}
- Comptes Render et Vercel actifs
- Vercel CLI installé
- Connexions GitHub établies
- Documentation

---

## 8.2 Architecture de Déploiement

### Objectifs {id="objectifs_2"}
- Concevoir l'architecture cloud cible
- Identifier les services nécessaires sur chaque plateforme

### Tâches {id="t-ches_2"}
- [ ] Schématiser l'architecture de déploiement
- [ ] Identifier les services Render nécessaires
- [ ] Identifier la configuration Vercel nécessaire
- [ ] Définir la stratégie de communication frontend ↔ backend
- [ ] Planifier les environnements (staging / production)
- [ ] Documenter l'architecture

### Architecture Cible

```
┌─────────────────────────────────────────────────────────┐
│                       INTERNET                          │
└───────────┬─────────────────────────────┬───────────────┘
            │                             │
            ▼                             ▼
┌───────────────────────┐   ┌─────────────────────────────┐
│       VERCEL           │   │          RENDER              │
│  (Frontend React)      │   │  (Backend Symfony)           │
│                        │   │                              │
│  - Build statique Vite │   │  ┌────────────────────────┐ │
│  - CDN global          │   │  │  Web Service (Docker)  │ │
│  - Variables d'env     │   │  │  PHP-FPM + Nginx       │ │
│  - Rewrites → /api     │   │  └────────────┬───────────┘ │
│                        │   │               │              │
└────────────────────────┘   │  ┌────────────▼───────────┐ │
                             │  │   PostgreSQL (Managed)  │ │
                             │  └────────────────────────┘ │
                             │  ┌────────────────────────┐ │
                             │  │   Redis (Managed)       │ │
                             │  └────────────────────────┘ │
                             └─────────────────────────────┘
```

```mermaid
graph TD
    NET(("🌐 Internet"))

    subgraph VER["VERCEL — Frontend React"]
        direction TB
        V1["Build Vite"] --> V2["CDN global"]
        V2 --> V3["Rewrites → /api"]
    end

    subgraph REN["RENDER&nbsp;—&nbsp;Backend Symfony"]
        direction TB
        subgraph DOC["Docker"]
            R1["PHP-FPM + Nginx"]
        end
        R2[("PostgreSQL")]
        R3[("Redis")]
        R1 --> R2
        R1 --> R3
    end

    NET --> VER
    NET --> REN
```



### Services Render
| Service | Type Render | Plan |
|---------|------------|------|
| Backend Symfony | Web Service (Docker) | Free / Starter |
| PostgreSQL | Managed Database | Free (90 jours) / Starter |
| Redis | Managed Redis | Free / Starter |

### Service Vercel
| Service | Type | Plan |
|---------|------|------|
| Frontend React | Static / Serverless | Hobby (gratuit) |

### Livrables {id="livrables_2"}
- Schéma d'architecture
- Choix des services justifié
- Documentation

---

## 8.3 Création des Services sur Render

### Objectifs {id="objectifs_3"}
- Créer la base de données PostgreSQL
- Créer l'instance Redis
- Créer le Web Service pour le backend

### Tâches {id="t-ches_3"}
- [ ] Créer une base de données PostgreSQL sur Render
- [ ] Noter l'URL de connexion (`Internal Database URL` et `External Database URL`)
- [ ] Créer une instance Redis sur Render
- [ ] Noter l'URL de connexion Redis
- [ ] Créer un Web Service de type Docker pour le backend
- [ ] Connecter le Web Service au dépôt GitHub
- [ ] Configurer le répertoire racine (`api/`)
- [ ] Documenter toutes les URLs et identifiants

### Création via le Dashboard Render

#### 1. PostgreSQL
```
Dashboard → New → PostgreSQL
  - Name : yapuka-db
  - Database : yapuka
  - User : yapuka
  - Region : Frankfurt (EU) ou Oregon (US)
  - Plan : Free (pour commencer)
```

#### 2. Redis
```
Dashboard → New → Redis
  - Name : yapuka-redis
  - Region : même région que PostgreSQL
  - Plan : Free (pour commencer)
```

#### 3. Web Service (Backend)
```
Dashboard → New → Web Service
  - Source : dépôt GitHub yapuka
  - Name : yapuka-api
  - Region : même région
  - Runtime : Docker
  - Root Directory : api
  - Plan : Free (pour commencer)
```

### Livrables {id="livrables_3"}
- Base PostgreSQL créée et accessible
- Instance Redis créée et accessible
- Web Service créé et connecté au dépôt
- URLs et identifiants documentés

---

## 8.4 Préparation du Backend Symfony pour Render

### Objectifs {id="objectifs_4"}
- Adapter le Dockerfile pour la production
- Créer la configuration Nginx de production
- Configurer les scripts de démarrage et de migration

### Tâches {id="t-ches_4"}
- [ ] Créer un `Dockerfile.prod` optimisé pour la production dans `api/`
- [ ] Créer un fichier `nginx.conf` de production dans `api/`
- [ ] Créer un script de démarrage `start.sh` dans `api/`
- [ ] Adapter `composer.json` si nécessaire (scripts post-install)
- [ ] Configurer la génération des clés JWT au démarrage
- [ ] Tester le build Docker en local
- [ ] Documenter les changements

### Dockerfile de Production (`api/Dockerfile.prod`)

```docker
# =============================================================================
# Dockerfile de production - Backend Symfony (Nginx + PHP-FPM)
# =============================================================================
# Image multi-stage optimisée pour la production :
#   1. Installation des dépendances Composer (sans dev)
#   2. Image finale avec Nginx + PHP-FPM
# =============================================================================

# --- Étape 1 : Installation des dépendances ---
FROM composer:latest AS composer_stage
WORKDIR /app
COPY composer.json composer.lock* ./
RUN composer install --no-dev --no-scripts --no-interaction --optimize-autoloader
COPY . .
RUN composer dump-autoload --optimize --no-dev

# --- Étape 2 : Image de production ---
FROM php:8.4-fpm-alpine

# Installer les dépendances système
RUN apk add --no-cache \
    nginx \
    supervisor \
    postgresql-dev \
    icu-dev \
    && docker-php-ext-install pdo_pgsql intl opcache

# Installer l'extension Redis
RUN apk add --no-cache $PHPIZE_DEPS \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && apk del $PHPIZE_DEPS

# Configuration PHP pour la production
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"
COPY docker/php/opcache.ini /usr/local/etc/php/conf.d/opcache.ini

# Configuration Nginx
COPY docker/nginx/nginx.prod.conf /etc/nginx/nginx.conf

# Configuration Supervisor (lance Nginx + PHP-FPM ensemble)
COPY docker/supervisor/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Copier le code source avec les dépendances
WORKDIR /var/www/api
COPY --from=composer_stage /app .

# Permissions
RUN mkdir -p var/cache var/log config/jwt \
    && chown -R www-data:www-data var/ config/jwt

# Script de démarrage
COPY start.sh /start.sh
RUN chmod +x /start.sh

# Port dynamique (Render injecte la variable $PORT)
EXPOSE 10000

CMD ["/start.sh"]
```

### Script de démarrage (`api/start.sh`)

```bash
#!/bin/sh
# =============================================================================
# Script de démarrage - Exécuté au lancement du conteneur sur Render
# =============================================================================

set -e

echo "=== Démarrage de Yapuka API ==="

# Générer les clés JWT si elles n'existent pas
if [ ! -f config/jwt/private.pem ]; then
    echo "Génération des clés JWT..."
    php bin/console lexik:jwt:generate-keypair --skip-if-exists --no-interaction
fi

# Vider le cache Symfony
echo "Nettoyage du cache..."
php bin/console cache:clear --env=prod --no-debug

# Exécuter les migrations
echo "Exécution des migrations..."
php bin/console doctrine:migrations:migrate --no-interaction --allow-no-migration

echo "=== Application prête ==="

# Remplacer le port Nginx par celui fourni par Render ($PORT)
sed -i "s/RENDER_PORT/${PORT:-10000}/g" /etc/nginx/nginx.conf

# Lancer Supervisor (Nginx + PHP-FPM)
exec /usr/bin/supervisord -c /etc/supervisor/conf.d/supervisord.conf
```

### Configuration Nginx de Production (`api/docker/nginx/nginx.prod.conf`)

```nginx
# =============================================================================
# Configuration Nginx de production pour Render
# =============================================================================

worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout 65;

    # Logs au format JSON (facilite le parsing dans les outils de monitoring)
    log_format json_combined escape=json '{"time":"$time_iso8601","remote_addr":"$remote_addr","method":"$request_method","uri":"$request_uri","status":$status,"body_bytes_sent":$body_bytes_sent}';
    access_log /dev/stdout json_combined;
    error_log /dev/stderr warn;

    server {
        # Render injecte le port via la variable $PORT
        listen RENDER_PORT;
        server_name _;
        root /var/www/api/public;

        location / {
            try_files $uri /index.php$is_args$args;
        }

        location ~ ^/index\.php(/|$) {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_split_path_info ^(.+\.php)(/.*)$;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            fastcgi_param DOCUMENT_ROOT $realpath_root;

            # Timeout étendu pour SSE
            fastcgi_read_timeout 300;
            fastcgi_buffering off;
            internal;
        }

        # Bloquer l'accès aux autres fichiers PHP
        location ~ \.php$ {
            return 404;
        }
    }
}
```

### Configuration Supervisor (`api/docker/supervisor/supervisord.conf`)

```ini
; =============================================================================
; Supervisor - Lance Nginx et PHP-FPM dans un même conteneur
; =============================================================================

[supervisord]
nodaemon=true
logfile=/dev/null
logfile_maxbytes=0
pidfile=/run/supervisord.pid

[program:php-fpm]
command=php-fpm -F
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:nginx]
command=nginx -g "daemon off;"
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

### Configuration OPcache Production (`api/docker/php/opcache.ini`)

```ini
; =============================================================================
; OPcache - Configuration optimisée pour la production
; =============================================================================
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.preload_user=www-data
```

### Livrables {id="livrables_4"}
- `Dockerfile.prod` créé et testé
- `start.sh` créé
- `nginx.prod.conf` créé
- `supervisord.conf` créé
- `opcache.ini` créé
- Build Docker fonctionnel en local
- Documentation

---

## 8.5 Préparation du Frontend React pour Vercel

### Objectifs {id="objectifs_5"}
- Adapter le frontend React/Vite pour Vercel
- Configurer les rewrites API
- Configurer le build de production

### Tâches {id="t-ches_5"}
- [ ] Créer le fichier `vercel.json` dans `front/`
- [ ] Adapter le client API pour la production (variable d'environnement)
- [ ] Configurer les rewrites pour les appels API
- [ ] Vérifier le build de production en local (`npm run build`)
- [ ] Tester le build avec `vercel dev` (optionnel)
- [ ] Documenter les changements

### Configuration Vercel (`front/vercel.json`)

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "installCommand": "npm install",
  "framework": "vite",
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://yapuka-api.onrender.com/api/:path*"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" }
      ]
    }
  ]
}
```

> **Note** : L'URL de destination dans `rewrites` sera remplacée par l'URL réelle de votre service Render.

### Adaptation du Client API (`front/src/api/client.js`)

```javascript
// En production sur Vercel, les appels /api sont redirigés via les rewrites
// En développement, on utilise VITE_API_URL
const API_BASE_URL = import.meta.env.VITE_API_URL || '';
```

> Le code existant est déjà compatible. La variable `VITE_API_URL` sera vide en production (les rewrites Vercel gèrent le routage), et définie en développement local.

### Vérification du Build Local

```bash
cd front

# Build de production
npm run build

# Prévisualiser le build
npm run preview
```

### Livrables {id="livrables_5"}
- `vercel.json` créé
- Build de production fonctionnel
- Documentation

---

## 8.6 Configuration des Variables d'Environnement

### Objectifs {id="objectifs_6"}
- Configurer toutes les variables d'environnement sur Render et Vercel
- Sécuriser les secrets

### Tâches {id="t-ches_6"}
- [ ] Lister toutes les variables nécessaires (backend et frontend)
- [ ] Récupérer `DATABASE_URL` depuis le service PostgreSQL Render
- [ ] Récupérer `REDIS_URL` depuis le service Redis Render
- [ ] Configurer les variables d'environnement sur Render (backend)
- [ ] Configurer les variables d'environnement sur Vercel (frontend)
- [ ] Vérifier la configuration
- [ ] Documenter toutes les variables

### Variables d'Environnement Backend (Render)

Configurer dans : **Render Dashboard → yapuka-api → Environment**

| Variable | Valeur | Description |
|----------|--------|-------------|
| `APP_ENV` | `prod` | Environnement Symfony |
| `APP_SECRET` | *(générer avec `openssl rand -hex 32`)* | Secret Symfony |
| `DATABASE_URL` | *(fournie automatiquement par Render PostgreSQL)* | URL de connexion PostgreSQL |
| `REDIS_URL` | *(fournie automatiquement par Render Redis)* | URL de connexion Redis |
| `CORS_ALLOW_ORIGIN` | `^https://yapuka-front\.vercel\.app$` | Origines CORS autorisées |
| `JWT_PASSPHRASE` | *(générer un passphrase aléatoire)* | Passphrase des clés JWT |

> **Important** : Si vous utilisez les services Render internes, les variables `DATABASE_URL` et `REDIS_URL` peuvent être liées automatiquement via les "Environment Groups" ou les "Internal URLs" dans le dashboard.

### Variables d'Environnement Frontend (Vercel)

Configurer dans : **Vercel Dashboard → yapuka-front → Settings → Environment Variables**

| Variable | Valeur | Environnement |
|----------|--------|---------------|
| `VITE_API_URL` | *(vide ou non définie)* | Production |
| `VITE_API_URL` | `http://localhost:8080` | Preview / Development |

> En production, les rewrites Vercel redirigent `/api/*` vers le backend Render. La variable `VITE_API_URL` n'est donc pas nécessaire.

### Vérification

```bash
# Vérifier les variables Render (via le dashboard)
# Render Dashboard → yapuka-api → Environment → vérifier toutes les variables

# Vérifier les variables Vercel
vercel env ls
```

### Livrables {id="livrables_6"}
- Variables d'environnement configurées sur Render
- Variables d'environnement configurées sur Vercel
- Secrets sécurisés (jamais dans le code source)
- Documentation complète

---

## 8.7 Premier Déploiement Manuel

### Objectifs {id="objectifs_7"}
- Déployer le backend sur Render
- Déployer le frontend sur Vercel
- Valider le fonctionnement de bout en bout

### Tâches {id="t-ches_7"}
- [ ] Déclencher le déploiement backend sur Render (push ou manuel)
- [ ] Observer les logs de build sur Render
- [ ] Vérifier que le backend répond (healthcheck)
- [ ] Déployer le frontend sur Vercel
- [ ] Observer les logs de build sur Vercel
- [ ] Tester l'application complète en production
- [ ] Documenter la procédure

### Déploiement Backend (Render)

```bash
# Option 1 : Déploiement automatique (si auto-deploy activé)
# Render détecte automatiquement les push sur la branche main

# Option 2 : Déploiement manuel via le dashboard
# Render Dashboard → yapuka-api → Manual Deploy → Deploy latest commit

# Vérification des logs
# Render Dashboard → yapuka-api → Logs
```

**Points à vérifier dans les logs Render :**
1. Build Docker réussi
2. Script `start.sh` exécuté
3. Clés JWT générées
4. Migrations exécutées
5. Nginx et PHP-FPM démarrés

### Déploiement Frontend (Vercel)

```bash
# Option 1 : Déploiement via CLI
cd front
vercel --prod

# Option 2 : Déploiement automatique
# Vercel détecte automatiquement les push sur la branche main
# si le projet est connecté au dépôt GitHub

# Option 3 : Via le dashboard Vercel
# Vercel Dashboard → yapuka-front → Deployments → Redeploy
```

**Configuration du projet Vercel (première fois) :**
```bash
cd front
vercel

# Répondre aux questions :
# - Set up and deploy? → Yes
# - Which scope? → votre compte
# - Link to existing project? → No (première fois)
# - Project name? → yapuka-front
# - Root directory? → ./ (on est déjà dans front/)
# - Override settings? → No
```

### Vérification Post-Déploiement

```bash
# Tester le backend
curl https://yapuka-api.onrender.com/api/docs
# → doit retourner la documentation Swagger

# Tester l'authentification
curl -X POST https://yapuka-api.onrender.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@yapuka.dev","password":"password"}'
# → doit retourner un JWT

# Tester le frontend
# Ouvrir https://yapuka-front.vercel.app dans le navigateur
# → doit afficher la page de connexion
```

### Livrables {id="livrables_7"}
- Backend déployé et accessible sur Render
- Frontend déployé et accessible sur Vercel
- Communication frontend ↔ backend fonctionnelle
- Documentation

---

## 8.8 Exécution des Migrations et Données Initiales

### Objectifs {id="objectifs_8"}
- Vérifier l'état de la base de données
- Charger les données de démonstration (fixtures)

### Tâches {id="t-ches_8"}
- [ ] Vérifier que les migrations ont été exécutées (via `start.sh`)
- [ ] Charger les fixtures si nécessaire
- [ ] Vérifier l'état de la base de données
- [ ] Tester la connexion avec les données de démo
- [ ] Documenter la procédure

### Commandes via Render Shell

Render permet d'exécuter des commandes dans le conteneur via le dashboard :

```
Render Dashboard → yapuka-api → Shell
```

```bash
# Vérifier l'état des migrations
php bin/console doctrine:migrations:status

# Exécuter les migrations manuellement (si nécessaire)
php bin/console doctrine:migrations:migrate --no-interaction

# Charger les fixtures (données de démonstration)
php bin/console doctrine:fixtures:load --no-interaction

# Vérifier que l'utilisateur de démo existe
php bin/console doctrine:query:sql "SELECT email, username FROM users LIMIT 5"
```

### Livrables {id="livrables_8"}
- Base de données initialisée
- Migrations exécutées
- Données de démonstration chargées
- Documentation

---

## 8.9 Configuration du CI/CD avec GitHub Actions

### Objectifs {id="objectifs_9"}
- Automatiser les tests avant chaque déploiement
- Automatiser le déploiement backend sur Render
- Automatiser le déploiement frontend sur Vercel
- Créer un pipeline complet : Tests → Build → Deploy

### Tâches {id="t-ches_9"}
- [ ] Créer le workflow GitHub Actions principal
- [ ] Configurer les secrets GitHub (clés API Render et Vercel)
- [ ] Configurer le job de tests backend (PHPUnit)
- [ ] Configurer le job de tests frontend (lint)
- [ ] Configurer le job de déploiement Render (deploy hook)
- [ ] Configurer le job de déploiement Vercel
- [ ] Tester le pipeline complet
- [ ] Documenter le workflow

### Secrets GitHub à Configurer

Configurer dans : **GitHub → Settings → Secrets and variables → Actions**

| Secret | Description | Où le trouver |
|--------|-------------|---------------|
| `RENDER_API_KEY` | Clé API Render | Render Dashboard → Account Settings → API Keys |
| `RENDER_SERVICE_ID` | ID du Web Service backend | Render Dashboard → yapuka-api → Settings (URL : `srv-xxxxx`) |
| `RENDER_DEPLOY_HOOK_URL` | URL du Deploy Hook | Render Dashboard → yapuka-api → Settings → Deploy Hook |
| `VERCEL_TOKEN` | Token d'accès Vercel | Vercel Dashboard → Settings → Tokens |
| `VERCEL_ORG_ID` | ID de l'organisation Vercel | Fichier `.vercel/project.json` après `vercel link` |
| `VERCEL_PROJECT_ID` | ID du projet Vercel | Fichier `.vercel/project.json` après `vercel link` |

### Workflow GitHub Actions (`.github/workflows/ci-cd.yml`)

```yaml
# =============================================================================
# Pipeline CI/CD - Tests, Build et Déploiement
# =============================================================================
# Ce workflow s'exécute à chaque push sur main et sur les pull requests.
# Il comprend 4 jobs :
#   1. test-backend   : Tests PHPUnit (unitaires + intégration)
#   2. test-frontend  : Lint et build du frontend
#   3. deploy-backend : Déploiement sur Render (uniquement sur main)
#   4. deploy-frontend: Déploiement sur Vercel (uniquement sur main)
# =============================================================================

name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # ===========================================================================
  # Job 1 : Tests Backend (PHPUnit)
  # ===========================================================================
  test-backend:
    name: Tests Backend
    runs-on: ubuntu-latest

    # Services Docker nécessaires pour les tests d'intégration
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: yapuka_test
          POSTGRES_USER: yapuka
          POSTGRES_PASSWORD: yapuka
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U yapuka"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    defaults:
      run:
        working-directory: api

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Installation de PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: pdo_pgsql, intl, redis
          coverage: xdebug

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: api/vendor
          key: composer-${{ hashFiles('api/composer.lock') }}
          restore-keys: composer-

      - name: Installation des dépendances
        run: composer install --no-interaction --prefer-dist

      - name: Génération des clés JWT
        run: php bin/console lexik:jwt:generate-keypair --skip-if-exists --no-interaction
        env:
          APP_ENV: test

      - name: 🗃️ Création de la base de test
        run: |
          php bin/console doctrine:database:create --if-not-exists --env=test
          php bin/console doctrine:migrations:migrate --no-interaction --env=test
        env:
          DATABASE_URL: "postgresql://yapuka:yapuka@localhost:5432/yapuka_test?serverVersion=16"

      - name: Tests unitaires
        run: php bin/phpunit tests/Unit --colors=always
        env:
          APP_ENV: test
          DATABASE_URL: "postgresql://yapuka:yapuka@localhost:5432/yapuka_test?serverVersion=16"
          REDIS_URL: "redis://localhost:6379"

      - name: Tests d'intégration
        run: php bin/phpunit tests/Integration --colors=always
        env:
          APP_ENV: test
          DATABASE_URL: "postgresql://yapuka:yapuka@localhost:5432/yapuka_test?serverVersion=16"
          REDIS_URL: "redis://localhost:6379"

  # ===========================================================================
  # Job 2 : Tests Frontend (Lint + Build)
  # ===========================================================================
  test-frontend:
    name: 🧪 Tests Frontend
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: front

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Installation de Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
          cache-dependency-path: front/package-lock.json

      - name: Installation des dépendances
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Build de production
        run: npm run build
        env:
          VITE_API_URL: ""

  # ===========================================================================
  # Job 3 : Déploiement Backend sur Render
  # ===========================================================================
  deploy-backend:
    name: Déploiement Backend (Render)
    runs-on: ubuntu-latest
    needs: [test-backend, test-frontend]
    # Uniquement sur la branche main (pas sur les PR)
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - name: Déclencher le déploiement Render
        run: |
          curl -X POST "${{ secrets.RENDER_DEPLOY_HOOK_URL }}"

      - name: Attendre le déploiement
        run: |
          echo "Déploiement déclenché sur Render."
          echo "Le déploiement prend généralement 2 à 5 minutes."
          echo "Vérifiez les logs sur : https://dashboard.render.com"

  # ===========================================================================
  # Job 4 : Déploiement Frontend sur Vercel
  # ===========================================================================
  deploy-frontend:
    name: Déploiement Frontend (Vercel)
    runs-on: ubuntu-latest
    needs: [test-backend, test-frontend]
    # Uniquement sur la branche main (pas sur les PR)
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Installation de Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Installation de Vercel CLI
        run: npm install -g vercel

      - name: Pull de la configuration Vercel
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
        working-directory: front
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

      - name: Build de production
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
        working-directory: front
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

      - name: Déploiement sur Vercel
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
        working-directory: front
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
```

### Schéma du Pipeline

```
┌──────────────┐     ┌───────────────┐
│ test-backend │     │ test-frontend │
│  (PHPUnit)   │     │  (Lint+Build) │
└──────┬───────┘     └──────┬────────┘
       │                    │
       └────────┬───────────┘
                │
        ┌───────▼────────┐
        │  Tests passés? │
        └───────┬────────┘
                │ (branche main uniquement)
       ┌────────┴────────┐
       │                 │
┌──────▼───────┐  ┌──────▼────────┐
│deploy-backend│  │deploy-frontend│
│   (Render)   │  │   (Vercel)    │
└──────────────┘  └───────────────┘
```

### Livrables {id="livrables_9"}
- Fichier `.github/workflows/ci-cd.yml` créé
- Secrets GitHub configurés
- Pipeline fonctionnel et testé
- Documentation

---

## 8.10 Configuration CORS et Communication Frontend ↔ Backend

### Objectifs {id="objectifs_10"}
- S'assurer que le frontend Vercel peut communiquer avec le backend Render
- Configurer correctement les CORS

### Tâches {id="t-ches_10"}
- [ ] Mettre à jour `CORS_ALLOW_ORIGIN` sur Render avec l'URL Vercel
- [ ] Vérifier les rewrites Vercel
- [ ] Tester la communication API depuis le frontend
- [ ] Tester l'authentification JWT de bout en bout
- [ ] Documenter la configuration

### Configuration CORS sur Render

```
Variable : CORS_ALLOW_ORIGIN
Valeur   : ^https://yapuka-front\.vercel\.app$
```

> Si vous utilisez un domaine personnalisé, adaptez la regex :
> ```
> ^https://(yapuka-front\.vercel\.app|www\.yapuka\.dev)$
> ```

### Tests de Communication

```bash
# Tester les CORS (depuis un autre domaine)
curl -X OPTIONS https://yapuka-api.onrender.com/api/tasks \
  -H "Origin: https://yapuka-front.vercel.app" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Authorization" \
  -v

# Vérifier les headers CORS dans la réponse :
# Access-Control-Allow-Origin: https://yapuka-front.vercel.app
# Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
# Access-Control-Allow-Headers: Content-Type, Authorization, Accept
```

### Livrables {id="livrables_10"}
- CORS correctement configurés
- Communication frontend ↔ backend validée
- Documentation

---

## 8.11 Monitoring et Logs

### Objectifs {id="objectifs_11"}
- Accéder aux logs des deux plateformes
- Mettre en place un monitoring basique

### Tâches {id="t-ches_11"}
- [ ] Accéder aux logs Render (backend)
- [ ] Accéder aux logs Vercel (frontend)
- [ ] Configurer les alertes Render (downtime)
- [ ] Vérifier les métriques Render (CPU, mémoire, requêtes)
- [ ] Vérifier les analytics Vercel (performances, trafic)
- [ ] Documenter l'accès aux logs et métriques

### Logs Backend (Render)

```
Render Dashboard → yapuka-api → Logs

Filtres disponibles :
  - Par période
  - Par type (deploy, runtime)
  - Recherche textuelle
```

### Logs Frontend (Vercel)

```
Vercel Dashboard → yapuka-front → Logs

Filtres disponibles :
  - Logs de fonctions serverless (si utilisées)
  - Logs de build
  - Analytics Web Vitals
```

### Métriques Render
- **CPU & Mémoire** : Dashboard → Service → Metrics
- **Temps de réponse** : Dashboard → Service → Metrics → Response Time
- **Alertes** : Dashboard → Service → Settings → Notifications

### Analytics Vercel
- **Web Vitals** : Dashboard → Analytics
- **Performances** : Dashboard → Speed Insights
- **Trafic** : Dashboard → Usage

### Livrables {id="livrables_11"}
- Accès aux logs documenté
- Métriques accessibles
- Alertes configurées (si applicable)
- Documentation

---

## 8.12 Backups et Restauration (Base de Données)

### Objectifs {id="objectifs_12"}
- Comprendre les sauvegardes automatiques de Render PostgreSQL
- Savoir effectuer une sauvegarde manuelle
- Tester la restauration

### Tâches {id="t-ches_12"}
- [ ] Vérifier les sauvegardes automatiques Render
- [ ] Effectuer une sauvegarde manuelle
- [ ] Télécharger un backup
- [ ] Tester la restauration (sur une base de staging)
- [ ] Documenter les procédures

### Sauvegardes Render PostgreSQL

```
Render Dashboard → yapuka-db → Backups

Options :
  - Backups automatiques (quotidiens sur les plans payants)
  - Backup manuel (clic sur "Create Backup")
  - Restauration depuis un backup existant
```

### Backup Manuel via psql

```bash
# Exporter la base de données
pg_dump -h <RENDER_DB_HOST> -U yapuka -d yapuka -F c -f backup.dump

# Restaurer
pg_restore -h <RENDER_DB_HOST> -U yapuka -d yapuka backup.dump
```

> **Note** : L'URL de connexion externe de la base PostgreSQL est disponible dans le dashboard Render.

### Livrables {id="livrables_12"}
- Procédure de backup documentée
- Restauration testée
- Documentation

---

## 8.13 Domaine Personnalisé (Optionnel)

### Objectifs {id="objectifs_13"}
- Configurer un domaine personnalisé
- Configurer le SSL

### Tâches {id="t-ches_13"}
- [ ] Choisir un domaine (ex : `yapuka.dev`)
- [ ] Configurer le domaine sur Vercel (frontend)
- [ ] Configurer le domaine sur Render (backend API)
- [ ] Configurer les DNS
- [ ] Vérifier les certificats SSL (automatiques)
- [ ] Mettre à jour `CORS_ALLOW_ORIGIN`
- [ ] Mettre à jour les rewrites Vercel
- [ ] Documenter la configuration

### Configuration sur Vercel

```
Vercel Dashboard → yapuka-front → Settings → Domains → Add Domain
  - Domaine : www.yapuka.dev
  - DNS : ajouter un CNAME pointant vers cname.vercel-dns.com
```

### Configuration sur Render

```
Render Dashboard → yapuka-api → Settings → Custom Domains → Add Domain
  - Domaine : api.yapuka.dev
  - DNS : ajouter un CNAME pointant vers l'URL fournie par Render
```

### Mise à Jour des Variables

```
# Render : mettre à jour CORS
CORS_ALLOW_ORIGIN=^https://(www\.yapuka\.dev|yapuka\.dev)$

# Vercel : mettre à jour les rewrites dans vercel.json
"destination": "https://api.yapuka.dev/api/:path*"
```

### Livrables {id="livrables_13"}
- Domaines configurés (si applicable)
- SSL actif (automatique sur les deux plateformes)
- Documentation

---

## 8.14 Tests Post-Déploiement

### Objectifs {id="objectifs_14"}
- Valider le fonctionnement complet de l'application en production
- Vérifier les performances

### Tâches {id="t-ches_14"}
- [ ] Tester l'inscription d'un nouvel utilisateur
- [ ] Tester la connexion avec les identifiants de démo
- [ ] Tester le CRUD complet des tâches (créer, lire, modifier, supprimer)
- [ ] Vérifier les statistiques du dashboard
- [ ] Vérifier la connexion SSE (notifications)
- [ ] Tester la déconnexion
- [ ] Vérifier les temps de réponse de l'API
- [ ] Vérifier le chargement de l'interface
- [ ] Documenter les résultats

### Checklist de Validation

| Test | URL / Action | Résultat attendu |
|------|-------------|-----------------|
| Page de connexion | `https://yapuka-front.vercel.app/login` | Formulaire affiché |
| Connexion démo | `demo@yapuka.dev` / `password` | Redirection vers le dashboard |
| Liste des tâches | Dashboard | Tâches affichées |
| Création de tâche | Formulaire "Nouvelle tâche" | Tâche ajoutée à la liste |
| Modification | Cliquer sur le crayon | Titre modifié |
| Suppression | Cliquer sur la corbeille | Tâche supprimée après confirmation |
| Statistiques | Widgets en haut du dashboard | Compteurs corrects |
| Graphique | Section "Répartition par statut" | Graphique affiché |
| Déconnexion | Bouton "Déconnexion" | Retour à la page de connexion |
| API Docs | `/api/docs` | Documentation Swagger |

### Livrables {id="livrables_14"}
- Tous les tests passés
- Rapport de validation
- Documentation

---

## 8.15 Scaling et Performances

### Objectifs {id="objectifs_15"}
- Comprendre les options de scaling sur Render et Vercel
- Optimiser les performances si nécessaire

### Tâches {id="t-ches_15"}
- [ ] Analyser les métriques de performance
- [ ] Comprendre le scaling sur Render (vertical et horizontal)
- [ ] Comprendre le scaling Vercel (automatique via CDN)
- [ ] Optimiser le cold start Render si nécessaire
- [ ] Documenter les options de scaling

### Scaling Backend (Render)

```
Render Dashboard → yapuka-api → Settings

Options :
  - Plan : Free → Starter → Standard → Pro
  - Instances : 1 → N (scaling horizontal, plans payants)
  - Auto-scaling : disponible sur les plans Standard+
```

> **Note sur le plan Free Render** : Le service s'endort après 15 minutes d'inactivité. Le premier appel après une période d'inactivité prend ~30 secondes (cold start). Les plans payants maintiennent le service actif en permanence.

### Scaling Frontend (Vercel)

Le frontend sur Vercel bénéficie automatiquement :
- Du CDN global (Edge Network)
- Du scaling automatique
- De la mise en cache des assets statiques

Aucune configuration manuelle nécessaire.

### Livrables
- Métriques analysées
- Options de scaling documentées
- Optimisations appliquées si nécessaire
- Documentation

---

## 8.16 Documentation Finale et Troubleshooting

### Objectifs
- Créer la documentation complète de déploiement
- Préparer un guide de dépannage

### Tâches
- [ ] Documenter toute la procédure de déploiement
- [ ] Créer un guide pas à pas pour un nouveau développeur
- [ ] Lister les commandes essentielles
- [ ] Créer un guide de dépannage (troubleshooting)
- [ ] Documenter les problèmes courants et leurs solutions
- [ ] Créer un runbook pour les opérations courantes

### Guide de Troubleshooting

#### Problème : Le backend ne démarre pas sur Render

```bash
# 1. Vérifier les logs
Render Dashboard → yapuka-api → Logs

# 2. Problèmes courants :
# - DATABASE_URL manquante → vérifier les variables d'environnement
# - Erreur de migration → vérifier la connexion PostgreSQL
# - Erreur JWT → vérifier que start.sh génère les clés
# - Port incorrect → vérifier que Nginx utilise $PORT
```

#### Problème : Le frontend ne communique pas avec le backend

```bash
# 1. Vérifier les CORS
# - CORS_ALLOW_ORIGIN doit correspondre à l'URL exacte du frontend Vercel
# - Attention aux trailing slashes

# 2. Vérifier les rewrites Vercel
# - L'URL de destination dans vercel.json doit être correcte
# - Tester directement l'API backend avec curl

# 3. Vérifier les variables d'environnement
# - VITE_API_URL doit être vide en production (rewrites)
```

#### Problème : Cold start lent sur Render (plan Free)

```
# Le plan Free met le service en veille après 15min d'inactivité
# Solutions :
# 1. Passer au plan Starter (7$/mois) pour un service toujours actif
# 2. Configurer un health check externe (ex : UptimeRobot) qui ping
#    le service toutes les 14 minutes pour le maintenir actif
```

#### Problème : Le déploiement GitHub Actions échoue

```bash
# 1. Vérifier les secrets GitHub
# GitHub → Settings → Secrets → vérifier que tous les secrets sont définis

# 2. Vérifier les logs du workflow
# GitHub → Actions → cliquer sur le workflow échoué → lire les logs

# 3. Problèmes courants :
# - RENDER_DEPLOY_HOOK_URL expiré → régénérer dans le dashboard Render
# - VERCEL_TOKEN expiré → régénérer dans Vercel Settings
# - Tests qui échouent → corriger les tests avant le déploiement
```

### Sections de Documentation Finale

1. Prérequis (comptes, CLI, accès)
2. Architecture de déploiement
3. Configuration Render (backend, PostgreSQL, Redis)
4. Configuration Vercel (frontend)
5. Variables d'environnement
6. Pipeline CI/CD (GitHub Actions)
7. Premier déploiement
8. Monitoring et logs
9. Backups et restauration
10. Scaling
11. Domaine personnalisé
12. Troubleshooting

### Commandes Essentielles

```bash
# === RENDER ===
# Les opérations se font via le dashboard ou l'API REST

# === VERCEL ===
vercel                    # Déploiement preview
vercel --prod             # Déploiement production
vercel env ls             # Lister les variables d'environnement
vercel logs               # Voir les logs
vercel domains ls         # Lister les domaines

# === GITHUB ACTIONS ===
# Déclencher un workflow manuellement (si configuré)
gh workflow run ci-cd.yml

# Voir les exécutions récentes
gh run list --workflow=ci-cd.yml

# Voir les logs d'une exécution
gh run view <run-id> --log
```
