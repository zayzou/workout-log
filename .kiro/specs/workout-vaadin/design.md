# Design — Workout Log (Vaadin / Spring Boot)

## 1. Vue d'ensemble & architecture

Architecture en couches calquée sur JTAF4 (vue → domaine → persistance), mais avec **Spring Data
JDBC** au lieu de jOOQ.

```
com.workout
├── WorkoutApplication.java            @SpringBootApplication, @PWA (installable)
├── configuration
│   └── security
│       ├── SecurityConfiguration.java  Spring Security (form login + prêt pour OAuth2)
│       ├── CurrentUser.java            accès à l'utilisateur courant (SecurityContext)
│       └── Role.java
├── domain                              entités + repositories + services
│   ├── user/        AppUser, AppUserRepository, UserService (register/login)
│   ├── catalog/     Exercise, ExerciseInfo, + repositories
│   ├── program/     Program, ProgramDay, DayExercise, + repositories, ProgramService
│   ├── session/     WorkoutSession, SessionExercise, SetLog, + repositories, WorkoutService
│   └── preferences/ UserPreferences, PreferencesRepository, Unit (enum), Theme (enum)
└── ui                                  vues Vaadin (routes) + composants
    ├── MainLayout.java                 shell : zone contenu + barre d'onglets bas (tabs)
    ├── HomeView.java                   "/"  accueil (progression + cartes de jours)
    ├── DayView.java                    "/day/{id}" écran de séance (le cœur)
    ├── HistoryView.java                "/history" historique + recherche
    ├── GuideView.java                  "/guide" principes + Appearance
    ├── PreferencesView.java            "/preferences" unité / repos / thème
    ├── ProgramEditorView.java          "/programs" création/édition de programme
    ├── LoginView.java / RegisterView.java
    ├── component/
    │   ├── RestTimer.java              timer de repos flottant (polling côté serveur)
    │   ├── DayPillStrip.java           strip horizontal de pills de jours
    │   └── SetRow.java                 ligne de set (poids / reps / ✓ / remove)
    └── dialog/
        ├── ConfirmDialog.java          confirmation réutilisable (start/finish/remove)
        ├── ExercisePickerDialog.java   choix d'un exercice du catalogue (+ créer à la volée)
        └── ExerciseInfoDialog.java     muscles + lien vidéo YouTube
```

### Décisions clés
- **Spring Data JDBC** : agrégats simples, repositories `CrudRepository` + requêtes `@Query`.
  Mapping explicite via annotations `@Table`, `@Id`, `@Column`, `@MappedCollection`.
- **H2 fichier** : `jdbc:h2:file:~/.workout/db` (chemin configurable). Flyway gère le schéma.
- **Theming** : dossier `frontend/themes/workout/` avec `theme.json` + `styles.css`, surcharge des
  **design tokens Aura** pour mapper la palette iOS actuelle. Aucune manipulation de Shadow DOM.
  > À confirmer via MCP Vaadin : nom exact du thème de base (Aura), chemin et API de theming 25.x.
- **PWA** : annotation `@PWA(name=..., shortName="Workout")` sur `WorkoutApplication`, icônes
  existantes (`icon-*.png`) copiées dans les ressources, offline shell géré par Vaadin.

---

## 2. Modèle de données (schéma H2 / Flyway)

```
app_user(id PK, email UNIQUE, password_hash, display_name, created_at)

exercise(id PK, name, owner_user_id NULL)          -- NULL = catalogue global (seed)
exercise_info(exercise_id PK/FK, primary_muscles, secondary_muscles, youtube_url)

program(id PK, owner_user_id FK, name, description, is_template)
program_day(id PK, program_id FK, position, title, focus, is_rest)
day_exercise(id PK, program_day_id FK, exercise_id FK, position,
             sets, rep_range, rest_seconds NULL, rir)

workout_session(id PK, user_id FK, program_day_id FK, day_title,
                status ENUM(NOT_STARTED,RUNNING,PAUSED,FINISHED),
                started_at, accumulated_seconds, finished_at, cycle)
session_exercise(id PK, session_id FK, position,
                 source_day_exercise_id NULL,       -- prescription d'origine
                 exercise_id FK,                     -- exercice effectif (peut différer = substitution)
                 substituted BOOLEAN,
                 rep_range, rest_seconds, rir)
set_log(id PK, session_exercise_id FK, set_number, weight, reps, done, is_warmup)

user_preferences(user_id PK/FK, unit ENUM(KG,LBS), default_rest_seconds, theme ENUM(AUTO,LIGHT,DARK))
```

### Correspondance avec le modèle statique actuel
- `D[]` (6 jours) → `program` « 4-day Upper/Lower » + 6 `program_day`.
- Chaque `['nom',sets,reps,restLabel,restSec,rir]` → une `exercise` (catalogue) + une `day_exercise`.
- `S.log` / `S.prev` → `set_log` par `session_exercise` ; « previous » = dernier `set_log` fini du
  même exercice.
- `S.hist` → `workout_session` au statut `FINISHED` (+ ses `session_exercise`/`set_log`).
- `S.theme`, unité, repos → `user_preferences`.

### Substitution temporaire (R6)
À la création de la séance, chaque `day_exercise` est copiée en `session_exercise`
(`source_day_exercise_id` renseigné, `substituted=false`). Remplacer un exercice met simplement à
jour `exercise_id` + `substituted=true` sur la ligne de séance — le `program`/`day_exercise`
d'origine n'est jamais modifié.

### Repos effectif (R7)
`session_exercise.rest_seconds` = `day_exercise.rest_seconds` s'il existe, sinon
`user_preferences.default_rest_seconds`.

---

## 3. Flux principaux

### Démarrer / Pause / Finish (R4, R8)
```
HomeView → DayView(dayId)
  [Start] → ConfirmDialog("Start workout?")
          → WorkoutService.start(user, dayId) : crée session RUNNING + session_exercise copiés
  [Pause] → accumulated_seconds += now - started_at ; status PAUSED
  [Resume]→ started_at = now ; status RUNNING
  [Finish]→ ConfirmDialog("Finish workout?") → status FINISHED, finished_at=now
          → previous (prev) recalculé pour la prochaine fois
```
Chrono affiché = `accumulated_seconds (+ now - started_at si RUNNING)`. Rafraîchi par polling
Vaadin (`@Push` ou `UI.setPollInterval`) comme le `setInterval` actuel.

### Set : add / tick / remove (R5)
- `+ Add Set` → nouveau `set_log` (vide) sur la `session_exercise`.
- Cocher ✓ → `done=true`, préremplit poids/reps depuis « previous » si vide, déclenche RestTimer.
- `Remove set` → supprime le `set_log` ; volume & compteur recalculés.

### Changer l'exercice (R6)
`DayView` → bouton « Change exercise » sur un `session_exercise` → `ExercisePickerDialog`
(liste catalogue filtrable + bouton « New exercise… ») → met à jour `session_exercise.exercise_id`.

### Infos exercice (R11)
Clic sur le nom d'un exercice → `ExerciseInfoDialog` affichant `exercise_info`
(muscles primaires/secondaires + lien YouTube ouvrant un nouvel onglet).

### Historique + recherche (R9)
`HistoryView` : trois contrôles **séparés** — un `DatePicker` de plage (from/to), un `Select` de
type de séance (distinct des `day_title`), un `TextField` texte libre (match sur nom d'exercice).
Chaque filtre interroge le repository indépendamment.

### Création de programme (R10)
`ProgramEditorView` : CRUD programme → jours (position, titre, focus, rest) → pour chaque jour,
liste ordonnée de `day_exercise` (exercice via picker, sets, reps, rest, RIR).

---

## 4. Theming (R1)

- `frontend/themes/workout/theme.json` déclare le thème.
- `styles.css` mappe les variables actuelles sur les tokens Aura :
  `--blue → couleur primaire`, `--green → succès`, radius, surfaces `--bg/--bg2`, lignes `--line`.
- Composants spécifiques recréés en CSS : barre d'onglets bas fixe, pills rondes de jours
  (états on/completed/in-progress/rest), timer flottant, cartes de jour, lignes de tableau de sets.
- Thème light/dark/auto piloté par préférence utilisateur + `prefers-color-scheme`, via attribut
  `theme` sur `<html>` (équivalent du `data-theme` actuel).

> Point ouvert : API exacte de theming Aura en Vaadin 25 à confirmer via MCP avant d'écrire le CSS.

---

## 5. Sécurité (R3)
- `SecurityConfiguration` : form login (`LoginView`), routes publiques = login/register, le reste protégé.
- Mots de passe hachés (BCrypt).
- `CurrentUser` expose l'utilisateur connecté ; tous les repositories filtrent par `user_id`.
- Extension OAuth2 : point d'ajout isolé (ajout d'un `oauth2Login` + mapping compte) sans toucher
  au reste. Non implémenté dans cette itération.

---

## 6. Tests & vérification
- Tests de repository/service Spring Boot sur H2 (register, start/pause/finish, add/remove set,
  substitution, recherche historique).
- Vérification build : `mvn -q verify` doit compiler et passer les tests.
- Vérification manuelle : installer la PWA sur mobile, parcours Start→sets→rest timer→Finish→History.

## 7. Risques / incertitudes
- API Vaadin 25 + thème Aura non confirmée en direct (MCP non connecté) → à valider avant le theming.
- Fidélité pixel du look iOS sur composants Vaadin : theming ciblé, itératif.
- H2 fichier + Flyway : s'assurer du dialecte et des types ENUM (stockés en VARCHAR + CHECK).
