# 4. Dockerisation production d'un frontend React

## De l'atelier au magasin : comprendre les deux modes d'une SPA

Imaginez une boulangerie artisanale. En coulisses, le boulanger dispose
d'un laboratoire complet : fours, pétrin, balance de précision, et tout
l'équipement pour expérimenter de nouvelles recettes. C'est bruyant,
encombrant, et ça consomme beaucoup d'énergie. Mais en vitrine, il
n'expose que le résultat final : des pains et des viennoiseries, prêts
à être consommés.

Une application React fonctionne exactement de cette manière.

- **En développement**, Vite est votre laboratoire : serveur local,
  rechargement automatique à chaque modification (Hot Module
  Replacement), sourcemaps, messages d'erreur détaillés. C'est lourd
  (~400 Mo d'outils), mais puissant pour travailler.

- **En production**, seul le résultat compte : un dossier `dist/`
  contenant quelques fichiers HTML, CSS et JavaScript compilés, qui
  tiennent en quelques centaines de kilo-octets. Node.js n'est plus
  nécessaire — n'importe quel serveur web peut les distribuer.

C'est cette distinction fondamentale qui rend possible une image Docker
de production très légère : environ 45 Mo au lieu de 400 Mo.

---

## Le processus de build Vite

Quand vous lancez `npm run build`, Vite exécute une transformation
complète de votre code source. Il résout toutes les dépendances,
compile le JSX en JavaScript standard, optimise et minifie le code,
puis génère un dossier `dist/` autonome.

```mermaid
flowchart TD
    A["Code source<br />(JSX, CSS, assets)"] --> B["npm run build<br />(Vite)"]
    B --> C["dist/"]
    C --> D["index.html<br />(point d'entrée)"]
    C --> E["assets/<br />index-a1b2c3.js<br />index-d4e5f6.css"]
```

Remarquez les noms des fichiers dans `assets/` : ils contiennent un
**hash** calculé à partir du contenu. Si vous modifiez une ligne de
code, le hash change et le nom du fichier change. Cela permet une
stratégie de cache très agressive : un navigateur peut conserver ces
fichiers en cache pendant un an, en sachant que si leur contenu change,
leur URL changera aussi.

`index.html`, lui, ne contient pas de hash. Il référence les autres
fichiers par leur nom exact. C'est lui qui doit toujours être servi
frais, sans cache.

---

## Le défi des variables d'environnement dans une SPA

Voici le piège classique, et l'une des difficultés centrales de ce lab.

Dans une application serveur (Node.js, PHP, Python), les variables
d'environnement sont lues au moment où le serveur démarre. Vous pouvez
les changer, redémarrer, et le comportement change. C'est souple.

Dans une SPA compilée par Vite, c'est différent. Imaginez un livre
imprimé : une fois sorti de l'imprimerie, vous ne pouvez plus changer
le texte. Les variables `VITE_*` sont lues par Vite **au moment du
build** et leur valeur est littéralement inscrite dans les fichiers
JavaScript générés. Après compilation, elles sont figées.

```mermaid
sequenceDiagram
    participant Dev as Développeur
    participant Vite as Vite (build)
    participant JS as Fichier JS compilé
    participant Browser as Navigateur
    Dev ->> Vite: npm run build
    Note over Vite: Lit VITE_API_URL=http://api.dev
    Vite ->> JS: Remplace import.meta.env.VITE_API_URL<br/>par "http://api.dev" en dur
    Browser ->> JS: Télécharge et exécute
    Note over Browser: La valeur est figée,<br/>impossible à changer
```

Cela pose un problème concret : si vous construisez votre image Docker
avec l'URL de votre API de développement, cette URL sera présente dans
le code de toutes vos instances, qu'elles tournent en staging ou en
production.

### La solution : l'injection runtime

Pour contourner cette contrainte, on utilise une astuce élégante : au
démarrage du conteneur, un script shell génère un fichier JavaScript
(`config.js`) qui expose les variables d'environnement Docker dans
l'objet global `window.__CONFIG__`. Ce fichier est chargé par le
navigateur avant tout le reste de l'application.

```mermaid
sequenceDiagram
    participant Docker as Docker run
    participant Script as docker-entrypoint.sh
    participant Nginx as Nginx
    participant Browser as Navigateur
    Docker ->> Script: Démarre le conteneur<br />avec API_URL=https://api.prod.com
    Script ->> Nginx: Génère config.js :<br />window.__CONFIG__ = { API_URL: "..." }
    Script ->> Nginx: Démarre Nginx
    Browser ->> Nginx: GET /config.js
    Nginx -->> Browser: window.__CONFIG__ = { API_URL: "..." }
    Browser ->> Nginx: GET /index.html et assets
    Note over Browser: L'application lit window.__CONFIG__.API_URL<br/>avant import.meta.env
```

Résultat : **une seule image Docker**, utilisable dans tous les
environnements. L'URL de l'API change via une variable d'environnement
au moment du `docker run`, sans jamais reconstruire l'image.

---

## Le multi-stage build : ne livrer que ce qui est nécessaire

Construire une image Docker pour une SPA en production, c'est comme
préparer un déménagement : vous n'emportez pas votre table de cuisine
dans votre nouveau bureau. Vous ne gardez que ce dont vous avez besoin
à destination.

Le **multi-stage build** de Docker permet d'utiliser plusieurs images
intermédiaires dans un seul Dockerfile. Seule la dernière image est
conservée dans l'image finale.

```mermaid
flowchart TD
    subgraph Stage1["Stage 1 : deps (node:22-alpine)"]
        S1A["package.json<br />package-lock.json"]
        S1B["npm ci<br />→ node_modules/"]
        S1A --> S1B
    end

    subgraph Stage2["Stage 2 : builder (node:22-alpine)"]
        S2A["node_modules/<br />(copié depuis deps)"]
        S2B["Code source<br />(JSX, CSS, assets)"]
        S2C["npm run build<br />→ dist/"]
        S2A --> S2C
        S2B --> S2C
    end

    subgraph Stage3["Stage 3 : production (nginx:alpine)"]
        S3A["dist/<br />(copié depuis builder)"]
        S3B["front.conf<br />(config Nginx)"]
        S3C["docker-entrypoint.sh"]
        S3D["Image finale ~45 Mo"]
        S3A --> S3D
        S3B --> S3D
        S3C --> S3D
    end

    Stage1 --> Stage2
    Stage2 --> Stage3
```

Les stages `deps` et `builder` servent uniquement pendant la
construction. Leurs centaines de mégaoctets de `node_modules/` et
d'outils Node.js n'entrent jamais dans l'image finale. Seul le
dossier `dist/` compilé est transféré vers le stage Nginx.

### ARG vs ENV dans Docker

Lors du stage `builder`, Vite a besoin de lire la variable
`VITE_API_URL`. Docker distingue deux types de variables :

| Type  | Disponible pendant le build | Disponible au runtime | Visible dans `docker inspect` |
|-------|-----------------------------|-----------------------|-------------------------------|
| `ARG` | Oui                         | Non                   | Non                           |
| `ENV` | Oui                         | Oui                   | Oui                           |

Pour que Vite accède à la valeur, il faut passer par les deux :

```docker
ARG VITE_API_URL=""
ENV VITE_API_URL=$VITE_API_URL
```

La valeur par défaut vide (`""`) signifie que Vite utilisera une URL
relative, ce qui est le comportement souhaité quand l'API et le
frontend partagent le même domaine derrière un reverse proxy.

---

## Nginx comme serveur de fichiers statiques

Nginx est ici utilisé dans son rôle le plus simple : distribuer des
fichiers. Pas de PHP, pas de Node.js, pas de logique applicative. Il
reçoit une requête HTTP, cherche le fichier correspondant sur le
disque, et le renvoie. C'est extrêmement rapide et peu gourmand en
ressources.

Mais une SPA introduit une subtilité importante.

### Le problème du routage SPA

Quand un utilisateur navigue vers `https://monapp.com/login`, il attend
de voir la page de connexion. React Router gère cette URL côté client.
Mais si l'utilisateur tape directement cette URL dans la barre
d'adresse, c'est Nginx qui reçoit la requête en premier, avant que
JavaScript ne soit chargé.

Nginx cherche alors un fichier `/login` sur le disque. Ce fichier
n'existe pas — seul `index.html` existe. Sans configuration adaptée,
Nginx renverrait une erreur 404.

La directive `try_files` résout ce problème :

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Elle indique à Nginx d'essayer dans l'ordre :

```mermaid
flowchart TD
    A["Requête : /login"] --> B{"Fichier /login<br />existe ?"}
    B -- " Oui " --> C["Servir le fichier"]
    B -- " Non " --> D{"Dossier /login/<br />existe ?"}
    D -- " Oui " --> E["Servir index.html<br />du dossier"]
    D -- " Non " --> F["Servir /index.html"]
    F --> G["React Router<br />prend le relais<br />et affiche /login"]
```

Les fichiers statiques réels (JS, CSS, images) sont servis directement.
Toutes les autres URLs renvoient `index.html`, laissant React Router
décider quoi afficher.

### La stratégie de cache

La configuration Nginx distingue deux catégories de fichiers selon leur
politique de cache :

```mermaid
flowchart LR
    subgraph "Cache 1 an"
        A["/assets/index-a1b2c3.js"]
        B["/assets/index-d4e5f6.css"]
    end
    subgraph "Pas de cache"
        C["/index.html"]
        D["/config.js"]
    end

    A -- " Hash dans le nom<br />Contenu immuable " --> E["Cache-Control: public, immutable"]
    B -- " Hash dans le nom<br />Contenu immuable " --> E
    C -- " Référence les assets<br />Doit pointer vers la dernière version " --> F["Cache-Control: no-cache"]
    D -- " Généré au démarrage<br />Contient l'URL d'API courante " --> F
```

`config.js` est particulièrement important à ne pas cacher : il est
regénéré à chaque démarrage du conteneur avec les variables
d'environnement courantes. Un cache navigateur le rendrait obsolète.

---

## L'entrypoint Docker : le chef d'orchestre du démarrage

Le script `docker-entrypoint.sh` est exécuté à chaque démarrage du
conteneur, avant Nginx. C'est lui qui lit les variables d'environnement
Docker et génère le fichier `config.js` dans le dossier servi par Nginx.

```mermaid
sequenceDiagram
    participant D as Docker
    participant E as entrypoint.sh
    participant F as /usr/share/nginx/html/config.js
    participant N as Nginx
    D ->> E: Démarre avec API_URL="https://api.prod.com"
    E ->> F: Écrit :<br />window.__CONFIG__ = { API_URL: "https://api.prod.com" }#59;
    E ->> N: exec nginx -g #39;daemon off#59;#39;
    Note over N: Prêt à servir les requêtes
```

L'instruction `exec` en dernière ligne est importante : elle remplace
le processus shell par le processus Nginx, de sorte que Nginx devienne
le processus principal (PID 1) du conteneur. C'est la convention
Docker pour une gestion correcte des signaux d'arrêt.

---

## Récapitulatif : la chaîne complète

Voici comment tous les éléments s'assemblent, de votre code source
jusqu'au navigateur de l'utilisateur :

```mermaid
flowchart TD
    subgraph Build["Phase de build (CI/CD)"]
        A["Code source React"] --> B["Stage deps<br />npm ci"]
        B --> C["Stage builder<br />npm run build"]
        C --> D["Stage production<br />nginx:alpine + dist/"]
        D --> E["Image Docker<br />yapuka-front:prod<br />~45 Mo"]
    end

    subgraph Deploy["Phase de déploiement"]
        E --> F["docker run<br />-e API_URL=https://api.prod.com"]
        F --> G["entrypoint.sh<br />génère config.js"]
        G --> H["Nginx démarre<br />port 80"]
    end

    subgraph Request["Requête utilisateur"]
        H --> I["GET /config.js<br />→ window.__CONFIG__"]
        I --> J["GET /index.html<br />→ point d'entrée React"]
        J --> K["GET /assets/index-abc.js<br />→ application compilée"]
        K --> L["React Router<br />gère la navigation"]
    end
```

Chaque étape a un rôle précis. Le lab vous guidera pour créer et
assembler ces pièces une à une. Bonne mise en pratique !