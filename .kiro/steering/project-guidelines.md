# Project Guidelines — Workout Log (Vaadin / Spring Boot)

Guidelines d'architecture, de structure et de conventions pour ce projet. Elles sont adaptées du
projet de référence **JTAF4** (https://github.com/martinellich/jtaf4) — même design, même structure
en couches, mêmes patterns UI — mais avec la **stack choisie ici** (voir ci-dessous). Quand une
librairie utile de JTAF4 existe pour notre stack, on la réutilise ; les librairies spécifiques à
jOOQ sont remplacées par leurs équivalents Spring Data JDBC.

> La spec détaillée vit dans `.kiro/specs/workout-vaadin/` (requirements, design, tasks).

---

## 1. Stack technique (verrouillée)

| Domaine        | JTAF4 (référence)              | **Ce projet**                              |
|----------------|--------------------------------|--------------------------------------------|
| Langage        | Java 25                        | **Java 25**                                |
| Framework      | Spring Boot 4.1                | **Spring Boot 4.1**                        |
| UI             | Vaadin 25 (Lumo)               | **Vaadin 25 (thème Aura)**                 |
| Persistance    | jOOQ + codegen Testcontainers  | **Spring Data JDBC**                       |
| Migrations     | Flyway                         | **Flyway** (identique)                     |
| Base de données| PostgreSQL                     | **H2 en mode fichier** (local, sans serveur)|
| Sécurité       | Spring Security + JWT + OAuth  | **Spring Security** (form login, extensible OAuth2 Google) |
| Build          | Maven                          | **Maven**                                  |

Librairies conservées de JTAF4 car utiles et compatibles : **Spring Security**, **Flyway**,
**spring-boot-starter-actuator**, **Vaadin Spring Boot starter**, **ArchUnit** (tests
d'architecture), **spring-javaformat** (formatage), **jspecify** (`@Nullable`), **Testcontainers**
seulement si une BD serveur est introduite plus tard (pas nécessaire avec H2).
Librairies **non reprises** (hors périmètre) : OpenPDF, Apache POI, jOOQ et `jooq-spring`, mail.

---

## 2. Structure du projet

Package racine : `com.workout`. Organisation en couches identique à JTAF4
(`configuration` / `domain` / `ui`), avec sous-packages par agrégat métier.

```
com.workout
├── WorkoutApplication.java              @SpringBootApplication + @PWA + AppShellConfigurator
├── configuration
│   └── security/                        SecurityConfiguration, CurrentUser, Role
├── domain                               entités + repositories + services (logique métier)
│   ├── user/        catalog/  program/  session/  preferences/
└── ui                                   vues Vaadin = routes
    ├── MainLayout.java                  shell (@Layout)
    ├── <Feature>View.java               une vue par écran, @Route
    ├── component/                       composants réutilisables (RestTimer, DayPillStrip…)
    └── dialog/                          ConfirmDialog, EditDialog, pickers…
```

Règles de dépendances entre couches (reprises de JTAF4, à faire respecter par un test ArchUnit) :
- `ui` ne doit être accédée par **aucune** autre couche.
- `domain` ne peut être accédée que par `ui`.
- Pas de cycles entre packages (`slices().should().beFreeOfCycles()`).

---

## 3. Conventions de persistance (Spring Data JDBC)

> Remplace le pattern jOOQ `*DAO extends JooqDAO<...>` de JTAF4.

- Un **agrégat** par racine métier ; les entités sont des `record` ou classes immuables annotées
  `@Table`, champs `@Id`, `@Column`, collections possédées via `@MappedCollection`.
- Un **repository** par agrégat : interface `extends CrudRepository<T, Long>` (ou
  `ListCrudRepository`), requêtes custom via `@Query` (SQL H2). Les repositories jouent le rôle des
  `*DAO` de JTAF4.
- La **logique métier** vit dans des `@Service` du package `domain` (ex. `WorkoutService`,
  `ProgramService`, `UserService`), équivalents des services JTAF4 (`UserService`, etc.).
- Toute donnée utilisateur est filtrée par `user_id` ; jamais de requête inter-utilisateurs.
- Schéma et données initiales **uniquement via migrations Flyway** (`src/main/resources/db/migration/`),
  `V0001__Initial.sql`, `V0002__Seed_program.sql`, etc. Jamais de `ddl-auto` destructif.
- Enums stockés en `VARCHAR` (+ `CHECK` si pertinent), pas en type natif.

---

## 4. Conventions UI Vaadin (reprises de JTAF4)

- Une **vue = une route** : classe `@Route`, nom en `XxxView`. Titre dynamique via
  `implements HasDynamicTitle` + `getPageTitle()`.
- **Shell** unique `MainLayout` annoté `@Layout`. JTAF4 utilise un `AppLayout` avec drawer ; ici le
  shell reproduit la **barre d'onglets bas** (History / Workout / Guide) de l'app d'origine, mais la
  logique (navigation, titre, visibilité selon login) suit le même modèle.
- **Services injectés** par constructeur, marqués `transient` (sérialisation Vaadin), comme dans
  JTAF4 (`private final transient XxxService xxx;`).
- **Dialogs réutilisables** dans `ui/dialog` :
  - `ConfirmDialog` générique (header/texte/confirm/cancel + events) — **à utiliser pour toute
    action sensible** (start/finish séance, suppression). Calqué sur `ch.jtaf.ui.dialog.ConfirmDialog`.
  - un `EditDialog<T>` abstrait (binder + Save/Cancel + fullscreen toggle) pour les formulaires
    d'édition (programme, exercice), calqué sur `ch.jtaf.ui.dialog.EditDialog`.
- **Styling** : privilégier `LumoUtility.*` (classes utilitaires) pour l'agencement, comme JTAF4.
  Le look custom (iOS) passe par le **theming officiel** (`frontend/themes/workout/` : `theme.json`
  + `styles.css`) en surchargeant les **design tokens Aura**. **Interdit** : hacker le Shadow DOM.
- **i18n / libellés** : UI en **anglais**. Utiliser `getTranslation("Key")` + fichiers
  `messages_en.properties` (comme `messages_*.properties` de JTAF4), même si une seule langue au
  départ — ne pas coder les libellés en dur dispersés.
- **IDs stables** sur les composants interactifs (`setId(...)`) pour la testabilité, comme JTAF4
  (`edit-save`, `logout`, `remove-results`…).
- `@Nullable` (jspecify) sur les champs/paramètres nullable ; éviter les NPE silencieuses.

---

## 5. Sécurité

- `SecurityConfiguration` : form login (`LoginView`), routes `login`/`register` publiques
  (`@AnonymousAllowed`), reste protégé par rôle (`@RolesAllowed`), comme JTAF4.
- Mots de passe hachés **BCrypt**. Accès à l'utilisateur courant via un `CurrentUser` /
  `SecurityContext` (équivalent du `SecurityContext` de JTAF4).
- **Extensibilité OAuth2 (Google)** : garder le branchement du provider isolé pour l'ajouter sans
  refonte. Préparé, pas implémenté dans l'itération courante.
- Ne jamais committer de secret ; secrets via variables d'environnement
  (`${VAR:default}` dans `application.properties`), comme JTAF4.

---

## 6. PWA

- Rendre l'app installable via `@PWA(name=..., shortName="Workout")` sur l'application + un
  `AppShellConfigurator` (comme `Jtaf4Application`). Réutiliser les **icônes existantes**
  (`icon-*.png`) et le thème. Offline shell géré par Vaadin.

---

## 7. Qualité, tests & build

- **Formatage** : `spring-javaformat` (plugin Maven, phase `validate`). Le code doit passer le
  format avant commit ; respecter le style (tabs, imports ordonnés) comme JTAF4.
- **Tests d'architecture** : un `ArchitectureTest` ArchUnit vérifie les couches et l'absence de
  cycles (cf. section 2) — reprendre le test de JTAF4 en adaptant les packages (`com.workout`).
- **Tests** : services/repositories testés sur H2 (register, start/pause/finish, add/remove set,
  substitution d'exercice, recherche d'historique). Vues testables via IDs stables.
- **Config** : `application.version=@project.version@`, `server.port=${PORT:8080}`,
  `vaadin.allowed-packages=com.vaadin,org.vaadin,com.workout`,
  `vaadin.launch-browser=true` en dev. BD H2 fichier : `spring.datasource.url=jdbc:h2:file:...`.
- **Vérification** : `mvn verify` doit compiler, passer ArchUnit + tests, et builder le frontend
  Vaadin (`vaadin:build-frontend`) avant de considérer une tâche terminée.

---

## 8. Règles de contribution

- Suivre la spec `.kiro/specs/workout-vaadin/` ; cocher les tâches au fur et à mesure.
- Avant tout travail de **theming** ou de **config PWA**, confirmer l'API exacte de Vaadin 25 /
  thème Aura via le **MCP Vaadin** (ou la doc officielle si le MCP est indisponible).
- Rester dans le périmètre : auth simple, pas de multi-organisation, pas de PDF/Excel/mail.
- Réutiliser un composant/dialog existant avant d'en créer un nouveau.
