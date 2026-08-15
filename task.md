# Tâches — Projet Gestion de Scolarité HEI

Package racine : `com.example.demo` (squelette Poja existant, cf. `notehei.zip`)
Stack : Spring Boot, Spring Security (JWT), Thymeleaf, PostgreSQL (Neon), JPA, Docker, Testcontainers, Jacoco (80%), Poja (event-driven)

## Convention de package (RSP strict + validator)

Un domaine = un package, avec **une classe = une responsabilité** :

```
com.example.demo.<domaine>/
 ├── <Entite>.java                → entité JPA (données uniquement)
 ├── <Entite>Repository.java      → accès DB uniquement (extends JpaRepository), AUCUNE logique métier
 ├── <Entite>Service.java         → orchestration : appelle le repository + le validator, AUCUN accès DB direct, AUCUNE règle de validation écrite en dur
 └── validator/
      └── <Entite>Validator.java  → règles métier du domaine (ex. "somme des crédits = 30", "ce teacher a-t-il ce TeachingAssignment ?"), lève une exception métier si invalide

com.example.demo.endpoint.rest.controller.<domaine>/
 └── <Entite>Controller.java      → HTTP uniquement + annotation @PreAuthorize("hasRole('...')"), AUCUNE logique métier

com.example.demo.endpoint.web.controller.<domaine>/     (Thymeleaf)
 └── <Entite>ViewController.java  → prépare le Model et retourne le nom de vue, AUCUNE logique métier

src/main/resources/templates/<domaine>/
 └── *.html
```

**Important — Security ≠ Validator, deux responsabilités différentes :**
- `com.example.demo.security` répond à *"qui es-tu et as-tu le bon rôle ?"* (authentification JWT, `@PreAuthorize("hasRole('TEACHER')")`). Ne contient **aucune** règle métier de domaine.
- `<domaine>/validator/` répond à *"as-tu le droit de faire CETTE action précise sur CETTE donnée ?"* (ex. un teacher authentifié avec le rôle TEACHER a-t-il réellement un `TeachingAssignment` pour ce cours ?). Ne contient **aucun** code JWT/session.

Flux type pour une note :
```
JWT → JwtAuthenticationFilter → Authentication (userId, ROLE_TEACHER)
   → GradeController (@PreAuthorize hasRole TEACHER)
   → GradeService
   → GradeValidator (vérifie le TeachingAssignment via TeachingAssignmentRepository)
   → autorisé → GradeRepository.save(...)
```

---

## 🔐 Socle commun (à faire ensemble avant de séparer)

- [ ] Ajout des dépendances `spring-boot-starter-data-jpa`, `spring-boot-starter-security`, `spring-boot-starter-thymeleaf`, `jjwt` (ou équivalent) dans `build.gradle`
- [ ] `docker-compose.yml` : app principale + app Poja + LocalStack (S3/EventBridge/SQS)
- [ ] `src/test/java/com/example/demo/conf/PostgresConf.java` : Testcontainers Postgres, branché dans `FacadeIT.java` (déjà fait sur `exam`, à garder comme référence commune)
- [ ] Config Jacoco : seuil fixé à 80% dans `jacocoTestCoverageVerification`
- [ ] Convention commune : noms des rôles (`ROLE_STUDENT`, `ROLE_TEACHER`, `ROLE_ADMIN`) figée dès le départ — Valisoa les implémente, mais A et B en ont besoin dès leurs premiers `@PreAuthorize`

---

## 🔑 Valisoa — Structure académique, administration & Spring Security

### `com.example.demo.security` (authentification & JWT — nouveau, priorité haute)
- [ ] `SecurityConfig.java` — chaîne de filtres, routes publiques (`/login`, `/ping`) vs protégées, séparation routes REST (`/api/**` → 401 JSON) vs routes Thymeleaf (→ redirection `/login`)
- [ ] `SecurityUser.java` — implémentation `UserDetails` qui enveloppe Student/Teacher/Admin + son rôle
- [ ] `StudentUserDetailsService.java`, `TeacherUserDetailsService.java`, `AdminUserDetailsService.java` — un par type de compte
- [ ] `JwtService.java` — génération + validation du token (signature, expiration)
- [ ] `JwtAuthenticationFilter.java` — extrait le token de la requête, peuple le `SecurityContext`
- [ ] `PasswordEncoderConfig.java` — bean `BCryptPasswordEncoder`, injecté par `StudentService`/`TeacherService`/`AdminService` lors de la création de compte (mot de passe jamais stocké en clair)
- [ ] `AuthenticationManagerConfig.java` — bean `AuthenticationManager`
- [ ] `RestAuthenticationEntryPoint.java` — 401 JSON propre sur `/api/**` (pas de redirection HTML)
- [ ] `CustomAccessDeniedHandler.java` — 403 propre (JSON sur API, page Thymeleaf sur le web)
- [ ] Configuration CSRF : activé pour les formulaires Thymeleaf (token caché dans chaque `<form>`), désactivé/adapté sur les routes API stateless JWT
- [ ] Tests Security : login réussi/échoué, token expiré/invalide, accès refusé par rôle (`@WithMockUser`, MockMvc)

### Auth Web (dans le même périmètre que Security)
- [ ] `endpoint.web.controller.auth.LoginViewController.java`
- [ ] `templates/auth/login.html` (+ gestion des messages d'erreur d'authentification)
- [ ] `templates/layout/` : fragments header/nav/footer par rôle (visibilité des menus selon `ROLE_*`)

### `com.example.demo.academicyear` (Année universitaire)
- [ ] `AcademicYear.java`, `AcademicYearRepository.java`, `AcademicYearService.java`
- [ ] `endpoint.rest.controller.academicyear.AcademicYearController.java`

### `com.example.demo.cohort` (Promotion)
- [ ] `Cohort.java`, `CohortRepository.java`, `CohortService.java`
- [ ] `endpoint.rest.controller.cohort.CohortController.java`

### `com.example.demo.semester` (Semestre)
- [ ] `Semester.java`, `SemesterRepository.java`, `SemesterService.java`
- [ ] `semester/validator/SemesterCreditValidator.java` — règle "somme des crédits `course_unit` = 30"
- [ ] `endpoint.rest.controller.semester.SemesterController.java`

### `com.example.demo.program` (Parcours)
- [ ] `Program.java`, `ProgramRepository.java`, `ProgramService.java`
- [ ] `endpoint.rest.controller.program.ProgramController.java`

### `com.example.demo.group` (Group + historique parcours)
- [ ] `StudentGroup.java`, `StudentGroupRepository.java`, `StudentGroupService.java`
- [ ] `GroupProgramHistory.java`, `GroupProgramHistoryRepository.java`, `GroupProgramHistoryService.java`
- [ ] `group/validator/GroupProgramHistoryValidator.java` — un seul parcours actif à la fois par groupe
- [ ] `endpoint.rest.controller.group.GroupController.java`

### `com.example.demo.groupmembership` (Appartenance_Groupe)
- [ ] `GroupMembership.java`, `GroupMembershipRepository.java`, `GroupMembershipService.java`
- [ ] `groupmembership/validator/GroupMembershipValidator.java` — matricule `STDyynnn`, immutabilité, un seul groupe actif, logique redoublement
- [ ] `endpoint.rest.controller.groupmembership.GroupMembershipController.java`

### `com.example.demo.courseunit` (UE)
- [ ] `CourseUnit.java`, `CourseUnitRepository.java`, `CourseUnitService.java`
- [ ] `CourseUnitProgram.java`, `CourseUnitProgramRepository.java`
- [ ] `courseunit/validator/CourseUnitValidator.java` — au moins un parcours, au moins un cours rattaché
- [ ] `endpoint.rest.controller.courseunit.CourseUnitController.java`

### `com.example.demo.course` (Cours)
- [ ] `Course.java`, `CourseRepository.java`, `CourseService.java`
- [ ] `CourseUnitCourse.java`, `CourseUnitCourseRepository.java`
- [ ] `course/validator/CourseUnitCourseValidator.java` — somme des crédits `UE_COURSE` = `UE.credits`
- [ ] `endpoint.rest.controller.course.CourseController.java`

### `com.example.demo.teachingassignment` (Affectation)
- [ ] `TeachingAssignment.java`, `TeachingAssignmentRepository.java`, `TeachingAssignmentService.java`
- [ ] `endpoint.rest.controller.teachingassignment.TeachingAssignmentController.java`

### `com.example.demo.teacher` / `com.example.demo.admin` (comptes — métier uniquement, PAS l'authentification)
- [ ] `Teacher.java`, `TeacherRepository.java`, `TeacherService.java` (CRUD, activation/désactivation, encode le mot de passe via le `PasswordEncoder` du module security)
- [ ] `Admin.java`, `AdminRepository.java`, `AdminService.java` (idem)
- [ ] `endpoint.rest.controller.teacher.TeacherController.java`, `endpoint.rest.controller.admin.AdminController.java`

### Vues Thymeleaf
- [ ] Écrans admin : promotions/semestres (`templates/cohort/`, `templates/semester/`)
- [ ] Écrans admin : groupes/parcours (`templates/group/`, `templates/program/`)
- [ ] Écrans admin : UE/cours/affectations (`templates/courseunit/`, `templates/course/`, `templates/teachingassignment/`)
- [ ] Vue organigramme (`templates/structure/organigramme.html`)
- [ ] Fiche étudiant (`templates/student/detail.html`)

### Tests
- [ ] Tests unitaires des `*Service` et `*Validator` (règles métier : crédits, matricule, historisation)
- [ ] Tests d'intégration des `*Repository` (Testcontainers)
- [ ] Tests MockMvc des `*Controller` / `*ViewController` (`@WithMockUser` par rôle)
- [ ] Tests Security (login, JWT, accès par rôle — cf. section Security ci-dessus)

---

## 📝 Fenohasina — Évaluation, notes & relevés PDF

### `com.example.demo.exam` (Examen)
- [ ] `Exam.java`, `ExamRepository.java`, `ExamService.java` ✅ déjà fait
- [ ] `exam/validator/ExamValidator.java` — extraire la règle "somme pondérations hors RATTRAPAGE = 100%" hors du service (actuellement dans `ExamService`, à déplacer pour respecter la nouvelle convention)
- [ ] `endpoint.rest.controller.exam.ExamController.java` ✅ déjà fait — ajouter `@PreAuthorize("hasAnyRole('TEACHER','ADMIN')")`

### `com.example.demo.grade` (Note)
- [ ] `Grade.java`, `GradeRepository.java`, `GradeService.java`
- [ ] `grade/validator/GradeValidator.java` — vérifie via `TeachingAssignmentRepository` (lecture seule, domaine de Valisoa) que le teacher authentifié possède bien l'affectation sur ce cours ; logique `MAX(note_normale, note_rattrapage)` ; calcul moyennes cours → UE → semestre → générale
- [ ] `endpoint.rest.controller.grade.GradeController.java` — `@PreAuthorize` par rôle, la vérification fine de propriété reste dans `GradeValidator`, pas dans le controller

### `com.example.demo.gradehistory` (Note_Historique)
- [ ] `GradeHistory.java`, `GradeHistoryRepository.java`, `GradeHistoryService.java` — log auto déclenché par `GradeService`, jamais écrit directement par le repository
- [ ] `gradehistory/validator/GradeHistoryValidator.java` — exactement un des deux (teacher_id/admin_id) renseigné

### `com.example.demo.transcript` (module Poja — RELEVE_PDF, app séparée)
- [ ] `Transcript.java` (entité `RELEVE_PDF`), `TranscriptRepository.java`
- [ ] `TranscriptService.java` (app principale) : déclenche `TranscriptRequestedEvent` via `com.example.demo.endpoint.event.EventProducer`
- [ ] `transcript/validator/TranscriptValidator.java` — l'étudiant/admin demandeur a-t-il le droit de générer ce relevé (propriétaire du dossier ou rôle admin) ?
- [ ] `com.example.demo.transcript.pdf.TranscriptPdfGenerator.java` (app Poja) : lecture des notes via `GradeRepository`/`TranscriptRepository`, génération PDFBox
- [ ] Upload S3 via `com.example.demo.file.bucket.BucketComponent` — appelé par `TranscriptPdfGenerator`, pas par le controller
- [ ] `com.example.demo.handler.TranscriptEventHandler.java` — consommateur d'événement (pattern `MailboxEventHandler.java`)
- [ ] `TranscriptService.java` (retour) : met à jour le statut, déclenche l'email via `com.example.demo.mail.Mailer.java`
- [ ] `endpoint.rest.controller.transcript.TranscriptController.java`

### Config AWS (rapatriée depuis Valisoa pour compenser sa charge Security)
- [ ] `com.example.demo.aws` : config des clients S3/EventBridge/SQS (beans), utilisés par le module Poja `transcript`

### Vues Thymeleaf
- [ ] Formulaire saisie notes (`templates/grade/form.html`)
- [ ] Consultation notes étudiant (`templates/grade/list.html`)
- [ ] Bulletin en ligne + bouton "générer PDF" + statut (`templates/transcript/detail.html`)
- [ ] Bouton "envoyer par email" (`templates/transcript/detail.html`)
- [ ] Dashboard notes manquantes (`templates/grade/dashboard.html`)

### Tests
- [ ] Tests unitaires des `*Service` et `*Validator` (calcul moyennes, rattrapage, pondérations, propriété du `TeachingAssignment`)
- [ ] Tests d'intégration des `*Repository` (Testcontainers)
- [ ] Tests d'intégration event-driven (`EventConf.java`, `BucketConf.java` — à réutiliser pour LocalStack)
- [ ] Tests MockMvc des `*Controller` / `*ViewController` (`@WithMockUser` par rôle)

---

## Répartition (mise à jour)

- **Valisoa : ~38 tâches** — 9 domaines structure/administration + module Security/JWT complet + Auth Web
- **Fenohasina : ~24 tâches** — 4 domaines (`exam`, `grade`, `gradehistory`, `transcript`) + module event-driven complet + config AWS (récupérée pour rééquilibrer)

⚠️ Le déséquilibre est assumé : Valisoa porte volontairement Security en plus de sa structure. Pour compenser un peu, la config AWS repasse côté Fenohasina (elle était chez Valisoa dans la version précédente). Si ça reste trop lourd pour Valisoa au fil du sprint, la partie "Auth Web" (LoginViewController + templates login) est la plus facile à redistribuer sans casser le découpage Security.

## Dépendances entre les deux (minimisées)

- Les noms de rôles (`ROLE_STUDENT`, `ROLE_TEACHER`, `ROLE_ADMIN`) sont figés dans le socle commun dès le départ → Fenohasina peut écrire ses `@PreAuthorize` sans attendre l'implémentation complète de Security, seulement la convention de nommage
- `GradeValidator` (Fenohasina) lit `TeachingAssignmentRepository` (Valisoa) en lecture seule → aucune dépendance sur les controllers/vues de Valisoa
- `TranscriptPdfGenerator` (Fenohasina) dépend de `com.example.demo.aws`, désormais dans son propre périmètre → plus de blocage croisé
- `TeacherService`/`AdminService` (Valisoa) dépendent du `PasswordEncoder` exposé par `com.example.demo.security` (Valisoa aussi) → dépendance interne à la même personne, pas de blocage inter-personnes
- Aucun controller ne dépend d'un autre controller ; chaque domaine expose son propre `*Controller`/`*ViewController`
- Security ne contient jamais de règle métier de domaine ; les `*Validator` ne contiennent jamais de code JWT/session — à surveiller en review de code pour ne pas finir avec un `SecurityValidator` fourre-tout