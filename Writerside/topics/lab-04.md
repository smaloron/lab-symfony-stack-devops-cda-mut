# Lab 4 — Dockerisation production du frontend React

> **Prérequis** : le projet Yapuka fonctionne en mode développement via `docker compose up -d`. Vous avez réalisé le lab
> précédent (Dockerisation Symfony) ou en comprenez les principes (multi-stage build, `.dockerignore`, user non-root).

## Contexte

Le frontend Yapuka est une **SPA** (Single Page Application) construite avec React 18 et Vite. En développement, Vite
fournit un serveur avec Hot Module Replacement. En production, Vite génère des fichiers statiques (HTML, JS, CSS) qu'un
serveur web comme Nginx peut servir directement — **pas besoin de Node.js en production**.

Cela donne une image Docker très légère : seul Nginx est nécessaire pour servir les fichiers compilés.

> **SPA vs SSR** : une SPA envoie un seul fichier HTML puis charge tout via JavaScript côté client. Le routage est géré
> par React Router dans le navigateur. Nginx doit renvoyer `index.html` pour toutes les routes qui ne correspondent pas à
> un fichier statique.

Dans ce lab, vous allez :

1. Analyser les spécificités du build Vite
2. Gérer les variables d'environnement (build-time vs runtime)
3. Créer un Dockerfile multi-stage optimisé
4. Configurer Nginx pour servir une SPA
5. Builder et tester l'image

## Partie 1 — Analyse de l'application (5 min)

### 1.1 Comprendre le build Vite

**À faire** : ouvrez un shell dans le conteneur frontend et explorez le processus de build.

```bash
docker compose exec front sh
```

1. Lancez un build de production et observez la sortie :

```bash
npm run build
```

2. Explorez le dossier généré (`dist/`). Listez les fichiers et notez leur structure.

3. Répondez aux questions :
    - Quel est le point d'entrée HTML ?
    - Comment sont nommés les fichiers JS et CSS ? (indice : ils contiennent un hash)
    - Le dossier `dist/` est-il autonome ? A-t-il besoin de Node.js pour être servi ?

#### Correction 1.1 {collapsible="true"}

```bash
$ npm run build
vite v6.x.x building for production...
✓ 42 modules transformed.
dist/index.html                  0.46 kB │ gzip:  0.30 kB
dist/assets/index-a1b2c3d4.css  12.34 kB │ gzip:  3.21 kB
dist/assets/index-e5f6g7h8.js  187.65 kB │ gzip: 61.23 kB
✓ built in 3.21s
```

```bash
$ ls dist/
assets/     index.html
$ ls dist/assets/
index-a1b2c3d4.css   index-e5f6g7h8.js
```

Réponses :

- **Point d'entrée** : `dist/index.html` — contient les balises `<script>` et `<link>` vers les assets.
- **Nommage** : les fichiers contiennent un **hash de contenu** (ex: `index-a1b2c3d4.css`). Cela permet le cache
  agressif : quand le contenu change, le nom change aussi.
- **Autonome** : oui, `dist/` ne contient que des fichiers statiques. Seul un serveur HTTP est nécessaire — pas de
  Node.js en production.

### 1.2 Comprendre les variables d'environnement Vite

C'est le piège principal de la dockerisation d'une SPA : les variables d'environnement sont injectées **au moment du
build**, pas au runtime.

**À faire** : examinez le fichier `front/.env` et le code dans `src/api/client.js`.

1. Quelle variable d'environnement est utilisée par le frontend ?
2. Comment Vite la rend-elle accessible dans le code ? (quel préfixe, quel objet JavaScript)
3. À quel moment cette variable est-elle remplacée par sa valeur : au build ou au runtime ?

#### Correction 1.2 {collapsible="true"}

1. **Variable** : `VITE_API_URL` (définie dans `front/.env` avec la valeur `http://localhost:8080`).

2. **Accès dans le code** : `import.meta.env.VITE_API_URL`. Vite expose toutes les variables préfixées par `VITE_` dans
   l'objet `import.meta.env`.

3. **Moment d'injection** : au **build** uniquement. Lors de `npm run build`, Vite remplace littéralement
   `import.meta.env.VITE_API_URL` par la valeur dans le fichier JS compilé. Après le build, la valeur est figée dans le
   JavaScript — impossible de la changer via une variable d'environnement Docker.

> **Conséquence** : en production, l'API sera servie par le même domaine que le frontend (derrière un reverse proxy). On
> peut donc utiliser une URL relative (`""` ou `/api`) comme valeur de `VITE_API_URL`, ce qui évite le problème de
> variable figée.

## Partie 2 — Gestion des variables d'environnement (10 min)

### 2.1 Stratégie de configuration runtime

Le problème des SPA est que les variables sont figées au build. En production, on veut pouvoir changer l'URL de l'API
sans rebuilder l'image Docker (par exemple, pour déployer la même image en staging et en production).

Il existe trois stratégies. Vous allez implémenter la plus robuste.

| Stratégie                      | Principe                                                                 | Avantage                      | Inconvénient                      |
|--------------------------------|--------------------------------------------------------------------------|-------------------------------|-----------------------------------|
| **URL relative**               | L'API est sur le même domaine                                            | Simple, pas de variable       | Impose un reverse proxy devant    |
| **Build-time ARG**             | `docker build --build-arg VITE_API_URL=...`                              | Simple                        | Rebuild pour chaque environnement |
| **Script d'injection runtime** | Un script remplace un placeholder dans le HTML au démarrage du conteneur | 1 image pour N environnements | Un peu plus complexe              |

**À faire** : implémentez la stratégie **script d'injection runtime**.

**Étape 1** — Modifiez `front/index.html` pour ajouter un script de configuration **avant** le script React :

```html

<script src="/config.js"></script>
```

Ce fichier sera généré dynamiquement au démarrage du conteneur.

**Étape 2** — Modifiez `front/src/api/client.js` pour lire l'URL de l'API depuis `window.__CONFIG__` en priorité, puis
depuis `import.meta.env.VITE_API_URL` en fallback.

> Le fichier `config.js` généré contiendra :
> ```js
> window.__CONFIG__ = { API_URL: "https://api.production.com" };
> ```

**Étape 3** — Créez un script shell `docker/front/docker-entrypoint.sh` qui :

1. Génère `/usr/share/nginx/html/config.js` avec la variable d'environnement `API_URL`
2. Puis démarre Nginx avec `exec nginx -g 'daemon off;'`

> En shell, pour écrire le fichier :
> ```bash
> cat <<EOF > /usr/share/nginx/html/config.js
> window.__CONFIG__ = { API_URL: "${API_URL:-}" };
> EOF
> ```

#### Correction 2.1 {collapsible="true"}

**`front/index.html`** — ajout du script de configuration :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Yapuka</title>
    <!-- Configuration runtime injectée au démarrage du conteneur -->
    <script src="/config.js"></script>
</head>
<body>
<div id="root"></div>
<script type="module" src="/src/main.jsx"></script>
</body>
</html>
```

**`front/src/api/client.js`** — modification de la base URL :

```js
// =============================================================================
// URL de base de l'API
// =============================================================================
// Priorité :
//   1. window.__CONFIG__.API_URL (injecté au runtime par Docker)
//   2. import.meta.env.VITE_API_URL (injecté au build par Vite)
//   3. '' (URL relative — même domaine, via reverse proxy)
// =============================================================================
const API_BASE_URL = window.__CONFIG__?.API_URL
    || import.meta.env.VITE_API_URL
    || '';
```

Puis dans la fonction `apiRequest`, remplacez l'ancienne référence par `API_BASE_URL` :

```js
async function apiRequest(endpoint, options = {}) {
    const url = `${API_BASE_URL}${endpoint}`;
    // ...
}
```

**`docker/front/docker-entrypoint.sh`** :

```bash
#!/bin/sh
# =============================================================================
# Entrypoint Docker — Injection des variables d'environnement runtime
# =============================================================================
# Génère /usr/share/nginx/html/config.js avec les variables d'environnement
# passées au conteneur, puis démarre Nginx.
# =============================================================================

set -e

# Générer le fichier de configuration runtime
cat <<EOF > /usr/share/nginx/html/config.js
window.__CONFIG__ = {
  API_URL: "${API_URL:-}"
};
EOF

echo "Configuration runtime générée :"
cat /usr/share/nginx/html/config.js

# Démarrer Nginx au premier plan
exec nginx -g 'daemon off;'
```

N'oubliez pas de le rendre exécutable :

```bash
chmod +x docker/front/docker-entrypoint.sh
```

## Partie 3 — Dockerfile multi-stage (15 min)

### 3.1 Créer le .dockerignore

**À faire** : créez un fichier `front/.dockerignore` qui exclut les fichiers inutiles au build.

Excluez : `node_modules/`, `dist/`, les fichiers Git/IDE, les fichiers Docker, les configurations Cypress, et les
fichiers `.env.local`.

#### Correction 3.1 {collapsible="true"}

```docker
# front/.dockerignore

# Dépendances (installées dans le build)
node_modules/

# Build précédent (recréé dans le build)
dist/

# Tests E2E
cypress/
cypress.config.js

# Git et IDE
.git/
.gitignore
.idea/
.vscode/

# Docker (éviter la récursion)
Dockerfile*

# Environnement local
.env.local
```

### 3.2 Stage 1 : installation des dépendances

**À faire** : créez un fichier `front/Dockerfile.prod` avec le premier stage.

**Consignes** :

- Partez de `node:22-alpine`
- Nommez ce stage `deps`
- Définissez le workdir `/app`
- Copiez **uniquement** `package.json` et `package-lock.json`
- Lancez `npm ci` (installation propre, plus rapide que `npm install`, adaptée au CI/CD)

> **`npm ci` vs `npm install`** : `npm ci` supprime `node_modules/` et installe exactement les versions du
`package-lock.json`. C'est plus rapide et reproductible — idéal pour les builds Docker.

#### Correction 3.2 {collapsible="true"}

```docker
# =============================================================================
# Stage 1 : Deps — Installation des dépendances Node
# =============================================================================
FROM node:22-alpine AS deps

WORKDIR /app

# Copier uniquement les fichiers de dépendances (optimisation cache Docker)
COPY package.json package-lock.json ./

# Installation propre et reproductible
RUN npm ci
```

### 3.3 Stage 2 : build de production

Le deuxième stage compile l'application React en fichiers statiques. C'est ici que les variables `VITE_*` sont
injectées.

**À faire** : ajoutez le stage de build.

**Consignes** :

- Partez de `node:22-alpine`, nommez-le `builder`
- Copiez `node_modules/` depuis le stage `deps`
- Copiez tout le code source
- Définissez un `ARG VITE_API_URL` avec une valeur par défaut vide (URL relative)
- Convertissez l'ARG en ENV pour que Vite y accède
- Lancez `npm run build`

> **ARG vs ENV dans Docker** :
> - `ARG` est disponible uniquement pendant le build (`docker build --build-arg ...`)
> - `ENV` est disponible au build ET au runtime
> - Il faut écrire `ARG VITE_API_URL` puis `ENV VITE_API_URL=$VITE_API_URL` pour que Vite puisse lire la variable

#### Correction 3.3 {collapsible="true"}

```docker
# =============================================================================
# Stage 2 : Builder — Compilation des assets de production
# =============================================================================
FROM node:22-alpine AS builder

WORKDIR /app

# Copier les node_modules depuis le stage deps
COPY --from=deps /app/node_modules ./node_modules

# Copier le code source
COPY . .

# Variable d'environnement Vite (injectée au build)
# Par défaut vide = URL relative (même domaine via reverse proxy)
ARG VITE_API_URL=""
ENV VITE_API_URL=$VITE_API_URL

# Build de production
RUN npm run build
```

### 3.4 Stage 3 : image Nginx de production

Le stage final ne contient que Nginx et les fichiers statiques compilés. Pas de Node.js, pas de `node_modules/` —
l'image est extrêmement légère.

**À faire** : ajoutez le stage de production.

**Consignes** :

1. Partez de `nginx:alpine`, nommez-le `production`
2. Supprimez la configuration Nginx par défaut
3. Copiez votre configuration Nginx personnalisée (vous la créerez à l'étape suivante)
4. Copiez le dossier `dist/` depuis le stage `builder` vers `/usr/share/nginx/html`
5. Copiez le script `docker-entrypoint.sh` et rendez-le exécutable
6. Exposez le port 80
7. Utilisez `ENTRYPOINT` pour lancer votre script au lieu du CMD par défaut

> **ENTRYPOINT vs CMD** :
> - `CMD` est la commande par défaut, facilement remplaçable
> - `ENTRYPOINT` est le point d'entrée obligatoire — utile quand on a un script d'initialisation qui doit toujours s'
    exécuter avant le serveur

#### Correction 3.4 {collapsible="true"}

```docker
# =============================================================================
# Stage 3 : Production — Nginx servant les fichiers statiques
# =============================================================================
FROM nginx:alpine AS production

# Supprimer la config par défaut
RUN rm /etc/nginx/conf.d/default.conf

# Copier notre configuration Nginx
COPY docker/nginx/front.conf /etc/nginx/conf.d/default.conf

# Copier les fichiers statiques compilés
COPY --from=builder /app/dist /usr/share/nginx/html

# Copier le script d'injection des variables runtime
COPY docker/front/docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

EXPOSE 80

ENTRYPOINT ["/docker-entrypoint.sh"]
```

**Dockerfile complet** (`front/Dockerfile.prod`) :

```docker
# =============================================================================
# Dockerfile de production — Yapuka Frontend
# =============================================================================
# Multi-stage build :
#   1. deps    → npm ci (cache optimisé)
#   2. builder → npm run build (compilation Vite)
#   3. prod    → Nginx Alpine + fichiers statiques
# =============================================================================

# --- Stage 1 : Dépendances ---
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# --- Stage 2 : Build ---
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG VITE_API_URL=""
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

# --- Stage 3 : Production ---
FROM nginx:alpine AS production
RUN rm /etc/nginx/conf.d/default.conf
COPY docker/nginx/front.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
COPY docker/front/docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh
EXPOSE 80
ENTRYPOINT ["/docker-entrypoint.sh"]
```

## Partie 4 — Configuration Nginx pour SPA (5 min)

### 4.1 Créer la configuration Nginx

Une SPA a une particularité : toutes les routes (ex: `/login`, `/register`) doivent renvoyer `index.html`, car le
routage est géré par React Router côté client. Seuls les fichiers statiques réels (JS, CSS, images) doivent être servis
directement.

**À faire** : créez le fichier `docker/nginx/front.conf`.

**Consignes** :

1. **Server block** sur le port 80
2. **Root** vers `/usr/share/nginx/html`
3. **Location /** avec `try_files $uri $uri/ /index.html` — c'est la règle clé pour une SPA
4. **Cache agressif** pour les assets avec hash (`/assets/`) : `expires 1y`, `Cache-Control: public, immutable`
5. **Pas de cache** pour `index.html` et `config.js` (ces fichiers changent à chaque déploiement)
6. **Headers de sécurité** : `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`
7. **Compression gzip** pour JS, CSS, JSON, HTML

> **Pourquoi cacher les assets mais pas `index.html` ?** Les fichiers dans `assets/` ont un hash dans leur nom (
`index-a1b2c3.js`). Si le contenu change, le nom change → on peut cacher indéfiniment. `index.html` référence ces
> fichiers par nom → il doit toujours être frais pour pointer vers les dernières versions.

#### Correction 4.1 {collapsible="true"}

```nginx
# docker/nginx/front.conf
# =============================================================================
# Configuration Nginx de production pour le frontend React (SPA)
# =============================================================================

server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # --- Compression ---
    gzip on;
    gzip_types text/html text/css application/javascript application/json;
    gzip_min_length 256;

    # --- Headers de sécurité ---
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # --- Assets avec hash : cache agressif (1 an) ---
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # --- index.html et config.js : jamais mis en cache ---
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";
    }

    location = /config.js {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";
    }

    # --- SPA : toutes les routes renvoient index.html ---
    location / {
        try_files $uri $uri/ /index.html;
    }

    # --- Bloquer les fichiers sensibles ---
    location ~ /\.(env|git|htaccess) {
        deny all;
        return 404;
    }

    # --- Logs ---
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
}
```

**Explication de `try_files $uri $uri/ /index.html`** :

1. Essaie de servir le fichier demandé (`$uri`) — ex: `/assets/index-abc.js` → sert le fichier JS
2. Essaie le dossier (`$uri/`) — ex: `/assets/` → cherche un `index.html` dans le dossier
3. Sinon, renvoie `/index.html` — ex: `/login` → sert `index.html`, React Router prend le relais

## Partie 5 — Build et validation (10 min)

### 5.1 Builder l'image

**À faire** :

1. Vérifiez que tous les fichiers sont en place :

```bash
ls front/Dockerfile.prod
ls docker/nginx/front.conf
ls docker/front/docker-entrypoint.sh
```

2. Buildez l'image depuis la racine du projet (car le Dockerfile référence `docker/` qui est en dehors de `front/`) :

```bash
docker build -f front/Dockerfile.prod -t yapuka-front:prod front/
```

> Si le build échoue parce que `docker/nginx/front.conf` n'est pas trouvé, c'est que le contexte de build est
`front/` mais le fichier Nginx est dans `docker/`. Deux solutions :
> - **Solution A** : déplacer `front.conf` dans `front/docker/nginx/` et adapter le COPY
> - **Solution B** : builder depuis la racine avec `-f front/Dockerfile.prod .` et adapter les COPY

3. Vérifiez la taille de l'image :

```bash
docker images yapuka-front:prod
```

#### Correction 5.1 {collapsible="true"}

Le problème de contexte est courant. La solution la plus propre est de copier les fichiers de configuration **dans le
dossier front** ou d'adapter les chemins.

**Solution recommandée** : placer les fichiers de config dans `front/` pour que le contexte de build soit autonome.

```bash
mkdir -p front/docker/nginx
cp docker/nginx/front.conf front/docker/nginx/front.conf
mkdir -p front/docker/front
cp docker/front/docker-entrypoint.sh front/docker/front/docker-entrypoint.sh
```

Le Dockerfile avec les chemins relatifs au contexte `front/` :

```docker
# Les COPY sont relatifs au contexte de build (front/)
COPY docker/nginx/front.conf /etc/nginx/conf.d/default.conf
COPY docker/front/docker-entrypoint.sh /docker-entrypoint.sh
```

Build :

```bash
$ docker build -f front/Dockerfile.prod -t yapuka-front:prod front/
[+] Building 25.3s (14/14) FINISHED
 => [deps] npm ci
 => [builder] npm run build
 => [production] COPY dist + config
 => exporting to image

$ docker images yapuka-front:prod
REPOSITORY      TAG    SIZE
yapuka-front    prod   ~45MB
```

L'image fait environ **40-50 MB** — c'est Nginx Alpine (~7 MB) + les fichiers statiques (~1 MB) + les couches de base.
Comparez avec les 300+ MB d'une image Node.js qui inclut tout `node_modules/`.

### 5.2 Tester le conteneur

**À faire** :

1. Lancez le conteneur en passant une URL d'API en variable d'environnement :

```bash
docker run --rm -d \
  --name yapuka-front-test \
  -p 3000:80 \
  -e API_URL="http://localhost:8080" \
  yapuka-front:prod
```

2. Vérifiez les logs — vous devez voir la configuration runtime générée :

```bash
docker logs yapuka-front-test
```

3. Testez dans le navigateur :
    - Ouvrez `http://localhost:3000` → la page de login doit s'afficher
    - Ouvrez `http://localhost:3000/login` → même page (SPA routing fonctionne)
    - Ouvrez `http://localhost:3000/nimportequoi` → redirigé vers login (route catch-all)
    - Ouvrez `http://localhost:3000/config.js` → doit afficher le contenu `window.__CONFIG__`

4. Vérifiez que le `config.js` contient la bonne URL :

```bash
docker exec yapuka-front-test cat /usr/share/nginx/html/config.js
```

5. Nettoyez :

```bash
docker stop yapuka-front-test
```

#### Correction 5.2 {collapsible="true"}

```bash
$ docker logs yapuka-front-test
Configuration runtime générée :
window.__CONFIG__ = {
  API_URL: "http://localhost:8080"
};
```

```bash
$ curl -s http://localhost:3000/config.js
window.__CONFIG__ = {
  API_URL: "http://localhost:8080"
};
```

```bash
$ curl -s http://localhost:3000/ | head -5
<!DOCTYPE html>
<html lang="fr">
  <head>
    <meta charset="UTF-8" />
    ...
```

```bash
# Vérifier que le SPA routing fonctionne (toute route retourne index.html)
$ curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/login
200
$ curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/nimportequoi
200
```

**Vérification des headers de cache** :

```bash
# Assets avec hash : cache 1 an
$ curl -sI http://localhost:3000/assets/index-*.js | grep -i cache
Cache-Control: public, immutable
Expires: (date dans 1 an)

# index.html : pas de cache
$ curl -sI http://localhost:3000/ | grep -i cache
Cache-Control: no-cache, no-store, must-revalidate
```

**Si la page est blanche** :

- Vérifiez `docker logs` pour des erreurs Nginx
- Vérifiez que `dist/` contient bien les fichiers : `docker exec yapuka-front-test ls /usr/share/nginx/html/`
- Vérifiez que `index.html` référence les bons fichiers dans `assets/`

**Si `config.js` retourne 404** :

- Le script `docker-entrypoint.sh` n'a pas été exécuté. Vérifiez que `ENTRYPOINT` est bien défini dans le Dockerfile et
  que le script est exécutable (`chmod +x`)

### 5.3 Tester avec une autre URL d'API (sans rebuild)

C'est le test décisif de la stratégie d'injection runtime : la même image Docker fonctionne avec une URL d'API
différente.

**À faire** :

```bash
# Lancer avec une URL de staging
docker run --rm -d \
  --name yapuka-staging \
  -p 3001:80 \
  -e API_URL="https://api.staging.yapuka.dev" \
  yapuka-front:prod

# Vérifier que config.js contient la nouvelle URL
curl -s http://localhost:3001/config.js

# Nettoyer
docker stop yapuka-staging
```

> Si `config.js` affiche `https://api.staging.yapuka.dev`, votre stratégie d'injection runtime fonctionne : **une
seule image, N environnements**.

#### Correction 5.3 {collapsible="true"}

```bash
$ curl -s http://localhost:3001/config.js
window.__CONFIG__ = {
  API_URL: "https://api.staging.yapuka.dev"
};
```

C'est le principal avantage par rapport à un simple `--build-arg` : vous buildez l'image **une seule fois** dans le
pipeline CI/CD, puis vous la déployez en staging, pré-production et production en changeant uniquement la variable
d'environnement.

### 5.4 Comparer les images dev vs prod

**À faire** : comparez les tailles des images.

```bash
docker images | grep yapuka-front
```

#### Correction 5.4 {collapsible="true"}

```bash
$ docker images | grep yapuka-front
yapuka-front   prod     ~45MB
```

Comparaison avec l'image de dev qui utilise `node:22-alpine` + tous les `node_modules/` + Vite dev server :

| Image | Base                       | Contenu                                 | Taille      |
|-------|----------------------------|-----------------------------------------|-------------|
| Dev   | `node:22-alpine` (~180 MB) | `node_modules/` (~200 MB) + code source | **~400 MB** |
| Prod  | `nginx:alpine` (~7 MB)     | Fichiers statiques (~1 MB)              | **~45 MB**  |

L'image de production est **~9x plus petite**. Elle démarre aussi instantanément (Nginx vs compilation Vite).

## Critères de validation

| #  | Critère                                                                                | Points  |
|----|----------------------------------------------------------------------------------------|---------|
| 1  | Le `.dockerignore` exclut `node_modules/`, `dist/`, `cypress/`, et les fichiers de dev | /2      |
| 2  | Le `docker-entrypoint.sh` génère `config.js` à partir de la variable `API_URL`         | /3      |
| 3  | Le `client.js` lit `window.__CONFIG__.API_URL` en priorité                             | /2      |
| 4  | Le Dockerfile utilise un multi-stage build avec 3 stages (deps, builder, production)   | /2      |
| 5  | Le stage `deps` utilise `npm ci` et copie d'abord `package.json` + `package-lock.json` | /2      |
| 6  | Le stage `production` utilise `nginx:alpine` (pas `node:alpine`)                       | /2      |
| 7  | La config Nginx implémente `try_files $uri $uri/ /index.html` pour le SPA routing      | /2      |
| 8  | Les assets ont un cache de 1 an, `index.html` et `config.js` n'ont pas de cache        | /2      |
| 9  | `docker build` réussit et le conteneur sert la page sur `http://localhost:3000`        | /1      |
| 10 | La même image fonctionne avec deux URLs d'API différentes (test 5.3)                   | /2      |
|    | **Total**                                                                              | **/20** |

## Pour aller plus loin

Si vous avez terminé en avance :

- **Healthcheck** : ajoutez une instruction `HEALTHCHECK` qui vérifie que Nginx répond sur `/` avec
  `curl -f http://localhost/ || exit 1`. Pensez à installer `curl` dans l'image Alpine.
- **CSP (Content Security Policy)** : ajoutez un header `Content-Security-Policy` dans la config Nginx pour restreindre
  les sources de scripts et de styles.
- **User non-root** : Nginx Alpine tourne en root par défaut. Configurez-le pour tourner en tant qu'utilisateur
  `nginx` (modifiez `nginx.conf` principal et les permissions des dossiers de cache/PID).
- **Variables multiples** : étendez le script d'injection pour supporter plusieurs variables (`API_URL`, `APP_TITLE`,
  `FEATURE_FLAGS`…) avec une boucle qui lit toutes les variables préfixées par `FRONT_`.