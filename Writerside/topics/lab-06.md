# Lab 6 — Pipeline CI/CD avec GitHub Actions

## Objectifs du lab

À l'issue de ce lab, vous serez capable de :

- Créer un workflow GitHub Actions complet pour un projet full-stack
- Configurer des jobs de linting, tests, build et sécurité
- Mettre en cache les dépendances pour accélérer le pipeline
- Builder et pusher des images Docker vers un registry
- Scanner les images pour détecter des vulnérabilités
- Déployer automatiquement sur un environnement de staging


**Prérequis :**
- Le projet Yapuka fonctionnel en local avec Docker Compose
- Un compte GitHub avec un repository contenant le projet
- Connaissances de base en YAML

> **Rappel de l'architecture Yapuka**
>
> Le projet est composé de :
> - Un backend **Symfony 7.2** (PHP 8.4, API Platform, PostgreSQL, Redis)
> - Un frontend **React 18** (Vite, Tailwind CSS)
> - Une orchestration **Docker Compose** (Nginx, PHP-FPM, Node, PostgreSQL, Redis)

---

## Étape 1 — Comprendre le pipeline cible

Avant d'écrire la moindre ligne de YAML, prenez 5 minutes pour comprendre l'architecture du pipeline que vous allez construire.

### Schéma du pipeline

```
  push / PR sur main ou develop
              │
     ┌────────┴────────┐
     ▼                  ▼
 lint-backend      lint-frontend
     │                  │
     └────────┬─────────┘
              ▼
        test-backend
              │
              ▼
        build-images
              │
              ▼
       security-scan
              │
              ▼
      deploy-staging
    (si branche develop)
```

**Règles importantes :**

- Les jobs de lint s'exécutent **en parallèle**
- Les tests ne démarrent que si **les deux lints** réussissent
- Le build ne démarre que si **les tests** réussissent
- Le déploiement ne se fait que sur la branche `develop`

### Questions de réflexion

Avant de passer à la suite, répondez mentalement à ces questions :

1. Pourquoi exécuter les lints en parallèle plutôt qu'en séquence ?
2. Pourquoi bloquer le build si les tests échouent ?
3. Quel est l'intérêt de scanner les images **après** le build ?

---

## Étape 2 — Créer la structure du workflow

### Consignes {id="consignes_1"}

1. Créez le répertoire `.github/workflows/` à la racine du projet
2. Créez un fichier `ci-cd.yml` dans ce répertoire
3. Définissez :
    - Le **nom** du workflow : `CI/CD Pipeline`
    - Les **triggers** : le workflow doit se déclencher sur les `push` et `pull_request` vers les branches `main` et `develop`
    - Une **variable d'environnement globale** `REGISTRY` avec la valeur `ghcr.io` (GitHub Container Registry)

> **Aide — Syntaxe des triggers**
>
> La clé `on` accepte plusieurs événements. Chaque événement peut filtrer sur des branches :
> ```yaml
> on:
>   evenement1:
>     branches: [ branche1, branche2 ]
> ```

### Correction {collapsible="true" id="correction_1"}

```yaml
# =============================================================================
# Pipeline CI/CD - Yapuka
# =============================================================================
# Ce workflow s'exécute à chaque push ou pull request sur main et develop.
# Il enchaîne : lint → tests → build → scan sécurité → déploiement.
# =============================================================================

name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  REGISTRY: ghcr.io
```

---

## Étape 3 — Job : Linting backend (PHP)

### Contexte {id="contexte_1"}

Le linting vérifie la **qualité du code** sans l'exécuter. Pour PHP, deux outils sont courants :

| Outil | Rôle |
|-------|------|
| **PHP CS Fixer** | Vérifie le respect des conventions de style (PSR-12) |
| **PHPStan** | Analyse statique — détecte les bugs potentiels sans exécuter le code |

### Consignes {id="consignes_2"}

Ajoutez un job `lint-backend` qui :

1. S'exécute sur `ubuntu-latest`
2. Utilise l'action `actions/checkout@v4` pour récupérer le code
3. Configure PHP 8.4 avec l'action `shivammathur/setup-php@v2` (extensions : `intl, pdo_pgsql, redis`)
4. **Met en cache** les dépendances Composer pour accélérer les exécutions suivantes
5. Installe les dépendances avec `composer install`
6. Exécute PHP CS Fixer en mode **vérification** (sans modifier les fichiers)
7. Exécute PHPStan

> **Aide — Cache Composer**
>
> L'action `actions/cache@v4` utilise une `key` basée sur un hash de fichier pour savoir si le cache est valide. Pour Composer, le fichier de référence est `composer.lock`. Le chemin à mettre en cache est `api/vendor`.

> **Aide — PHP CS Fixer**
>
> PHP CS Fixer n'est pas encore installé dans le projet. Vous devrez l'installer via Composer (`require-dev`), puis créer un fichier de configuration `.php-cs-fixer.dist.php` dans le dossier `api/`.
>
> En mode vérification (dry-run), la commande est :
> ```bash
> vendor/bin/php-cs-fixer fix --dry-run --diff
> ```

> **Aide — PHPStan**
>
> De même, PHPStan doit être installé via Composer. Il nécessite un fichier `phpstan.neon` pour sa configuration. Un bon point de départ est le niveau d'analyse `5` sur le dossier `src`.

### Correction {collapsible="true" id="correction_2"}

**Fichier `api/.php-cs-fixer.dist.php` :**

```php
<?php

// =============================================================================
// Configuration PHP CS Fixer - Règles de style PSR-12
// =============================================================================

$finder = (new PhpCsFixer\Finder())
    ->in(__DIR__ . '/src')
    ->in(__DIR__ . '/tests');

return (new PhpCsFixer\Config())
    ->setRules([
        '@PSR12' => true,
        '@Symfony' => true,
    ])
    ->setFinder($finder);
```

**Fichier `api/phpstan.neon` :**

```neon
# =============================================================================
# Configuration PHPStan - Analyse statique du code PHP
# =============================================================================

parameters:
    # Niveau d'analyse (0 = permissif, 9 = strict)
    level: 5
    # Dossiers à analyser
    paths:
        - src
```

**Job dans `ci-cd.yml` :**

```yaml
jobs:
  # ===========================================================================
  # Job : Vérification de la qualité du code PHP
  # ===========================================================================
  lint-backend:
    name: Lint Backend (PHP)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: api

    steps:
      # --- Récupérer le code source ---
      - name: Checkout du code
        uses: actions/checkout@v4

      # --- Installer PHP avec les extensions nécessaires ---
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: intl, pdo_pgsql, redis
          tools: composer

      # --- Mettre en cache les dépendances Composer ---
      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: api/vendor
          key: composer-${{ hashFiles('api/composer.lock') }}
          restore-keys: composer-

      # --- Installer les dépendances ---
      - name: Installation des dépendances
        run: composer install --no-interaction --prefer-dist

      # --- Vérifier le style du code (PSR-12) ---
      - name: PHP CS Fixer
        run: vendor/bin/php-cs-fixer fix --dry-run --diff

      # --- Analyse statique du code ---
      - name: PHPStan
        run: vendor/bin/phpstan analyse
```

---

## Étape 4 — Job : Linting frontend (JavaScript)

### Contexte {id="contexte_2"}

Le projet React utilise déjà ESLint (déclaré dans `package.json`). Vous allez ajouter un job qui vérifie le code JavaScript.

### Consignes {id="consignes_3"}

Ajoutez un job `lint-frontend` qui :

1. S'exécute sur `ubuntu-latest`
2. Récupère le code
3. Configure Node.js 22 avec l'action `actions/setup-node@v4`
4. Met en cache les dépendances npm (le chemin à cacher est `front/node_modules`, la clé se base sur `package-lock.json`)
5. Installe les dépendances avec `npm ci` (plus rapide et reproductible que `npm install`)
6. Exécute ESLint

> **Attention :** Ce job ne doit **pas** dépendre du job `lint-backend`. Les deux lints s'exécutent en parallèle (pas de mot-clé `needs`).

> **Aide — `npm ci` vs `npm install`**
>
> `npm ci` :
> - Supprime `node_modules` et installe depuis le `package-lock.json` exact
> - Plus rapide en CI car pas de résolution de dépendances
> - Échoue si `package-lock.json` n'est pas à jour

### Correction {collapsible="true" id="correction_3"}

```yaml
  # ===========================================================================
  # Job : Vérification de la qualité du code JavaScript
  # ===========================================================================
  lint-frontend:
    name: Lint Frontend (JS)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: front

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      # --- Installer Node.js ---
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      # --- Mettre en cache les dépendances npm ---
      - name: Cache npm
        uses: actions/cache@v4
        with:
          path: front/node_modules
          key: npm-${{ hashFiles('front/package-lock.json') }}
          restore-keys: npm-

      - name: Installation des dépendances
        run: npm ci

      # --- Vérifier la qualité du code ---
      - name: ESLint
        run: npm run lint
```

---

## Étape 5 — Job : Tests unitaires backend

### Contexte {id="contexte_3"}

Les tests PHPUnit du projet nécessitent une base de données PostgreSQL et un serveur Redis. GitHub Actions permet de démarrer des **services** (conteneurs Docker) accessibles par les jobs.

### Consignes {id="consignes_4"}

Ajoutez un job `test-backend` qui :

1. **Dépend** des deux jobs de lint (`lint-backend` et `lint-frontend`)
2. Déclare deux **services** :
    - `postgres` : image `postgres:16-alpine`, avec les variables `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` correspondant à la configuration du projet, et un **healthcheck** via `pg_isready`
    - `redis` : image `redis:7-alpine`
3. Définit les **variables d'environnement** nécessaires au job (DATABASE_URL pointant vers le service `postgres`, REDIS_URL, APP_ENV=test, APP_SECRET)
4. Installe PHP, les dépendances Composer
5. Génère les clés JWT (commande Symfony : `php bin/console lexik:jwt:generate-keypair`)
6. Crée le schéma de la base de données de test
7. Exécute PHPUnit avec génération de couverture au format Clover
8. Upload le rapport de couverture en tant qu'**artifact** avec `actions/upload-artifact@v4`

> **Aide — Services GitHub Actions**
>
> Les services sont des conteneurs accessibles via `localhost` depuis le runner. Les ports doivent être mappés explicitement :
> ```yaml
> services:
>   mon-service:
>     image: mon-image
>     ports:
>       - 5432:5432
>     options: >-
>       --health-cmd "commande"
>       --health-interval 10s
>       --health-timeout 5s
>       --health-retries 5
> ```

> **Aide — DATABASE_URL**
>
> En CI, le host de la base de données est `localhost` (pas `database` comme dans Docker Compose). Adaptez l'URL en conséquence.

> **Aide — Couverture de code**
>
> PHPUnit génère un rapport Clover avec l'option `--coverage-clover`. Xdebug doit être activé dans les extensions PHP (ajoutez `xdebug` dans le setup-php avec `coverage: xdebug`).

### Correction {collapsible="true" id="correction_4"}

```yaml
  # ===========================================================================
  # Job : Tests unitaires PHP avec PHPUnit
  # ===========================================================================
  test-backend:
    name: Tests Backend (PHPUnit)
    runs-on: ubuntu-latest
    # --- Ce job attend la fin des deux lints ---
    needs: [ lint-backend, lint-frontend ]
    defaults:
      run:
        working-directory: api

    # --- Services nécessaires (conteneurs Docker) ---
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
          --health-cmd "pg_isready -U yapuka"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    # --- Variables d'environnement pour Symfony ---
    env:
      DATABASE_URL: "postgresql://yapuka:yapuka@localhost:5432/yapuka_test?serverVersion=16&charset=utf8"
      REDIS_URL: "redis://localhost:6379"
      APP_ENV: test
      APP_SECRET: ci-test-secret

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: intl, pdo_pgsql, redis
          coverage: xdebug
          tools: composer

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: api/vendor
          key: composer-${{ hashFiles('api/composer.lock') }}
          restore-keys: composer-

      - name: Installation des dépendances
        run: composer install --no-interaction --prefer-dist

      # --- Générer les clés JWT pour l'authentification ---
      - name: Génération des clés JWT
        run: php bin/console lexik:jwt:generate-keypair --skip-if-exists

      # --- Créer le schéma de la base de données de test ---
      - name: Création du schéma de base de données
        run: php bin/console doctrine:schema:create --env=test

      # --- Exécuter les tests avec rapport de couverture ---
      - name: Exécution de PHPUnit
        run: php bin/phpunit --coverage-clover coverage.xml

      # --- Sauvegarder le rapport de couverture ---
      - name: Upload du rapport de couverture
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: backend-coverage
          path: api/coverage.xml
```

---

## Étape 6 — Job : Build et push des images Docker

### Contexte {id="contexte_4"}

Une fois les tests validés, il faut construire les images Docker et les envoyer sur un **registry** pour pouvoir les déployer. Vous utiliserez le **GitHub Container Registry** (ghcr.io), intégré à GitHub.

### Consignes {id="consignes_5"}

Ajoutez un job `build-images` qui :

1. **Dépend** du job `test-backend`
2. Ne s'exécute que sur un **push** (pas sur une PR) — utilisez la condition `if: github.event_name == 'push'`
3. Se connecte au registry GHCR avec l'action `docker/login-action@v3` (le username est `${{ github.actor }}`, le password est `${{ secrets.GITHUB_TOKEN }}`)
4. Configure Docker Buildx avec `docker/setup-buildx-action@v3`
5. Build et push l'image **backend** avec `docker/build-push-action@v5` :
    - Contexte : `./api`
    - Tags : `ghcr.io/${{ github.repository }}/backend:latest` et `ghcr.io/${{ github.repository }}/backend:${{ github.sha }}`
    - Activez le cache des layers Docker (`cache-from` et `cache-to` de type `gha`)
6. Faites de même pour l'image **frontend** (contexte `./front`)

> **Aide — Tags d'images**
>
> Deux tags par image :
> - `latest` : pointe toujours vers la dernière version
> - `<sha du commit>` : permet de tracer exactement quelle version est déployée
>
> L'utilisation de `github.sha` dans le tag garantit l'unicité.

> **Aide — Cache des layers Docker**
>
> Docker Buildx peut utiliser le cache de GitHub Actions :
> ```yaml
> cache-from: type=gha
> cache-to: type=gha,mode=max
> ```
> Cela évite de reconstruire les layers inchangés à chaque pipeline.

### Correction {collapsible="true" id="correction_5"}

```yaml
  # ===========================================================================
  # Job : Build et push des images Docker vers GHCR
  # ===========================================================================
  build-images:
    name: Build Images Docker
    runs-on: ubuntu-latest
    needs: [ test-backend ]
    # --- Ne builder que sur un push (pas sur les PR) ---
    if: github.event_name == 'push'

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      # --- Configurer Docker Buildx (builder amélioré) ---
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      # --- Se connecter au GitHub Container Registry ---
      - name: Login GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # --- Build et push de l'image backend ---
      - name: Build & Push Backend
        uses: docker/build-push-action@v5
        with:
          context: ./api
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/backend:latest
            ghcr.io/${{ github.repository }}/backend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # --- Build et push de l'image frontend ---
      - name: Build & Push Frontend
        uses: docker/build-push-action@v5
        with:
          context: ./front
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/frontend:latest
            ghcr.io/${{ github.repository }}/frontend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Étape 7 — Job : Scan de sécurité avec Trivy

### Contexte {id="contexte_5"}

**Trivy** est un scanner de vulnérabilités open-source qui analyse les images Docker pour détecter :
- Des vulnérabilités connues (CVE) dans les packages système
- Des dépendances applicatives obsolètes ou vulnérables
- Des mauvaises configurations

### Consignes {id="consignes_6"}

Ajoutez un job `security-scan` qui :

1. **Dépend** du job `build-images`
2. Scanne l'image **backend** avec l'action `aquasecurity/trivy-action@master` :
    - Image à scanner : celle que vous venez de pusher (tag `latest`)
    - Format de sortie : `table` (lisible dans les logs)
    - Sévérités à vérifier : `CRITICAL,HIGH`
    - **Exit code `1`** pour les vulnérabilités CRITICAL (le job échoue si une vulnérabilité critique est trouvée)
3. Scanne l'image **frontend** de la même manière
4. Génère un rapport au format `sarif` et l'uploade en artifact

> **Aide — Exit code**
>
> Le paramètre `exit-code: '1'` fait échouer le step (et donc le job) si des vulnérabilités du niveau de sévérité indiqué sont trouvées. Utilisez `exit-code: '0'` si vous voulez simplement signaler sans bloquer.

> **Réflexion :** En pratique, il est courant de ne bloquer que sur les vulnérabilités `CRITICAL` et de simplement alerter pour les `HIGH`. Comment feriez-vous cela avec deux steps distincts ?

### Correction {collapsible="true" id="correction_6"}

```yaml
  # ===========================================================================
  # Job : Scan de sécurité des images Docker
  # ===========================================================================
  security-scan:
    name: Security Scan (Trivy)
    runs-on: ubuntu-latest
    needs: [ build-images ]

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      # --- Scanner l'image backend (bloque si vulnérabilité CRITICAL) ---
      - name: Scan image backend
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}/backend:latest
          format: table
          severity: CRITICAL
          exit-code: '1'

      # --- Scanner l'image frontend ---
      - name: Scan image frontend
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}/frontend:latest
          format: table
          severity: CRITICAL
          exit-code: '1'

      # --- Générer un rapport détaillé (ne bloque pas) ---
      - name: Rapport SARIF backend
        uses: aquasecurity/trivy-action@master
        if: always()
        with:
          image-ref: ghcr.io/${{ github.repository }}/backend:latest
          format: sarif
          output: backend-trivy.sarif
          severity: CRITICAL,HIGH

      # --- Sauvegarder le rapport ---
      - name: Upload rapport de sécurité
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: trivy-reports
          path: "*.sarif"
```

---

## Étape 8 — Job : Déploiement staging

### Contexte

Le déploiement automatique sur staging permet de tester chaque changement dans un environnement proche de la production. Ce job ne doit se déclencher que pour la branche `develop`.

Pour simplifier ce lab, le déploiement consistera à vérifier que tout est prêt et à exécuter un **smoke test** (vérification basique que l'application répond).

> Dans un projet réel, vous utiliseriez ici un outil comme **Heroku**, **Railway**, **Fly.io** ou un déploiement SSH/Kubernetes.

### Consignes {id="consignes_7"}

Ajoutez un job `deploy-staging` qui :

1. **Dépend** du job `security-scan`
2. Ne s'exécute que sur la branche `develop` — condition : `if: github.ref == 'refs/heads/develop'`
3. Déclare un **environment** GitHub nommé `staging` (cela apparaîtra dans l'onglet Environments du repository)
4. Affiche les images qui seraient déployées (tags backend et frontend)
5. Exécute un smoke test avec `curl` contre l'URL de staging (si elle existe) ou simule le déploiement

> **Aide — Environments GitHub**
>
> La clé `environment` dans un job permet de :
> - Suivre les déploiements dans l'interface GitHub
> - Ajouter des règles de protection (approval manuelle, restrictions de branche)
> ```yaml
> environment:
>   name: staging
>   url: https://staging.yapuka.dev
> ```

### Correction {collapsible="true"}

```yaml
  # ===========================================================================
  # Job : Déploiement sur l'environnement de staging
  # ===========================================================================
  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: [ security-scan ]
    # --- Uniquement sur la branche develop ---
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.yapuka.dev

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      # --- Afficher les images à déployer ---
      - name: Résumé du déploiement
        run: |
          echo "=== Déploiement Staging ==="
          echo "Backend:  ghcr.io/${{ github.repository }}/backend:${{ github.sha }}"
          echo "Frontend: ghcr.io/${{ github.repository }}/frontend:${{ github.sha }}"
          echo "Commit:   ${{ github.sha }}"
          echo "Branche:  ${{ github.ref_name }}"

      # --- Ici viendrait le déploiement réel ---
      # Exemple avec Heroku, Railway, SSH, Kubernetes, etc.
      - name: Simulation du déploiement
        run: |
          echo "Déploiement des images vers staging..."
          echo "Déploiement simulé avec succès"

      # --- Smoke test : vérifier que l'application répond ---
      - name: Smoke test
        run: |
          echo "Vérification de la disponibilité de l'application..."
          # En conditions réelles :
          # curl --fail --retry 5 --retry-delay 10 https://staging.yapuka.dev/api/docs
          echo "Smoke test réussi"
```

---

## Étape 9 — Fichier complet et validation

### Consignes

1. Assemblez tous les jobs dans un seul fichier `ci-cd.yml`
2. Vérifiez la syntaxe YAML (l'indentation est cruciale !)
3. Commitez et poussez sur une branche `develop` pour déclencher le pipeline
4. Observez l'exécution dans l'onglet **Actions** de votre repository GitHub

### Points de vérification

- [ ] Les jobs `lint-backend` et `lint-frontend` s'exécutent en **parallèle**
- [ ] Le job `test-backend` attend la fin des **deux** lints
- [ ] Le job `build-images` ne s'exécute **pas** sur les Pull Requests
- [ ] Le job `deploy-staging` ne s'exécute que sur la branche **develop**
- [ ] Les artifacts (couverture, rapports Trivy) sont bien uploadés
- [ ] Les caches Composer et npm sont utilisés à la deuxième exécution

### Dépannage courant

| Problème | Solution probable |
|----------|-------------------|
| `Error: Process completed with exit code 2` sur PHP CS Fixer | Le code ne respecte pas PSR-12. Exécutez `vendor/bin/php-cs-fixer fix` localement |
| `Connection refused` sur PostgreSQL | Vérifiez que le healthcheck du service est bien configuré et que le port 5432 est mappé |
| `Permission denied` sur les clés JWT | Ajoutez `--skip-if-exists` à la commande de génération |
| Job `build-images` skipped | Normal sur une PR — il ne s'exécute que sur un push |
| Erreur YAML `mapping values are not allowed here` | Problème d'indentation — utilisez des espaces, jamais des tabulations |

### Correction — Fichier complet {collapsible="true"}

```yaml
# =============================================================================
# Pipeline CI/CD - Yapuka
# =============================================================================
# Workflow complet : lint → tests → build → sécurité → déploiement
#
# Déclencheurs : push ou PR sur main et develop
# Registry : GitHub Container Registry (ghcr.io)
# =============================================================================

name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  REGISTRY: ghcr.io

jobs:
  # ===========================================================================
  # Lint Backend - Qualité du code PHP
  # ===========================================================================
  lint-backend:
    name: Lint Backend (PHP)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: api

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: intl, pdo_pgsql, redis
          tools: composer

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: api/vendor
          key: composer-${{ hashFiles('api/composer.lock') }}
          restore-keys: composer-

      - name: Installation des dépendances
        run: composer install --no-interaction --prefer-dist

      - name: PHP CS Fixer
        run: vendor/bin/php-cs-fixer fix --dry-run --diff

      - name: PHPStan
        run: vendor/bin/phpstan analyse

  # ===========================================================================
  # Lint Frontend - Qualité du code JavaScript
  # ===========================================================================
  lint-frontend:
    name: Lint Frontend (JS)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: front

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Cache npm
        uses: actions/cache@v4
        with:
          path: front/node_modules
          key: npm-${{ hashFiles('front/package-lock.json') }}
          restore-keys: npm-

      - name: Installation des dépendances
        run: npm ci

      - name: ESLint
        run: npm run lint

  # ===========================================================================
  # Tests Backend - PHPUnit avec PostgreSQL et Redis
  # ===========================================================================
  test-backend:
    name: Tests Backend (PHPUnit)
    runs-on: ubuntu-latest
    needs: [ lint-backend, lint-frontend ]
    defaults:
      run:
        working-directory: api

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
          --health-cmd "pg_isready -U yapuka"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    env:
      DATABASE_URL: "postgresql://yapuka:yapuka@localhost:5432/yapuka_test?serverVersion=16&charset=utf8"
      REDIS_URL: "redis://localhost:6379"
      APP_ENV: test
      APP_SECRET: ci-test-secret

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: intl, pdo_pgsql, redis
          coverage: xdebug
          tools: composer

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: api/vendor
          key: composer-${{ hashFiles('api/composer.lock') }}
          restore-keys: composer-

      - name: Installation des dépendances
        run: composer install --no-interaction --prefer-dist

      - name: Génération des clés JWT
        run: php bin/console lexik:jwt:generate-keypair --skip-if-exists

      - name: Création du schéma de base de données
        run: php bin/console doctrine:schema:create --env=test

      - name: Exécution de PHPUnit
        run: php bin/phpunit --coverage-clover coverage.xml

      - name: Upload du rapport de couverture
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: backend-coverage
          path: api/coverage.xml

  # ===========================================================================
  # Build - Construction et push des images Docker
  # ===========================================================================
  build-images:
    name: Build Images Docker
    runs-on: ubuntu-latest
    needs: [ test-backend ]
    if: github.event_name == 'push'

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push Backend
        uses: docker/build-push-action@v5
        with:
          context: ./api
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/backend:latest
            ghcr.io/${{ github.repository }}/backend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build & Push Frontend
        uses: docker/build-push-action@v5
        with:
          context: ./front
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/frontend:latest
            ghcr.io/${{ github.repository }}/frontend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ===========================================================================
  # Sécurité - Scan des images avec Trivy
  # ===========================================================================
  security-scan:
    name: Security Scan (Trivy)
    runs-on: ubuntu-latest
    needs: [ build-images ]

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Scan image backend
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}/backend:latest
          format: table
          severity: CRITICAL
          exit-code: '1'

      - name: Scan image frontend
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}/frontend:latest
          format: table
          severity: CRITICAL
          exit-code: '1'

      - name: Rapport SARIF backend
        uses: aquasecurity/trivy-action@master
        if: always()
        with:
          image-ref: ghcr.io/${{ github.repository }}/backend:latest
          format: sarif
          output: backend-trivy.sarif
          severity: CRITICAL,HIGH

      - name: Upload rapport de sécurité
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: trivy-reports
          path: "*.sarif"

  # ===========================================================================
  # Déploiement Staging - Branche develop uniquement
  # ===========================================================================
  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: [ security-scan ]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.yapuka.dev

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Résumé du déploiement
        run: |
          echo "=== Déploiement Staging ==="
          echo "Backend:  ghcr.io/${{ github.repository }}/backend:${{ github.sha }}"
          echo "Frontend: ghcr.io/${{ github.repository }}/frontend:${{ github.sha }}"
          echo "Commit:   ${{ github.sha }}"
          echo "Branche:  ${{ github.ref_name }}"

      - name: Simulation du déploiement
        run: |
          echo "Déploiement des images vers staging..."
          echo "Déploiement simulé avec succès"

      - name: Smoke test
        run: |
          echo "Vérification de la disponibilité..."
          # curl --fail --retry 5 --retry-delay 10 https://staging.yapuka.dev/api/docs
          echo "Smoke test réussi"
```

---

## Pour aller plus loin

Si vous avez terminé en avance, voici des améliorations à explorer :

1. **Deploy production avec approval :** Ajoutez un job `deploy-production` conditionné à `main` avec un environment protégé par une approval manuelle dans les settings GitHub.

2. **Matrix strategy :** Testez le backend sur plusieurs versions de PHP (8.3 et 8.4) en utilisant une matrice :
   ```yaml
   strategy:
     matrix:
       php-version: ['8.3', '8.4']
   ```

3. **Notifications Slack :** Ajoutez une notification en cas d'échec du pipeline avec l'action `8398a7/action-slack@v3`.

4. **Badge de statut :** Ajoutez un badge dans votre `README.md` pour afficher le statut du pipeline :
   ```markdown
   ![CI/CD](https://github.com/<user>/<repo>/actions/workflows/ci-cd.yml/badge.svg)
   ```

5. **Concurrency :** Empêchez deux pipelines de se déployer simultanément :
   ```yaml
   concurrency:
     group: deploy-${{ github.ref }}
     cancel-in-progress: true
   ```