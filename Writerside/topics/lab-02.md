# Lab 2 — Tests BDD avec Behat

> **Prérequis** : le projet Yapuka tourne localement via `docker compose up -d`, les fixtures sont chargées, et vous pouvez vous connecter avec `demo@yapuka.dev` / `password`. Le lab précédent (Cypress) n'est pas requis.
> **Corrections** : voir le document compagnon `lab-behat-bdd-corrections.md`.

## Contexte

Le Behavior-Driven Development (BDD) est une méthodologie où les tests sont écrits **en langage naturel** avant d'être automatisés. L'idée est qu'un product owner, un testeur ou un développeur puissent tous lire et comprendre les spécifications.

**Behat** est le framework BDD de référence en PHP. Il utilise le langage **Gherkin** (Given / When / Then) pour décrire des scénarios métier, puis les relie à du code PHP qui les exécute.

Dans ce lab, vous allez :
1. Installer et configurer Behat dans le backend Symfony
2. Écrire des scénarios Gherkin pour l'API de gestion des tâches
3. Implémenter les steps en PHP
4. Exécuter les tests et valider le tout

> **Différence avec les tests Cypress** : Cypress teste l'application **du point de vue du navigateur** (clic, saisie, affichage). Behat teste l'API et la logique métier **du point de vue du comportement attendu**, sans navigateur.

## Partie 1 — Installation et configuration (20 min)

### 1.1 Installer Behat et ses extensions

Behat s'installe via Composer comme dépendance de développement. Vous aurez besoin de plusieurs packages.

**À faire** :

1. Ouvrez un shell dans le conteneur PHP :

```bash
docker compose exec php bash
```

2. Installez les packages suivants via Composer :

| Package                              | Rôle                                                                  |
|--------------------------------------|-----------------------------------------------------------------------|
| `behat/behat`                        | Framework BDD principal                                               |
| `friends-of-behat/symfony-extension` | Intègre Behat dans Symfony (accès au kernel, aux services, à la base) |
| `behatch/contexts`                   | Collection de contextes prêts à l'emploi (JSON, REST, etc.)           |

```bash
composer require --dev behat/behat friends-of-behat/symfony-extension behatch/contexts
```

3. Vérifiez l'installation :

```bash
vendor/bin/behat --version
```

#### Correction 1.1 — Installation {collapsible="true"}

La sortie attendue après `vendor/bin/behat --version` :

```
behat 3.x.x
```

Si Composer refuse l'installation à cause de conflits de dépendances, essayez :

```bash
composer require --dev behat/behat friends-of-behat/symfony-extension behatch/contexts --with-all-dependencies
```


### 1.2 Créer la structure de dossiers

Behat attend une arborescence conventionnelle à la racine du projet Symfony.

**À faire** : créez la structure suivante dans le dossier `api/` :

```
api/
├── features/
│   └── (vos fichiers .feature iront ici)
├── tests/
│   └── Behat/
│       └── FeatureContext.php
└── behat.yml
```

> Le dossier `tests/Behat/` contient les classes PHP de contexte (convention PSR-4). Le fichier `behat.yml` est le fichier de configuration principal de Behat.

#### Correction 1.2 — Structure de dossiers {collapsible="true"}

```bash
mkdir -p features
mkdir -p tests/Behat
touch tests/Behat/FeatureContext.php
touch behat.yml
```


### 1.3 Configurer behat.yml

Le fichier `behat.yml` définit comment Behat se connecte à Symfony, quels contextes utiliser, et où trouver les features.

**À faire** : configurez `behat.yml` avec les éléments suivants :

- **Suite par défaut** nommée `default`
- **Contextes** : votre `FeatureContext` personnalisé + le contexte REST de Behatch + le contexte JSON de Behatch
- **Extension Symfony** : chemin vers le kernel Symfony (`src/Kernel.php`) et l'environnement `test`
- **Extension Behatch** : activer avec l'URL de base de l'API

> **Indices** :
> - L'extension Symfony se déclare sous `extensions` > `FriendsOfBehat\SymfonyExtension`
> - L'extension Behatch se déclare sous `extensions` > `Behatch\Extension`
> - Les contextes Behatch sont : `Behatch\Context\RestContext` et `Behatch\Context\JsonContext`
> - Le kernel Symfony est dans `src/Kernel.php` avec la classe `App\Kernel`

> **Référence** : [Friends of Behat Symfony Extension](https://github.com/FriendsOfBehat/SymfonyExtension)

#### Correction 1.3 — behat.yml {collapsible="true"}

```yaml
# behat.yml
# =============================================================================
# Configuration Behat pour Yapuka API
# =============================================================================
# Définit la suite de tests, les contextes et les extensions utilisées.
# =============================================================================

default:
  suites:
    default:
      contexts:
        # Contexte personnalisé : nos steps sur mesure
        - App\Tests\Behat\FeatureContext
        # Contexte REST de Behatch : steps pour les requêtes HTTP
        - Behatch\Context\RestContext
        # Contexte JSON de Behatch : steps pour vérifier les réponses JSON
        - Behatch\Context\JsonContext

  extensions:
    # Intégration Symfony : accès au kernel, au container, à Doctrine
    FriendsOfBehat\SymfonyExtension:
      kernel:
        class: App\Kernel
        environment: test
        debug: true

    # Behatch : URL de base pour les requêtes REST
    Behatch\Extension: ~
```


### 1.4 Créer le FeatureContext de base

Le `FeatureContext` est la classe PHP qui contient vos steps personnalisés. Grâce à l'extension Symfony, elle peut accéder au container de services, à Doctrine, et à tous les services de l'application.

**À faire** : créez la classe `FeatureContext` dans `tests/Behat/FeatureContext.php`.

Votre contexte doit :
1. Implémenter `Behat\Behat\Context\Context`
2. Recevoir par injection le `EntityManagerInterface` de Doctrine, le `UserPasswordHasherInterface` et le `KernelInterface`
3. Stocker dans des propriétés privées : le token JWT courant, les données de la dernière réponse, l'id de la dernière tâche créée
4. Contenir un hook `@BeforeScenario` qui nettoie la base de données avant chaque scénario (supprimer toutes les tâches, puis tous les utilisateurs), réinitialise l'état interne, et crée un nouveau `KernelBrowser`

> **Indice d'injection** : avec l'extension Symfony, les services sont injectés automatiquement dans le constructeur si le contexte est déclaré en autowire.

> **Important** : pour que l'autoloading fonctionne, vérifiez que `composer.json` contient :
> ```json
> "autoload-dev": {
>     "psr-4": {
>         "App\\Tests\\": "tests/"
>     }
> }
> ```
> Puis lancez `composer dump-autoload`.

> **Important** : pour que l'injection fonctionne, ajoutez dans `config/services.yaml` :
> ```yaml
> when@test:
>     services:
>         App\Tests\Behat\FeatureContext:
>             autowire: true
>             autoconfigure: true
> ```

#### Correction 1.4 — FeatureContext de base {collapsible="true"}

Fichier `tests/Behat/FeatureContext.php` :

```php
<?php

// =============================================================================
// FeatureContext — Contexte principal des tests Behat
// =============================================================================
// Cette classe contient les steps personnalisés qui relient le langage
// Gherkin (Given/When/Then) au code PHP de l'application Symfony.
//
// Elle reçoit les services Symfony par injection de dépendances et
// nettoie la base de données avant chaque scénario.
// =============================================================================

namespace App\Tests\Behat;

use App\Entity\Task;
use App\Entity\User;
use Behat\Behat\Context\Context;
use Behat\Behat\Hook\Scope\BeforeScenarioScope;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\KernelBrowser;
use Symfony\Component\HttpKernel\KernelInterface;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

class FeatureContext implements Context
{
    // Client HTTP interne Symfony (pas de réseau)
    private ?KernelBrowser $client = null;

    // Token JWT de l'utilisateur connecté
    private ?string $jwtToken = null;

    // Données JSON de la dernière réponse (décodées)
    private ?array $responseData = null;

    // Id de la dernière tâche créée (pour les scénarios update/delete)
    private ?int $lastTaskId = null;

    public function __construct(
        private EntityManagerInterface $entityManager,
        private UserPasswordHasherInterface $passwordHasher,
        private KernelInterface $kernel,
    ) {
    }

    /**
     * Nettoie la base de données avant chaque scénario.
     * Garantit que chaque test part d'un état propre.
     *
     * @BeforeScenario
     */
    public function cleanDatabase(BeforeScenarioScope $scope): void
    {
        // Supprimer toutes les tâches puis tous les utilisateurs
        // (les tâches ont une FK vers users, donc les supprimer d'abord)
        $connection = $this->entityManager->getConnection();
        $connection->executeStatement('DELETE FROM tasks');
        $connection->executeStatement('DELETE FROM users');
        $this->entityManager->clear();

        // Réinitialiser l'état interne
        $this->jwtToken = null;
        $this->responseData = null;
        $this->lastTaskId = null;

        // Créer un nouveau client HTTP pour chaque scénario
        $this->client = new KernelBrowser($this->kernel);
    }
}
```

Autoloading dans `composer.json` :

```json
"autoload-dev": {
    "psr-4": {
        "App\\Tests\\": "tests/"
    }
}
```

Injection dans `config/services.yaml` :

```yaml
when@test:
    services:
        App\Tests\Behat\FeatureContext:
            autowire: true
            autoconfigure: true
```

Puis :

```bash
composer dump-autoload
```


### 1.5 Vérifier la configuration

**À faire** : lancez Behat en mode "dry run" pour vérifier que la configuration est correcte sans exécuter de test :

```bash
vendor/bin/behat --dry-run
```

Vous devriez voir : `No scenarios / No steps`. Pas d'erreur = la config est bonne.

#### Correction 1.5 — Vérification {collapsible="true"}

Si vous obtenez une erreur `FeatureContext class not found` :
- Vérifiez le namespace dans `behat.yml` : `App\Tests\Behat\FeatureContext`
- Vérifiez que le fichier est dans `tests/Behat/FeatureContext.php`
- Relancez `composer dump-autoload`

Si vous obtenez `Service not found` :
- Vérifiez que le bloc `when@test` est bien dans `services.yaml`
- Vérifiez que `APP_ENV=test` est bien défini (l'extension Symfony utilise `environment: test`)


## Partie 2 — Écrire les features en Gherkin (20 min)

### 2.1 Comprendre le Gherkin

Le Gherkin est un langage structuré avec des mots-clés :

| Mot-clé            | Rôle                                   | Exemple                               |
|--------------------|----------------------------------------|---------------------------------------|
| `Feature`          | Décrit la fonctionnalité testée        | `Feature: Gestion des tâches`         |
| `Background`       | Étapes exécutées avant chaque scénario | Créer un utilisateur de test          |
| `Scenario`         | Un cas de test concret                 | `Scenario: Créer une tâche`           |
| `Given`            | Pré-condition (état initial)           | `Given un utilisateur existe`         |
| `When`             | Action effectuée                       | `When je crée une tâche "CI/CD"`      |
| `Then`             | Résultat attendu                       | `Then le code de réponse est 201`     |
| `And` / `But`      | Suite de Given/When/Then               | `And la tâche apparaît dans la liste` |
| `Scenario Outline` | Scénario paramétré                     | Teste plusieurs combinaisons          |
| `Examples`         | Tableau de données pour Outline        | Valeurs injectées dans le scénario    |

> **Règle d'or** : un non-développeur doit pouvoir lire un fichier `.feature` et comprendre le comportement attendu de l'application.

### 2.2 Feature d'authentification

**À faire** : créez un fichier `features/auth.feature` qui décrit le comportement de l'API d'authentification.

Écrivez les éléments suivants :

**En-tête Feature** : décrivez la fonctionnalité avec le format "En tant que / Je veux / Afin de".

**Scénarios à écrire** :

1. **Inscription réussie** : un utilisateur envoie un POST avec email, username et password → il reçoit un 201 avec un token JWT et ses informations.

2. **Inscription avec email déjà existant** : un utilisateur qui existe déjà tente de s'inscrire avec le même email → il reçoit un 409.

3. **Inscription avec données invalides** : un utilisateur envoie un POST sans email → il reçoit un 400.

4. **Connexion réussie** : un utilisateur existant envoie ses identifiants → il reçoit un 200 avec un token.

5. **Connexion avec mauvais mot de passe** : un utilisateur envoie un mauvais mot de passe → il reçoit un 401.

<note title="Conseils de rédaction">

 - Utilisez des steps expressifs : `Given un utilisateur existe avec l'email "demo@yapuka.dev"` plutôt que `Given il y a un utilisateur`
 - Mettez les valeurs importantes entre guillemets : `"demo@yapuka.dev"`, `"password"`
 - Pour envoyer un corps JSON, utilisez les triple-guillemets Gherkin (DocString) :
 
```gherkin
   When j'envoie une requête POST sur "/api/auth/register" avec le corps :
     """
     { "email": "test@test.com" }
     """
   ```

</note>

#### Correction 2.2 — auth.feature {collapsible="true"}

```gherkin
# features/auth.feature

# =============================================================================
# Feature : Authentification de l'API Yapuka
# =============================================================================
# Teste l'inscription et la connexion via JWT.
# Chaque scénario part d'une base de données vide (nettoyée par @BeforeScenario).
# =============================================================================

Feature: Authentification API
  En tant qu'utilisateur de Yapuka
  Je veux pouvoir m'inscrire et me connecter
  Afin d'accéder à mes tâches de manière sécurisée

  # ---------------------------------------------------------------------------
  # Inscription
  # ---------------------------------------------------------------------------

  Scenario: Inscription réussie avec des données valides
    When j'envoie une requête POST sur "/api/auth/register" avec le corps :
      """
      {
        "email": "nouveau@yapuka.dev",
        "username": "Nouveau User",
        "password": "password123"
      }
      """
    Then le code de réponse est 201
    And la réponse JSON contient la clé "token"
    And la réponse JSON contient "email" avec la valeur "nouveau@yapuka.dev"

  Scenario: Inscription refusée si l'email existe déjà
    Given un utilisateur existe avec l'email "existant@yapuka.dev" et le mot de passe "password"
    When j'envoie une requête POST sur "/api/auth/register" avec le corps :
      """
      {
        "email": "existant@yapuka.dev",
        "username": "Doublon",
        "password": "password123"
      }
      """
    Then le code de réponse est 409

  Scenario: Inscription refusée si des champs sont manquants
    When j'envoie une requête POST sur "/api/auth/register" avec le corps :
      """
      {
        "password": "password123"
      }
      """
    Then le code de réponse est 400

  # ---------------------------------------------------------------------------
  # Connexion
  # ---------------------------------------------------------------------------

  Scenario: Connexion réussie avec des identifiants valides
    Given un utilisateur existe avec l'email "demo@yapuka.dev" et le mot de passe "password"
    When j'envoie une requête POST sur "/api/auth/login" avec le corps :
      """
      {
        "email": "demo@yapuka.dev",
        "password": "password"
      }
      """
    Then le code de réponse est 200
    And la réponse JSON contient la clé "token"

  Scenario: Connexion refusée avec un mauvais mot de passe
    Given un utilisateur existe avec l'email "demo@yapuka.dev" et le mot de passe "password"
    When j'envoie une requête POST sur "/api/auth/login" avec le corps :
      """
      {
        "email": "demo@yapuka.dev",
        "password": "mauvais_mot_de_passe"
      }
      """
    Then le code de réponse est 401
```

### 2.3 Feature de gestion des tâches

**À faire** : créez un fichier `features/tasks.feature` qui décrit le CRUD des tâches.

**Background** : tous les scénarios nécessitent un utilisateur connecté. Utilisez un `Background` qui :
1. Crée un utilisateur
2. Le connecte (obtient un JWT)

**Scénarios à écrire** :

1. **Créer une tâche** : envoi d'un POST avec titre et priorité → 201, la tâche est retournée avec le statut "todo" par défaut.

2. **Créer une tâche sans titre** : envoi d'un POST sans titre → 422 avec erreur de validation.

3. **Lister ses tâches** : après avoir créé 2 tâches, un GET retourne un tableau de 2 éléments.

4. **Modifier une tâche** : créer une tâche, puis envoyer un PUT pour changer son statut en "done" → 200.

5. **Supprimer une tâche** : créer une tâche, puis envoyer un DELETE → 200, la tâche n'apparaît plus dans la liste.

6. **Accès interdit sans JWT** : un GET sur `/api/tasks` sans token → 401.

7. **(Bonus) Scenario Outline** : tester la création avec différentes priorités (low, medium, high) dans un seul scénario paramétré avec un tableau `Examples`.

> Pour le Background :
> ```gherkin
> Background:
>   Given un utilisateur existe avec l'email "demo@yapuka.dev" et le mot de passe "password"
>   And je suis connecté en tant que "demo@yapuka.dev" avec le mot de passe "password"
> ```

#### Correction 2.3 — tasks.feature {collapsible="true"}

```gherkin
# features/tasks.feature

# =============================================================================
# Feature : Gestion des tâches (CRUD)
# =============================================================================
# Teste la création, la lecture, la modification et la suppression des tâches
# via l'API REST. Tous les scénarios nécessitent un utilisateur authentifié.
# =============================================================================

Feature: Gestion des tâches
  En tant qu'utilisateur connecté
  Je veux pouvoir gérer mes tâches
  Afin d'organiser mon travail efficacement

  Background:
    Given un utilisateur existe avec l'email "demo@yapuka.dev" et le mot de passe "password"
    And je suis connecté en tant que "demo@yapuka.dev" avec le mot de passe "password"

  # ---------------------------------------------------------------------------
  # Création
  # ---------------------------------------------------------------------------

  Scenario: Créer une tâche avec des données valides
    When j'envoie une requête authentifiée POST sur "/api/tasks" avec le corps :
      """
      {
        "title": "Configurer le pipeline CI/CD",
        "description": "Mettre en place GitHub Actions",
        "priority": "high"
      }
      """
    Then le code de réponse est 201
    And la réponse JSON contient "title" avec la valeur "Configurer le pipeline CI/CD"
    And la réponse JSON contient "status" avec la valeur "todo"
    And la réponse JSON contient "priority" avec la valeur "high"

  Scenario: Créer une tâche sans titre échoue
    When j'envoie une requête authentifiée POST sur "/api/tasks" avec le corps :
      """
      {
        "description": "Pas de titre"
      }
      """
    Then le code de réponse est 422

  # ---------------------------------------------------------------------------
  # Lecture
  # ---------------------------------------------------------------------------

  Scenario: Lister les tâches de l'utilisateur connecté
    Given une tâche existe avec le titre "Tâche 1"
    And une tâche existe avec le titre "Tâche 2"
    When j'envoie une requête authentifiée GET sur "/api/tasks"
    Then le code de réponse est 200
    And la réponse JSON est un tableau de 2 éléments

  # ---------------------------------------------------------------------------
  # Modification
  # ---------------------------------------------------------------------------

  Scenario: Modifier le statut d'une tâche
    Given une tâche existe avec le titre "Tâche à terminer"
    When je modifie la dernière tâche créée avec le corps :
      """
      {
        "status": "done"
      }
      """
    Then le code de réponse est 200
    And la réponse JSON contient "status" avec la valeur "done"

  # ---------------------------------------------------------------------------
  # Suppression
  # ---------------------------------------------------------------------------

  Scenario: Supprimer une tâche
    Given une tâche existe avec le titre "Tâche à supprimer"
    When je supprime la dernière tâche créée
    Then le code de réponse est 200
    When j'envoie une requête authentifiée GET sur "/api/tasks"
    Then la réponse JSON est un tableau de 0 éléments

  # ---------------------------------------------------------------------------
  # Sécurité
  # ---------------------------------------------------------------------------

  Scenario: Accès refusé sans authentification
    When j'envoie une requête POST sur "/api/tasks" avec le corps :
      """
      {
        "title": "Test sans auth"
      }
      """
    Then le code de réponse est 401

  # ---------------------------------------------------------------------------
  # Scenario Outline : tester plusieurs priorités
  # ---------------------------------------------------------------------------

  Scenario Outline: Créer une tâche avec différentes priorités
    When j'envoie une requête authentifiée POST sur "/api/tasks" avec le corps :
      """
      {
        "title": "Tâche priorité <priority>",
        "priority": "<priority>"
      }
      """
    Then le code de réponse est 201
    And la réponse JSON contient "priority" avec la valeur "<priority>"

    Examples:
      | priority |
      | low      |
      | medium   |
      | high     |
```


### 2.4 Feature des statistiques

**À faire** : créez un fichier `features/stats.feature` court qui teste l'endpoint `/api/tasks/stats`.

Utilisez le même `Background` que pour les tâches (utilisateur connecté).

Écrivez 2 scénarios :

1. **Stats vides** : aucune tâche → tous les compteurs à 0.
2. **Stats après création** : créer 2 tâches "todo" et 1 tâche "done" → total=3, todo=2, done=1.

> Vous aurez besoin d'un step qui crée une tâche avec un statut spécifique :
> `Given une tâche existe avec le titre "..." et le statut "done"`

#### Correction 2.4 — stats.feature {collapsible="true"}

```gherkin
# features/stats.feature

# =============================================================================
# Feature : Statistiques des tâches
# =============================================================================
# Teste l'endpoint GET /api/tasks/stats qui retourne les compteurs
# par statut (total, todo, in_progress, done, overdue).
# =============================================================================

Feature: Statistiques des tâches
  En tant qu'utilisateur connecté
  Je veux voir les statistiques de mes tâches
  Afin de suivre ma progression

  Background:
    Given un utilisateur existe avec l'email "demo@yapuka.dev" et le mot de passe "password"
    And je suis connecté en tant que "demo@yapuka.dev" avec le mot de passe "password"

  Scenario: Statistiques avec aucune tâche
    When j'envoie une requête authentifiée GET sur "/api/tasks/stats"
    Then le code de réponse est 200
    And la réponse JSON contient "total" avec la valeur entière 0
    And la réponse JSON contient "todo" avec la valeur entière 0
    And la réponse JSON contient "done" avec la valeur entière 0

  Scenario: Statistiques après création de tâches
    Given une tâche existe avec le titre "Tâche A" et le statut "todo"
    And une tâche existe avec le titre "Tâche B" et le statut "todo"
    And une tâche existe avec le titre "Tâche C" et le statut "done"
    When j'envoie une requête authentifiée GET sur "/api/tasks/stats"
    Then le code de réponse est 200
    And la réponse JSON contient "total" avec la valeur entière 3
    And la réponse JSON contient "todo" avec la valeur entière 2
    And la réponse JSON contient "done" avec la valeur entière 1
```


## Partie 3 — Implémenter les steps en PHP (35 min)

### 3.1 Générer les steps manquants

Behat peut analyser vos fichiers `.feature` et vous montrer les steps qu'il ne sait pas encore exécuter.

**À faire** : lancez la commande suivante pour voir tous les steps à implémenter :

```bash
vendor/bin/behat --dry-run --append-snippets
```

Behat vous proposera des squelettes de méthodes PHP pour chaque step manquant. Ne copiez pas tout aveuglément : lisez la partie suivante pour comprendre ce que chaque step doit faire.

> Vous pouvez aussi lister les steps déjà disponibles (via Behatch) :
> ```bash
> vendor/bin/behat -dl
> ```

#### Correction 3.1 — Générer les steps {collapsible="true"}

La sortie de `vendor/bin/behat --dry-run --append-snippets` ressemble à :

```
--- App\Tests\Behat\FeatureContext has missing steps.
    Add these to your context class:

    /**
     * @Given un utilisateur existe avec l'email :email et le mot de passe :password
     */
    public function unUtilisateurExisteAvecLemailEtLeMotDePasse($email, $password)
    {
        throw new PendingException();
    }

    /**
     * @When j'envoie une requête POST sur :url avec le corps :
     */
    public function jEnvoieUneRequetePostSurAvecLeCorps($url, PyStringNode $body)
    {
        throw new PendingException();
    }
    ...
```

Ces squelettes vous donnent la signature exacte des méthodes. Vous devez remplacer `throw new PendingException()` par l'implémentation réelle.

Certains steps comme "le code de réponse est 200" sont peut-être déjà couverts par `Behatch\Context\RestContext`. Lancez `vendor/bin/behat -dl` pour vérifier. Vous n'avez à implémenter que ceux qui manquent.


### 3.2 Implémenter les steps d'authentification et d'action

Votre `FeatureContext` doit contenir des méthodes pour chaque step Gherkin. Vous allez utiliser le `KernelBrowser` de Symfony pour effectuer les requêtes internes (sans passer par le réseau).

**À faire** : implémentez les steps suivants dans `FeatureContext`.

**Steps `@Given` (pré-conditions)** :

| Step Gherkin                                                             | Comportement attendu                                                                                                            |
|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `un utilisateur existe avec l'email :email et le mot de passe :password` | Créer un `User`, hasher le password, persister en base. Le username peut être dérivé de l'email (partie avant `@`).             |
| `je suis connecté en tant que :email avec le mot de passe :password`     | Envoyer un POST `/api/auth/login` via le `KernelBrowser`, extraire le `token` de la réponse, le stocker dans `$this->jwtToken`. |
| `une tâche existe avec le titre :title`                                  | Envoyer un POST authentifié `/api/tasks`, stocker l'`id` retourné dans `$this->lastTaskId`.                                     |
| `une tâche existe avec le titre :title et le statut :status`             | Créer la tâche, puis si le statut n'est pas "todo", envoyer un PUT pour le changer.                                             |

**Steps `@When` (actions)** :

| Step Gherkin | Comportement attendu |
|---|---|
| `j'envoie une requête POST sur :url avec le corps :` | Requête sans JWT. Corps passé en `PyStringNode`. Stocker la réponse décodée. |
| `j'envoie une requête authentifiée :method sur :url` | Requête avec header `Authorization: Bearer <token>`. Pas de corps. |
| `j'envoie une requête authentifiée :method sur :url avec le corps :` | Idem avec un corps JSON. |
| `je modifie la dernière tâche créée avec le corps :` | PUT sur `/api/tasks/{lastTaskId}`. |
| `je supprime la dernière tâche créée` | DELETE sur `/api/tasks/{lastTaskId}`. |

**Steps `@Then` (assertions)** :

| Step Gherkin | Comportement attendu |
|---|---|
| `le code de réponse est :code` | Comparer `$response->getStatusCode()` avec `:code`. |
| `la réponse JSON contient la clé :key` | Vérifier que la clé existe dans les données décodées. |
| `la réponse JSON contient :key avec la valeur :value` | Vérifier l'égalité (string). |
| `la réponse JSON contient :key avec la valeur entière :value` | Vérifier l'égalité (int) — pour les compteurs des stats. |
| `la réponse JSON est un tableau de :count éléments` | Vérifier `count($responseData) === $count`. |

> **Indice pour le KernelBrowser** :
> ```php
> $this->client = new KernelBrowser($this->kernel);
> $this->client->request('POST', $url, [], [], [
>     'CONTENT_TYPE' => 'application/json',
> ], $jsonBody);
> $response = $this->client->getResponse();
> ```

> **Piège** : pour les requêtes authentifiées, le header HTTP en Symfony s'écrit `HTTP_AUTHORIZATION` (pas `Authorization`).

> **Conseil** : créez une méthode privée `sendAuthenticatedRequest(method, url, data)` pour ne pas dupliquer le code d'injection du JWT dans chaque step.

#### Correction 3.2 — FeatureContext complet {collapsible="true"}

Voici le `FeatureContext` complet avec tous les steps implémentés :

```php
<?php

// =============================================================================
// FeatureContext — Contexte principal des tests Behat
// =============================================================================
// Contient tous les steps personnalisés pour tester l'API Yapuka.
// Utilise le KernelBrowser de Symfony pour les requêtes internes.
// =============================================================================

namespace App\Tests\Behat;

use App\Entity\Task;
use App\Entity\User;
use Behat\Behat\Context\Context;
use Behat\Behat\Hook\Scope\BeforeScenarioScope;
use Behat\Gherkin\Node\PyStringNode;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\KernelBrowser;
use Symfony\Component\HttpKernel\KernelInterface;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

class FeatureContext implements Context
{
    private ?KernelBrowser $client = null;
    private ?string $jwtToken = null;
    private ?array $responseData = null;
    private ?int $lastTaskId = null;

    public function __construct(
        private EntityManagerInterface $entityManager,
        private UserPasswordHasherInterface $passwordHasher,
        private KernelInterface $kernel,
    ) {
    }

    // =========================================================================
    // Hook : nettoyage avant chaque scénario
    // =========================================================================

    /**
     * @BeforeScenario
     */
    public function cleanDatabase(BeforeScenarioScope $scope): void
    {
        $connection = $this->entityManager->getConnection();
        $connection->executeStatement('DELETE FROM tasks');
        $connection->executeStatement('DELETE FROM users');
        $this->entityManager->clear();

        $this->jwtToken = null;
        $this->responseData = null;
        $this->lastTaskId = null;
        $this->client = new KernelBrowser($this->kernel);
    }

    // =========================================================================
    // Steps @Given — Pré-conditions
    // =========================================================================

    /**
     * Crée un utilisateur en base de données avec un mot de passe hashé.
     *
     * @Given un utilisateur existe avec l'email :email et le mot de passe :password
     */
    public function unUtilisateurExisteAvecLemailEtLeMotDePasse(
        string $email,
        string $password,
    ): void {
        $user = new User();
        $user->setEmail($email);
        $user->setUsername(explode('@', $email)[0]);
        $user->setPassword(
            $this->passwordHasher->hashPassword($user, $password)
        );

        $this->entityManager->persist($user);
        $this->entityManager->flush();
    }

    /**
     * Se connecte à l'API et stocke le token JWT pour les requêtes suivantes.
     *
     * @Given je suis connecté en tant que :email avec le mot de passe :password
     */
    public function jeSuisConnecteEnTantQue(string $email, string $password): void
    {
        $this->client->request('POST', '/api/auth/login', [], [], [
            'CONTENT_TYPE' => 'application/json',
        ], json_encode([
            'email' => $email,
            'password' => $password,
        ]));

        $data = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );

        if (!isset($data['token'])) {
            throw new \RuntimeException(
                'Login échoué : pas de token. Réponse : '
                . $this->client->getResponse()->getContent()
            );
        }

        $this->jwtToken = $data['token'];
    }

    /**
     * Crée une tâche via l'API (nécessite d'être connecté).
     *
     * @Given une tâche existe avec le titre :title
     */
    public function uneTacheExisteAvecLeTitre(string $title): void
    {
        $this->sendAuthenticatedRequest('POST', '/api/tasks', [
            'title' => $title,
        ]);

        $data = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );
        $this->lastTaskId = $data['id'] ?? null;
    }

    /**
     * Crée une tâche puis modifie son statut si différent de "todo".
     *
     * @Given une tâche existe avec le titre :title et le statut :status
     */
    public function uneTacheExisteAvecLeTitreEtLeStatut(
        string $title,
        string $status,
    ): void {
        // Créer la tâche (statut par défaut : "todo")
        $this->sendAuthenticatedRequest('POST', '/api/tasks', [
            'title' => $title,
        ]);

        $data = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );
        $taskId = $data['id'];
        $this->lastTaskId = $taskId;

        // Si le statut souhaité n'est pas "todo", on le modifie
        if ($status !== 'todo') {
            $this->sendAuthenticatedRequest('PUT', "/api/tasks/{$taskId}", [
                'status' => $status,
            ]);
        }
    }

    // =========================================================================
    // Steps @When — Actions
    // =========================================================================

    /**
     * Envoie une requête POST sans authentification (inscription, login).
     *
     * @When j'envoie une requête POST sur :url avec le corps :
     */
    public function jEnvoieUneRequetePostSurAvecLeCorps(
        string $url,
        PyStringNode $body,
    ): void {
        $this->client->request('POST', $url, [], [], [
            'CONTENT_TYPE' => 'application/json',
        ], $body->getRaw());

        $this->responseData = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );
    }

    /**
     * Envoie une requête authentifiée sans corps (GET, DELETE).
     *
     * @When j'envoie une requête authentifiée :method sur :url
     */
    public function jEnvoieUneRequeteAuthentifiee(
        string $method,
        string $url,
    ): void {
        $this->sendAuthenticatedRequest($method, $url);

        $this->responseData = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );
    }

    /**
     * Envoie une requête authentifiée avec un corps JSON (POST, PUT).
     *
     * @When j'envoie une requête authentifiée :method sur :url avec le corps :
     */
    public function jEnvoieUneRequeteAuthentifieeAvecLeCorps(
        string $method,
        string $url,
        PyStringNode $body,
    ): void {
        $this->sendAuthenticatedRequest(
            $method,
            $url,
            json_decode($body->getRaw(), true),
        );

        $this->responseData = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );
    }

    /**
     * Modifie la dernière tâche créée via PUT.
     *
     * @When je modifie la dernière tâche créée avec le corps :
     */
    public function jeModifieLaDerniereTacheAvecLeCorps(PyStringNode $body): void
    {
        if (!$this->lastTaskId) {
            throw new \RuntimeException('Aucune tâche créée précédemment.');
        }

        $this->sendAuthenticatedRequest(
            'PUT',
            "/api/tasks/{$this->lastTaskId}",
            json_decode($body->getRaw(), true),
        );

        $this->responseData = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );
    }

    /**
     * Supprime la dernière tâche créée via DELETE.
     *
     * @When je supprime la dernière tâche créée
     */
    public function jeSupprimeLaDerniereTacheCree(): void
    {
        if (!$this->lastTaskId) {
            throw new \RuntimeException('Aucune tâche créée précédemment.');
        }

        $this->sendAuthenticatedRequest(
            'DELETE',
            "/api/tasks/{$this->lastTaskId}",
        );

        $this->responseData = json_decode(
            $this->client->getResponse()->getContent(),
            true,
        );
    }

    // =========================================================================
    // Steps @Then — Assertions
    // =========================================================================

    /**
     * Vérifie le code HTTP de la réponse.
     *
     * @Then le code de réponse est :code
     */
    public function leCodeDeReponseEst(int $code): void
    {
        $actual = $this->client->getResponse()->getStatusCode();

        if ($actual !== $code) {
            throw new \RuntimeException(
                "Code attendu : {$code}, reçu : {$actual}. "
                . "Body : " . $this->client->getResponse()->getContent()
            );
        }
    }

    /**
     * Vérifie qu'une clé existe dans la réponse JSON.
     *
     * @Then la réponse JSON contient la clé :key
     */
    public function laReponseJsonContientLaCle(string $key): void
    {
        if (!is_array($this->responseData) || !array_key_exists($key, $this->responseData)) {
            throw new \RuntimeException(
                "Clé '{$key}' absente. Clés présentes : "
                . implode(', ', array_keys($this->responseData ?? []))
            );
        }
    }

    /**
     * Vérifie qu'une clé contient une valeur string.
     * Supporte l'accès imbriqué avec un point (ex: "user.email").
     *
     * @Then la réponse JSON contient :key avec la valeur :value
     */
    public function laReponseJsonContientAvecLaValeur(
        string $key,
        string $value,
    ): void {
        $actual = $this->getNestedValue($this->responseData, $key);

        if ($actual === null && !array_key_exists($key, $this->responseData ?? [])) {
            throw new \RuntimeException("Clé '{$key}' absente de la réponse.");
        }

        if ((string) $actual !== $value) {
            throw new \RuntimeException(
                "Clé '{$key}' : attendu '{$value}', reçu '{$actual}'."
            );
        }
    }

    /**
     * Vérifie qu'une clé contient une valeur entière (pour les compteurs).
     *
     * @Then la réponse JSON contient :key avec la valeur entière :value
     */
    public function laReponseJsonContientAvecLaValeurEntiere(
        string $key,
        int $value,
    ): void {
        if (!array_key_exists($key, $this->responseData ?? [])) {
            throw new \RuntimeException("Clé '{$key}' absente de la réponse.");
        }

        $actual = $this->responseData[$key];

        if ((int) $actual !== $value) {
            throw new \RuntimeException(
                "Clé '{$key}' : attendu {$value}, reçu {$actual}."
            );
        }
    }

    /**
     * Vérifie que la réponse est un tableau de N éléments.
     *
     * @Then la réponse JSON est un tableau de :count éléments
     */
    public function laReponseJsonEstUnTableauDeElements(int $count): void
    {
        if (!is_array($this->responseData)) {
            throw new \RuntimeException(
                'La réponse n\'est pas un tableau. Type reçu : '
                . gettype($this->responseData)
            );
        }

        $actual = count($this->responseData);

        if ($actual !== $count) {
            throw new \RuntimeException(
                "Tableau de {$count} éléments attendu, {$actual} trouvés."
            );
        }
    }

    // =========================================================================
    // Méthodes utilitaires privées
    // =========================================================================

    /**
     * Envoie une requête HTTP avec le header Authorization JWT.
     */
    private function sendAuthenticatedRequest(
        string $method,
        string $url,
        ?array $data = null,
    ): void {
        $headers = [
            'CONTENT_TYPE' => 'application/json',
            'HTTP_AUTHORIZATION' => 'Bearer ' . $this->jwtToken,
        ];

        $this->client->request(
            $method,
            $url,
            [],
            [],
            $headers,
            $data ? json_encode($data) : null,
        );
    }

    /**
     * Accède à une valeur imbriquée via la notation pointée (ex: "user.email").
     */
    private function getNestedValue(array $data, string $key): mixed
    {
        if (array_key_exists($key, $data)) {
            return $data[$key];
        }

        $parts = explode('.', $key);
        $current = $data;

        foreach ($parts as $part) {
            if (!is_array($current) || !array_key_exists($part, $current)) {
                return null;
            }
            $current = $current[$part];
        }

        return $current;
    }
}
```


### 3.3 Lancer un premier test

**À faire** : exécutez Behat sur un seul fichier pour valider vos steps au fur et à mesure :

```bash
vendor/bin/behat features/auth.feature
```

Puis passez aux tâches et aux stats :

```bash
vendor/bin/behat features/tasks.feature
vendor/bin/behat features/stats.feature
```

> **Astuce de debug** : pour n'exécuter qu'un seul scénario, utilisez le numéro de ligne :
> ```bash
> vendor/bin/behat features/auth.feature:15
> ```

> **Commandes à exécuter avant le premier lancement** (si ce n'est pas déjà fait) :
> ```bash
> php bin/console doctrine:database:create --env=test --if-not-exists
> php bin/console doctrine:schema:create --env=test
> ```

#### Correction 3.3 — Steps tâches et stats {collapsible="true"}

Si vous avez utilisé la correction 3.2, tous les steps nécessaires sont déjà implémentés. Vérifiez que vous avez :

- `uneTacheExisteAvecLeTitre()` — avec stockage de `$this->lastTaskId`
- `uneTacheExisteAvecLeTitreEtLeStatut()` — crée puis modifie
- `jeModifieLaDerniereTacheAvecLeCorps()` — PUT
- `jeSupprimeLaDerniereTacheCree()` — DELETE
- `laReponseJsonEstUnTableauDeElements()` — vérification count
- `laReponseJsonContientAvecLaValeurEntiere()` — vérification int

**Problème courant** : différence entre steps **avec corps** et **sans corps** :
- `j'envoie une requête authentifiée GET sur "/api/tasks"` → sans `PyStringNode`
- `j'envoie une requête authentifiée POST sur "/api/tasks" avec le corps :` → avec `PyStringNode`

Ce sont deux méthodes PHP différentes (annotations Behat différentes). Si Behat dit "ambiguous step", c'est que les deux annotations matchent le même texte. Vérifiez que le texte se termine bien par `avec le corps :` pour la version avec corps.


## Partie 4 — Exécution et validation (15 min)

### 4.1 Exécuter tous les tests

**À faire** :

1. Lancez Behat sur toutes les features :

```bash
vendor/bin/behat
```

2. Observez la sortie : chaque scénario doit afficher des steps en **vert**.
3. Si un step est en rouge, lisez le message d'erreur et corrigez.

#### Correction 4.1 — Sortie attendue {collapsible="true"}

```
Feature: Authentification API

  Scenario: Inscription réussie avec des données valides          ✔
  Scenario: Inscription refusée si l'email existe déjà            ✔
  Scenario: Inscription refusée si des champs sont manquants      ✔
  Scenario: Connexion réussie avec des identifiants valides       ✔
  Scenario: Connexion refusée avec un mauvais mot de passe        ✔

Feature: Gestion des tâches

  Scenario: Créer une tâche avec des données valides              ✔
  Scenario: Créer une tâche sans titre échoue                     ✔
  Scenario: Lister les tâches de l'utilisateur connecté           ✔
  Scenario: Modifier le statut d'une tâche                        ✔
  Scenario: Supprimer une tâche                                   ✔
  Scenario: Accès refusé sans authentification                    ✔
  Scenario Outline: Créer une tâche avec différentes priorités
    Examples:
      | priority |
      | low      | ✔
      | medium   | ✔
      | high     | ✔

Feature: Statistiques des tâches

  Scenario: Statistiques avec aucune tâche                        ✔
  Scenario: Statistiques après création de tâches                 ✔

15 scenarios (15 passed)
~60 steps (all passed)
```

Si le `Scenario Outline` ne matche pas les steps, c'est que les placeholders `<priority>` sont substitués dans le texte **avant** le matching. Vérifiez que l'annotation `@When` correspond au texte final (ex: `"Tâche priorité low"`).


### 4.2 Options utiles de Behat

**À faire** : testez ces options de la ligne de commande :

```bash
# Voir tous les steps disponibles (ceux de votre contexte + Behatch)
vendor/bin/behat -dl

# Exécuter avec une sortie détaillée
vendor/bin/behat --format=pretty

# Arrêter au premier échec
vendor/bin/behat --stop-on-failure
```

#### Correction 4.2 — Options Behat {collapsible="true"}

La commande `vendor/bin/behat -dl` affiche la liste complète des steps disponibles. Vous devriez voir vos steps personnalisés ET les steps de Behatch.

Si des steps Behat font doublon avec les vôtres (par exemple, deux définitions pour "le code de réponse est :code"), il y aura un conflit `Ambiguous`. Solutions :
- Retirer les contextes Behat de `behat.yml` et n'utiliser que votre `FeatureContext`
- Ou retirer vos steps redondants et utiliser ceux de Behatch

Pour ce lab, la solution la plus simple est de n'utiliser que votre `FeatureContext` si des conflits apparaissent.


### 4.3 Ajouter des scripts Composer

**À faire** : ajoutez une section `scripts` dans le `composer.json` du backend pour simplifier le lancement :

| Script | Commande | Usage |
|--------|----------|-------|
| `test:behat` | Lance Behat avec les couleurs | Tests BDD seuls |
| `test:unit` | Lance PHPUnit sur `tests/Unit` | Tests unitaires seuls |
| `test:integration` | Lance PHPUnit sur `tests/Integration` | Tests d'intégration seuls |
| `test:all` | Exécute les 3 suites séquentiellement | Tout d'un coup |

Testez avec `composer test:behat` puis `composer test:all`.

#### Correction 4.3 — Scripts Composer {collapsible="true"}

Ajoutez dans `composer.json` :

```json
{
    "scripts": {
        "test:behat": "vendor/bin/behat --colors",
        "test:unit": "php bin/phpunit tests/Unit",
        "test:integration": "php bin/phpunit tests/Integration",
        "test:all": [
            "@test:unit",
            "@test:integration",
            "@test:behat"
        ]
    }
}
```

Lancement :

```bash
composer test:behat      # Behat seul
composer test:all        # Les 3 suites séquentiellement
```

La sortie de `composer test:all` exécute les suites dans l'ordre. Si une suite échoue, Composer s'arrête avec un code d'erreur non-zéro (utile pour le CI/CD).


### 4.4 Rapport de tests

**À faire** : créez un fichier `TESTS-BEHAT.md` à la racine de `api/` contenant :

- Le nombre de features
- Le nombre total de scénarios
- Le nombre de steps
- Le temps d'exécution
- Les difficultés rencontrées

#### Correction 4.4 — Rapport de tests {collapsible="true"}

Exemple de `TESTS-BEHAT.md` :

```markdown
# Rapport des tests BDD (Behat) — Yapuka API

## Résumé

| Feature | Scénarios | Steps |
|---------|-----------|-------|
| `auth.feature` | 5 | ~20 |
| `tasks.feature` | 7 (dont 3 via Outline) | ~35 |
| `stats.feature` | 2 | ~12 |
| **Total** | **14** | **~67** |

## Résultat

✅ 14/14 scénarios passent (`vendor/bin/behat`)

## Durée d'exécution

~5 secondes (les tests utilisent le KernelBrowser interne,
sans passer par le réseau)

## Difficultés rencontrées

1. **Conflits de steps avec Behatch** : certains steps étaient définis
   à la fois dans FeatureContext et dans RestContext. Résolu en retirant
   les contextes Behatch redondants.

2. **Base de données de test** : il faut créer le schéma avant le premier
   lancement (`doctrine:schema:create --env=test`).

3. **Accents dans les annotations** : les accents français dans les
   annotations `@Given`/`@When`/`@Then` doivent correspondre exactement
   au texte du fichier `.feature`.

4. **Stats avec cache Redis** : les stats étant cachées 60 secondes,
   le test peut retourner des valeurs périmées. Résolu en utilisant
   l'environnement `test` sans cache ou en invalidant le cache.
```

## Critères de validation

| #  | Critère                                                                  | Points  |
|----|--------------------------------------------------------------------------|---------|
| 1  | Behat est installé et `vendor/bin/behat --version` fonctionne            | /1      |
| 2  | `behat.yml` est correctement configuré avec l'extension Symfony          | /2      |
| 3  | Le `FeatureContext` nettoie la base avant chaque scénario                | /1      |
| 4  | `auth.feature` contient les 5 scénarios d'authentification               | /3      |
| 5  | `tasks.feature` contient les 7 scénarios CRUD (dont le Scenario Outline) | /4      |
| 6  | `stats.feature` contient les 2 scénarios de statistiques                 | /2      |
| 7  | Tous les steps sont implémentés dans `FeatureContext`                    | /3      |
| 8  | `vendor/bin/behat` exécute tous les scénarios avec 0 échec               | /2      |
| 9  | Les scripts Composer `test:behat` et `test:all` fonctionnent             | /1      |
| 10 | Le fichier `TESTS-BEHAT.md` est complet                                  | /1      |
|    | **Total**                                                                | **/20** |


<!--
## Pour aller plus loin

Si vous avez terminé en avance, voici des défis supplémentaires :

- **Tags** : ajoutez des tags `@smoke`, `@security`, `@crud` à vos scénarios et lancez uniquement une catégorie avec `vendor/bin/behat --tags=@security`.
- **Isolation** : écrivez un scénario où un utilisateur A crée une tâche, puis un utilisateur B tente de la modifier → 403 Forbidden.
- **Tâche en retard** : écrivez un scénario qui crée une tâche avec une date d'échéance passée, puis vérifie que `overdue` vaut 1 dans les stats.
- **Rapport HTML** : configurez un formateur HTML dans `behat.yml` et générez un rapport visuel.
- **Hook @AfterScenario** : ajoutez un hook qui logge le résultat de chaque scénario dans un fichier.
-->