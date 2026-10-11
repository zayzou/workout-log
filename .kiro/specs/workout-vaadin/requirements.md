# Requirements — Workout Log (Vaadin / Spring Boot)

## Contexte

Portage de la PWA mono-fichier `index.html` (HTML + JS + `localStorage`) vers une application
Vaadin + Spring Boot, en reprenant **exactement le même UI/UX** (look iOS : onglets bas, strip de
pills de jours, timer flottant, thème clair/sombre) et en ajoutant des améliorations de persistance
et de flexibilité.

> **Langue de l'UI : anglais** (labels, titres, boutons), comme l'app actuelle. La documentation de
> la spec est en français.

### Stack verrouillée
- Java 25, Spring Boot 4.1, Vaadin 25 (thème **Aura**).
- Persistance : **Spring Data JDBC** + **Flyway**, base **H2 en mode fichier** (install locale, sans serveur).
- Auth : login multi-utilisateur **simple**, conçu pour accueillir OAuth2 (Google…) plus tard.
- PWA installable sur mobile (via `@PWA` Vaadin + manifeste + service worker).

> Note build : on abandonne jOOQ (et sa génération de code via Testcontainers) de la référence JTAF4
> au profit de Spring Data JDBC pour simplifier le build. L'**architecture en couches** et
> l'organisation du code de JTAF4 restent la référence (packages `domain` / `ui` / `ui.dialog` /
> `configuration.security`, vues = routes, repositories, dialogs de confirmation réutilisables).

---

## Exigences

### R1 — Fidélité UI/UX
**En tant qu'**utilisateur de l'app actuelle, **je veux** retrouver le même écran et les mêmes
interactions, **afin de** ne pas être dépaysé.

Critères (EARS) :
- QUAND l'app s'affiche, ELLE DOIT reproduire : onglets bas (History / Workout / Guide), l'accueil
  avec barre de progression + grille de cartes de jours, l'écran de jour avec strip de pills,
  stats (Duration / Volume / Sets), tableau de sets (SET / PREVIOUS / LBS|KG / REPS / ✓), et le
  timer de repos flottant.
- ELLE DOIT fournir les thèmes auto / light / dark avec la même palette (`--blue`, `--green`,
  teintes iOS) via le mécanisme de theming recommandé par Vaadin (tokens Aura + CSS custom), sans
  hacks de Shadow DOM.
- ELLE DOIT être mobile-first et respecter les safe-areas (encoche).

### R2 — Modèle de données persistant
**En tant qu'**utilisateur, **je veux** que mes programmes, séances et logs soient stockés en base,
**afin de** les conserver durablement et de permettre des programmes personnalisés.

Critères :
- LE SYSTÈME DOIT modéliser : `app_user`, `exercise` (catalogue), `exercise_info`
  (muscles + URL vidéo), `program`, `program_day`, `day_exercise` (prescription : sets, reps,
  rest, RIR, position), `workout_session`, `session_exercise` (incl. substitution temporaire),
  `set_log`, `user_preferences`.
- LE SYSTÈME DOIT charger le programme 4-day Upper/Lower existant (6 jours, ~40 exercices) comme
  **données initiales** via migration Flyway (`V...__Seed`).
- Les écrans DOIVENT lire/écrire ces entités (plus de modèle statique en dur dans le front).

### R3 — Authentification simple
**En tant qu'**utilisateur, **je veux** me connecter, **afin que** mes données me soient propres.
- LE SYSTÈME DOIT fournir login + inscription (email + mot de passe haché).
- LES données (programmes custom, séances, logs, préférences) DOIVENT être rattachées à l'utilisateur.
- LA configuration de sécurité DOIT être structurée pour ajouter un provider OAuth2 (Google) sans
  refonte (séparation claire, pas de couplage fort au provider).

### R4 — Contrôles de séance Start / Pause / Finish
- QUAND l'utilisateur ouvre un jour non démarré, IL DOIT voir un bouton **Start**.
- QUAND il appuie sur **Start**, LE SYSTÈME DOIT demander confirmation puis démarrer le chrono de séance.
- QUAND la séance tourne, IL DOIT voir **Pause** (met le chrono en pause, temps cumulé) et **Finish**.
- QUAND il appuie sur **Finish**, LE SYSTÈME DOIT demander confirmation avant de clôturer.
- LE timer de **repos** entre sets reste séparé et se déclenche automatiquement quand un set est coché.

### R5 — Gestion des sets (ajouter / supprimer)
- IL DOIT exister un bouton **+ Add Set** (déjà présent) ET un bouton **Remove set** par série.
- LA suppression DOIT mettre à jour volume et compteur de sets.

### R6 — Changer l'exercice en cours de séance
- PENDANT une séance, l'utilisateur DOIT pouvoir remplacer un exercice prescrit par un autre
  **chargé depuis le catalogue (BD)**.
- LE remplacement DOIT être **temporaire** : il n'affecte que la séance en cours, pas le programme.
- LE picker DOIT lister **tous** les exercices du catalogue ET permettre d'**en créer un nouveau à la volée**.

### R7 — Temps de repos configurable
- LA page Preferences DOIT permettre de configurer le temps de repos.
- LE réglage DOIT proposer une **valeur par défaut globale** utilisée quand une prescription n'a pas
  de repos spécifique ; les repos par exercice du programme restent prioritaires.

### R8 — Confirmations anti-erreur
- Démarrer une séance ET terminer une séance DOIVENT ouvrir un **écran de confirmation**
  (réutiliser un `ConfirmDialog` à la JTAF4) pour éviter les actions accidentelles.

### R9 — Historique enrichi avec recherche
- L'historique DOIT permettre de chercher, **de façon séparée** :
  - par **plage de dates** (début/fin),
  - par **type de séance** (Upper A / Lower A / Upper B / Lower B …),
  - par **texte libre** (ex. nom d'exercice).

### R10 — Création de programme (UI)
- L'utilisateur DOIT pouvoir **créer/éditer un programme** : jours, et pour chaque jour la liste
  ordonnée d'exercices avec prescription (sets, reps, rest, RIR).
- LES exercices proviennent du catalogue ; création à la volée possible (cf. R6).

### R11 — Infos exercice (muscles + vidéo)
- QUAND l'utilisateur sélectionne un exercice (ex. « Incline Dumbbell Press »), LE SYSTÈME DOIT
  afficher les **muscles concernés** et un **lien vidéo YouTube**.
- LES infos initiales des ~40 exercices existants seront saisies dans le seed (best effort).

### R12 — Préférences
- La page Preferences DOIT gérer : **unité (kg / lbs)**, temps de repos par défaut, thème (auto/light/dark).
- LE changement d'unité DOIT s'appliquer à l'affichage et à la saisie du volume/poids.

### R13 — PWA installable
- L'app DOIT être installable sur mobile comme PWA (manifeste, icônes existantes réutilisées,
  service worker, offline shell), via la configuration PWA de Vaadin.

---

## Hors périmètre (non-goals)
- Pas de multi-organisation / rôles complexes (JTAF4) — auth simple seulement.
- Pas d'import Excel, PDF, rapports (fonctions JTAF4 non demandées).
- Pas de synchronisation cloud/serveur distant — base locale H2 fichier.
- OAuth2 Google : seulement **préparé** structurellement, pas implémenté dans cette itération.

## Points à vérifier à l'implémentation
- Via le **MCP Vaadin** : version exacte 25.x, API/nom du thème **Aura**, recommandations
  officielles de theming et de configuration PWA. (MCP non connecté au moment de la rédaction.)
