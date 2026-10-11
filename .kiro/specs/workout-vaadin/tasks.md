# Tasks — Workout Log (Vaadin / Spring Boot)

> Chaque tâche référence les exigences couvertes. Cocher au fur et à mesure.
> Avant le theming (tâche 9) et la config PWA (tâche 10), **confirmer l'API Vaadin 25 / Aura via le
> MCP Vaadin**.

## Phase 0 — Scaffold & build
- [ ] 0.1 Créer le projet Maven : Spring Boot 4.1, Vaadin 25, Java 25, H2, Flyway, Spring Security.
      `pom.xml` + `WorkoutApplication` + `application.properties` (H2 fichier, Flyway on). _(R*)_
- [ ] 0.2 Vérifier que l'app démarre à vide (`mvn spring-boot:run`), page Vaadin par défaut OK.

## Phase 1 — Schéma & données initiales
- [ ] 1.1 Migration `V0001__Initial.sql` : toutes les tables du design (section 2). _(R2)_
- [ ] 1.2 Migration `V0002__Seed_program.sql` : programme 4-day Upper/Lower, 6 jours, ~40 exercices
      (depuis `D[]` de `index.html`), + `exercise_info` (muscles + YouTube, best effort). _(R2, R11)_
- [ ] 1.3 Entités Spring Data JDBC + repositories pour chaque agrégat. _(R2)_

## Phase 2 — Auth
- [ ] 2.1 `AppUser` + `UserService` (register, BCrypt) + `UserPreferences` créées à l'inscription. _(R3, R12)_
- [ ] 2.2 `SecurityConfiguration` form login, `LoginView`, `RegisterView`, `CurrentUser`. _(R3)_
- [ ] 2.3 Point d'extension OAuth2 documenté/isolé (pas d'implémentation). _(R3)_

## Phase 3 — Shell & navigation (fidélité UI)
- [ ] 3.1 `MainLayout` : zone contenu + barre d'onglets bas (History / Workout / Guide). _(R1)_
- [ ] 3.2 `HomeView` : progression de cycle + grille de cartes de jours (états). _(R1, R2)_

## Phase 4 — Écran de séance (cœur)
- [ ] 4.1 `DayView` : stats (Duration/Volume/Sets), pill strip, liste d'exercices + tableaux de sets. _(R1, R2)_
- [ ] 4.2 `WorkoutService` + contrôles **Start / Pause / Resume / Finish** avec `ConfirmDialog`. _(R4, R8)_
- [ ] 4.3 Sets : **Add set** / tick (préremplissage « previous ») / **Remove set**. _(R5)_
- [ ] 4.4 `RestTimer` flottant auto au tick (+ bouton +15 / Skip), via polling/Push. _(R1, R7)_
- [ ] 4.5 **Change exercise** : `ExercisePickerDialog` (catalogue + créer à la volée), substitution temporaire. _(R6)_
- [ ] 4.6 `ExerciseInfoDialog` : muscles + lien YouTube au clic sur l'exercice. _(R11)_

## Phase 5 — Historique
- [ ] 5.1 `HistoryView` : cartes de séances finies (durée, volume, sets, détail exercices). _(R1, R2)_
- [ ] 5.2 Recherche **séparée** : plage de dates / type de séance / texte libre. _(R9)_

## Phase 6 — Préférences & Guide
- [ ] 6.1 `PreferencesView` : unité (kg/lbs), repos par défaut, thème ; persistées. _(R7, R12)_
- [ ] 6.2 Appliquer l'unité à l'affichage/saisie du poids & volume partout. _(R12)_
- [ ] 6.3 `GuideView` : principes (double progression, RIR, warm-up) + sélecteur de thème. _(R1)_

## Phase 7 — Création de programme
- [ ] 7.1 `ProgramEditorView` : CRUD programme → jours → `day_exercise` (sets/reps/rest/RIR, ordre). _(R10)_

## Phase 8 — Theming (fidélité look iOS)  ⚠️ confirmer MCP Vaadin d'abord
- [ ] 8.1 `frontend/themes/workout/` : mapper palette iOS sur tokens Aura. _(R1)_
- [ ] 8.2 CSS composants : onglets bas, pills, timer flottant, cartes, tableaux de sets. _(R1)_
- [ ] 8.3 Thèmes auto / light / dark pilotés par préférence + `prefers-color-scheme`. _(R1, R12)_

## Phase 9 — PWA  ⚠️ confirmer MCP Vaadin d'abord
- [ ] 9.1 `@PWA` + manifeste + icônes existantes + offline shell ; installable mobile. _(R13)_

## Phase 10 — Vérification
- [ ] 10.1 Tests service/repo sur H2 (auth, start/pause/finish, add/remove set, substitution, recherche).
- [ ] 10.2 `mvn verify` vert ; parcours manuel complet ; install PWA mobile.
