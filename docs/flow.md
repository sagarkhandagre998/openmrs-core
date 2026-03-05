🏗️ OpenMRS — Complete Service Execution Flow (From App Start to DB Commit)

I've created two detailed Mermaid diagram files for you:
- `openmrs-core/docs/01-architecture-and-startup.md` — Architecture, Maven modules, Startup sequence, Spring wiring, Domain model, Service map
- `openmrs-core/docs/02-execution-flows.md` — AOP chain, HTTP filters, Auth, Registration, Visit/Encounter, Obs, Order, Privilege, Scheduler

Here's the complete written explanation of every flow:

---

## 🚀 Phase 1 — Application Startup (7 Phases)

When Tomcat starts and loads the WAR, the very first thing that fires is `Listener.java` (`ServletContextListener`). It runs in **7 phases**:

| Phase | Class | What Happens |
|---|---|---|
| **1 — Pre-flight** | `Listener.java` | Validates Java version, reads `runtime.properties`, loads CSRF config, sets application data directory |
| **2 — Spring Boot** | `XmlWebApplicationContext` | Reads `applicationContext-service.xml`, builds `HibernateSessionFactoryBean` (loads `hibernate.cfg.xml`, scans `org.openmrs` for `@Entity` classes, collects all `Interceptor` beans sorted alphabetically), wires all `@Service`, `@Repository`, `@Component`, `@Handler` beans |
| **3 — Daemon Boot** | `WebDaemon` → `Daemon` | Spawns a privileged `DaemonThread` (bypasses auth), starts OpenMRS inside it |
| **4 — Load Modules** | `ModuleFactory` | Scans `WEB-INF/bundledModules/*.omod` files, creates a `ModuleClassLoader` per module |
| **5 — Context.startup()** | `Context` → `HibernateContextDAO` → `DatabaseUpdater` (Liquibase) | Runs all 3 Liquibase changelogs (schema-only → core-data → update-to-latest), opens session, calls `checkCoreDataset()` to ensure core Roles/Privileges exist, rebuilds Lucene/ES search index, then starts all modules via `ModuleUtil.startup()` calling `ModuleActivator.started()` per module |
| **6 — Web Modules** | `Listener.performWebStartOfModules()` | Registers module-contributed servlets, filters, and DWR endpoints |
| **7 — Scheduler** | `TimerSchedulerServiceImpl` | Loads all `TaskDefinition` from DB, schedules any with `startOnStartup=true` via `java.util.Timer` inside `Daemon` threads |

---

## 🌐 Phase 2 — Every HTTP Request (7-Filter Chain)

Every single HTTP request passes through this chain in order:

```
CSRFGuardFilter → StartupFilter → UpdateFilter → OpenmrsFilter → GZIPFilter → OpenSessionInViewFilter → JspClassLoaderFilter → DispatcherServlet
```

The **two most important filters** are:

**`OpenmrsFilter`** — This is the identity linchpin. It reads the `UserContext` from the `HttpSession` and calls `Context.setUserContext(userContext)` which puts it into a `ThreadLocal`. Every subsequent service call on this thread can now call `Context.getAuthenticatedUser()` and get back the logged-in user. After the response is sent, `Context.clearUserContext()` removes it from the `ThreadLocal` (prevents memory leaks).

**`OpenSessionInViewFilter`** — Opens a Hibernate `Session` and binds it to the thread via `TransactionSynchronizationManager`. This is what enables **lazy loading** — if a controller reads a `Patient` and later accesses `patient.getVisits()`, Hibernate can still load those visits because the session is still open. After the response, the session is closed.

---

## 🔐 Phase 3 — Authentication Flow

When a user logs in (`POST /ws/rest/v1/session`):

1. `Context.authenticate(credentials)` calls `HibernateContextDAO.authenticate(username, password)`
2. It looks up the user by username or system ID in the `users` table
3. Checks the lockout timestamp — if the account is locked (7 failed attempts by default), it rejects and updates the lockout time
4. Fetches the `password` and `salt` from the DB
5. Calls `Security.hashMatches(passwordOnRecord, inputPassword + salt)` — SHA-512 hash comparison
6. On success: hydrates all `Role`s, `Privilege`s, and `UserProperty` objects, resets the login attempt counter, updates `last_login_time`, and returns the `User` object
7. `UserContext.setUserIfStillAuthenticated(user)` stores it on the `UserContext` which lives in the HTTP session

---

## 🔒 Phase 4 — The AOP Interceptor Chain (Every Service Call)

Every call to any service method passes through **3 Spring AOP advisors** applied as `MethodBeforeAdvice`:

### Interceptor 1 — `AuthorizationAdvice`
- Reads the `@Authorized({"privilege1", "privilege2"})` annotation on the method
- If it's a `Daemon` thread → skip entirely (scheduled tasks bypass auth)
- Otherwise checks `Context.hasPrivilege()` which looks at the `UserContext` → walks all `Role`s (including inherited parent roles) and their `Privilege`s
- Throws `APIAuthenticationException` if denied

### Interceptor 2 — `RequiredDataAdvice`
- For `save*` methods: calls `recursivelyHandle(SaveHandler.class, entity, changeMessage)` which sets `creator`, `dateCreated`, `uuid` on the entity AND all child collections (names, addresses, identifiers, obs group members, etc.)
- For `void*` methods: calls `VoidHandler` — sets `voidedBy`, `dateVoided`, `voidReason`
- For `retire*` methods: calls `RetireHandler` — sets `retiredBy`, `dateRetired`, `retireReason`
- Then calls `ValidateUtil.validate(entity)` which runs the appropriate Spring `Validator` (e.g., `PatientValidator` → `PersonValidator` → `PatientIdentifierValidator`)

### Interceptor 3 — `LoggingAdvice`
- Just logs method entry/exit at `DEBUG` level

---

## 📝 Phase 5 — Patient Registration (12 DB Writes)

```
POST /ws/rest/v1/patient
    ↓ AOP: AuthorizationAdvice (checks ADD_PATIENTS)
    ↓ AOP: RequiredDataAdvice (sets creator/dateCreated/uuid recursively on Patient, PersonName, PersonAddress, PatientIdentifier)
    ↓ AOP: PatientValidator + PatientIdentifierValidator (format regex, Luhn check digit, uniqueness)
    ↓ PatientServiceImpl.savePatient()
        ↓ checkPatientIdentifiers() — check required types, no duplicates
        ↓ setPreferredPatientIdentifier/Name/Address()
        ↓ HibernatePatientDAO.savePatient()
            ↓ session.saveOrUpdate(patient)
                ↓ AuditableInterceptor.onSave() — double-check creator/dateCreated
                ↓ INSERT INTO person (...)
                ↓ INSERT INTO patient (...)
                ↓ INSERT INTO person_name (...)
                ↓ INSERT INTO person_address (...)
                ↓ INSERT INTO patient_identifier (...)
                ↓ Hibernate Envers → INSERT INTO person_aud, patient_aud, patient_identifier_aud
                ↓ @Transactional COMMIT
```

---

## 🏥 Phase 6 — Visit → Encounter → Obs Chain

**Visit** (`encounterService.saveEncounter` can auto-create one via `EncounterVisitHandler`):
```
INSERT INTO visit (patient_id, visit_type_id, location_id, date_started)
```

**Encounter** (groups all interactions in one visit):
```
EncounterServiceImpl.saveEncounter()
    → dao.saveEncounter()   → INSERT INTO encounter (...)
    → for each Obs          → Context.getObsService().saveObs(obs, null)
    → for each Order        → Context.getOrderService().saveOrder(order, null)
    → for each Condition    → Context.getConditionService().saveCondition(c)
    → for each Allergy      → Context.getPatientService().saveAllergy(a)
    → for each Diagnosis    → Context.getDiagnosisService().save(d)
```

**Obs** (the immutable EAV data point):
```
ObsServiceImpl.saveObs(obs, changeMessage)
    If obs is NEW:         → dao.saveObs(obs) → INSERT INTO obs (concept_id, value_numeric, ...)
    If obs is EXISTING:    → saveExistingObs()
        → Obs newObs = Obs.newInstance(obs)   copy all fields
        → newObs.setPreviousVersion(obs)      link to old
        → dao.saveObs(newObs)                 INSERT new row
        → obsService.voidObs(obs, reason)     void the old row
        ↓ ImmutableObsInterceptor prevents direct field edits
```

**Order** (immutable clinical instructions):
```
OrderServiceImpl.saveOrder(order, context)
    → failOnExistingOrder()     CANNOT edit an existing order
    → ensureDateActivatedIsSet()
    → ensureConceptIsSet()      drug → get concept from Drug entity
    → ensureDrugOrderAutoExpirationDateIsSet()  calculate from duration
    → ensureOrderTypeIsSet()    DrugOrder → Drug order type
    → ensureCareSettingIsSet()  OUTPATIENT or INPATIENT
    → if Action=REVISE          stopOrder(previousOrder)
    → if Action=DISCONTINUE     discontinueExistingOrdersIfNecessary()
    → check for duplicate active drug orders (AmbiguousOrderException)
    → saveOrderInternal()       → INSERT INTO orders (...)
```

---

## ⏰ Phase 7 — Scheduler and Daemon Threads

`TimerSchedulerServiceImpl` uses `java.util.Timer` to schedule tasks. Each task runs inside a `Daemon.executeScheduledTask()` call which:
1. Marks the thread as `isDaemonThread = true` (bypasses all `AuthorizationAdvice` checks)
2. Sets the daemon `User` as the authenticated user
3. Opens a Context session
4. Runs `task.execute()`
5. Closes the session

This means background tasks (like "Auto-close visits", "Concept word index rebuild", "Send HL7 messages") can call any service method without having a login session.

---

## 🗃️ Phase 8 — Hibernate Interceptor Chain (Inside the ORM)

When `session.saveOrUpdate(entity)` is called, Hibernate fires these interceptors **before writing to DB**:

| Interceptor | Trigger | Action |
|---|---|---|
| `AuditableInterceptor` | `onSave()` INSERT | Sets `creator`/`dateCreated` if null |
| `AuditableInterceptor` | `onFlushDirty()` UPDATE | Sets `changedBy`/`dateChanged` |
| `ImmutableObsInterceptor` | `onFlushDirty()` UPDATE on `Obs` | Throws if any field other than `voided`/`dateVoided`/`voidedBy`/`voidReason`/`groupMembers` changed |
| `ImmutableOrderInterceptor` | `onFlushDirty()` UPDATE on `Order` | Throws if any immutable field changed |
| `DropMillisecondsHibernateInterceptor` | `onSave` / `onFlushDirty` | Truncates milliseconds from dates (MySQL compatibility) |
| **Hibernate Envers** | After any write | Captures `_aud` table entry with revision number, changed fields, and `OpenmrsRevisionEntity` |

The `ChainingInterceptor` collects all registered interceptors and calls them in sorted bean-name order, ensuring `AuditableInterceptor` (alphabetically first) always fires before others.