# Tâches — Projet Gestion de Scolarité HEI

Package racine : `com.example.demo` (squelette Poja existant, cf. `notehei.zip`)
Stack : Spring Boot, Spring Security, Thymeleaf, PostgreSQL (Neon), Docker, Testcontainers, Jacoco (80%), Poja (event-driven)

## Convention de package (RSP strict)

Un domaine = un package, avec **une classe = une responsabilité** :

```
com.example.demo.<domaine>/
 ├── <Entite>.java                → entité JPA (données uniquement)
 ├── <Entite>Repository.java      → accès DB uniquement (extends JpaRepository), AUCUNE logique métier
 └── <Entite>Service.java         → logique métier/validation, appelle le repository, AUCUN accès DB direct

com.example.demo.endpoint.rest.controller.<domaine>/
 └── <Entite>Controller.java      → HTTP uniquement (reçoit la requête, appelle le service, renvoie la réponse), AUCUNE logique métier

com.example.demo.endpoint.web.controller.<domaine>/     (nouveau, pour Thymeleaf)
 └── <Entite>ViewController.java  → prépare le Model et retourne le nom de vue, AUCUNE logique métier

src/main/resources/templates/<domaine>/                 (nouveau, pour Thymeleaf)
 └── *.html
```

Règle : un controller ne fait **jamais** de requête SQL/JPA directe, un repository ne contient **jamais** de règle métier (ex. "somme des crédits = 30"), et un service ne construit **jamais** de réponse HTTP ni de vue.

---

## 🔐 Socle commun (à faire ensemble avant de séparer)

- [ ] Entités JPA de base dans leurs packages `com.example.demo.<domaine>` (mapping des 18 tables + `RELEVE_PDF`)
- [ ] Ajout des dépendances `spring-boot-starter-security`, `spring-boot-starter-thymeleaf` dans `build.gradle`
- [ ] `com.example.demo.security` : config Spring Security (3 `UserDetailsService` : student/teacher/admin), filtre JWT ou session selon choix
- [ ] `com.example.demo.endpoint.web.controller.auth` : `LoginViewController`
- [ ] `src/main/resources/templates/layout/` : fragments header/nav/footer par rôle
- [ ] `docker-compose.yml` : app principale + app Poja + LocalStack (S3/EventBridge/SQS)
- [ ] `src/test/java/com/example/demo/conf/` : classe abstraite `AbstractIntegrationTest` (Testcontainers), réutilise `FacadeIT.java` déjà présent
- [ ] Config Jacoco déjà scaffoldée dans `build.gradle` → fixer le seuil à 80% dans `jacocoTestCoverageVerification`

---

## 👤 Valisoa — Structure académique & administration

### `com.example.demo.academicyear` (Année universitaire)
- [ ] `AcademicYear.java` (entité)
- [ ] `AcademicYearRepository.java`
- [ ] `AcademicYearService.java`
- [ ] `endpoint.rest.controller.academicyear.AcademicYearController.java`

### `com.example.demo.cohort` (Promotion)
- [ ] `Cohort.java`, `CohortRepository.java`, `CohortService.java`
- [ ] `endpoint.rest.controller.cohort.CohortController.java`

### `com.example.demo.semester` (Semestre)
- [ ] `Semester.java`, `SemesterRepository.java`
- [ ] `SemesterService.java` — inclut la validation "somme des crédits `course_unit` = 30" (logique métier, pas dans le repository)
- [ ] `endpoint.rest.controller.semester.SemesterController.java`

### `com.example.demo.program` (Parcours)
- [ ] `Program.java`, `ProgramRepository.java`, `ProgramService.java`
- [ ] `endpoint.rest.controller.program.ProgramController.java`

### `com.example.demo.group` (Group + historique parcours)
- [ ] `StudentGroup.java` (entité, nom Java car `group` est un mot réservé), `StudentGroupRepository.java`, `StudentGroupService.java`
- [ ] `GroupProgramHistory.java`, `GroupProgramHistoryRepository.java`, `GroupProgramHistoryService.java` (gestion parcours actif unique)
- [ ] `endpoint.rest.controller.group.GroupController.java`

### `com.example.demo.groupmembership` (Appartenance_Groupe)
- [ ] `GroupMembership.java`, `GroupMembershipRepository.java`
- [ ] `GroupMembershipService.java` — génération matricule `STDyynnn`, immutabilité, logique redoublement
- [ ] `endpoint.rest.controller.groupmembership.GroupMembershipController.java`

### `com.example.demo.courseunit` (UE)
- [ ] `CourseUnit.java`, `CourseUnitRepository.java`, `CourseUnitService.java`
- [ ] `CourseUnitProgram.java`, `CourseUnitProgramRepository.java` (association multi-parcours)
- [ ] `endpoint.rest.controller.courseunit.CourseUnitController.java`

### `com.example.demo.course` (Cours)
- [ ] `Course.java`, `CourseRepository.java`, `CourseService.java`
- [ ] `CourseUnitCourse.java`, `CourseUnitCourseRepository.java`, validation "somme crédits = UE.credits" dans `CourseUnitCourseService.java`
- [ ] `endpoint.rest.controller.course.CourseController.java`

### `com.example.demo.teachingassignment` (Affectation)
- [ ] `TeachingAssignment.java`, `TeachingAssignmentRepository.java`, `TeachingAssignmentService.java`
- [ ] `endpoint.rest.controller.teachingassignment.TeachingAssignmentController.java`

### `com.example.demo.teacher` / `com.example.demo.admin` (comptes)
- [ ] `TeacherService.java` (CRUD, activation/désactivation), `AdminService.java`
- [ ] `endpoint.rest.controller.teacher.TeacherController.java`, `endpoint.rest.controller.admin.AdminController.java`

### Infra partagée (compense la charge Poja de B)
- [ ] `com.example.demo.aws` : config des clients S3/EventBridge/SQS (beans), utilisés par le module Poja de B

### Vues Thymeleaf (`endpoint.web.controller.<domaine>` + `templates/<domaine>/`)
- [ ] Écrans admin : promotions/semestres (`templates/cohort/`, `templates/semester/`)
- [ ] Écrans admin : groupes/parcours (`templates/group/`, `templates/program/`)
- [ ] Écrans admin : UE/cours/affectations (`templates/courseunit/`, `templates/course/`, `templates/teachingassignment/`)
- [ ] Vue organigramme (`templates/structure/organigramme.html`)
- [ ] Fiche étudiant (`templates/student/detail.html`)

### Tests (packages miroirs sous `src/test/java/com/example/demo/<domaine>/`)
- [ ] Tests unitaires des `*Service` (règles métier : crédits, matricule, historisation)
- [ ] Tests d'intégration des `*Repository` (Testcontainers)
- [ ] Tests MockMvc des `*Controller` / `*ViewController` (`@WithMockUser` par rôle)

---

## 📝 Fenohasina — Évaluation, notes & relevés PDF

### `com.example.demo.exam` (Examen)
- [ ] `Exam.java`, `ExamRepository.java`
- [ ] `ExamService.java` — validation "somme pondérations hors RATTRAPAGE = 100%"
- [ ] `endpoint.rest.controller.exam.ExamController.java`

### `com.example.demo.grade` (Note)
- [ ] `Grade.java`, `GradeRepository.java`
- [ ] `GradeService.java` — restriction teacher à ses `TeachingAssignment`, logique `MAX(note_normale, note_rattrapage)`, calcul moyennes cours → UE → semestre → générale
- [ ] `endpoint.rest.controller.grade.GradeController.java`

### `com.example.demo.gradehistory` (Note_Historique)
- [ ] `GradeHistory.java`, `GradeHistoryRepository.java`
- [ ] `GradeHistoryService.java` — log automatique déclenché par `GradeService` (jamais écrit directement par le repository)

### `com.example.demo.transcript` (module Poja — RELEVE_PDF, app séparée)
- [ ] `Transcript.java` (entité `RELEVE_PDF`), `TranscriptRepository.java`
- [ ] `TranscriptService.java` (côté app principale) : déclenche l'événement `TranscriptRequestedEvent` via `com.example.demo.endpoint.event.EventProducer` (déjà présent dans le squelette)
- [ ] `com.example.demo.transcript.pdf.TranscriptPdfGenerator.java` (app Poja) : lecture des notes via `GradeRepository`/`TranscriptRepository`, génération PDFBox
- [ ] Upload S3 via `com.example.demo.file.bucket.BucketComponent` (déjà présent dans le squelette) — appelé par `TranscriptPdfGenerator`, pas par le controller
- [ ] `com.example.demo.handler` : nouveau `TranscriptEventHandler.java` (consommateur d'événement, suit le pattern de `MailboxEventHandler.java` déjà présent)
- [ ] `TranscriptService.java` (retour) : met à jour le statut dans `TranscriptRepository`, déclenche l'email via `com.example.demo.mail.Mailer.java` (déjà présent)
- [ ] `endpoint.rest.controller.transcript.TranscriptController.java`

### Vues Thymeleaf (`endpoint.web.controller.<domaine>` + `templates/<domaine>/`)
- [ ] Formulaire saisie notes (`templates/grade/form.html`)
- [ ] Consultation notes étudiant (`templates/grade/list.html`)
- [ ] Bulletin en ligne + bouton "générer PDF" + statut (`templates/transcript/detail.html`)
- [ ] Bouton "envoyer par email" (dans `templates/transcript/detail.html`, action vers `TranscriptController`)
- [ ] Dashboard notes manquantes (`templates/grade/dashboard.html`)

### Tests (packages miroirs sous `src/test/java/com/example/demo/<domaine>/`)
- [ ] Tests unitaires des `*Service` (calcul moyennes, rattrapage, pondérations)
- [ ] Tests d'intégration des `*Repository` (Testcontainers)
- [ ] Tests d'intégration event-driven (`src/test/java/com/example/demo/conf/EventConf.java`, `BucketConf.java` déjà présents — à réutiliser pour LocalStack)
- [ ] Tests MockMvc des `*Controller` / `*ViewController` (`@WithMockUser` par rôle)

---

## Répartition (rééquilibrée)

- **Valisoa : ~25 tâches** — 9 domaines (`academicyear`, `cohort`, `semester`, `program`, `group`, `groupmembership`, `courseunit`, `course`, `teachingassignment`, `teacher`, `admin`) + config AWS
- **Fenohasina : ~25 tâches** — 4 domaines (`exam`, `grade`, `gradehistory`, `transcript`) mais avec le module event-driven complet (handler, générateur PDF, bucket, mail)

## Dépendances entre A et B (minimisées)

- B dépend uniquement des **entités JPA** de A (`Student`, `Course`, `StudentGroup`, `TeachingAssignment`, `CourseUnit`) → posées dans le socle commun, donc B peut coder avec des données de seed sans attendre les services/controllers de A
- `GradeService` (B) lit `TeachingAssignmentRepository` (A) uniquement en lecture — aucune dépendance sur le controller ou la vue de A
- `TranscriptPdfGenerator` (B) dépend de `com.example.demo.aws` (config clients, livrée par A) — à prioriser tôt par A pour ne pas bloquer B
- Aucun controller ne dépend d'un autre controller — chaque domaine expose son propre `*Controller`/`*ViewController`, pas d'appel croisé entre packages `endpoint.rest.controller.*`