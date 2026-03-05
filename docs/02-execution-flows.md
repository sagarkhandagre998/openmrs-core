# OpenMRS Core — Complete Flow Reference
## Part 2: Execution Flows — AOP, HTTP, Auth, Registration, Clinical Data, Infrastructure

---

## 7. AOP Interceptor Chain (Detailed)

```mermaid
sequenceDiagram
    participant CALLER  as REST Controller
    participant PROXY   as Spring AOP Proxy
    participant AUTH    as AuthorizationAdvice
    participant REQDATA as RequiredDataAdvice
    participant LOG     as LoggingAdvice
    participant HANDLER as SaveHandler  VoidHandler  RetireHandler
    participant VALID   as Spring Validator
    participant IMPL    as PatientServiceImpl
    participant DAO     as HibernatePatientDAO
    participant HIB     as Hibernate Session

    CALLER ->> PROXY: patientService.savePatient(patient)

    Note over PROXY: Spring AOP proxy intercepts the call
    Note over PROXY: Runs all registered MethodBeforeAdvice in order

    rect rgb(255,235,235)
        Note over AUTH: INTERCEPTOR 1 — AuthorizationAdvice
        PROXY  ->> AUTH: before(method=savePatient, args=[patient], target)
        AUTH   ->> AUTH: read @Authorized on savePatient
        Note over AUTH: @Authorized({ADD_PATIENTS, EDIT_PATIENTS})
        AUTH   ->> AUTH: Daemon.isDaemonThread()
        alt Daemon thread — background scheduled task
            AUTH -->> PROXY: skip auth entirely — daemon bypasses all privilege checks
        else Normal HTTP request thread
            AUTH ->> AUTH: Context.addProxyPrivilege(GET_ROLES)
            loop each privilege in @Authorized annotation
                AUTH ->> AUTH: Context.hasPrivilege(privilege)
                AUTH ->> AUTH: UserContext.hasPrivilege(privilege)
                AUTH ->> AUTH: check user.getAllRoles() contains privilege
                alt User lacks ALL required privileges
                    AUTH -->> CALLER: throw APIAuthenticationException 401
                end
            end
            AUTH ->> AUTH: Context.removeProxyPrivilege(GET_ROLES)
            AUTH -->> PROXY: authorised — continue
        end
    end

    rect rgb(235,245,255)
        Note over REQDATA: INTERCEPTOR 2 — RequiredDataAdvice
        PROXY   ->> REQDATA: before(method=savePatient, args=[patient], target)
        REQDATA ->> REQDATA: methodName.startsWith("save") = true
        REQDATA ->> REQDATA: args[0] instanceof OpenmrsObject = true
        REQDATA ->> REQDATA: methodNameEndsWithClassName("savePatient", Patient) = true
        REQDATA ->> HANDLER: recursivelyHandle(SaveHandler.class, patient, null)

        loop Each SaveHandler that supports Patient.class
            HANDLER ->> HANDLER: handler.handle(patient, currentUser, date, other)
            HANDLER ->> HANDLER: if patient.creator == null  set creator = currentUser
            HANDLER ->> HANDLER: if patient.dateCreated == null  set dateCreated = now
            HANDLER ->> HANDLER: if patient.uuid == null  set uuid = UUID.randomUUID()
        end

        Note over HANDLER: Recurse into child collections
        loop patient.getNames() — each PersonName
            HANDLER ->> HANDLER: recursivelyHandle(SaveHandler, personName, null)
            HANDLER ->> HANDLER: set personName.creator, dateCreated, uuid
        end
        loop patient.getAddresses() — each PersonAddress
            HANDLER ->> HANDLER: recursivelyHandle(SaveHandler, personAddress, null)
        end
        loop patient.getIdentifiers() — each PatientIdentifier
            HANDLER ->> HANDLER: recursivelyHandle(SaveHandler, identifier, null)
        end

        REQDATA ->> VALID: ValidateUtil.validate(patient)
        VALID   ->> VALID: PersonValidator.validate()
        VALID   ->> VALID: check gender not blank
        VALID   ->> VALID: check birthdate not in future
        VALID   ->> VALID: check birthdate not > 140 years ago
        VALID   ->> VALID: PatientValidator.validate()
        VALID   ->> VALID: check at least one active identifier
        VALID   ->> VALID: check exactly one preferred identifier
        loop each PatientIdentifier
            VALID ->> VALID: PatientIdentifierValidator.validateIdentifier(pi)
            VALID ->> VALID: check identifier not blank
            VALID ->> VALID: check identifier matches format regex
            VALID ->> VALID: run configured IdentifierValidator  e.g. LuhnIdentifierValidator
            VALID ->> VALID: check uniqueness per UniquenessBehavior
        end
        alt Validation errors exist
            VALID -->> CALLER: throw ValidationException with BindingResult errors
        else No errors
            VALID -->> REQDATA: valid
        end
        REQDATA -->> PROXY: continue
    end

    rect rgb(235,255,235)
        Note over LOG: INTERCEPTOR 3 — LoggingAdvice
        PROXY ->> LOG: before(method, args, target)
        LOG   ->> LOG: log.debug entering PatientServiceImpl.savePatient
        LOG   -->> PROXY: continue
    end

    Note over PROXY: All advisors passed — invoke real method
    PROXY ->> IMPL: PatientServiceImpl.savePatient(patient)

    rect rgb(255,250,230)
        Note over IMPL: PatientServiceImpl Business Logic
        IMPL ->> IMPL: requireAppropriatePatientModificationPrivilege(patient)
        IMPL ->> IMPL: patient.id == null  require ADD_PATIENTS
        IMPL ->> IMPL: patient.id != null  require EDIT_PATIENTS
        IMPL ->> IMPL: patient.voided == true  require DELETE_PATIENTS
        IMPL ->> IMPL: if single identifier  force preferred=true
        IMPL ->> IMPL: checkPatientIdentifiers(patient)
        IMPL ->> IMPL: validate format, duplicates, required types
        IMPL ->> IMPL: setPreferredPatientIdentifier(patient)
        IMPL ->> IMPL: setPreferredPatientName(patient)
        IMPL ->> IMPL: setPreferredPatientAddress(patient)
        IMPL ->> DAO:  dao.savePatient(patient)
    end

    rect rgb(240,240,255)
        Note over DAO: HibernatePatientDAO
        DAO ->> HIB: sessionFactory.getCurrentSession()
        DAO ->> HIB: session.saveOrUpdate(patient)
    end

    rect rgb(255,240,240)
        Note over HIB: Hibernate fires Interceptors before flush
        HIB ->> HIB: AuditableInterceptor.onSave(entity, ...)
        HIB ->> HIB: set creator, dateCreated if null on INSERT
        HIB ->> HIB: set changedBy, dateChanged on UPDATE
        HIB ->> HIB: Envers captures revision to patient_aud table
        HIB ->> HIB: flush to DB  INSERT INTO person patient patient_identifier ...
    end

    DAO  -->> IMPL: saved Patient with generated IDs
    IMPL -->> PROXY: Patient
    PROXY -->> CALLER: Patient
```

---

## 8. Per-Request HTTP Filter Chain

```mermaid
sequenceDiagram
    participant CLIENT  as HTTP Client  Browser or REST
    participant CSRF    as CSRFGuardFilter
    participant STARTUP as StartupFilter
    participant UPDATE  as UpdateFilter
    participant OPENMRS as OpenmrsFilter
    participant GZIP    as GZIPFilter
    participant OSIV    as OpenSessionInViewFilter
    participant JSP_CL  as JspClassLoaderFilter
    participant SPRING  as DispatcherServlet
    participant CTX     as Context.java
    participant SESS    as Hibernate Session

    CLIENT ->> CSRF: HTTP Request

    rect rgb(255,235,235)
        Note over CSRF: CSRF protection check
        CSRF ->> CSRF: validate CSRF token on POST PUT DELETE
        alt CSRF token invalid
            CSRF -->> CLIENT: 403 Forbidden
        else token valid or GET request
            CSRF -->> STARTUP: chain.doFilter()
        end
    end

    rect rgb(255,245,225)
        Note over STARTUP: StartupFilter — Is app initialised?
        STARTUP ->> STARTUP: Listener.isSetupNeeded()
        alt Setup wizard needed  first time run
            STARTUP ->> STARTUP: render InitializationFilter wizard pages
            STARTUP -->> CLIENT: 200 Setup wizard HTML
            Note over STARTUP: Request does NOT continue to Spring
        else App already set up
            STARTUP -->> UPDATE: chain.doFilter()
        end
    end

    rect rgb(245,255,225)
        Note over UPDATE: UpdateFilter — DB migrations pending?
        UPDATE ->> UPDATE: DatabaseUpdater.updatesRequired()
        alt Updates needed and not auto-approved
            UPDATE ->> UPDATE: render UpdateFilter wizard
            UPDATE -->> CLIENT: 200 Update wizard HTML
        else No updates needed
            UPDATE -->> OPENMRS: chain.doFilter()
        end
    end

    rect rgb(225,245,255)
        Note over OPENMRS: OpenmrsFilter — User Context setup
        OPENMRS ->> OPENMRS: httpSession.getAttribute(OPENMRS_USER_CONTEXT)
        alt UserContext not on session yet
            OPENMRS ->> OPENMRS: new UserContext(authenticationScheme)
            OPENMRS ->> OPENMRS: httpSession.setAttribute(OPENMRS_USER_CONTEXT, userContext)
        end
        OPENMRS ->> OPENMRS: httpSession.setAttribute("username", user or -anonymous-)
        OPENMRS ->> OPENMRS: httpSession.setAttribute("locale", userContext.getLocale())
        OPENMRS ->> CTX: Context.setUserContext(userContext)
        Note over CTX: userContextHolder ThreadLocal = userContext
        OPENMRS ->> OPENMRS: Thread.currentThread().setContextClassLoader(OpenmrsClassLoader)
        OPENMRS -->> GZIP: chain.doFilter()
        Note over OPENMRS: finally block runs after response
    end

    rect rgb(235,255,235)
        Note over GZIP: GZIPFilter — Response compression
        GZIP ->> GZIP: check Accept-Encoding: gzip header
        alt Client supports gzip
            GZIP ->> GZIP: wrap response in GZIPResponseWrapper
        end
        GZIP -->> OSIV: chain.doFilter()
    end

    rect rgb(245,235,255)
        Note over OSIV: OpenSessionInViewFilter — Hibernate Session
        OSIV ->> OSIV: lookupSessionFactory() from WebApplicationContext
        OSIV ->> OSIV: TransactionSynchronizationManager.hasResource(sf)?
        alt Session already bound  e.g. async dispatch
            OSIV ->> OSIV: participate = true  reuse existing session
        else No session on thread
            OSIV ->> SESS: sf.openSession()
            SESS -->> OSIV: new Hibernate Session
            OSIV ->> OSIV: session.setHibernateFlushMode(MANUAL)
            OSIV ->> OSIV: TransactionSynchronizationManager.bindResource(sf, SessionHolder)
        end
        OSIV -->> JSP_CL: chain.doFilter()
        Note over OSIV: finally block closes session after response
    end

    rect rgb(255,235,245)
        Note over JSP_CL: JspClassLoaderFilter
        JSP_CL ->> JSP_CL: set OpenmrsClassLoader on thread for JSP compilation
        JSP_CL -->> SPRING: chain.doFilter()
    end

    rect rgb(235,235,255)
        Note over SPRING: DispatcherServlet — Spring MVC routing
        SPRING ->> SPRING: HandlerMapping — find controller method
        SPRING ->> SPRING: HandlerAdapter — invoke controller
        SPRING ->> SPRING: controller calls Context.getXxxService().method()
        SPRING -->> JSP_CL: response written
    end

    Note over OSIV: finally — tear down
    OSIV ->> OSIV: unbindResource(sf)
    OSIV ->> SESS: session.close()

    Note over OPENMRS: finally — tear down
    OPENMRS ->> CTX: Context.clearUserContext()
    Note over CTX: userContextHolder ThreadLocal = null  prevent memory leak

    OPENMRS -->> CLIENT: HTTP Response
```

---

## 9. Authentication and Session Flow

```mermaid
sequenceDiagram
    participant CLIENT  as HTTP Client
    participant FILTER  as OpenmrsFilter
    participant CTX     as Context.java
    participant UCTX    as UserContext
    participant CTXDAO  as HibernateContextDAO
    participant SESSION as Hibernate Session
    participant DB      as Database users table

    CLIENT ->> FILTER: POST /ws/rest/v1/session  {username, password}

    rect rgb(230,245,255)
        Note over FILTER: OpenmrsFilter runs first
        FILTER ->> UCTX: new UserContext(authenticationScheme)
        FILTER ->> CTX:  Context.setUserContext(userContext)
        Note over CTX: stored in ThreadLocal
    end

    rect rgb(255,245,230)
        Note over CTX: Context.authenticate(credentials)
        CTX    ->> CTX:  getAuthenticationScheme()
        Note over CTX: Default = UsernamePasswordAuthenticationScheme
        CTX    ->> CTXDAO: authenticate(username, password)
    end

    rect rgb(240,255,240)
        Note over CTXDAO: HibernateContextDAO.authenticate()
        CTXDAO ->> SESSION: openSession
        CTXDAO ->> DB: SELECT * FROM users WHERE username=? AND retired=false
        DB     -->> CTXDAO: candidateUser row

        alt User not found
            CTXDAO -->> CTX: throw ContextAuthenticationException
        end

        CTXDAO ->> CTXDAO: check lockout timestamp  USER_PROPERTY_LOCKOUT_TIMESTAMP
        alt Account locked  lockout period not expired
            CTXDAO ->> DB: UPDATE user_property SET lockout_timestamp=now
            CTXDAO -->> CTX: throw ContextAuthenticationException  locked out
        end

        CTXDAO ->> DB: SELECT password, salt FROM users WHERE user_id=?
        DB     -->> CTXDAO: passwordOnRecord, saltOnRecord
        CTXDAO ->> CTXDAO: Security.hashMatches(passwordOnRecord, password + salt)

        alt Password does not match
            CTXDAO ->> CTXDAO: attempts = getUsersLoginAttempts(user) + 1
            CTXDAO ->> DB: UPDATE user_property SET login_attempts = attempts
            alt attempts >= allowedFailedLoginCount  default 7
                CTXDAO ->> DB: SET lockout_timestamp = now
            end
            CTXDAO -->> CTX: throw ContextAuthenticationException  bad password
        else Password matches
            CTXDAO ->> CTXDAO: user.getAllRoles().size()  hydrate roles
            CTXDAO ->> CTXDAO: user.getUserProperties().size()  hydrate props
            CTXDAO ->> CTXDAO: user.getPrivileges().size()  hydrate privileges
            CTXDAO ->> CTXDAO: reset login_attempts = 0
            CTXDAO ->> DB: UPDATE user_property SET last_login_time = now
            CTXDAO -->> CTX: return authenticated User object
        end
    end

    rect rgb(255,240,255)
        Note over CTX: Context sets authenticated user into UserContext
        CTX  ->> UCTX: userContext.setUserIfStillAuthenticated(user)
        UCTX ->> UCTX: this.user = user
        UCTX ->> UCTX: this.authenticatedRole = Role  Authenticated
        Note over UCTX: locale resolved from user or system default
        CTX  ->> UCTX: notify UserSessionListeners  Event.LOGIN
    end

    CTX -->> CLIENT: 200 OK  session token  authenticated user info

    Note over CLIENT: Subsequent requests carry session cookie
    Note over CLIENT: OpenmrsFilter retrieves UserContext from HttpSession
    Note over CLIENT: No re-authentication needed per request
```

---

## 10. Privilege Check Flow

```mermaid
flowchart TD
    A([Service method called\ne.g. patientService.savePatient]) --> B

    B{Is Daemon\nthread?}
    B -->|Yes\nScheduled task background thread| C([ALLOW — no privilege check])
    B -->|No\nNormal HTTP thread| D

    D[Read @Authorized annotation\nfrom method signature]
    D --> E{Has @Authorized\nannotation?}

    E -->|No annotation at all| F([ALLOW — public method])
    E -->|Annotation present| G

    G{requireAll\nflag?}
    G -->|false\nAny one privilege sufficient| H
    G -->|true\nAll privileges required| I

    H[Loop through privileges]
    H --> H1{Context.hasPrivilege\nprivilege?}
    H1 -->|Yes| H2([ALLOW — first match])
    H1 -->|No more to check| H3([DENY — throw\nAPIAuthenticationException])

    I[Loop through ALL privileges]
    I --> I1{Context.hasPrivilege\nprivilege?}
    I1 -->|Yes — keep checking| I
    I1 -->|No| I2([DENY — throw\nAPIAuthenticationException])
    I -->|All checked and passed| I3([ALLOW])

    subgraph PRIV_CHECK["Context.hasPrivilege internals"]
        P1[Get current UserContext\nfrom ThreadLocal]
        P2{User is null\nnot logged in?}
        P2 -->|Yes| P3[Check Anonymous role's privileges]
        P2 -->|No| P4[Collect all user roles\nincluding inherited parent roles]
        P4 --> P5[Collect all privileges\nfrom all roles]
        P4 --> P6[Add proxy privileges\nContext.addProxyPrivilege called]
        P5 --> P7{privilege in\ncombined set?}
        P6 --> P7
        P3 --> P7
        P7 -->|Yes| P8([return true])
        P7 -->|No| P9([return false])
    end

    H1 --> P1
    I1 --> P1
```

---

## 11. Patient Registration — Full Execution Flow

```mermaid
sequenceDiagram
    participant CLIENT   as Receptionist Browser
    participant REST     as REST Controller
    participant CTX      as Context.java
    participant AOP      as AOP Proxy
    participant AUTH     as AuthorizationAdvice
    parameter  RDA      as RequiredDataAdvice
    participant IMPL     as PatientServiceImpl
    participant VALID    as PatientValidator
    participant ID_VALID as PatientIdentifierValidator
    participant DAO      as HibernatePatientDAO
    participant HIB      as Hibernate  AuditableInterceptor  Envers
    participant DB       as Database

    CLIENT ->> REST: POST /ws/rest/v1/patient\n{names, gender, birthdate, addresses, identifiers}

    REST ->> REST: deserialize JSON to Patient + Person + PatientIdentifier objects
    REST ->> CTX:  Context.getPatientService()
    CTX  -->> REST: AOP-proxied PatientService bean

    REST ->> AOP: patientService.savePatient(patient)

    rect rgb(255,235,235)
        AOP ->> AUTH: AuthorizationAdvice.before()
        AUTH ->> AUTH: check @Authorized({ADD_PATIENTS, EDIT_PATIENTS})
        AUTH ->> AUTH: new patient  patient.id == null  check ADD_PATIENTS
        AUTH -->> AOP: authorised
    end

    rect rgb(235,245,255)
        AOP ->> RDA: RequiredDataAdvice.before()
        RDA ->> RDA: recursivelyHandle(SaveHandler, patient)

        Note over RDA: Set audit fields on Patient
        RDA ->> RDA: patient.creator     = receptionist User
        RDA ->> RDA: patient.dateCreated = NOW
        RDA ->> RDA: patient.uuid        = new UUID

        Note over RDA: Set audit fields on each PersonName
        loop patient.getNames()
            RDA ->> RDA: name.creator = receptionist User
            RDA ->> RDA: name.dateCreated = NOW
            RDA ->> RDA: name.uuid = new UUID
        end

        Note over RDA: Set audit fields on each PersonAddress
        loop patient.getAddresses()
            RDA ->> RDA: address.creator = receptionist User
            RDA ->> RDA: address.dateCreated = NOW
        end

        Note over RDA: Set audit fields on each PatientIdentifier
        loop patient.getIdentifiers()
            RDA ->> RDA: identifier.creator = receptionist User
            RDA ->> RDA: identifier.dateCreated = NOW
        end

        RDA ->> VALID: ValidateUtil.validate(patient)
        VALID ->> VALID: gender not blank
        VALID ->> VALID: birthdate not in future
        VALID ->> VALID: birthdate not > 140 years ago
        VALID ->> VALID: cause of death set if dead=true
        VALID ->> VALID: preferred identifier chosen
        loop each PatientIdentifier
            VALID ->> ID_VALID: validateIdentifier(pi)
            ID_VALID ->> ID_VALID: identifier not blank
            ID_VALID ->> ID_VALID: matches format regex of PatientIdentifierType
            ID_VALID ->> ID_VALID: run configured Validator  e.g. LuhnIdentifierValidator
            ID_VALID ->> ID_VALID: check uniqueness against DB  if UNIQUE behavior
            ID_VALID -->> VALID: valid or BindingResult errors
        end
        VALID -->> RDA: valid
        RDA  -->> AOP: continue
    end

    AOP ->> IMPL: PatientServiceImpl.savePatient(patient)

    rect rgb(255,250,230)
        IMPL ->> IMPL: requireAppropriatePatientModificationPrivilege
        IMPL ->> IMPL: single identifier  force preferred=true
        IMPL ->> IMPL: checkPatientIdentifiers(patient)
        IMPL ->> IMPL: validate identifiers again  check required types
        IMPL ->> IMPL: setPreferredPatientIdentifier
        IMPL ->> IMPL: setPreferredPatientName
        IMPL ->> IMPL: setPreferredPatientAddress
        IMPL ->> DAO: dao.savePatient(patient)
    end

    rect rgb(240,255,240)
        DAO ->> DAO: sessionFactory.getCurrentSession()
        DAO ->> HIB: session.saveOrUpdate(patient)
    end

    rect rgb(240,240,255)
        Note over HIB: Hibernate fires before flush
        HIB ->> HIB: AuditableInterceptor.onSave()
        HIB ->> HIB: set creator dateCreated if still null  double check
        HIB ->> DB: INSERT INTO person (person_id, gender, birthdate, ...) VALUES (...)
        HIB ->> DB: INSERT INTO patient (patient_id, allergy_status) VALUES (...)
        HIB ->> DB: INSERT INTO person_name (person_id, given_name, family_name, ...) VALUES (...)
        HIB ->> DB: INSERT INTO person_address (person_id, address1, city_village, ...) VALUES (...)
        HIB ->> DB: INSERT INTO patient_identifier (patient_id, identifier, identifier_type, ...) VALUES (...)
        HIB ->> DB: INSERT INTO person_aud  audit revision
        HIB ->> DB: INSERT INTO patient_aud  audit revision
        HIB ->> DB: INSERT INTO patient_identifier_aud  audit revision
        Note over HIB: Spring @Transactional commits
        HIB -->> DAO: patient with generated IDs
    end

    DAO  -->> IMPL: saved Patient
    IMPL -->> AOP:  saved Patient
    AOP  -->> REST: saved Patient

    REST ->> REST: serialize Patient to JSON response
    REST -->> CLIENT: 201 Created  {uuid, patientId, names, identifiers, ...}

    Note over CLIENT: Patient registered successfully
    Note over CLIENT: Hospital ID printed on patient card
```

---

## 12. Visit and Encounter Creation Flow

```mermaid
sequenceDiagram
    participant NURSE    as Nurse  Triage
    participant REST     as REST Controller
    participant CTX      as Context.java
    participant VS_PROXY as VisitService AOP Proxy
    participant VSI      as VisitServiceImpl
    participant ES_PROXY as EncounterService AOP Proxy
    participant ESI      as EncounterServiceImpl
    participant EVH      as EncounterVisitHandler
    participant VD       as HibernateVisitDAO
    participant ED       as HibernateEncounterDAO
    participant DB       as Database

    Note over NURSE: Patient arrives — receptionist opens visit

    NURSE ->> REST: POST /ws/rest/v1/visit\n{patient, visitType, location, startDatetime}
    REST  ->> CTX: Context.getVisitService()
    REST  ->> VS_PROXY: visitService.saveVisit(visit)

    rect rgb(235,245,255)
        Note over VS_PROXY: AOP — auth + requiredData
        VS_PROXY ->> VS_PROXY: AuthorizationAdvice  check ADD_VISITS
        VS_PROXY ->> VS_PROXY: RequiredDataAdvice  set creator dateCreated uuid
        VS_PROXY ->> VSI: VisitServiceImpl.saveVisit(visit)
        VSI      ->> VD: dao.saveVisit(visit)
        VD       ->> DB: INSERT INTO visit (patient_id, visit_type_id, location_id, date_started)
        DB       -->> VD: visit_id generated
        VD       -->> VSI: saved Visit
        VSI      -->> VS_PROXY: Visit
        VS_PROXY -->> REST: Visit
    end

    REST -->> NURSE: 201 Created  {visitId, uuid, ...}

    Note over NURSE: Now record triage encounter with vitals

    NURSE ->> REST: POST /ws/rest/v1/encounter\n{patient, visit, encounterType=TRIAGE,\n encounterDatetime, location, provider,\n obs=[{concept=WEIGHT, valueNumeric=72},\n     {concept=BP_SYSTOLIC, valueNumeric=120},\n     {concept=BP_DIASTOLIC, valueNumeric=80}]}

    REST ->> CTX: Context.getEncounterService()
    REST ->> ES_PROXY: encounterService.saveEncounter(encounter)

    rect rgb(255,240,235)
        Note over ES_PROXY: AOP layer
        ES_PROXY ->> ES_PROXY: AuthorizationAdvice  check ADD_ENCOUNTERS
        ES_PROXY ->> ES_PROXY: RequiredDataAdvice  set creator dateCreated uuid
        ES_PROXY ->> ES_PROXY: recursively handle all child Obs
        ES_PROXY ->> ESI: EncounterServiceImpl.saveEncounter(encounter)
    end

    rect rgb(240,255,240)
        Note over ESI: EncounterServiceImpl business logic
        ESI ->> ESI: failIfDeniedToEdit(encounter)
        ESI ->> ESI: check encounterType.editPrivilege
        ESI ->> EVH: createVisitForNewEncounter(encounter)
        Note over EVH: Only if encounter has no visit yet
        EVH ->> EVH: getActiveEncounterVisitHandler()
        Note over EVH: e.g. ExistingOrNewVisitAssignmentHandler
        EVH ->> EVH: find active visit for patient or create new one
        alt Encounter already has a Visit assigned
            EVH -->> ESI: no-op
        else No visit  handler assigns one
            EVH ->> VS_PROXY: visitService.saveVisit(newVisit)
            VS_PROXY -->> EVH: saved Visit
            EVH ->> ESI: encounter.setVisit(visit)
        end

        ESI ->> ESI: requirePrivilege(encounter)
        ESI ->> ESI: new encounter  Context.requirePrivilege(ADD_ENCOUNTERS)

        Note over ESI: For existing encounter — sync obs dates if date changed
        ESI ->> ESI: check if encounterDatetime changed
        loop each Obs whose obsDatetime == old encounterDatetime
            ESI ->> ESI: obs.setObsDatetime(newDate)
        end

        Note over ESI: Sync patient across encounter, obs, orders
        loop each Obs in encounter
            ESI ->> ESI: if obs.person != encounter.patient  fix it
        end
        loop each Order in encounter
            ESI ->> ESI: if order.patient != encounter.patient  fix it
        end

        ESI ->> ED: dao.saveEncounter(encounter)
        ED  ->> DB: INSERT INTO encounter (patient_id, encounter_datetime, encounter_type_id, visit_id, location_id)
        DB  -->> ED: encounter_id generated

        Note over ESI: Save all Obs via ObsService
        loop each Obs at top level
            ESI ->> CTX: Context.getObsService().saveObs(obs, null)
            Note over CTX: Full ObsService AOP cycle runs for each Obs
            CTX ->> DB: INSERT INTO obs (encounter_id, concept_id, value_numeric, ...)
        end

        Note over ESI: Save OrderGroups and Orders
        loop each OrderGroup
            ESI ->> CTX: Context.getOrderService().saveOrderGroup(og)
        end
        loop each Order without group
            ESI ->> CTX: Context.getOrderService().saveOrder(order, null)
        end

        Note over ESI: Save Conditions, Allergies, Diagnoses
        loop encounter.getConditions()
            ESI ->> CTX: Context.getConditionService().saveCondition(c)
        end
        loop encounter.getAllergies()
            ESI ->> CTX: Context.getPatientService().saveAllergy(a)
        end
        loop encounter.getDiagnoses()
            ESI ->> CTX: Context.getDiagnosisService().save(d)
        end

        ESI -->> ES_PROXY: saved Encounter
    end

    ES_PROXY -->> REST: Encounter
    REST     -->> NURSE: 201 Created  encounter with all obs saved
```

---

## 13. Observation (Obs) Save Flow

```mermaid
sequenceDiagram
    participant CALLER  as EncounterServiceImpl or direct REST
    participant OS      as ObsService AOP Proxy
    participant AUTH    as AuthorizationAdvice
    participant RDA     as RequiredDataAdvice
    participant IMPL    as ObsServiceImpl
    participant DAO     as HibernateObsDAO
    participant INT     as ImmutableObsInterceptor
    participant ENVERS  as Hibernate Envers
    participant DB      as Database

    CALLER ->> OS: obsService.saveObs(obs, changeMessage)

    rect rgb(255,235,235)
        OS ->> AUTH: AuthorizationAdvice.before()
        AUTH ->> AUTH: obs.obsId == null  check ADD_OBS
        AUTH ->> AUTH: obs.obsId != null  check EDIT_OBS
        AUTH -->> OS: authorised
    end

    rect rgb(235,245,255)
        OS ->> RDA: RequiredDataAdvice.before()
        RDA ->> RDA: set obs.creator, dateCreated, uuid
        RDA ->> RDA: if obs is group  recursively handle each groupMember
        RDA -->> OS: continue
    end

    OS ->> IMPL: ObsServiceImpl.saveObs(obs, changeMessage)

    rect rgb(240,255,240)
        IMPL ->> IMPL: null check on obs
        IMPL ->> IMPL: if obs.id != null and changeMessage == null  throw
        Note over IMPL: Editing an existing obs REQUIRES a change reason
        IMPL ->> IMPL: ens