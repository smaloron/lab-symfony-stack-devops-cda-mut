# Lab 1 — Tests End-to-End avec Cypress

> **Prérequis** : le projet Yapuka tourne localement via `docker compose up -d`, les fixtures sont chargées, et vous
> pouvez vous connecter avec `demo@yapuka.dev` / `password`.

---

## Contexte

Votre équipe vient de terminer le développement de Yapuka. Avant de mettre en place le pipeline CI/CD (prochain lab),
vous devez vous assurer que les parcours utilisateurs critiques fonctionnent de bout en bout. Un test unitaire vérifie
qu'une fonction marche ; un test E2E vérifie que **l'utilisateur peut accomplir sa tâche** dans un vrai navigateur.

Vous allez écrire une suite de tests Cypress qui couvre le parcours complet : **s'inscrire → créer des tâches → les
gérer → les supprimer**.

---

## Partie 1 — Installation et configuration (20 min)

### 1.1 Installer Cypress

Cypress s'installe comme une dépendance de développement dans le projet frontend.

**À faire** :

1. Ouvrez un terminal dans le dossier `front/`.
2. Installez Cypress et `start-server-and-test` :

```bash
npm install --save-dev cypress start-server-and-test
```

3. Ajoutez les scripts suivants dans votre `package.json` :

| Script     | Commande                                                                | Usage                   |
|------------|-------------------------------------------------------------------------|-------------------------|
| `cy:open`  | Lance Cypress en mode interactif (UI)                                   | Développement des tests |
| `cy:run`   | Lance Cypress en mode headless                                          | CI/CD                   |
| `test:e2e` | Utilise `start-server-and-test` pour démarrer Vite puis lancer `cy:run` | Exécution autonome      |

> **Indice** : la syntaxe de `start-server-and-test` est
`start-server-and-test <script-serveur> <url-à-attendre> <script-tests>`. Votre serveur Vite tourne sur le port 5173.

#### Correction 1.1 {collapsible="true"}

Dans `package.json`, ajoutez dans le bloc `"scripts"` :

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "cy:open": "cypress open",
    "cy:run": "cypress run",
    "test:e2e": "start-server-and-test dev http://localhost:5173 cy:run"
  }
}
```



### 1.2 Première ouverture

Lancez `npm run cy:open` une première fois. Cypress va créer automatiquement son arborescence de dossiers.

**À faire** :

1. Lancez Cypress, choisissez "E2E Testing", puis sélectionnez le navigateur Chrome.
2. Observez les dossiers créés. Fermez Cypress.
3. Nettoyez : supprimez les fichiers d'exemple générés par Cypress dans `cypress/e2e/`.


#### Correction 1.2 {collapsible="true"}

Après la première ouverture, Cypress crée cette structure :

```
front/
├── cypress/
│   ├── e2e/              ← supprimer les fichiers d'exemple ici
│   ├── fixtures/
│   │   └── example.json  ← supprimer aussi
│   └── support/
│       ├── commands.js
│       └── e2e.js
└── cypress.config.js
```

Nettoyage :

```bash
rm -f cypress/e2e/*.cy.js
rm -f cypress/fixtures/example.json
```



### 1.3 Configurer Cypress

Le fichier `cypress.config.js` a été créé à la racine de `front/`. Il faut l'adapter au projet.

**À faire** : modifiez `cypress.config.js` pour configurer les éléments suivants.

| Paramètre                          | Valeur                         | Pourquoi                                                              |
|------------------------------------|--------------------------------|-----------------------------------------------------------------------|
| `baseUrl`                          | L'URL de votre frontend en dev | Évite de répéter l'URL dans chaque `cy.visit()`                       |
| `viewportWidth` / `viewportHeight` | 1280 × 720                     | Taille de fenêtre cohérente                                           |
| `video`                            | `false`                        | Désactiver les vidéos pendant le développement (accélère l'exécution) |
| `defaultCommandTimeout`            | 8000                           | Votre API peut être lente en environnement Docker                     |

> **Référence** : [Configuration Cypress](https://docs.cypress.io/guides/references/configuration)


#### Correction 1.3 {collapsible="true"}

```js
// cypress.config.js
const {defineConfig} = require('cypress');

module.exports = defineConfig({
    e2e: {
        // URL de base : tous les cy.visit('/login') deviendront http://localhost:5173/login
        baseUrl: 'http://localhost:5173',

        // Taille du viewport pour des tests cohérents
        viewportWidth: 1280,
        viewportHeight: 720,

        // Désactiver les vidéos en dev (accélère les tests)
        video: false,

        // Timeout étendu pour les commandes (API lente en Docker)
        defaultCommandTimeout: 8000,

        // Dossier pour les fichiers de spec
        specPattern: 'cypress/e2e/**/*.cy.js',
    },
});
```



### 1.4 Variables d'environnement

Les tests doivent pouvoir se connecter à l'application. Plutôt que de coder en dur les identifiants dans chaque test,
Cypress permet de définir des variables d'environnement.

**À faire** : dans `cypress.config.js`, ajoutez un bloc `env` contenant :

- `apiUrl` : l'URL de base de l'API
- `testUserEmail` : l'email du compte de démo
- `testUserPassword` : le mot de passe du compte de démo

> On accède à ces valeurs dans les tests avec `Cypress.env('testUserEmail')`.


#### Correction 1.4 {collapsible="true"}

Ajoutez le bloc `env` à l'intérieur de `e2e` dans `cypress.config.js` :

```js
module.exports = defineConfig({
    e2e: {
        baseUrl: 'http://localhost:5173',
        viewportWidth: 1280,
        viewportHeight: 720,
        video: false,
        defaultCommandTimeout: 8000,
        specPattern: 'cypress/e2e/**/*.cy.js',

        // Variables d'environnement accessibles via Cypress.env('clé')
        env: {
            apiUrl: 'http://localhost:8080',
            testUserEmail: 'demo@yapuka.dev',
            testUserPassword: 'password',
        },
    },
});
```



---

## Partie 2 — Préparer les outils de test (20 min)

### 2.1 Ajouter des `data-testid` aux composants React

Cypress peut cibler des éléments par texte, par classe CSS ou par sélecteur. Mais les classes Tailwind changent souvent,
et le texte peut être traduit. La bonne pratique est d'utiliser des attributs **`data-testid`** : stables, explicites,
et dédiés aux tests.

**À faire** : ajoutez des `data-testid` sur les éléments clés des composants suivants.

**`LoginPage.jsx`** — identifiez et annotez :

- le champ email
- le champ mot de passe
- le bouton de soumission
- le lien vers l'inscription

**`RegisterPage.jsx`** — identifiez et annotez :

- le champ username
- le champ email
- le champ mot de passe
- le champ confirmation
- le bouton de soumission

**`TaskForm.jsx`** — identifiez et annotez :

- le champ titre
- le champ description
- le sélecteur de priorité
- le champ date d'échéance
- le bouton d'ajout

**`TaskCard.jsx`** — identifiez et annotez :

- la checkbox de statut
- le bouton modifier
- le bouton supprimer
- le titre de la tâche (en mode affichage)

**`TaskList.jsx`** — identifiez et annotez :

- le conteneur de la liste
- le message d'empty state

**`ConfirmDialog.jsx`** — identifiez et annotez :

- le bouton de confirmation
- le bouton d'annulation

**`StatsPanel.jsx`** — identifiez et annotez :

- chaque widget compteur (Total, À faire, Terminées, En retard)

**`Layout.jsx`** — identifiez et annotez :

- le bouton de déconnexion

> **Convention de nommage** : utilisez un format cohérent, par exemple `data-testid="login-email-input"`,
`data-testid="task-delete-button"`. Préfixez par le contexte (login, register, task, confirm).

> **Important** : pour les `TaskCard`, chaque carte représente une tâche différente. Comment faire pour distinguer le
> bouton "supprimer" de la tâche 1 de celui de la tâche 3 ? Pensez à intégrer l'`id` de la tâche dans le `data-testid`.


#### Correction 2.1 {collapsible="true"}

**`LoginPage.jsx`** — modifications dans le JSX du formulaire :

```jsx
<input
    id="email"
    type="email"
    data-testid="login-email-input"
    placeholder="demo@yapuka.dev"
    /* ... */
    {...register('email')}
/>

<input
    id="password"
    type="password"
    data-testid="login-password-input"
    placeholder="••••••••"
    /* ... */
    {...register('password')}
/>

<button
    type="submit"
    data-testid="login-submit-button"
    disabled={isLoading}
    /* ... */
>

    <Link to="/register" data-testid="login-register-link" className="...">
        Créer un compte
    </Link>
```

**`RegisterPage.jsx`** :

```jsx
<input id="username" data-testid="register-username-input" /* ... */ />
<input id="email" data-testid="register-email-input" /* ... */ />
<input id="password" data-testid="register-password-input" /* ... */ />
<input id="confirmPassword" data-testid="register-confirm-input" /* ... */ />
<button type="submit" data-testid="register-submit-button" /* ... */ />
```

**`TaskForm.jsx`** :

```jsx
<input
    type="text"
    placeholder="Que devez-vous faire ?"
    data-testid="task-title-input"
    /* ... */
/>

<button type="submit" data-testid="task-add-button" /* ... */ />

<input
    type="text"
    placeholder="Description (optionnelle)"
    data-testid="task-description-input"
    /* ... */
/>

<select data-testid="task-priority-select" /* ... */ >

    <input type="date" data-testid="task-duedate-input" /* ... */ />
```

**`TaskCard.jsx`** — l'`id` de la tâche est injecté dynamiquement :

```jsx
{/* Conteneur de la carte */
}
<div data-testid={`task-card-${task.id}`} className={`bg-white rounded-xl ...`}>

    {/* Checkbox */}
    <button
        onClick={handleToggleStatus}
        data-testid={`task-toggle-${task.id}`}
        /* ... */
    >

        {/* Titre en mode affichage */}
        <p data-testid={`task-title-${task.id}`} className={/* ... */}>
            {task.title}
        </p>

        {/* Bouton modifier */}
        <button
            onClick={() => setIsEditing(true)}
            data-testid={`task-edit-${task.id}`}
            /* ... */
        >

            {/* Bouton supprimer */}
            <button
                onClick={() => setShowConfirm(true)}
                data-testid={`task-delete-${task.id}`}
                /* ... */
            >
```

Pour le mode édition inline, ajoutez également :

```jsx
<input
    type="text"
    value={editTitle}
    data-testid={`task-edit-title-${task.id}`}
    /* ... */
/>
<button onClick={handleSaveEdit} data-testid={`task-save-${task.id}`} /* ... */ />
<button onClick={handleCancelEdit} data-testid={`task-cancel-${task.id}`} /* ... */ />
```

**`TaskList.jsx`** :

```jsx
{/* Conteneur de la liste */
}
<div data-testid="task-list" className="space-y-3">

    {/* Empty state */}
    <div data-testid="task-empty-state" className="bg-white rounded-xl ...">
```

**`ConfirmDialog.jsx`** :

```jsx
<button onClick={onCancel} data-testid="confirm-cancel-button" /* ... */ />
<button onClick={onConfirm} data-testid="confirm-ok-button" /* ... */ />
```

**`StatsPanel.jsx`** — ajoutez un `data-testid` à chaque widget compteur. Le plus simple est d'utiliser le label en
minuscule :

```jsx
{
    counters.map((counter) => {
        const Icon = counter.icon;
        return (
            <div
                key={counter.label}
                data-testid={`stat-${counter.label.toLowerCase().replace(/ /g, '-')}`}
                className="bg-white rounded-xl ..."
            >
                {/* ... */}
                <p data-testid={`stat-value-${counter.label.toLowerCase().replace(/ /g, '-')}`}
                   className="text-2xl font-bold ...">
                    {counter.value}
                </p>
```

Cela produit : `stat-total`, `stat-à-faire`, `stat-terminées`, `stat-en-retard` et les valeurs correspondantes.

**`Layout.jsx`** :

```jsx
<button
    onClick={handleLogout}
    data-testid="logout-button"
    /* ... */
>
```



### 2.2 Créer une commande de login réutilisable

Le login est un prérequis pour presque tous les tests. Plutôt que de remplir le formulaire à chaque fois, créez une *
*commande personnalisée** Cypress.

**À faire** : dans `cypress/support/commands.js`, créez une commande `cy.login(email, password)`.

Cette commande doit :

1. Appeler directement l'endpoint `POST /api/auth/login` via `cy.request()` (pas via l'UI — c'est beaucoup plus rapide)
2. Récupérer le `token` et le `user` de la réponse
3. Les stocker dans `localStorage` avec les bonnes clés

>  **Indice** : regardez dans `authStore.js` quelles clés sont utilisées dans `localStorage`. Votre commande doit
> reproduire exactement ce que fait le store après un login réussi. Utilisez `cy.window()` pour accéder au `localStorage`
> du navigateur.

> **Référence** : [Custom Commands](https://docs.cypress.io/api/cypress-api/custom-commands), [
`cy.request()`](https://docs.cypress.io/api/commands/request)


#### Correction 2.2 {collapsible="true"}

```js
// cypress/support/commands.js

// =============================================================================
// Commande cy.login() — Authentification programmatique
// =============================================================================
// Se connecte via l'API (pas via l'UI) pour gagner du temps.
// Stocke le token et l'utilisateur dans localStorage exactement comme
// le fait le authStore de l'application.
//
// Utilisation :
//   cy.login()                              → identifiants par défaut (env)
//   cy.login('autre@mail.com', 'pass123')   → identifiants personnalisés
// =============================================================================

Cypress.Commands.add('login', (email, password) => {
    // Utiliser les variables d'environnement par défaut si non fournies
    const userEmail = email || Cypress.env('testUserEmail');
    const userPassword = password || Cypress.env('testUserPassword');

    cy.request({
        method: 'POST',
        url: `${Cypress.env('apiUrl')}/api/auth/login`,
        body: {
            email: userEmail,
            password: userPassword,
        },
    }).then((response) => {
        // Vérifier que le login a fonctionné
        expect(response.status).to.eq(200);
        expect(response.body).to.have.property('token');

        // Stocker dans localStorage (mêmes clés que authStore.js)
        window.localStorage.setItem('yapuka_token', response.body.token);
        window.localStorage.setItem('yapuka_user', JSON.stringify(response.body.user));
    });
});
```



### 2.3 Créer les fixtures de données

Les fixtures Cypress sont des fichiers JSON réutilisables dans les tests.

**À faire** : créez les fichiers suivants dans `cypress/fixtures/` :

**`user.json`** — les identifiants du compte de démo :

```json
{
  "email": "demo@yapuka.dev",
  "password": "password"
}
```

**`task.json`** — une tâche de test que vous allez créer, modifier, puis supprimer. Définissez un titre unique et
reconnaissable (pour pouvoir le retrouver dans la liste), une description, une priorité `high`, et une date d'échéance
future.

>  Utilisez un titre avec un identifiant aléatoire ou un timestamp pour éviter les collisions si les tests tournent
> plusieurs fois sans reset des fixtures. Exemple : `"Tâche Cypress ${Date.now()}"` — mais dans un fichier JSON statique,
> choisissez simplement un titre suffisamment unique.


#### Correction 2.3 {collapsible="true"}

**`cypress/fixtures/user.json`** :

```json
{
  "email": "demo@yapuka.dev",
  "password": "password",
  "username": "Démo User"
}
```

**`cypress/fixtures/task.json`** :

```json
{
  "title": "Tâche Cypress E2E - À supprimer",
  "description": "Tâche créée automatiquement par les tests Cypress",
  "priority": "high",
  "dueDate": "2026-12-31",
  "updatedTitle": "Tâche Cypress E2E - Modifiée"
}
```

> On inclut `updatedTitle` dans la fixture pour le scénario de modification — cela évite de coder le nouveau titre en
> dur dans le test.



---

## Partie 3 — Écrire les tests E2E (35 min)

Vous allez écrire **3 fichiers de tests** couvrant les parcours critiques de l'application. Chaque fichier correspond à
une fonctionnalité.

> **Commandes Cypress essentielles** :
>
> | Commande | Usage |
> |----------|-------|
> | `cy.visit(url)` | Naviguer vers une page |
> | `cy.get(selector)` | Sélectionner un élément |
> | `cy.get('[data-testid="xxx"]')` | Sélectionner par data-testid |
> | `.type('texte')` | Saisir du texte |
> | `.click()` | Cliquer |
> | `.should('exist')` | Vérifier l'existence |
> | `.should('contain', 'texte')` | Vérifier le contenu |
> | `.should('not.exist')` | Vérifier l'absence |
> | `cy.url().should('include', '/path')` | Vérifier l'URL |
> | `cy.intercept()` | Intercepter une requête réseau |
> | `cy.wait('@alias')` | Attendre une requête interceptée |

### 3.1 Test d'authentification — `cypress/e2e/auth.cy.js`

Ce fichier teste le parcours d'authentification complet.

**À faire** : écrivez les scénarios suivants dans un bloc `describe('Authentification', ...)`.

**Scénario 1 — Affichage de la page de login** (`it('affiche le formulaire de connexion', ...)`)

- Visitez `/login`
- Vérifiez que le champ email, le champ mot de passe et le bouton de soumission sont présents
- Vérifiez que le lien vers l'inscription est visible

**Scénario 2 — Échec de connexion** (`it('affiche une erreur avec de mauvais identifiants', ...)`)

- Visitez `/login`
- Remplissez le formulaire avec un email valide mais un **mauvais mot de passe**
- Soumettez le formulaire
- Vérifiez qu'un message d'erreur apparaît
- Vérifiez que l'URL reste sur `/login`

**Scénario 3 — Connexion réussie** (`it('redirige vers le dashboard après connexion', ...)`)

- Visitez `/login`
- Remplissez avec les identifiants du compte de démo (utilisez votre fixture)
- Soumettez le formulaire
- Vérifiez que l'URL change vers `/`
- Vérifiez qu'un élément spécifique au dashboard est visible (le titre "Tableau de bord", le header avec le nom de
  l'utilisateur, etc.)

**Scénario 4 — Protection des routes** (`it('redirige vers /login si non authentifié', ...)`)

- Assurez-vous qu'il n'y a pas de token dans le localStorage (`cy.clearLocalStorage()`)
- Visitez `/`
- Vérifiez que l'URL est redirigée vers `/login`

**Scénario 5 — Déconnexion** (`it('déconnecte l\'utilisateur et redirige vers /login', ...)`)

- Connectez-vous via `cy.login()` (votre commande custom)
- Visitez `/`
- Cliquez sur le bouton de déconnexion
- Vérifiez que l'URL change vers `/login`
- Vérifiez que le localStorage ne contient plus le token

>  **Astuce** : utilisez `beforeEach(() => { cy.clearLocalStorage(); })` pour garantir un état propre entre chaque
> test.


#### Correction 3.1 {collapsible="true"}

```js
// cypress/e2e/auth.cy.js

// =============================================================================
// Tests E2E — Authentification
// =============================================================================
// Couvre : affichage login, échec, succès, protection de routes, déconnexion.
// =============================================================================

describe('Authentification', () => {
    // Nettoyer le localStorage avant chaque test pour partir d'un état vierge
    beforeEach(() => {
        cy.clearLocalStorage();
    });

    // ---------------------------------------------------------------------------
    // Scénario 1 : la page de login affiche tous les éléments attendus
    // ---------------------------------------------------------------------------
    it('affiche le formulaire de connexion', () => {
        cy.visit('/login');

        cy.get('[data-testid="login-email-input"]').should('exist');
        cy.get('[data-testid="login-password-input"]').should('exist');
        cy.get('[data-testid="login-submit-button"]').should('exist');
        cy.get('[data-testid="login-register-link"]').should('contain', 'Créer un compte');
    });

    // ---------------------------------------------------------------------------
    // Scénario 2 : un mauvais mot de passe affiche une erreur
    // ---------------------------------------------------------------------------
    it('affiche une erreur avec de mauvais identifiants', () => {
        cy.visit('/login');

        cy.get('[data-testid="login-email-input"]').type('demo@yapuka.dev');
        cy.get('[data-testid="login-password-input"]').type('mauvais_mot_de_passe');
        cy.get('[data-testid="login-submit-button"]').click();

        // Vérifier qu'un message d'erreur apparaît (le toast ou le bloc d'erreur)
        cy.contains('Identifiants invalides').should('be.visible');

        // L'URL ne doit pas avoir changé
        cy.url().should('include', '/login');
    });

    // ---------------------------------------------------------------------------
    // Scénario 3 : des identifiants corrects redirigent vers le dashboard
    // ---------------------------------------------------------------------------
    it('redirige vers le dashboard après connexion', () => {
        cy.fixture('user').then((user) => {
            cy.visit('/login');

            cy.get('[data-testid="login-email-input"]').type(user.email);
            cy.get('[data-testid="login-password-input"]').type(user.password);
            cy.get('[data-testid="login-submit-button"]').click();

            // Vérifier la redirection vers la racine
            cy.url().should('eq', Cypress.config('baseUrl') + '/');

            // Vérifier qu'un élément du dashboard est visible
            cy.contains('Tableau de bord').should('be.visible');
        });
    });

    // ---------------------------------------------------------------------------
    // Scénario 4 : un utilisateur non connecté est redirigé vers /login
    // ---------------------------------------------------------------------------
    it('redirige vers /login si non authentifié', () => {
        // clearLocalStorage déjà fait dans beforeEach
        cy.visit('/');

        cy.url().should('include', '/login');
    });

    // ---------------------------------------------------------------------------
    // Scénario 5 : le bouton de déconnexion nettoie la session
    // ---------------------------------------------------------------------------
    it("déconnecte l'utilisateur et redirige vers /login", () => {
        // Se connecter via la commande programmatique
        cy.login();
        cy.visit('/');

        // Vérifier qu'on est bien sur le dashboard
        cy.contains('Tableau de bord').should('be.visible');

        // Cliquer sur le bouton de déconnexion
        cy.get('[data-testid="logout-button"]').click();

        // Vérifier la redirection
        cy.url().should('include', '/login');

        // Vérifier que le token a été supprimé du localStorage
        cy.window().then((win) => {
            expect(win.localStorage.getItem('yapuka_token')).to.be.null;
        });
    });
});
```



### 3.2 Test CRUD des tâches — `cypress/e2e/tasks.cy.js`

Ce fichier teste le cycle de vie complet d'une tâche.

**À faire** : écrivez les scénarios suivants dans un `describe('Gestion des tâches', ...)`.

**Setup** : dans un `beforeEach`, connectez-vous via `cy.login()`, puis visitez `/`. Chargez votre fixture `task.json`
via `cy.fixture('task')`.

**Scénario 1 — Affichage de la liste** (`it('affiche la liste des tâches', ...)`)

- Vérifiez que le conteneur de la liste est présent
- Vérifiez qu'il y a au moins une tâche affichée (les fixtures Symfony en ont créé 10 pour le user démo)

>  `cy.get('[data-testid="task-list"]').children().should('have.length.greaterThan', 0)`

**Scénario 2 — Création d'une tâche** (`it('crée une nouvelle tâche', ...)`)

- Remplissez le formulaire de création avec les données de votre fixture
- Soumettez
- Vérifiez que la nouvelle tâche apparaît dans la liste (cherchez le titre)
- **Bonus** : interceptez la requête `POST /api/tasks` avec `cy.intercept()` et vérifiez que la réponse a un status 201

**Scénario 3 — Modification d'une tâche** (`it('modifie le titre d\'une tâche', ...)`)

C'est le scénario le plus délicat. Voici le flux à reproduire :

1. Trouvez la tâche que vous venez de créer dans la liste (par son titre)
2. Cliquez sur son bouton "modifier"
3. Effacez le titre actuel et tapez un nouveau titre
4. Sauvegardez
5. Vérifiez que le nouveau titre apparaît dans la liste

>  **Problème** : comment cibler la bonne `TaskCard` parmi toutes ? Utilisez `.contains()` pour trouver la carte
> contenant votre titre, puis naviguez dans le DOM avec `.parent()` ou `.closest()` pour accéder aux boutons d'action.
> Alternativement, si vous avez bien ajouté les `data-testid` dynamiques avec l'id de la tâche, interceptez la création
> pour récupérer l'id.

**Scénario 4 — Changement de statut** (`it('marque une tâche comme terminée', ...)`)

- Trouvez votre tâche modifiée
- Cliquez sur sa checkbox
- Vérifiez que le style de la tâche change (classe `line-through`, ou vérifiez que le statut dans le sélecteur passe à "
  Terminée")

**Scénario 5 — Suppression d'une tâche** (`it('supprime une tâche après confirmation', ...)`)

- Trouvez votre tâche
- Cliquez sur le bouton supprimer
- Vérifiez que la modale de confirmation s'affiche
- Cliquez sur le bouton de confirmation
- Vérifiez que la tâche n'apparaît plus dans la liste

>  **Ordre des tests** : dans ce `describe`, les tests dépendent les uns des autres (le scénario 3 modifie la tâche
> créée au scénario 2, etc.). Réfléchissez à l'ordre d'exécution. Cypress exécute les `it()` dans l'ordre du fichier, mais
> par défaut **chaque test est indépendant**. Vous avez deux approches :
> - **Approche A** : un seul gros `it()` qui fait tout le CRUD d'affilée
> - **Approche B** : plusieurs `it()` mais en utilisant une variable partagée (via `let` au niveau du `describe`) pour
    stocker l'id ou le titre de la tâche créée
>
> Choisissez celle qui vous semble la plus lisible. La correction ci-dessous utilise l'approche B.


#### Correction 3.2 {collapsible="true"}

```js
// cypress/e2e/tasks.cy.js

// =============================================================================
// Tests E2E — Gestion des tâches (CRUD complet)
// =============================================================================
// Couvre : affichage liste, création, modification, changement de statut,
// suppression avec confirmation.
//
// Stratégie : on utilise une variable `createdTaskId` partagée entre les tests
// pour cibler la tâche créée dans les scénarios suivants.
// On intercepte la requête POST pour capturer l'id retourné par l'API.
// =============================================================================

describe('Gestion des tâches', () => {
    // Variable partagée pour stocker l'id de la tâche créée
    let createdTaskId;

    beforeEach(() => {
        // Se connecter et aller sur le dashboard avant chaque test
        cy.login();
        cy.visit('/');
        // Attendre que la liste soit chargée
        cy.get('[data-testid="task-list"]', {timeout: 10000}).should('exist');
    });

    // ---------------------------------------------------------------------------
    // Scénario 1 : la liste affiche les tâches existantes (fixtures Symfony)
    // ---------------------------------------------------------------------------
    it('affiche la liste des tâches', () => {
        cy.get('[data-testid="task-list"]')
            .children()
            .should('have.length.greaterThan', 0);
    });

    // ---------------------------------------------------------------------------
    // Scénario 2 : créer une tâche via le formulaire
    // ---------------------------------------------------------------------------
    it('crée une nouvelle tâche', () => {
        cy.fixture('task').then((task) => {
            // Intercepter la requête POST pour capturer l'id de la tâche créée
            cy.intercept('POST', '**/api/tasks').as('createTask');

            // Remplir le formulaire
            cy.get('[data-testid="task-title-input"]').type(task.title);
            cy.get('[data-testid="task-description-input"]').type(task.description);
            cy.get('[data-testid="task-priority-select"]').select(task.priority);
            cy.get('[data-testid="task-duedate-input"]').type(task.dueDate);

            // Soumettre
            cy.get('[data-testid="task-add-button"]').click();

            // Attendre la réponse API et capturer l'id
            cy.wait('@createTask').then((interception) => {
                expect(interception.response.statusCode).to.eq(201);
                createdTaskId = interception.response.body.id;
            });

            // Vérifier que la tâche apparaît dans la liste
            cy.contains(task.title).should('be.visible');
        });
    });

    // ---------------------------------------------------------------------------
    // Scénario 3 : modifier le titre de la tâche créée
    // ---------------------------------------------------------------------------
    it("modifie le titre d'une tâche", () => {
        cy.fixture('task').then((task) => {
            // Cliquer sur le bouton modifier de la bonne tâche
            cy.get(`[data-testid="task-edit-${createdTaskId}"]`).click();

            // Effacer le titre actuel et saisir le nouveau
            cy.get(`[data-testid="task-edit-title-${createdTaskId}"]`)
                .clear()
                .type(task.updatedTitle);

            // Sauvegarder
            cy.get(`[data-testid="task-save-${createdTaskId}"]`).click();

            // Vérifier que le nouveau titre est affiché
            cy.get(`[data-testid="task-title-${createdTaskId}"]`)
                .should('contain', task.updatedTitle);
        });
    });

    // ---------------------------------------------------------------------------
    // Scénario 4 : cocher la tâche comme terminée
    // ---------------------------------------------------------------------------
    it('marque une tâche comme terminée', () => {
        // Cliquer sur la checkbox
        cy.get(`[data-testid="task-toggle-${createdTaskId}"]`).click();

        // Vérifier que le titre est barré (classe line-through)
        cy.get(`[data-testid="task-title-${createdTaskId}"]`)
            .should('have.class', 'line-through');
    });

    // ---------------------------------------------------------------------------
    // Scénario 5 : supprimer la tâche avec confirmation
    // ---------------------------------------------------------------------------
    it('supprime une tâche après confirmation', () => {
        cy.fixture('task').then((task) => {
            // Cliquer sur le bouton supprimer
            cy.get(`[data-testid="task-delete-${createdTaskId}"]`).click();

            // Vérifier que la modale de confirmation apparaît
            cy.get('[data-testid="confirm-ok-button"]').should('be.visible');

            // Confirmer la suppression
            cy.get('[data-testid="confirm-ok-button"]').click();

            // Vérifier que la tâche n'est plus dans la liste
            cy.contains(task.updatedTitle).should('not.exist');
        });
    });
});
```

> **Note importante** : cette correction utilise l'approche B (variable partagée `createdTaskId`). Les tests **doivent**
> s'exécuter dans l'ordre. Cela fonctionne car Cypress exécute les `it()` séquentiellement dans un même fichier. Mais si
> un test échoue, tous les suivants échoueront aussi — c'est un compromis accepté pour la lisibilité.



### 3.3 Test des statistiques — `cypress/e2e/stats.cy.js`

**À faire** : écrivez un fichier court qui vérifie le bon fonctionnement du tableau de bord statistiques.

**Scénario 1 — Affichage des compteurs** (`it('affiche les widgets de statistiques', ...)`)

- Connectez-vous et visitez `/`
- Vérifiez que les 4 compteurs sont présents (Total, À faire, Terminées, En retard)
- Vérifiez que les valeurs sont des nombres (pas NaN, pas vide)

**Scénario 2 — Mise à jour après création** (`it('met à jour les stats après création d\'une tâche', ...)`)

- Interceptez `GET /api/tasks/stats` et donnez-lui un alias (`cy.intercept(...).as('getStats')`)
- Récupérez la valeur du compteur "Total" avant la création
- Créez une nouvelle tâche
- Attendez que l'appel stats soit refait (`cy.wait('@getStats')`)
- Vérifiez que le compteur "Total" a augmenté de 1

>  Pour lire la valeur d'un compteur, utilisez `.invoke('text')` sur l'élément, puis `.then(text => ...)` pour
> travailler avec la valeur.


#### Correction 3.3 {collapsible="true"}

```js
// cypress/e2e/stats.cy.js

// =============================================================================
// Tests E2E — Statistiques du tableau de bord
// =============================================================================
// Couvre : affichage des 4 compteurs, mise à jour après création d'une tâche.
// =============================================================================

describe('Statistiques', () => {
    beforeEach(() => {
        cy.login();
        cy.visit('/');
    });

    // ---------------------------------------------------------------------------
    // Scénario 1 : les 4 widgets de statistiques sont visibles avec des nombres
    // ---------------------------------------------------------------------------
    it('affiche les widgets de statistiques', () => {
        // Vérifier la présence des 4 compteurs
        cy.get('[data-testid="stat-total"]').should('exist');
        cy.get('[data-testid="stat-à-faire"]').should('exist');
        cy.get('[data-testid="stat-terminées"]').should('exist');
        cy.get('[data-testid="stat-en-retard"]').should('exist');

        // Vérifier que la valeur de "Total" est un nombre et pas vide
        cy.get('[data-testid="stat-value-total"]')
            .invoke('text')
            .then((text) => {
                const value = parseInt(text.trim(), 10);
                expect(value).to.be.a('number');
                expect(value).to.not.be.NaN;
                expect(value).to.be.greaterThan(0);
            });
    });

    // ---------------------------------------------------------------------------
    // Scénario 2 : le compteur "Total" augmente de 1 après création d'une tâche
    // ---------------------------------------------------------------------------
    it("met à jour les stats après création d'une tâche", () => {
        // Lire la valeur initiale du compteur Total
        cy.get('[data-testid="stat-value-total"]')
            .invoke('text')
            .then((initialText) => {
                const initialTotal = parseInt(initialText.trim(), 10);

                // Intercepter le prochain appel aux stats (déclenché après la création)
                cy.intercept('GET', '**/api/tasks/stats').as('getStats');

                // Créer une tâche
                cy.get('[data-testid="task-title-input"]').type('Tâche stats test');
                cy.get('[data-testid="task-add-button"]').click();

                // Attendre que les stats soient rafraîchies
                cy.wait('@getStats');

                // Vérifier que le compteur a augmenté de 1
                cy.get('[data-testid="stat-value-total"]')
                    .invoke('text')
                    .should((newText) => {
                        const newTotal = parseInt(newText.trim(), 10);
                        expect(newTotal).to.eq(initialTotal + 1);
                    });
            });
    });
});
```



---

## Partie 4 — Exécution et validation (15 min)

### 4.1 Mode interactif

**À faire** :

1. Lancez `npm run cy:open`
2. Exécutez chaque fichier de test un par un
3. Observez l'exécution en temps réel dans le navigateur Cypress
4. Si un test échoue :
    - Lisez le message d'erreur
    - Utilisez le "time travel" de Cypress (cliquez sur chaque étape dans le panneau gauche pour voir l'état du DOM à
      cet instant)
    - Corrigez et relancez


#### Correction 4.1 — Problèmes fréquents et solutions {collapsible="true"}

| Symptôme                                                 | Cause probable                                                            | Solution                                                                                                                                             |
|----------------------------------------------------------|---------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Timed out retrying: cy.get() could not find element`    | Le `data-testid` est mal orthographié ou l'élément n'est pas encore rendu | Vérifier l'orthographe exacte dans le composant React. Augmenter le `timeout` si besoin : `cy.get('...', { timeout: 15000 })`                        |
| `cy.request() failed: 401 Unauthorized`                  | L'URL de l'API est incorrecte dans `Cypress.env('apiUrl')`                | Vérifier que `apiUrl` pointe vers `http://localhost:8080` (Nginx) et non vers `http://localhost:5173` (Vite)                                         |
| Le test de login réussit mais la page reste sur `/login` | Le `localStorage` est écrit mais React ne re-rend pas                     | Après `cy.login()`, faire `cy.visit('/')` pour que React relise le store                                                                             |
| `createdTaskId` est `undefined` dans le scénario 3       | Les tests Cypress sont isolés par défaut et `let` est réinitialisé        | Vérifier que le scénario 2 (création) s'exécute avant le scénario 3. S'assurer que `createdTaskId` est bien assigné dans le `.then()` du `cy.wait()` |
| Le clic sur la checkbox ne change rien visuellement      | L'Optimistic UI met à jour le store mais le DOM n'a pas encore re-rendu   | Utiliser `.should()` avec retry automatique plutôt que de vérifier immédiatement                                                                     |
| `cy.intercept()` ne capture aucune requête               | Le pattern d'URL ne matche pas                                            | Utiliser `**/api/tasks` (double wildcard) pour matcher n'importe quel domaine/port                                                                   |



### 4.2 Mode headless

**À faire** :

1. Lancez `npm run cy:run`
2. Observez la sortie console : chaque test affiche ✅ ou ❌
3. Vérifiez le résumé final : tous les tests doivent passer


#### Correction 4.2 — Sortie attendue {collapsible="true"}

```
  Authentification
    ✓ affiche le formulaire de connexion (1234ms)
    ✓ affiche une erreur avec de mauvais identifiants (2345ms)
    ✓ redirige vers le dashboard après connexion (3456ms)
    ✓ redirige vers /login si non authentifié (567ms)
    ✓ déconnecte l'utilisateur et redirige vers /login (2345ms)

  Gestion des tâches
    ✓ affiche la liste des tâches (1234ms)
    ✓ crée une nouvelle tâche (2345ms)
    ✓ modifie le titre d'une tâche (2345ms)
    ✓ marque une tâche comme terminée (1234ms)
    ✓ supprime une tâche après confirmation (2345ms)

  Statistiques
    ✓ affiche les widgets de statistiques (1234ms)
    ✓ met à jour les stats après création d'une tâche (3456ms)

  12 passing (25s)
```

Si `npm run test:e2e` échoue avec une erreur de connexion au port 5173 :

- Vérifiez que le serveur Vite n'est pas déjà lancé dans un autre terminal
- Vérifiez la syntaxe de `start-server-and-test` dans `package.json`



### 4.3 Rapport

**À faire** : notez dans un fichier `TESTS.md` à la racine de `front/` :

- Le nombre total de tests
- Le nombre de tests par fichier
- Le temps d'exécution total
- Les difficultés rencontrées et comment vous les avez résolues


#### Correction 4.3 — Exemple de TESTS.md {collapsible="true"}

```markdown
# Rapport des tests E2E — Yapuka

## Résumé

| Fichier | Tests | Durée |
|---------|-------|-------|
| `auth.cy.js` | 5 | ~10s |
| `tasks.cy.js` | 5 | ~12s |
| `stats.cy.js` | 2 | ~6s |
| **Total** | **12** | **~28s** |

## Résultat

✅ 12/12 tests passent en mode headless (`npm run cy:run`)

## Difficultés rencontrées

1. **Ciblage des TaskCard** : les data-testid dynamiques (`task-edit-{id}`)
   nécessitent de capturer l'id lors de la création via `cy.intercept()`.

2. **Ordre des tests CRUD** : les scénarios sont interdépendants. La variable
   `createdTaskId` doit être partagée au niveau du `describe`.

3. **Timing des stats** : après la création d'une tâche, il faut attendre
   que l'appel `GET /api/tasks/stats` soit terminé avant de vérifier le compteur.
   Résolu avec `cy.intercept().as('getStats')` + `cy.wait('@getStats')`.
```



---

## Critères de validation

| # | Critère                                                                              | Points  |
|---|--------------------------------------------------------------------------------------|---------|
| 1 | Cypress est installé et `cy:open` / `cy:run` fonctionnent                            | /2      |
| 2 | `cypress.config.js` est correctement configuré (baseUrl, env, timeouts)              | /1      |
| 3 | Les `data-testid` sont ajoutés sur tous les composants listés en partie 2.1          | /2      |
| 4 | La commande `cy.login()` fonctionne et utilise `cy.request()` (pas l'UI)             | /2      |
| 5 | Les 5 tests d'authentification passent                                               | /3      |
| 6 | Les 5 tests CRUD des tâches passent                                                  | /5      |
| 7 | Les 2 tests de statistiques passent                                                  | /2      |
| 8 | Les tests sont lisibles : noms explicites, assertions claires, pas de valeurs en dur | /2      |
| 9 | `npm run cy:run` exécute tous les tests avec 0 échec                                 | /1      |
|   | **Total**                                                                            | **/20** |

---

<!--
## Pour aller plus loin

Si vous avez terminé en avance, voici des défis supplémentaires :

- **Inscription** : écrivez un test qui crée un nouveau compte via le formulaire d'inscription, vérifie la redirection
  vers `/`, puis supprime le compte (ou utilisez un email unique à chaque exécution).
- **Empty state** : écrivez un test qui supprime toutes les tâches et vérifie que le message "Bravo, vous n'avez rien à
  faire !" s'affiche.
- **Responsive** : ajoutez un `describe` qui redimensionne le viewport en mobile (`cy.viewport('iphone-x')`) et vérifie
  que le formulaire et la liste restent utilisables.
- **Intercepts avancés** : simulez une erreur 500 du serveur avec
  `cy.intercept('GET', '/api/tasks', { statusCode: 500 })` et vérifiez que le toast d'erreur s'affiche.

-->