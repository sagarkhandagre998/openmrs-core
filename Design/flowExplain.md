🏥 OpenMRS Core — Complete Architecture & Flow Analysis

## 📦 Project Structure (Maven Multi-Module)

The project is a **Maven multi-module** project (`pom.xml` at root) with these modules:

| Module | Purpose |
|---|---|
| `api` | Core business logic, domain model, services, DAOs |
| `web` | Web layer — Servlets, Filters, Startup Listener |
| `webapp` | WAR packaging — assembles everything into a deployable app |
| `liquibase` | Database migration tooling |
| `test` | Test utilities |
| `test-suite` | Integration/system tests |
| `tools` | Build-time tools (checkstyle, etc.) |
| `bom` | Bill of Materials (dependency version management) |

---

## 🗄️ DATABASE LAYER

### 1. Engine Support
The `docker-compose.yml` shows support for **MySQL**, **MariaDB**, and **PostgreSQL**. The database engine is configured at runtime via `runtime.properties`.

### 2. EAV (Entity-Attribute-Value) Data Model
OpenMRS uses a famous **EAV model** for clinical observations. This is the heart of the OpenMRS Data Model:

```openmrs-core/api/src/main/java/org/openmrs/Obs.java#L77-100
@Audited
public class Obs extends BaseFormRecordableOpenmrsData {
    ...
    protected Integer obsId;
    protected Concept concept;       // the "Attribute" (what was measured)
    protected Date obsDatetime;
    ...
```

Every clinical data point (blood pressure, weight, diagnosis, etc.) is stored as an `Obs` (Observation) row with:
- **Entity** → the `Patient`
- **Attribute** → the `Concept` (e.g., "Blood Pressure")
- **Value** → the actual value (numeric, text, coded, datetime, etc.)

### 3. Liquibase — Schema Management
All schema creation and migrations are handled by **Liquibase**:

```openmrs-core/api/src/main/resources/liquibase-update-to-latest.xml#L1-5
(references chain of changelogs for every version upgrade)
```

Key files:
- `liquibase-schema-only.xml` — fresh schema creation
- `liquibase-core-data.xml` — seed/reference data
- `liquibase-update-to-latest.xml` — upgrade chain
- `liquibase/scripts/` — shell scripts to create/drop the database

### 4. Hibernate ORM — Mapping
Hibernate is configured in `hibernate.cfg.xml`. All domain objects are mapped either via:
- **`.hbm.xml` files** (legacy XML mapping) for complex cases like `Concept`, `Obs`, `Patient`, `Person`
- **JPA annotations** (`@Entity`, `@Table`, `@Column`, `@ManyToOne`, etc.) for newer entities like `Encounter`, `Visit`, `Condition`

```openmrs-core/api/src/main/java/org/openmrs/Encounter.java#L55-80
@Entity
@Table(name = "encounter")
@BatchSize(size = 25)
@Audited
public class Encounter extends BaseChangeableOpenmrsData {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "encounter_id")
    private Integer encounterId;
    ...
    @ManyToOne(optional = false)
    @JoinColumn(name = "patient_id")
    private Patient patient;
```

### 5. Hibernate Envers — Audit Trail
Every entity annotated with `@Audited` (which is nearly all of them) gets a full **change history** tracked by **Hibernate Envers**. This writes to audit tables (`*_aud`) automatically.

```openmrs-core/api/src/main/java/org/openmrs/api/db/hibernate/envers/OpenmrsRevisionEntity.java
```

### 6. Hibernate Search (Full-Text Search)
`Concept` names and `Patient` names support **full-text search** via Hibernate Search, backed by either **Lucene** (local) or **Elasticsearch** (distributed). The `docker-compose.es.yml` shows the Elasticsearch setup.

---

## ⚙️ BACKEND LAYER (Spring + Hibernate)

### Overall Backend Architecture Pattern

```
REST/FHIR Request
        ↓
  Web Filter Chain  (OpenmrsFilter, OpenSessionInViewFilter, etc.)
        ↓
  Spring DispatcherServlet
        ↓
  AOP Interceptors  (AuthorizationAdvice → RequiredDataAdvice → LoggingAdvice)
        ↓
  Service Interface  (e.g., PatientService)
        ↓
  Service Implementation  (e.g., PatientServiceImpl)  @Transactional
        ↓
  DAO Interface  (e.g., PatientDAO)
        ↓
  Hibernate DAO Implementation  (e.g., HibernatePatientDAO)  @Repository
        ↓
  Hibernate SessionFactory → Database
```

---

### 1. Domain Model (`org.openmrs.*`)

The domain model is a rich object hierarchy. Every object ultimately extends:

```
OpenmrsObject (interface: has UUID, id)
    └── BaseOpenmrsObject  (abstract: uuid, id)
            ├── BaseOpenmrsData  (abstract: + creator, dateCreated, voided, voidedBy...)
            │       ├── Patient  extends Person
            │       ├── Encounter
            │       ├── Obs
            │       ├── Order / DrugOrder / TestOrder
            │       ├── Visit
            │       └── ... (all patient-centric data)
            └── BaseOpenmrsMetadata  (abstract: + name, description, retired...)
                    ├── Concept
                    ├── Location
                    ├── EncounterType
                    ├── Drug
                    └── ... (all reference/config data)
```

**Key distinction:**
- `OpenmrsData` = patient-specific records (can be **voided** — soft delete for clinical records)
- `OpenmrsMetadata` = reference/config data (can be **retired** — soft disable for config)

The `BaseOpenmrsData` carries these audit fields automatically:
```openmrs-core/api/src/main/java/org/openmrs/BaseOpenmrsData.java#L33-60
protected User creator;
private Date dateCreated;
private User changedBy;
private Date dateChanged;
private Boolean voided = Boolean.FALSE;
private Date dateVoided;
private User voidedBy;
private String voidReason;
```

**Core clinical data flow:**

```
Person  ──(extends)──►  Patient
                             │
                    has many VisitAttribute
                             │
                         Visit  (a time period: e.g. "Outpatient visit Jan 10")
                             │
                    has many Encounter  (a single interaction: "nurse triage")
                             │
                    has many Obs  (individual data point: "weight = 72kg")
                    has many Order  (clinical order: "prescribe Amoxicillin")
                             │
                        DrugOrder / TestOrder / ReferralOrder / ServiceOrder
```

### 2. Service Layer (`org.openmrs.api`)

Every domain area has a **Service interface** + **Impl class**:

| Service Interface | Impl Class | Responsibility |
|---|---|---|
| `PatientService` | `PatientServiceImpl` | CRUD for patients, identifiers, allergies |
| `EncounterService` | `EncounterServiceImpl` | Encounters, encounter types, providers |
| `ObsService` | `ObsServiceImpl` | Observations (clinical data points) |
| `OrderService` | `OrderServiceImpl` | Clinical orders (drugs, tests, referrals) |
| `ConceptService` | `ConceptServiceImpl` | Concept dictionary management |
| `UserService` | `UserServiceImpl` | User accounts, roles, privileges |
| `PersonService` | `PersonServiceImpl` | Person demographics |
| `LocationService` | `LocationServiceImpl` | Clinic/hospital locations |
| `VisitService` | `VisitServiceImpl` | Patient visits |
| `ProgramWorkflowService` | `ProgramWorkflowServiceImpl` | Patient programs & care pathways |
| `ConditionService` | `ConditionServiceImpl` | Clinical conditions/problems list |
| `DiagnosisService` | `DiagnosisServiceImpl` | Diagnoses |
| `SchedulerService` | timer-based impl | Background task scheduling |
| `AdministrationService` | `AdministrationServiceImpl` | Global properties, settings |
| `FormService` | `FormServiceImpl` | Form definitions |
| `ProviderService` | `ProviderServiceImpl` | Healthcare provider management |

Services are accessed **only** through the static `Context` class:

```openmrs-core/api/src/main/java/org/openmrs/api/context/Context.java#L433-435
public static PatientService getPatientService() {
    return getServiceContext().getPatientService();
}
```

### 3. The Context System (`org.openmrs.api.context`)

This is the central access point for everything:

| Class | Role |
|---|---|
| `Context` | **Static facade** — the single entry point to all services, the current user, sessions |
| `ServiceContext` | **Singleton** — holds Spring-wired service beans, manages Spring `ApplicationContext` |
| `UserContext` | **Per-thread** (stored in `ThreadLocal`) — holds the authenticated user, locale, proxy privileges |
| `ContextDAO` / `HibernateContextDAO` | Handles login/logout, session management |

The `Context` class stores one `UserContext` **per thread** via `ThreadLocal<UserContext>`. This means every HTTP request thread has its own user session.

### 4. AOP Layer (`org.openmrs.aop`) — Cross-Cutting Concerns

Every service method call passes through **3 AOP interceptors** (Spring AOP, applied in order):

**① `AuthorizationAdvice`** (`MethodBeforeAdvice`)
- Fires **before** every service method
- Reads `@Authorized({"privilege1", "privilege2"})` annotations
- Checks `Context.getUserContext().hasPrivilege()`
- Throws `APIAuthenticationException` if denied

```openmrs-core/api/src/main/java/org/openmrs/aop/AuthorizationAdvice.java#L52-70
public void before(Method method, Object[] args, Object target) throws Throwable {
    if (Daemon.isDaemonThread()) return; // background tasks bypass auth
    ...
    for (String privilege : privileges) {
        if (!Context.hasPrivilege(privilege)) throwUnauthorized(...);
    }
}
```

**② `RequiredDataAdvice`** (`MethodBeforeAdvice`)
- Fires before `save*`, `void*`, `retire*`, `unvoid*`, `unretire*` methods
- Automatically sets `creator`, `dateCreated`, `voidedBy`, `dateVoided`, etc.
- Delegates to **handler classes** (e.g., `SaveHandler`, `VoidHandler`, `RetireHandler`)
- Recurses into child collections (e.g., saves the patient's name/address too)

**③ `LoggingAdvice`** — logs method entry/exit for debug purposes

**Hibernate-level interceptors** (additional layer inside the ORM):
- `AuditableInterceptor` — fills `creator`/`dateCreated` on INSERT, `changedBy`/`dateChanged` on UPDATE
- `ImmutableObsInterceptor` — prevents direct edits to `Obs`; forces void-and-replace pattern
- `ImmutableOrderInterceptor` — same for `Order`

### 5. DAO Layer (`org.openmrs.api.db` + `org.openmrs.api.db.hibernate`)

Each domain has:
- **DAO Interface** (e.g., `PatientDAO`) — defines DB operations
- **Hibernate Implementation** (e.g., `HibernatePatientDAO`) — uses JPA Criteria API + HQL

```openmrs-core/api/src/main/java/org/openmrs/api/db/hibernate/HibernatePatientDAO.java#L67-80
@Repository("patientDAO")
public class HibernatePatientDAO implements PatientDAO {
    private final SessionFactory sessionFactory;
    ...
```

The `DbSessionFactory` wraps Hibernate's `SessionFactory` and is injected into all DAOs. Sessions are managed by **Spring's `@Transactional`** on the service impl classes.

### 6. Module System (`org.openmrs.module`)

This is what makes OpenMRS extensible:
- Modules are **`.omod` files** (essentially JARs with metadata)
- `ModuleFactory` loads, starts, and stops modules at runtime
- `ModuleClassLoader` — each module gets its own class loader (isolation)
- Modules can: add new Services, extend existing ones via AOP `Advisor`, add Liquibase changelogs for new tables, add REST endpoints, etc.
- `ModuleActivator` — lifecycle hooks (`started()`, `stopped()`, `contextRefreshed()`)

### 7. Web Layer (`org.openmrs.web`)

| Class | Role |
|---|---|
| `Listener` | `ServletContextListener` — bootstraps the entire app on startup; starts Spring, runs DB migrations, loads modules |
| `OpenmrsFilter` | Master servlet filter — sets user context per request |
| `OpenSessionInViewFilter` | Keeps Hibernate session open for the full request (lazy loading works) |
| `StartupFilter` | Shows the setup wizard on first run (before app is configured) |
| `InitializationFilter` | Handles database initialization wizard |
| `UpdateFilter` | Handles DB upgrade prompts |
| `DispatcherServlet` | Spring MVC dispatcher — routes to REST/FHIR controllers |
| `GZIPFilter` | Compresses responses |

**Startup sequence:**

```
Tomcat starts WAR
    ↓
Listener.contextInitialized()
    ├── Read runtime.properties
    ├── Start Spring ApplicationContext (loads applicationContext-service.xml)
    │       ├── Creates Hibernate SessionFactory
    │       ├── Wires all Service beans
    │       └── Wires all DAO beans
    ├── Context.startup() → run Liquibase migrations
    ├── Load & start all .omod modules (ModuleFactory)
    └── App is ready
```

### 8. Security Model

- **Roles** → contain **Privileges**
- **Users** → assigned **Roles** (hierarchical — roles can inherit from parent roles)
- Special built-in roles: `Anonymous`, `Authenticated`, `System Developer`
- `@Authorized({"privilege"})` annotation on every service method
- Privilege checked at AOP layer before method executes
- `Daemon` thread bypasses auth (used for scheduled tasks)

---

## 🔄 Complete Request Flow — Example: "Save a new Observation"

```
1. HTTP POST /openmrs/ws/rest/v1/obs
        ↓
2. OpenmrsFilter  — opens UserContext for thread
        ↓
3. OpenSessionInViewFilter  — opens Hibernate session
        ↓
4. Spring DispatcherServlet  — routes to REST controller
        ↓
5. REST Controller calls:  Context.getObsService().saveObs(obs, reason)
        ↓
6. AOP: AuthorizationAdvice.before()
   → checks @Authorized({"Add Observations"})
   → throws 403 if user lacks privilege
        ↓
7. AOP: RequiredDataAdvice.before()
   → calls SaveHandler → sets obs.creator, obs.dateCreated, obs.uuid
        ↓
8. ObsServiceImpl.saveObs()  [@Transactional]
   → validates obs (ObsValidator)
   → voids old obs, creates new (immutable obs pattern)
   → calls obsDAO.saveObs(obs)
        ↓
9. HibernateObsDAO.saveObs()
   → session.saveOrUpdate(obs)
        ↓
10. Hibernate fires AuditableInterceptor
    → sets creator/dateCreated if null
        ↓
11. Hibernate Envers writes to obs_aud audit table
        ↓
12. @Transactional commits → SQL INSERT to obs table
        ↓
13. Response returned through filter chain
        ↓
14. OpenSessionInViewFilter closes Hibernate session
15. OpenmrsFilter clears UserContext from thread
```

---

## 🗂️ Key Design Patterns Used

| Pattern | Where |
|---|---|
| **Service Layer** | `*Service` interfaces + `*ServiceImpl` classes |
| **Repository / DAO** | `*DAO` interfaces + `Hibernate*DAO` implementations |
| **Proxy / AOP** | `AuthorizationAdvice`, `RequiredDataAdvice`, `LoggingAdvice` |
| **Template Method** | `BaseOpenmrsService`, `ImmutableEntityInterceptor` |
| **Observer / Event** | `GlobalPropertyListener`, `UserSessionListener`, `PrivilegeListener` |
| **Factory** | `ModuleFactory`, `TaskFactory`, `ServiceContext` (singleton factory) |
| **Strategy** | `AuthenticationScheme`, `IdentifierValidator`, `DosingInstructions` |
| **Decorator** | `ChainingInterceptor` (chains Hibernate interceptors) |
| **EAV** | The entire `Obs`/`Concept` model for clinical data