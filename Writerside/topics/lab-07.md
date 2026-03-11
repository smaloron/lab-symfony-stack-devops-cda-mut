# Lab 7 — Orchestration avec Docker Swarm

## Objectifs du lab

À l'issue de ce lab, vous serez capable de :

- Créer un cluster Docker Swarm multi-nœuds avec Multipass
- Adapter un fichier `docker-compose.yml` en stack Swarm
- Déployer, scaler et mettre à jour une application en production
- Tester la haute disponibilité et le self-healing



> **Prérequis** : Docker, Docker Compose, notions de réseau, lab précédent (Yapuka) terminé

---

## 7.1 — Comprendre Docker Swarm

Avant de mettre les mains dans le terminal, prenons 10 minutes pour comprendre ce que nous allons construire.

### Concepts clés

Docker Swarm est le moteur d'orchestration natif de Docker. Il transforme un ensemble de machines (physiques ou
virtuelles) en un **cluster unifié** capable de :

- **Distribuer** les conteneurs sur plusieurs machines
- **Équilibrer** la charge automatiquement (routing mesh)
- **Réparer** les pannes sans intervention (self-healing)
- **Mettre à jour** les services sans interruption (rolling update)

Un cluster Swarm est composé de deux types de nœuds :

- **Manager** : orchestre le cluster, prend les décisions de placement, expose l'API Swarm. C'est ici qu'on exécute les
  commandes `docker service`, `docker stack`, etc.
- **Worker** : exécute les conteneurs (appelés *tasks*). Ne prend aucune décision, obéit au manager.

### Architecture cible

```
┌─────────────────────────────────────────────────┐
│                  Votre machine                  │
│                                                 │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐  │
│  │   manager   │ │  worker-1  │ │  worker-2  │  │
│  │ (VM - 2CPU) │ │ (VM - 2CPU)│ │ (VM - 2CPU)│  │
│  │ 4GB / 20GB  │ │ 4GB / 20GB │ │ 4GB / 20GB │  │
│  │             │ │            │ │            │  │
│  │ nginx ×1    │ │ php ×1     │ │ php ×1     │  │
│  │ postgres ×1 │ │ front ×1   │ │ front ×1   │  │
│  │ redis ×1    │ │            │ │            │  │
│  └─────────────┘ └────────────┘ └────────────┘  │
└─────────────────────────────────────────────────┘
```

### Exercice de réflexion

Avant de continuer, répondez à ces questions dans un fichier `notes-swarm.md` :

1. Pourquoi le manager ne devrait-il pas exécuter les conteneurs applicatifs en production ?
2. Pourquoi recommande-t-on un nombre **impair** de managers (1, 3, 5) ?
3. Quelle est la différence entre un réseau `bridge` (Docker Compose) et un réseau `overlay` (Swarm) ?

### Correction {collapsible="true" id="correction_1"}

1. Le manager gère l'état du cluster (base Raft). S'il est surchargé par des conteneurs applicatifs, il peut devenir
   lent à prendre des décisions de placement ou à répondre aux heartbeats des workers, ce qui peut déstabiliser tout le
   cluster.

2. Swarm utilise l'algorithme de consensus **Raft** pour élire un leader parmi les managers. Il faut une majorité (
   quorum) pour prendre des décisions. Avec 3 managers, on tolère 1 panne. Avec 2, aucune panne tolérée (pas de
   majorité). Un nombre impair optimise donc le ratio tolérance/ressources.

3. Un réseau `bridge` est local à une seule machine Docker. Un réseau `overlay` s'étend sur **plusieurs nœuds** du
   cluster Swarm, permettant aux conteneurs de communiquer entre machines comme s'ils étaient sur le même réseau local.
   L'overlay encapsule les paquets via VXLAN.

---

## 7.2 — Création des VMs avec Multipass

[Multipass](https://multipass.run/) est un outil de Canonical qui permet de créer des VMs Ubuntu légères en quelques
secondes. Nous l'utilisons pour simuler un cluster de 3 machines.

### Installation de Multipass

Installez Multipass selon votre système d'exploitation depuis [multipass.run/install](https://multipass.run/install).

Vérifiez l'installation :

```bash
multipass version
```

### Création des 3 VMs

Nous avons besoin de 3 VMs : un manager et deux workers.

Créez-les avec les ressources suivantes :

| VM       | CPU | RAM | Disque |
|----------|-----|-----|--------|
| manager  | 2   | 4G  | 20G    |
| worker-1 | 2   | 4G  | 20G    |
| worker-2 | 2   | 4G  | 20G    |

> **À vous de jouer** : utilisez la commande `multipass launch` avec les options `--name`, `--cpus`, `--memory` et
`--disk` pour créer les 3 VMs.

Une fois créées, vérifiez leur état et **notez les adresses IP** de chaque VM — vous en aurez besoin tout au long du
lab.

> **Astuce** : `multipass list` affiche l'état et l'IP de chaque VM.

### Correction {collapsible="true" id="correction_2"}

```bash
# Création des 3 VMs
multipass launch --name manager --cpus 2 --memory 4G --disk 20G
multipass launch --name worker-1 --cpus 2 --memory 4G --disk 20G
multipass launch --name worker-2 --cpus 2 --memory 4G --disk 20G

# Vérification
multipass list
```

Résultat attendu (les IPs varieront) :

```
Name        State    IPv4            Image
manager     Running  192.168.64.2    Ubuntu 24.04 LTS
worker-1    Running  192.168.64.3    Ubuntu 24.04 LTS
worker-2    Running  192.168.64.4    Ubuntu 24.04 LTS
```

Vous pouvez aussi tester la connectivité entre VMs :

```bash
# Depuis le manager, pinger un worker
multipass exec manager -- ping -c 2 <IP_WORKER_1>
```

---

## 7.3 — Installation de Docker sur les VMs

Docker doit être installé sur **chacune des 3 VMs**. Nous utilisons le script d'installation officiel.

### À vous de jouer {id="vous-de-jouer_1"}

Pour chaque VM, exécutez le script d'installation Docker officiel via `multipass exec`. Le script se trouve à l'URL
`https://get.docker.com` et s'exécute avec `sh`.

Après l'installation, ajoutez l'utilisateur `ubuntu` au groupe `docker` pour éviter d'utiliser `sudo` :

```bash
multipass exec <nom-vm> -- sudo usermod -aG docker ubuntu
```

> **Important** : après avoir ajouté l'utilisateur au groupe docker, il faut redémarrer la session. Utilisez
`multipass restart <nom-vm>` ou `multipass exec <nom-vm> -- newgrp docker`.

Vérifiez l'installation sur chaque VM avec `docker --version`.

### Correction {collapsible="true" id="correction_3"}

```bash
# Installation de Docker sur les 3 VMs
multipass exec manager -- bash -c "curl -fsSL https://get.docker.com | sh"
multipass exec worker-1 -- bash -c "curl -fsSL https://get.docker.com | sh"
multipass exec worker-2 -- bash -c "curl -fsSL https://get.docker.com | sh"

# Ajout de l'utilisateur ubuntu au groupe docker
multipass exec manager -- sudo usermod -aG docker ubuntu
multipass exec worker-1 -- sudo usermod -aG docker ubuntu
multipass exec worker-2 -- sudo usermod -aG docker ubuntu

# Redémarrage pour appliquer le changement de groupe
multipass restart manager worker-1 worker-2

# Vérification (sans sudo)
multipass exec manager -- docker --version
multipass exec worker-1 -- docker --version
multipass exec worker-2 -- docker --version
```

---

## 7.4 — Initialisation du cluster Swarm

C'est le moment de transformer ces 3 machines indépendantes en un cluster Swarm.

### Étape 1 : Initialiser le Swarm sur le manager

Connectez-vous au manager :

```bash
multipass shell manager
```

Initialisez le Swarm. La commande `docker swarm init` accepte l'option `--advertise-addr` qui indique aux autres nœuds
sur quelle IP contacter le manager.

> **À vous de jouer** : initialisez le Swarm sur le manager en utilisant son adresse IP.

Lorsque l'initialisation réussit, Docker affiche une commande `docker swarm join` avec un token. **Copiez cette commande
** — elle servira à connecter les workers.

### Étape 2 : Connecter les workers

Ouvrez un nouveau terminal pour chaque worker (`multipass shell worker-1`, etc.) et exécutez la commande
`docker swarm join` copiée précédemment.

### Étape 3 : Vérification {id="tape-3-v-rification_1"}

De retour sur le manager, vérifiez l'état du cluster.

> **À vous de jouer** : quelle commande affiche la liste des nœuds du cluster avec leur rôle et leur statut ?

Vous devez voir 3 nœuds : 1 Leader (manager) et 2 workers, tous au statut `Ready`.

### Correction {collapsible="true" id="correction_4"}

```bash
# Sur le manager
multipass shell manager

# Initialisation (remplacez par l'IP réelle du manager)
docker swarm init --advertise-addr 192.168.64.2

# La commande affiche quelque chose comme :
# docker swarm join --token SWMTKN-1-xxxxx 192.168.64.2:2377

# Sur worker-1
multipass shell worker-1
docker swarm join --token SWMTKN-1-xxxxx 192.168.64.2:2377

# Sur worker-2
multipass shell worker-2
docker swarm join --token SWMTKN-1-xxxxx 192.168.64.2:2377

# Vérification sur le manager
docker node ls
```

Résultat attendu :

```
ID             HOSTNAME   STATUS   AVAILABILITY   MANAGER STATUS   ENGINE VERSION
abc123 *       manager    Ready    Active         Leader           27.x.x
def456         worker-1   Ready    Active                          27.x.x
ghi789         worker-2   Ready    Active                          27.x.x
```

> **Astuce** : si vous perdez le token, récupérez-le sur le manager avec :
> ```bash
> docker swarm join-token worker
> ```

---

## 7.5 — Préparation des images Docker

En mode Docker Compose local, Docker construit les images à la volée. En mode Swarm, les workers doivent **pouvoir
accéder aux images**. Comme nous n'avons pas de registry distant, nous allons mettre en place un **registry privé** sur le
manager.

### Étape 1 : Lancer un registry local

Sur le manager, déployez un registry Docker en tant que service Swarm :

```bash
docker service create --name registry --publish published=5000,target=5000 registry:2
```

Vérifiez qu'il tourne :

```bash
docker service ls
curl http://localhost:5000/v2/_catalog
```

> **Pourquoi `localhost:5000` fonctionne depuis les workers ?**
>
> Le registry est déployé en tant que **service Swarm** avec un port publié. Grâce au **routing mesh**, le port 5000 est
> accessible sur **tous les nœuds** du cluster, pas seulement sur celui qui héberge le conteneur. Quand un worker accède
> à `localhost:5000`, la requête est routée automatiquement vers le nœud qui exécute le registry.
>
> De plus, Docker autorise par défaut les connexions HTTP (non-TLS) vers `localhost`. Si vous utilisiez l'IP du manager
> au lieu de `localhost` dans les tags d'images, il faudrait configurer Docker pour accepter ce registry comme
> « insecure registry » sur chaque nœud.

### Étape 2 : Builder et pousser les images

Toujours sur le manager, vous devez copier le code source du projet Yapuka, builder les images et les pousser dans le
registry local.

> **À vous de jouer** :
>
> 1. Transférez le dossier du projet sur le manager (utilisez `multipass transfer` ou `multipass mount`)
> 2. Buildez les images pour le backend PHP et le frontend React en les taguant avec le préfixe `localhost:5000/` (ex :
     `localhost:5000/yapuka-php:v1`)
> 3. Poussez-les dans le registry avec `docker push`

### Correction {collapsible="true" id="correction_5"}

```bash
# Depuis votre machine hôte — transférer le code
# Option 1 : monter le dossier
multipass mount /chemin/vers/yapuka manager:/home/ubuntu/yapuka

# Option 2 : copier les fichiers
multipass transfer -r /chemin/vers/yapuka manager:/home/ubuntu/yapuka

# Sur le manager
multipass shell manager
cd ~/yapuka

# Builder les images avec le tag du registry local
docker build -t localhost:5000/yapuka-php:v1 ./api
docker build -t localhost:5000/yapuka-front:v1 ./front

# Pousser les images dans le registry
docker push localhost:5000/yapuka-php:v1
docker push localhost:5000/yapuka-front:v1

# Vérifier que les images sont dans le registry
curl http://localhost:5000/v2/_catalog
# Résultat attendu : {"repositories":["yapuka-php","yapuka-front"]}
```

> **Note** : les images `nginx:alpine`, `postgres:16-alpine` et `redis:7-alpine` sont des images publiques. Swarm les
> tirera automatiquement depuis Docker Hub sur chaque nœud.

---

## 7.6 — Adaptation du docker-compose.yml en Stack Swarm

Le fichier `docker-compose.yml` utilisé en développement ne convient pas directement à Swarm. Il faut l'adapter en
ajoutant des directives `deploy` et en remplaçant les `build` par des `image`.

### Différences clés entre Compose et Stack

| Aspect            | Docker Compose       | Docker Stack (Swarm)                                        |
|-------------------|----------------------|-------------------------------------------------------------|
| Build             | `build: ./api`       | ❌ Non supporté — utiliser `image:`                          |
| Volumes bind      | `./api:/var/www/api` | ❌ Déconseillé — les fichiers ne sont pas sur tous les nœuds |
| Réseau par défaut | `bridge`             | `overlay`                                                   |
| Réplication       | Non                  | `deploy.replicas`                                           |
| Placement         | Non                  | `deploy.placement.constraints`                              |
| Restart           | `restart: always`    | `deploy.restart_policy`                                     |
| depends_on        | ✅ Supporté           | ❌ Ignoré silencieusement                                    |

### À vous de jouer {id="vous-de-jouer_2"}

Créez un fichier `stack.yml` à la racine du projet en adaptant le `docker-compose.yml` existant. Voici les consignes :

**Services et replicas :**

| Service  | Image                             | Replicas | Contrainte de placement |
|----------|-----------------------------------|----------|-------------------------|
| nginx    | `nginx:alpine`                    | 1        | manager                 |
| php      | `localhost:5000/yapuka-php:v1`    | 2        | worker                  |
| front    | `localhost:5000/yapuka-front:v1`  | 2        | worker                  |
| database | `postgres:16-alpine`              | 1        | manager                 |
| redis    | `redis:7-alpine`                  | 1        | manager                 |

**Règles à respecter :**

1. Remplacer tous les `build:` par des `image:`
2. Supprimer les volumes bind mounts de développement (`./api:/var/www/api`, etc.)
3. Supprimer les `depends_on` — ils sont **ignorés par Swarm**. Vos services doivent être résilients et gérer eux-mêmes
   les retries si une dépendance n'est pas encore prête
4. Ajouter une section `deploy` à chaque service avec : `replicas`, `restart_policy` (condition: on-failure),
   `placement.constraints`
5. Pour `php` et `front`, ajouter une `update_config` avec `parallelism: 1` et `delay: 10s`
6. Pour `php` et `front`, ajouter une `rollback_config` avec `parallelism: 1` et `delay: 5s` pour configurer le
   comportement en cas de rollback
7. Remplacer le réseau `yapuka` par un réseau overlay (il suffit de changer le driver)
8. Conserver les volumes nommés pour `database` et `redis`
9. Le `healthcheck` de PostgreSQL doit rester
10. La configuration nginx doit être fournie via un **Docker config** ou montée différemment (voir astuce ci-dessous)

> **Astuce pour la config Nginx** : en Swarm, on utilise `docker config` pour distribuer des fichiers de configuration à
> tous les nœuds :
> ```bash
> docker config create nginx_conf ./docker/nginx/default.conf
> ```
> Puis dans le stack.yml :
> ```yaml
> configs:
>   nginx_conf:
>     external: true
>
> services:
>   nginx:
>     configs:
>       - source: nginx_conf
>         target: /etc/nginx/conf.d/default.conf
> ```

> **Important — `depends_on` et Swarm**
>
> Contrairement à Docker Compose, `docker stack deploy` **ignore silencieusement** la directive `depends_on`. Swarm
> démarre tous les services simultanément. C'est pourquoi vos applications doivent être conçues pour **tolérer
> l'indisponibilité temporaire** de leurs dépendances (retry de connexion à la base de données, etc.). Le healthcheck
> de PostgreSQL aide : Swarm ne considère le conteneur comme prêt qu'une fois le healthcheck passé, mais cela n'empêche
> pas les autres services de démarrer en parallèle.

### Correction {collapsible="true" id="correction_6"}

```yaml
# =============================================================================
# stack.yml - Déploiement Swarm de Yapuka
# =============================================================================
# Fichier adapté depuis docker-compose.yml pour Docker Swarm.
# Déploiement : docker stack deploy -c stack.yml yapuka
# =============================================================================

services:
  # ===========================================================================
  # Nginx - Reverse proxy (déployé sur le manager)
  # ===========================================================================
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == manager
    networks:
      - yapuka

  # ===========================================================================
  # PHP-FPM - Backend Symfony (répliqué sur les workers)
  # ===========================================================================
  php:
    image: localhost:5000/yapuka-php:v1
    environment:
      APP_ENV: prod
      DATABASE_URL: "postgresql://yapuka:yapuka@database:5432/yapuka?serverVersion=16"
      REDIS_URL: "redis://redis:6379"
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 5s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
    networks:
      - yapuka

  # ===========================================================================
  # Frontend React (répliqué sur les workers)
  # ===========================================================================
  front:
    image: localhost:5000/yapuka-front:v1
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 5s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
    networks:
      - yapuka

  # ===========================================================================
  # PostgreSQL - Base de données (sur le manager pour la persistence)
  # ===========================================================================
  database:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: yapuka
      POSTGRES_USER: yapuka
      POSTGRES_PASSWORD: yapuka
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U yapuka" ]
      interval: 5s
      timeout: 5s
      retries: 5
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == manager
    networks:
      - yapuka

  # ===========================================================================
  # Redis - Cache (sur le manager)
  # ===========================================================================
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == manager
    networks:
      - yapuka

# =============================================================================
# Configs (fichiers distribués à travers le cluster)
# =============================================================================
configs:
  nginx_conf:
    external: true

# =============================================================================
# Volumes persistants
# =============================================================================
volumes:
  db_data:
  redis_data:

# =============================================================================
# Réseau overlay (communication inter-nœuds)
# =============================================================================
networks:
  yapuka:
    driver: overlay
```

> **Note sur les volumes** : en Swarm, les volumes `db_data` et `redis_data` sont **locaux au nœud** où le conteneur
> tourne. C'est pourquoi on contraint PostgreSQL et Redis sur le manager — ainsi le volume reste toujours sur la même
> machine. En production, on utiliserait un volume driver distribué (NFS, Ceph, etc.) ou un service de base de données
> managé.

---

## 7.7 — Déploiement de la Stack

Tout est prêt. Déployons l'application sur le cluster.

### Étape 1 : Créer la config Nginx

Depuis le manager, créez la config Docker pour Nginx :

```bash
docker config create nginx_conf ./docker/nginx/default.conf
```

Vérifiez :

```bash
docker config ls
```

### Étape 2 : Déployer la stack

> **À vous de jouer** : utilisez la commande `docker stack deploy` avec l'option `-c` pour déployer le fichier
`stack.yml` sous le nom `yapuka`.

### Étape 3 : Vérification

Après le déploiement, exécutez les commandes suivantes **depuis le manager** et interprétez les résultats :

1. Listez les stacks déployées
2. Listez les services de la stack `yapuka`
3. Affichez la distribution des tasks du service `yapuka_php` (sur quels nœuds tournent-elles ?)
4. Accédez à l'application via `http://<IP_MANAGER>:8080` depuis votre machine hôte

> **Patience** : le premier déploiement peut prendre 1-2 minutes le temps que les workers tirent les images depuis le
> registry.

> **Routing mesh** : grâce au routing mesh de Swarm, l'application est accessible via le port 8080 de **n'importe quel
> nœud** du cluster (manager ou workers), pas seulement celui qui héberge Nginx.

### Correction {collapsible="true" id="correction_7"}

```bash
# Créer la config Nginx
docker config create nginx_conf ./docker/nginx/default.conf

# Déployer la stack
docker stack deploy -c stack.yml yapuka

# Vérifications (depuis le manager)
docker stack ls
# NAME      SERVICES   ORCHESTRATOR
# yapuka    5          Swarm

docker stack services yapuka
# ID        NAME              MODE        REPLICAS   IMAGE
# xxx       yapuka_nginx      replicated  1/1        nginx:alpine
# xxx       yapuka_php        replicated  2/2        localhost:5000/yapuka-php:v1
# xxx       yapuka_front      replicated  2/2        localhost:5000/yapuka-front:v1
# xxx       yapuka_database   replicated  1/1        postgres:16-alpine
# xxx       yapuka_redis      replicated  1/1        redis:7-alpine

# Voir où tournent les tasks PHP
docker service ps yapuka_php
# ID        NAME            NODE       DESIRED STATE   CURRENT STATE
# xxx       yapuka_php.1    worker-1   Running         Running 30 seconds ago
# xxx       yapuka_php.2    worker-2   Running         Running 28 seconds ago

# Test d'accès (depuis le manager)
curl -s http://localhost:8080/api/docs | head -5

# Test d'accès (depuis la machine hôte — utilisez l'IP du manager)
# curl -s http://<IP_MANAGER>:8080/api/docs | head -5
```

> **Résolution de problèmes** : si des services restent à 0/N replicas, consultez les logs :
> ```bash
> docker service logs yapuka_php
> ```

---

## 7.8 — Scaling des services

L'un des intérêts majeurs de Swarm est le **scaling horizontal** : augmenter ou diminuer le nombre de replicas d'un
service à la volée.

### À vous de jouer {id="vous-de-jouer_3"}

1. Scalez le service `yapuka_php` de 2 à **4 replicas**
2. Observez la distribution des nouvelles tasks avec `docker service ps`
3. Vérifiez que l'application reste accessible
4. Redescendez à **2 replicas**
5. Observez ce qui se passe (quelles tasks sont arrêtées ?)

> **Indice** : la commande est `docker service scale <service>=<nombre>`

### Questions de réflexion

- Les nouvelles tasks sont-elles réparties équitablement entre les workers ?
- Que se passe-t-il si on scale à un nombre supérieur au nombre de workers ?

### Correction {collapsible="true" id="correction_8"}

```bash
# Scaler PHP à 4 replicas
docker service scale yapuka_php=4

# Observer la distribution
docker service ps yapuka_php
# ID        NAME            NODE       DESIRED STATE   CURRENT STATE
# xxx       yapuka_php.1    worker-1   Running         Running 5 minutes ago
# xxx       yapuka_php.2    worker-2   Running         Running 5 minutes ago
# xxx       yapuka_php.3    worker-1   Running         Running 10 seconds ago
# xxx       yapuka_php.4    worker-2   Running         Running 8 seconds ago

# Tester l'accès (depuis le manager)
curl http://localhost:8080/api/tasks/stats

# Réduire à 2 replicas
docker service scale yapuka_php=2

# Observer : les tasks les plus récentes sont supprimées en premier
docker service ps yapuka_php
```

**Réponses aux questions** :

- Swarm utilise une stratégie de **spread** par défaut : il répartit les tasks sur les nœuds ayant le moins de tasks de
  ce service, donc oui, la répartition est équitable.
- Si on scale à plus que le nombre de workers, plusieurs tasks tourneront sur le même nœud. Swarm n'interdit pas cela —
  il place simplement les tasks là où c'est possible.

---

## 7.9 — Rolling Update (mise à jour sans interruption)

Simulons une mise à jour de l'application backend sans interruption de service.

### Étape 1 : Modifier l'application

Sur le manager, faites une modification visible dans le code PHP (par exemple, changez le message de réponse dans un
contrôleur, ou ajoutez un header personnalisé).

### Étape 2 : Builder et pousser la nouvelle version

> **À vous de jouer** :
>
> 1. Rebuilder l'image PHP avec un **nouveau tag** (ex : `localhost:5000/yapuka-php:v2`)
> 2. Poussez cette image dans le registry
> 3. Mettez à jour le service pour utiliser la nouvelle image avec `docker service update`

> **Indice** : `docker service update --image <nouvelle-image> <nom-du-service>`

### Étape 3 : Observer le rolling update

Pendant la mise à jour, exécutez dans un autre terminal :

```bash
watch docker service ps yapuka_php
```

Vous devriez voir les anciennes tasks s'arrêter une par une et les nouvelles démarrer, conformément à la configuration
`update_config` (parallelism: 1, delay: 10s).

### Correction {collapsible="true" id="correction_9"}

```bash
# Builder la v2
docker build -t localhost:5000/yapuka-php:v2 ./api

# Pousser dans le registry
docker push localhost:5000/yapuka-php:v2

# Mettre à jour le service (rolling update)
docker service update --image localhost:5000/yapuka-php:v2 yapuka_php

# Observer en temps réel (dans un autre terminal)
watch docker service ps yapuka_php

# On voit les tasks se remplacer une par une :
# yapuka_php.1  localhost:5000/yapuka-php:v2  worker-1  Running
# yapuka_php.1  localhost:5000/yapuka-php:v1  worker-1  Shutdown
# yapuka_php.2  localhost:5000/yapuka-php:v2  worker-2  Running   (après 10s)
# yapuka_php.2  localhost:5000/yapuka-php:v1  worker-2  Shutdown
```

> **Point clé** : pendant le rolling update, au moins 1 replica de l'ancienne version reste active tant que la nouvelle
> n'est pas prête. L'application reste donc accessible en permanence.

---

## 7.10 — Rollback

Que se passe-t-il si la v2 est cassée ? Swarm permet de revenir à la version précédente instantanément.

### À vous de jouer {id="vous-de-jouer_4"}

1. Effectuez un rollback du service `yapuka_php`
2. Vérifiez que le service est revenu à l'image précédente
3. Consultez l'historique des tasks pour voir les deux versions

> **Indice** : la commande est `docker service rollback <nom-du-service>`

### Correction {collapsible="true" id="correction_10"}

```bash
# Rollback
docker service rollback yapuka_php

# Vérifier l'image utilisée
docker service inspect --pretty yapuka_php | grep Image
# Image: localhost:5000/yapuka-php:v1

# Voir l'historique complet
docker service ps yapuka_php
# Montre les tasks v2 (Shutdown) et les tasks v1 restaurées (Running)
```

> **Note** : le rollback utilise la `rollback_config` définie dans le `stack.yml`. Dans notre cas, les replicas sont
> restaurés un par un (`parallelism: 1`) avec un délai de 5 secondes entre chaque.

---

## 7.11 — Tests de haute disponibilité

C'est le moment de casser des choses volontairement pour vérifier que Swarm tient ses promesses.

### Test 1 : Panne d'un worker

> **À vous de jouer** :
>
> 1. Depuis votre machine hôte, arrêtez brutalement `worker-1` avec `multipass stop worker-1`
> 2. Sur le manager, observez l'état des nœuds (`docker node ls`)
> 3. Observez la redistribution des tasks (`docker service ps yapuka_php`)
> 4. Testez que l'application est toujours accessible via `http://<IP_MANAGER>:8080`
> 5. Redémarrez `worker-1` avec `multipass start worker-1`
> 6. Observez sa réintégration dans le cluster

### Test 2 : Panne d'un conteneur

> **À vous de jouer** :
>
> 1. Identifiez un conteneur PHP qui tourne sur un worker : `docker service ps yapuka_php`
> 2. Connectez-vous au worker concerné et tuez le conteneur avec `docker kill <id>`
> 3. Observez sur le manager : Swarm relance-t-il automatiquement un nouveau conteneur ?

### Questions

- Combien de temps faut-il à Swarm pour détecter la panne du worker ?
- Les tasks sont-elles renvoyées sur `worker-1` quand il revient en ligne ?

### Correction {collapsible="true" id="correction_11"}

```bash
# --- Test 1 : Panne d'un worker ---

# Arrêter worker-1 (depuis la machine hôte)
multipass stop worker-1

# Sur le manager : observer (attendre ~30 secondes)
docker node ls
# worker-1 passe en status "Down"

docker service ps yapuka_php
# Les tasks de worker-1 sont relancées sur worker-2
# NAME               NODE       DESIRED STATE   CURRENT STATE
# yapuka_php.1       worker-2   Running         Running 10 seconds ago
# \_ yapuka_php.1   worker-1   Shutdown        Running 5 minutes ago
# yapuka_php.2       worker-2   Running         Running 5 minutes ago

# L'application reste accessible !
# Depuis le manager :
curl http://localhost:8080/api/tasks/stats
# Depuis la machine hôte :
# curl http://<IP_MANAGER>:8080/api/tasks/stats

# Redémarrer worker-1 (depuis la machine hôte)
multipass start worker-1

# Observer la réintégration (le nœud revient en "Ready")
docker node ls

# --- Test 2 : Panne d'un conteneur ---

# Sur le worker, trouver l'ID du conteneur
multipass exec worker-2 -- docker ps --filter name=yapuka_php

# Tuer le conteneur
multipass exec worker-2 -- docker kill <CONTAINER_ID>

# Sur le manager, observer le redémarrage automatique
docker service ps yapuka_php
# Swarm relance un nouveau conteneur en quelques secondes
```

**Réponses :**

- Swarm détecte la panne d'un nœud en environ **5 à 30 secondes** (configurable). Il replanifie ensuite les tasks sur
  les nœuds disponibles.
- Non, les tasks ne sont **pas automatiquement rééquilibrées** quand un nœud revient. Elles restent sur le nœud où elles
  ont été replanifiées. Pour forcer un rééquilibrage, on peut utiliser `docker service update --force yapuka_php`.

---

## 7.12 — Monitoring et logs

En production, il est essentiel de pouvoir consulter les logs et l'état des services.

### À vous de jouer

Depuis le manager, exécutez les commandes nécessaires pour :

1. Afficher les logs du service `yapuka_php` (les 50 dernières lignes)
2. Suivre les logs en temps réel (mode `follow`)
3. Inspecter la configuration détaillée du service `yapuka_php`
4. Afficher les statistiques de consommation de ressources des conteneurs sur le manager
5. Lister les tasks de **tous** les services de la stack

> **Indices** : `docker service logs`, `docker service inspect --pretty`, `docker stats`, `docker stack ps`

### Correction {collapsible="true" id="correction_12"}

```bash
# 1. Dernières 50 lignes de logs
docker service logs --tail 50 yapuka_php

# 2. Suivi en temps réel
docker service logs -f yapuka_php
# Ctrl+C pour arrêter

# 3. Inspection du service
docker service inspect --pretty yapuka_php

# 4. Stats de ressources
docker stats --no-stream

# 5. Toutes les tasks de la stack
docker stack ps yapuka
```

---

## 7.13 — Nettoyage

Le lab est terminé. Nettoyons l'environnement.

### Étapes de nettoyage

```bash
# 1. Supprimer la stack (sur le manager)
multipass exec manager -- docker stack rm yapuka

# 2. Supprimer la config et le registry
multipass exec manager -- docker config rm nginx_conf
multipass exec manager -- docker service rm registry

# 3. Quitter le Swarm (sur chaque nœud)
multipass exec worker-1 -- docker swarm leave
multipass exec worker-2 -- docker swarm leave
multipass exec manager -- docker swarm leave --force

# 4. Supprimer les VMs
multipass delete manager worker-1 worker-2
multipass purge
```

---

## Récapitulatif

Voici les commandes essentielles vues dans ce lab :

| Action               | Commande                                        |
|----------------------|-------------------------------------------------|
| Initialiser Swarm    | `docker swarm init --advertise-addr <IP>`       |
| Rejoindre un cluster | `docker swarm join --token <TOKEN> <IP>:2377`   |
| Lister les nœuds     | `docker node ls`                                |
| Déployer une stack   | `docker stack deploy -c stack.yml <nom>`        |
| Lister les services  | `docker stack services <nom>`                   |
| Voir les tasks       | `docker service ps <service>`                   |
| Scaler un service    | `docker service scale <service>=N`              |
| Mettre à jour        | `docker service update --image <img> <service>` |
| Rollback             | `docker service rollback <service>`             |
| Logs                 | `docker service logs <service>`                 |
| Supprimer une stack  | `docker stack rm <nom>`                         |

### Points clés à retenir

- Swarm transforme plusieurs machines en un cluster unifié piloté par des **managers**
- Les **workers** exécutent les conteneurs, les **managers** décident du placement
- Le **routing mesh** permet d'accéder à un service depuis **n'importe quel nœud**
- Le **rolling update** met à jour les replicas une par une sans interruption
- Le **self-healing** redémarre automatiquement les conteneurs tombés et replanifie les tasks en cas de panne d'un nœud
- Les **volumes locaux** ne sont pas partagés entre nœuds — c'est pourquoi on contraint les services avec état (DB,
  cache) sur un nœud spécifique
- `depends_on` est **ignoré en Swarm** — les services doivent être résilients face à l'indisponibilité temporaire de
  leurs dépendances
