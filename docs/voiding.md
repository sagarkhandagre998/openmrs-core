Voiding in OpenMRS — A Complete Deep Dive

## 1. Why Voiding Exists — The Philosophy

In an **Electronic Medical Record (EMR)** system, deleting clinical data is dangerous:

- A physician entered the wrong blood pressure reading → you can't just `DELETE` it, because regulators and auditors must see that it was recorded and *then* corrected.
- Two duplicate patient records are merged → the wrong record must be *preserved* for history but hidden from the active view.
- A medication order was placed erroneously → removing it from the database leaves no trace of the mistake.

OpenMRS solves this with a **"soft-delete" pattern** called **Voiding**. Instead of physically deleting rows, the system **marks them as void** — the data stays in the database forever but is treated as if it doesn't exist for clinical purposes.

> **Core rule:** In OpenMRS, data is almost NEVER permanently deleted. It is either **voided** (for clinical data) or **retired** (for configuration/metadata).

---

## 2. The Three-Tier Deletion Model

```openmrs-core/api/src/main/java/org/openmrs/Voidable.java#L14-27
/**
 * In OpenMRS, data are rarely fully deleted (purged) from the system; rather, they are either
 * voided or retired. When data can be removed (effectively deleted from the user's perspective),
 * then they are voidable. Voided data are no longer valid...
 */
```

OpenMRS has **three ways** to "remove" data, depending on what type of data it is:

```/dev/null/openmrs-data-removal.md#L1-20
┌─────────────────────────────────────────────────────────────────────┐
│                   Three Data Removal Strategies                     │
├───────────────┬─────────────────────────────┬───────────────────────┤
│  Strategy     │  Used For                   │  Effect               │
├───────────────┼─────────────────────────────┼───────────────────────┤
│  VOID         │  Clinical Data              │  Hidden from UI;      │
│               │  (Patient, Encounter, Obs,  │  still in DB;         │
│               │   Order, Visit, etc.)       │  cascade to children  │
├───────────────┼─────────────────────────────┼───────────────────────┤
│  RETIRE       │  Metadata/Config            │  Still usable for old │
│               │  (Concept, EncounterType,   │  data; can't be used  │
│               │   Form, IdentifierType)     │  for new entries      │
├───────────────┼─────────────────────────────┼───────────────────────┤
│  PURGE        │  TEST/ADMIN only            │  Real SQL DELETE.     │
│               │  (Never in production)      │  Permanent. Gone.     │
└───────────────┴─────────────────────────────┴───────────────────────┘
```

---

## 3. The `Voidable` Interface — The Foundation

Every voidable object in OpenMRS implements this interface:

```openmrs-core/api/src/main/java/org/openmrs/Voidable.java#L27-70
public interface Voidable extends OpenmrsObject {
    // Was this object voided?
    public Boolean getVoided();
    public void setVoided(Boolean voided);

    // Who voided it?
    public User getVoidedBy();
    public void setVoidedBy(User voidedBy);

    // When was it voided?
    public Date getDateVoided();
    public void setDateVoided(Date dateVoided);

    // Why was it voided?
    public String getVoidReason();
    public void setVoidReason(String voidReason);
}
```

These **four fields** are stored in every voidable database table:

```/dev/null/db-columns.sql#L1-10
-- Every voidable table has these 4 columns:
voided        TINYINT(1) NOT NULL DEFAULT 0,  -- The "soft-delete" flag
voided_by     INT(11),                         -- FK → users.user_id
date_voided   DATETIME,                        -- Timestamp
void_reason   VARCHAR(255)                     -- Mandatory reason
```

---

## 4. The Inheritance Chain — Who Is Voidable?

```/dev/null/hierarchy.md#L1-35
                    ┌──────────────┐
                    │ OpenmrsObject│  (uuid, id)
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │                         │
       ┌──────▼──────┐         ┌────────▼────────┐
       │  OpenmrsData│         │ OpenmrsMetadata  │
       │  (Voidable) │         │  (Retireable)    │
       └──────┬───────┘         └────────┬────────┘
              │                          │
   ┌──────────▼──────────┐      ┌────────▼──────────┐
   │  BaseOpenmrsData    │      │  BaseOpenmrsMetadata│
   │  (voided,           │      │  (retired,         │
   │   voidedBy,         │      │   retiredBy,       │
   │   dateVoided,       │      │   dateRetired,     │
   │   voidReason)       │      │   retireReason)    │
   └──────────┬──────────┘      └────────────────────┘
              │
    ┌─────────┼──────────┬──────────────┐
    │         │          │              │
  Visit   Encounter   Order           Obs
  Patient PatientId.  Visit     PersonName
  Person  Cohort      Allergy   PersonAddress
```

`BaseOpenmrsData` is where the actual fields live:

```openmrs-core/api/src/main/java/org/openmrs/BaseOpenmrsData.java#L51-64
@Column(name = "voided", nullable = false)
@GenericField
private Boolean voided = Boolean.FALSE;

@Column(name = "date_voided")
private Date dateVoided;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "voided_by")
private User voidedBy;

@Column(name = "void_reason", length = 255)
private String voidReason;
```

---

## 5. The AOP Engine — How Voiding Is Triggered Automatically

This is the **most critical piece** of the architecture. You never manually set `voided=true`. Instead, you call a service method like `voidEncounter(encounter, reason)`, and Spring AOP intercepts it.

### The Flow

```/dev/null/aop-flow.md#L1-30
Developer calls:
   encounterService.voidEncounter(encounter, "Wrong patient")
                    │
                    ▼
         ┌──────────────────────┐
         │  Spring AOP Proxy    │ ← wraps all service beans
         └──────────┬───────────┘
                    │ intercepts before the real method runs
                    ▼
         ┌──────────────────────┐
         │ RequiredDataAdvice   │ ← MethodBeforeAdvice
         │  .before()           │
         └──────────┬───────────┘
                    │ sees method starts with "void"
                    ▼
         ┌──────────────────────────────────────────────┐
         │  recursivelyHandle(VoidHandler.class, ...)   │
         │  1. Find all @Handler(supports=X) handlers   │
         │  2. Call handler.handle() on main object     │
         │  3. Loop all child collections               │
         │  4. Recursively call on each child           │
         └──────────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Real Service Method  │ ← now runs with voided flag set
         │  (saves to DB)       │
         └──────────────────────┘
```

Here is the actual AOP code that detects `void*` methods:

```openmrs-core/api/src/main/java/org/openmrs/aop/RequiredDataAdvice.java#L160-172
if (methodName.startsWith("void")) {
    Voidable voidable = (Voidable) args[0];
    Date dateVoided = voidable.getDateVoided() == null ? new Date() : voidable.getDateVoided();
    String voidReason = (String) args[1];
    recursivelyHandle(VoidHandler.class, voidable, Context.getAuthenticatedUser(), dateVoided, voidReason,
        (Set<OpenmrsObject>) null);

} else if (methodName.startsWith("unvoid")) {
    Voidable voidable = (Voidable) args[0];
    Date originalDateVoided = voidable.getDateVoided();
    User originalVoidingUser = voidable.getVoidedBy();
    recursivelyHandle(UnvoidHandler.class, voidable, originalVoidingUser, originalDateVoided, 
        null, (Set<OpenmrsObject>) null);
}
```

---

## 6. The Handler Chain — Who Does What

When `voidEncounter(...)` is called, `RequiredDataAdvice` finds **all handlers** annotated with `@Handler(supports=Encounter.class)` (or its supertype `Voidable.class`) and runs them:

```/dev/null/handlers.md#L1-25
For voidEncounter():
┌─────────────────────────────────────────────────────┐
│  Handler                        │  order │  Does     │
├─────────────────────────────────┼────────┼───────────┤
│  RequireVoidReasonVoidHandler   │   1    │  Validates│
│  (supports Patient,Encounter,   │  FIRST │  voidReason│
│   Obs, Cohort, Order)           │        │  is not   │
│                                 │        │  blank    │
├─────────────────────────────────┼────────┼───────────┤
│  BaseVoidHandler                │ default│  Sets:    │
│  (supports all Voidable)        │        │  voided=T │
│                                 │        │  voidedBy │
│                                 │        │  dateVoided│
│                                 │        │  voidReason│
└─────────────────────────────────┴────────┴───────────┘
```

`RequireVoidReasonVoidHandler` runs **first** (order=1) to validate:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/RequireVoidReasonVoidHandler.java#L36-46
@Handler(supports = { Patient.class, Encounter.class, Obs.class, Cohort.class, Order.class }, order = 1)
public class RequireVoidReasonVoidHandler implements VoidHandler<Voidable> {

    @Override
    public void handle(Voidable voidableObject, User voidingUser, Date voidedDate, String voidReason) {
        if (StringUtils.isBlank(voidReason)) {
            throw new IllegalArgumentException("The 'reason' argument is required");
        }
    }
}
```

`BaseVoidHandler` then sets all the fields:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/BaseVoidHandler.java#L55-70
@Override
public void handle(Voidable voidableObject, User voidingUser, Date voidedDate, String voidReason) {
    // skip over all work if the object is already voided
    if (!voidableObject.getVoided() || voidableObject.getVoidedBy() == null) {
        voidableObject.setVoided(true);
        voidableObject.setVoidReason(voidReason);
        if (voidableObject.getVoidedBy() == null) {
            voidableObject.setVoidedBy(voidingUser);
        }
        if (voidableObject.getDateVoided() == null) {
            voidableObject.setDateVoided(voidedDate);
        }
    }
}
```

---

## 7. Cascading Void — The Hierarchy

When you void a **parent**, OpenMRS **cascades** the void down to all children. This is done by two mechanisms:

### Mechanism A: Recursive child-collection traversal (AOP)
`RequiredDataAdvice.recursivelyHandle()` introspects all fields of the voided object using reflection, finds any child collections of `OpenmrsObject`, and calls the VoidHandler on each one.

### Mechanism B: Domain-specific VoidHandlers

For complex cascades, specific handlers exist:

---

### Cascade: Voiding a `Visit`

```openmrs-core/api/src/main/java/org/openmrs/api/handler/VisitVoidHandler.java#L36-44
@Handler(supports = Visit.class)
public class VisitVoidHandler implements VoidHandler<Visit> {
    @Override
    public void handle(Visit voidableObject, User voidingUser, Date voidedDate, String voidReason) {
        List<Encounter> encountersByVisit = Context.getEncounterService()
            .getEncountersByVisit(voidableObject, false);
        for (Encounter encounter : encountersByVisit) {
            encounter.setDateVoided(voidedDate);
            Context.getEncounterService().voidEncounter(encounter, voidReason);
        }
    }
}
```

### Cascade: Voiding an `Encounter`

```openmrs-core/api/src/main/java/org/openmrs/api/impl/EncounterServiceImpl.java#L421-457
public Encounter voidEncounter(Encounter encounter, String reason) {
    // Void all Obs in this encounter
    ObsService os = Context.getObsService();
    for (Obs o : encounter.getObsAtTopLevel(false)) {
        if (!o.getVoided()) {
            os.voidObs(o, reason);
        }
    }
    // Void all Orders in this encounter
    OrderService orderService = Context.getOrderService();
    for (Order o : encounter.getOrders()) {
        orderService.voidOrder(o, reason);
    }
    encounter.setVoided(true);
    encounter.setVoidedBy(Context.getAuthenticatedUser());
    encounter.setVoidReason(reason);
    Context.getEncounterService().saveEncounter(encounter);
    return encounter;
}
```

### Cascade: Voiding a `Patient`

```openmrs-core/api/src/main/java/org/openmrs/api/handler/PatientDataVoidHandler.java#L47-61
@Handler(supports = Patient.class)
public class PatientDataVoidHandler implements VoidHandler<Patient> {
    @Override
    public void handle(Patient patient, User voidingUser, Date voidedDate, String voidReason) {
        // Void ALL encounters (which cascades to Obs + Orders)
        EncounterService es = Context.getEncounterService();
        List<Encounter> encounters = es.getEncountersByPatient(patient);
        for (Encounter encounter : encounters) {
            if (!encounter.getVoided()) {
                encounter.setDateVoided(patient.getDateVoided()); // match timestamp!
                es.voidEncounter(encounter, voidReason);
            }
        }
        // Also notify CohortService
        Context.getCohortService().notifyPatientVoided(patient);
    }
}
```

### The Full Cascade Diagram

```/dev/null/cascade-diagram.md#L1-40
   voidPatient(patient, reason)
         │
         ▼
   ┌─────────────┐
   │   Patient   │ voided=true, voidedBy=user, dateVoided=now, voidReason=reason
   └──────┬──────┘
          │  PatientDataVoidHandler cascades to:
          │
          ├──► ┌─────────────┐
          │    │  Encounter 1│ voided=true (same dateVoided as Patient)
          │    └──────┬──────┘
          │           │ EncounterService.voidEncounter() cascades to:
          │           ├──► ┌─────────┐
          │           │    │  Obs 1  │ voided=true
          │           │    └─────────┘
          │           ├──► ┌─────────┐
          │           │    │  Obs 2  │ voided=true
          │           │    └─────────┘
          │           └──► ┌─────────┐
          │                │ Order 1 │ voided=true
          │                └─────────┘
          │
          ├──► ┌─────────────┐
          │    │  Encounter 2│ voided=true
          │    └─────────────┘
          │
          └──► PersonVoidHandler also retires
               all User accounts for this Person
```

The **key guarantee**: Every entity voided because of a parent void operation gets the **same `dateVoided` timestamp** as the parent. This makes unvoiding reliable — you only unvoid children that share the same `dateVoided`.

---

## 8. Special Case: The `Person` and `Patient` Split

`Person` has **its own set of void fields** (`personVoided`, `personVoidedBy`, `personDateVoided`, `personVoidReason`) because `Person` is a supertype of `Patient` and `User`, and the Hibernate mapping stores them in the same `person` table with differently-named columns. The `PersonVoidHandler` handles this:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/PersonVoidHandler.java#L48-65
@Handler(supports = Person.class)
public class PersonVoidHandler implements VoidHandler<Person> {
    @Override
    public void handle(Person person, User voidingUser, Date voidedDate, String voidReason) {
        if (!person.getPersonVoided()) {
            if (person.getPersonId() != null) {
                // Also RETIRE all User accounts linked to this person!
                UserService us = Context.getUserService();
                for (User user : us.getUsersByPerson(person, false)) {
                    us.retireUser(user, voidReason);
                }
            }
            person.setPersonVoided(true);
            person.setPersonVoidReason(voidReason);
            // ...
        }
    }
}
```

---

## 9. Unvoiding — Reversing the Operation

Unvoiding is the mirror operation. The same AOP intercepts `unvoid*` methods. `BaseUnvoidHandler` clears all void fields:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/BaseUnvoidHandler.java#L50-62
@Override
public void handle(Voidable voidableObject, User voidingUser, Date origParentVoidedDate, String unused) {
    // Only unvoid if: (1) currently voided AND (2) dateVoided matches the parent's void timestamp
    if (voidableObject.getVoided()
            && (origParentVoidedDate == null || origParentVoidedDate.equals(voidableObject.getDateVoided()))) {
        voidableObject.setVoided(false);
        voidableObject.setVoidedBy(null);
        voidableObject.setDateVoided(null);
        voidableObject.setVoidReason(null);
    }
}
```

The **matching timestamp** rule is critical: when you unvoid an Encounter, you don't want to also unvoid Obs that were **already voided before** the Encounter was voided. Only the ones voided *at the same time* (same `dateVoided`) get restored.

### Unvoid Cascade for `Visit`:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/VisitUnvoidHandler.java#L36-44
@Override
public void handle(Visit visit, User voidingUser, Date origParentVoidedDate, String unused) {
    List<Encounter> encountersByVisit = Context.getEncounterService()
        .getEncountersByVisit(visit, true); // includeVoided=true
    for (Encounter encounter : encountersByVisit) {
        // Only unvoid if same timestamp AND same reason as visit
        if (encounter.getVoided() 
                && encounter.getDateVoided().equals(visit.getDateVoided())
                && encounter.getVoidReason().equals(visit.getVoidReason())) {
            Context.getEncounterService().unvoidEncounter(encounter);
        }
    }
}
```

---

## 10. The `VoidSaveHandler` — Guarding Against Inconsistent Saves

If someone bypasses the service method and directly sets `voided=true` and then calls `savePatient(...)`, this handler ensures the metadata is filled in:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/VoidSaveHandler.java#L60-76
@Handler(supports = Voidable.class)
public class VoidSaveHandler implements SaveHandler<Voidable> {
    @Override
    public void handle(Voidable voidableObject, User currentUser, Date currentDate, String notUsed) {
        if (voidableObject.getVoided()) {
            // Fill in voidedBy and dateVoided if missing
            if (voidableObject.getVoidedBy() == null) {
                voidableObject.setVoidedBy(currentUser);
            }
            if (voidableObject.getDateVoided() == null) {
                voidableObject.setDateVoided(currentDate);
            }
        } else {
            // If voided is cleared to false, wipe out all void metadata
            voidableObject.setVoidedBy(null);
            voidableObject.setDateVoided(null);
            voidableObject.setVoidReason(null);
        }
    }
}
```

---

## 11. The Special Obs "Void-and-Replace" Pattern

For `Obs`, voiding has a special additional semantic: **edit-by-replacement**. Since Obs are clinical observations that form part of the medical record, you can't just edit them in place. Instead you:

1. Create a new Obs with the corrected value
2. Link it to the old Obs via `previousVersion`
3. Void the old Obs with a reason

This is the documented pattern right at the top of the `Obs` class:

```openmrs-core/api/src/main/java/org/openmrs/Obs.java#L56-64
// When you need to edit an Obs:
Obs newObs = Obs.newInstance(oldObs);    // copy all values
newObs.setPreviousVersion(oldObs);       // link to old version
Context.getObsService().saveObs(newObs, "Your reason for the change here");
Context.getObsService().voidObs(oldObs, "Your reason for the change here");
```

The `ObsServiceImpl.saveObs()` does this **automatically** for existing Obs:

```openmrs-core/api/src/main/java/org/openmrs/api/impl/ObsServiceImpl.java#L160-180
private Obs saveExistingObs(Obs obs, String changeMessage) {
    // 1. Make a new copy (new row in DB, new obs_id)
    Obs newObs = Obs.newInstance(obs);
    unsetVoidedAndCreationProperties(newObs, obs);

    // 2. Save the new row
    dao.saveObs(newObs);
    saveObsGroup(newObs, null);

    // 3. Void the old row (with the changeMessage as reason)
    voidExistingObs(obs, changeMessage, newObs);

    return newObs;
}
```

The audit trail looks like this in the DB:

```/dev/null/obs-audit.sql#L1-10
-- Original obs (voided=TRUE, audit trail preserved)
obs_id=101, concept=WEIGHT, value=70kg, voided=1, void_reason="Correction", voided_by=user5

-- New corrected obs (active, links back to old)
obs_id=102, concept=WEIGHT, value=72kg, voided=0, previous_version=101
```

---

## 12. How Voiding Integrates with Security

Every void service method is protected with `@Authorized`:

```/dev/null/security-table.md#L1-15
Method                              │ Required Privilege
────────────────────────────────────┼──────────────────────────
patientService.voidPatient()        │ "Delete Patients"
patientService.unvoidPatient()      │ "Delete Patients"
encounterService.voidEncounter()    │ "Edit Encounters"
encounterService.unvoidEncounter()  │ "Edit Encounters"
obsService.voidObs()                │ "Edit Observations"
obsService.unvoidObs()              │ "Edit Observations"
orderService.voidOrder()            │ "Delete Orders"
visitService.voidVisit()            │ "Delete Visits"
patientService.purgePatient()       │ "Purge Patients"  ← SEPARATE privilege
```

The `AuthorizationAdvice` AOP interceptor checks these **before** `RequiredDataAdvice` sets the void fields. So the full call stack is:

```/dev/null/full-call-stack.md#L1-18
encounterService.voidEncounter(encounter, reason)
    │
    ├─► AuthorizationAdvice.before()          [checks "Edit Encounters" privilege]
    │       throws APIAuthenticationException if denied
    │
    ├─► RequiredDataAdvice.before()           [AOP detects void*, runs handlers]
    │       └─ RequireVoidReasonVoidHandler   [validates reason is not blank]
    │       └─ BaseVoidHandler               [sets voided, voidedBy, dateVoided]
    │
    └─► EncounterServiceImpl.voidEncounter() [actual business logic]
            ├─ voids each Obs in encounter
            ├─ voids each Order in encounter
            └─ saves the encounter to DB
```

---

## 13. Voiding at the DAO Layer — Filtering Voided Records

The DAO layer automatically **filters out voided records** when querying:

```openmrs-core/api/src/main/java/org/openmrs/api/db/hibernate/HibernateOpenmrsDataDAO.java#L54-66
@Override
public List<T> getAll(boolean includeVoided) {
    Session session = sessionFactory.getCurrentSession();
    CriteriaBuilder cb = session.getCriteriaBuilder();
    CriteriaQuery<T> cq = cb.createQuery(mappedClass);
    Root<T> root = cq.from(mappedClass);

    if (!includeVoided) {
        // Adds WHERE voided = false to every query
        cq.where(cb.isFalse(root.get(getVoidedAttributeName(root))));
    }
    return session.createQuery(cq).getResultList();
}
```

This means: by default, every `getAll()` call hides voided records. You must explicitly pass `includeVoided=true` to see them (typically done by admin or unvoid workflows only).

---

## 14. Real-World Scenarios — When and Why Each Void Happens

```/dev/null/real-world.md#L1-45
┌──────────────────────────────────┬─────────────────────────────────────────────┐
│  Real Scenario                   │  What Gets Voided                           │
├──────────────────────────────────┼─────────────────────────────────────────────┤
│ Duplicate patient record found   │ voidPatient(wrongPatient, "Duplicate of    │
│ during registration              │  patient #1234")                            │
│                                  │ → cascades to ALL encounters, obs, orders   │
│                                  │ → users linked to person are RETIRED        │
├──────────────────────────────────┼─────────────────────────────────────────────┤
│ Nurse entered vitals on wrong    │ voidEncounter(encounter, "Wrong patient")   │
│ patient's chart                  │ → cascades to all Obs in encounter          │
│                                  │ → cascades to all Orders in encounter       │
├──────────────────────────────────┼─────────────────────────────────────────────┤
│ Clinician corrects a BP reading  │ saveObs(newObs, "Corrected transcription    │
│                                  │  error") → internally voids old Obs         │
│                                  │ → new Obs links to old via previousVersion  │
├──────────────────────────────────┼─────────────────────────────────────────────┤
│ Medication order entered in      │ voidOrder(order, "Entered in error")        │
│ error                            │ → order hidden from active orders list      │
│                                  │ → if it was a DISCONTINUE/REVISE order,     │
│                                  │   the previous order is re-activated        │
├──────────────────────────────────┼─────────────────────────────────────────────┤
│ Patient transferred to another   │ transferEncounter(encounter, newPatient)    │
│ patient's record                 │ → copies encounter to new patient           │
│                                  │ → voids original encounter                  │
│                                  │ → if visit had only that encounter,         │
│                                  │   the visit is also voided                  │
├──────────────────────────────────┼─────────────────────────────────────────────┤
│ Patient identifier was wrong     │ voidPatientIdentifier(id, "Typo in MRN")   │
│ (e.g., typo in medical #)        │ → identifier hidden; record kept for audit  │
└──────────────────────────────────┴─────────────────────────────────────────────┘
```

---


15. Complete End-to-End Flow Diagram (Full)

```/dev/null/complete-flow.md#L1-80
════════════════════════════════════════════════════════
        COMPLETE VOID LIFECYCLE (voidEncounter)
════════════════════════════════════════════════════════

[1] DEVELOPER/UI CALL
    encounterService.voidEncounter(encounter, "Wrong patient")
    │
    ▼
[2] SPRING AOP PROXY intercepts the call
    │  (every service bean is wrapped in a proxy at startup)
    │
    ├──[2a] AuthorizationAdvice.before()
    │         checks: does the caller have "Edit Encounters"?
    │         YES → continue
    │         NO  → throw APIAuthenticationException ✗ STOP
    │
    ├──[2b] RequiredDataAdvice.before()
    │         methodName.startsWith("void") → TRUE
    │         │
    │         ├── RequireVoidReasonVoidHandler.handle()  [order=1]
    │         │      reason blank? → throw IllegalArgumentException ✗ STOP
    │         │
    │         └── BaseVoidHandler.handle()               [order=default]
    │                sets: voided=true
    │                      voidedBy=currentUser
    │                      dateVoided=now
    │                      voidReason="Wrong patient"
    │
    │         then RequiredDataAdvice recursively walks all
    │         child collections (via reflection) on Encounter:
    │           → Set<Obs>     → BaseVoidHandler on each Obs
    │           → Set<Order>   → BaseVoidHandler on each Order
    │           (Set<EncounterProvider> is SKIPPED because of
    │            @DisableHandlers(VoidHandler.class))
    │
    ▼
[3] EncounterServiceImpl.voidEncounter() (real method runs)
    │  (by now encounter.voided is already TRUE from AOP)
    │
    │  explicit cascade to Obs (top-level only):
    │  for each Obs in encounter.getObsAtTopLevel(false):
    │      obsService.voidObs(obs, reason)
    │          → AOP runs VoidHandler on each Obs
    │
    │  explicit cascade to Orders:
    │  for each Order in encounter.getOrders():
    │      orderService.voidOrder(order, reason)
    │          → AOP runs VoidHandler on each Order
    │          → if order is DISCONTINUE/REVISE:
    │              re-activates the previousOrder (dateStopped=null)
    │
    │  saves the encounter to DB
    │      → saveEncounter(encounter)
    │          → VoidSaveHandler runs (fill dateVoided/voidedBy if null)
    │          → DB row: voided=1, voided_by=X, date_voided=T, void_reason="..."
    │
    ▼
[4] DATABASE STATE
    encounter row:  voided=1, void_reason="Wrong patient"
    obs rows:       voided=1 (same timestamp)
    order rows:     voided=1 (same timestamp)
    EncounterProvider rows: NOT voided (due to @DisableHandlers)
```

---

## 16. The `@DisableHandlers` and `@Independent` Annotations — Controlling Cascade Scope

Not every child collection should cascade void operations. OpenMRS gives you two escape hatches:

### `@DisableHandlers` — disable a specific handler type on a collection

Used in `Encounter` to prevent `EncounterProvider` from being auto-voided by the AOP recursion (the service handles provider logic separately):

```openmrs-core/api/src/main/java/org/openmrs/Encounter.java#L109-112
@OneToMany(mappedBy = "encounter", cascade = CascadeType.ALL)
@OrderBy("provider_id")
@DisableHandlers(handlerTypes = { VoidHandler.class })
private Set<EncounterProvider> encounterProviders = new LinkedHashSet<>();
```

### `@Independent` — skip the collection entirely from ALL handler recursion

Used in `Location` to prevent `LocationTag` references from being voided when a Location is voided (tags are shared metadata, not owned children):

```openmrs-core/api/src/main/java/org/openmrs/Location.java#L143-149
@ManyToMany(fetch = FetchType.LAZY)
@JoinTable(name = "location_tag_map", ...)
@Independent
private Set<LocationTag> tags;
```

The AOP recursion checks for both:

```openmrs-core/api/src/main/java/org/openmrs/aop/RequiredDataAdvice.java#L302-312
for (Field field : allInheritedFields) {
    // skip if @Independent — don't touch this collection at all
    if (Reflect.isAnnotationPresent(openmrsObjectClass, field.getName(), Independent.class)) {
        continue;
    }
    // skip if this handler type is disabled via @DisableHandlers
    if (reflect.isCollectionField(field) && !isHandlerMarkedAsDisabled(handlerType, field)) {
        Collection<OpenmrsObject> childCollection = getChildCollection(openmrsObject, field);
        // recurse into children...
    }
}
```

---

## 17. Immutability Meets Voiding — The Hibernate Interceptors

For `Obs` and `Order`, data immutability is enforced at the **Hibernate flush level** by `ImmutableEntityInterceptor`. These classes cannot be edited freely — but voiding is explicitly **allowed** as a mutable operation.

### `ImmutableObsInterceptor`

```openmrs-core/api/src/main/java/org/openmrs/api/db/hibernate/ImmutableObsInterceptor.java#L22-50
@Component("immutableObsInterceptor")
public class ImmutableObsInterceptor extends ImmutableEntityInterceptor {

    // These are the ONLY fields you're allowed to change on an existing Obs
    private static final String[] MUTABLE_PROPERTY_NAMES = new String[] {
        "voided", "dateVoided", "voidedBy", "voidReason",  // ← void fields explicitly whitelisted
        "groupMembers"
    };

    @Override
    protected Class<?> getSupportedType() { return Obs.class; }

    @Override
    protected String[] getMutablePropertyNames() { return MUTABLE_PROPERTY_NAMES; }

    @Override
    protected boolean ignoreVoidedOrRetiredObjects() {
        return true; // once an Obs is already voided, it can be edited freely
    }
}
```

### `ImmutableOrderInterceptor`

```openmrs-core/api/src/main/java/org/openmrs/api/db/hibernate/ImmutableOrderInterceptor.java#L22-44
@Component("immutableOrderInterceptor")
public class ImmutableOrderInterceptor extends ImmutableEntityInterceptor {

    private static final String[] MUTABLE_PROPERTY_NAMES = new String[] {
        "dateStopped",
        "voided", "dateVoided", "voidedBy", "voidReason",  // ← void fields whitelisted
        "changedBy", "dateChanged",
        "patient", "fulfillerStatus", "fulfillerComment", "accessionNumber"
    };
    // ...
    @Override
    protected boolean ignoreVoidedOrRetiredObjects() {
        return true; // once voided, can be freely updated
    }
}
```

This creates an important guarantee:

```/dev/null/immutability-rule.md#L1-12
┌──────────────────────────────────────────────────────────┐
│  For Obs and Orders:                                     │
│                                                          │
│  - You CANNOT edit clinical/value fields (immutable)     │
│  - You CAN only:                                         │
│    • void the record (voided/dateVoided/voidedBy/reason) │
│    • link group members (obsGroupMembers)                │
│    • stop an order (dateStopped)                         │
│  - The only way to "edit" an Obs is:                     │
│    create a new one + void the old one                   │
└──────────────────────────────────────────────────────────┘
```

---

## 18. Voiding and the Hibernate Envers Audit Trail

`BaseOpenmrsData` is annotated with `@Audited` from Hibernate Envers:

```openmrs-core/api/src/main/java/org/openmrs/BaseOpenmrsData.java#L31-33
@MappedSuperclass
@Audited
public abstract class BaseOpenmrsData extends BaseOpenmrsObject implements OpenmrsData {
```

This means every single database change — including a void — is logged in a **separate audit table** (`_AUD` tables). When you void an Obs, you get:

```/dev/null/envers-log.sql#L1-12
-- In obs table (main):
obs_id=101, concept=WEIGHT, value=70, voided=1, void_reason="Correction"

-- In obs_AUD table (Hibernate Envers history):
obs_id=101, REV=55, REVTYPE=1,  -- revision 55, type 1 = MODIFY
  voided=0, ... (old snapshot before void)

obs_id=101, REV=56, REVTYPE=1,  -- revision 56, type 1 = MODIFY
  voided=1, void_reason="Correction", voided_by=3, date_voided=...
```

So there are **two overlapping audit mechanisms**:
1. **Voiding** — soft-delete with reason (instantly visible to querying code)
2. **Envers** — full revision history of every field change, including the act of voiding

---

## 19. The Cohort Side Effect — Voiding Cascades to Cohort Memberships

When a Patient is voided, it also notifies `CohortService` to void the patient's membership in any cohorts — because a voided patient should not appear in clinical cohorts either:

```openmrs-core/api/src/main/java/org/openmrs/api/impl/CohortServiceImpl.java#L246-257
@Override
public void notifyPatientVoided(Patient patient) throws APIException {
    List<CohortMembership> memberships = Context.getCohortService()
            .getCohortMemberships(patient.getPatientId(), null, false);
    memberships.forEach(m -> {
        m.setVoided(patient.getVoided());
        m.setDateVoided(patient.getDateVoided());
        m.setVoidedBy(patient.getVoidedBy());
        m.setVoidReason(patient.getVoidReason());
        dao.saveCohortMembership(m);
    });
}
```

And the unvoid counterpart precisely reverses only the memberships that were voided at the same time as the patient:

```openmrs-core/api/src/main/java/org/openmrs/api/impl/CohortServiceImpl.java#L262-272
@Override
public void notifyPatientUnvoided(Patient patient, User originallyVoidedBy, Date originalDateVoided) {
    List<CohortMembership> memberships = getCohortMemberships(patient.getPatientId(), null, true);
    List<CohortMembership> toUnvoid = memberships.stream().filter(
        m -> m.getVoided()
          && m.getVoidedBy().equals(originallyVoidedBy)
          && OpenmrsUtil.compare(
               truncateToSeconds(m.getDateVoided()),
               truncateToSeconds(originalDateVoided)) == 0)
        .collect(Collectors.toList());
    // ... then unvoids each one
}
```

---

## 20. The Order Unvoid — The Most Complex Case

Unvoiding an Order has side effects beyond simply clearing the void flag, because orders have a `previousOrder` chain:

```openmrs-core/api/src/main/java/org/openmrs/api/impl/OrderServiceImpl.java#L536-548
@Override
public Order unvoidOrder(Order order) throws APIException {
    Order previousOrder = order.getPreviousOrder();
    if (previousOrder != null && isDiscontinueOrReviseOrder(order)) {
        // Cannot unvoid if the order you're trying to reinstate
        // (previousOrder) is no longer active
        if (!previousOrder.isActive()) {
            final String action = DISCONTINUE == order.getAction() ? "discontinuation" : "revision";
            throw new CannotUnvoidOrderException(action);
        }
        // Re-stop the previous order (it was re-activated when we voided this)
        stopOrder(previousOrder, aMomentBefore(order.getDateActivated()), false);
    }
    return saveOrderInternal(order, null);
}
```

This is why:

```/dev/null/order-void-logic.md#L1-25
Order chain:  [Drug A] ──REVISE──► [Drug A v2] ──DISCONTINUE──► [stopped]

Case: voidOrder(DISCONTINUE order)
  → Drug A v2's dateStopped is cleared (re-activated)
  → DISCONTINUE order is marked voided

Case: unvoidOrder(DISCONTINUE order)
  → checks: is Drug A v2 still active? (it was re-activated by the void)
  → if YES: re-stops Drug A v2 (restores the discontinuation)
  → if NO:  throws CannotUnvoidOrderException
              (e.g., Drug A v2 expired or was superseded while the
               DISCONTINUE was voided — can't undo history)
```

---

## 21. The VoidSaveHandler Safety Net

There is a subtle scenario: what if someone manually sets `voided=true` on an object and then calls `savePatient()` (instead of calling `voidPatient()`)? This bypasses the VoidHandler chain entirely. `VoidSaveHandler` is the safety net — it runs on every `save*` call and ensures the metadata stays consistent:

```/dev/null/voidsavehandler.md#L1-25
savePatient(patient) is called
  │
  ▼
RequiredDataAdvice.before() sees "save*"
  → runs all SaveHandler implementations including VoidSaveHandler
  │
  ▼
VoidSaveHandler.handle():
  if (patient.getVoided() == TRUE):
      if voidedBy is null  → fill in currentUser
      if dateVoided is null → fill in now
      (but does NOT set voidReason — that must be set before calling save)
  else (voided == FALSE):
      CLEARS voidedBy = null
      CLEARS dateVoided = null
      CLEARS voidReason = null
      → This ensures: if someone un-sets voided=false manually,
        all the void metadata is wiped clean too
```

---

## 22. Full Voiding Architecture — Master Diagram

```/dev/null/master-diagram.md#L1-100
═══════════════════════════════════════════════════════════════════════
           VOIDING ARCHITECTURE IN OPENMRS — MASTER DIAGRAM
═══════════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────────┐
  │                     INTERFACE LAYER                             │
  │                                                                 │
  │  Voidable interface          Retireable interface               │
  │  ├─ getVoided()              ├─ getRetired()                    │
  │  ├─ setVoided()              ├─ setRetired()                    │
  │  ├─ getVoidedBy()            ├─ getRetiredBy()                  │
  │  ├─ getDateVoided()          ├─ getDateRetired()                │
  │  └─ getVoidReason()          └─ getRetireReason()               │
  └───────────────────────┬─────────────────────────────────────────┘
                          │ implemented by
  ┌───────────────────────▼─────────────────────────────────────────┐
  │                    BASE CLASSES                                 │
  │                                                                 │
  │  BaseOpenmrsData (@Audited)    BaseOpenmrsMetadata              │
  │  ├─ voided (DB: voided)        ├─ retired (DB: retired)        │
  │  ├─ voidedBy (voided_by)       ├─ retiredBy (retired_by)       │
  │  ├─ dateVoided (date_voided)   ├─ dateRetired (date_retired)   │
  │  └─ voidReason (void_reason)   └─ retireReason (retire_reason) │
  └───────────────────────┬─────────────────────────────────────────┘
                          │ extended by
  ┌───────────────────────▼─────────────────────────────────────────┐
  │                   DOMAIN ENTITIES                               │
  │  Visit │ Encounter │ Obs │ Order │ Patient │ Person │ Cohort   │
  │  PatientIdentifier │ PersonName │ PersonAddress │ ...           │
  └─────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │                      AOP LAYER                                  │
  │                                                                 │
  │  RequiredDataAdvice (MethodBeforeAdvice)                        │
  │  ─────────────────────────────────────────────────             │
  │  Detects:  void*()   → runs VoidHandler chain                   │
  │            unvoid*() → runs UnvoidHandler chain                 │
  │            save*()   → runs SaveHandler chain (incl VoidSave)   │
  │                                                                 │
  │  Recursion: walks all child collections via reflection          │
  │             skips @Independent, skips @DisableHandlers          │
  └────────────────────────┬────────────────────────────────────────┘
                           │ selects handlers via HandlerUtil
  ┌────────────────────────▼────────────────────────────────────────┐
  │                   VOID HANDLER CHAIN                            │
  │                                                                 │
  │  order=1  RequireVoidReasonVoidHandler                          │
  │           supports: Patient, Encounter, Obs, Cohort, Order      │
  │           action: validates reason is not blank                  │
  │                                                                 │
  │  default  BaseVoidHandler                                       │
  │           supports: ALL Voidable                                │
  │           action: sets voided=T, voidedBy, dateVoided, reason   │
  │                                                                 │
  │  special  PersonVoidHandler                                     │
  │           supports: Person                                      │
  │           action: sets personVoided* fields, retires Users      │
  │                                                                 │
  │  special  PatientDataVoidHandler                                │
  │           supports: Patient                                     │
  │           action: voids all encounters (→ cascade to obs+orders)│
  │                   notifies CohortService                        │
  │                                                                 │
  │  special  VisitVoidHandler                                      │
  │           supports: Visit                                       │
  │           action: voids all encounters in the visit             │
  └────────────────────────┬────────────────────────────────────────┘
                           │ mirrors
  ┌────────────────────────▼────────────────────────────────────────┐
  │                  UNVOID HANDLER CHAIN                           │
  │                                                                 │
  │  order=10  BaseUnvoidHandler                                    │
  │            clears voided, voidedBy, dateVoided, voidReason      │
  │            ONLY if dateVoided matches parent's dateVoided        │
  │                                                                 │
  │  special   PersonUnvoidHandler    — clears personVoided*        │
  │  special   PatientDataUnvoidHandler — unvoids encounters/orders │
  │            that share same dateVoided+voidedBy as patient       │
  │  special   VisitUnvoidHandler     — unvoids matching encounters  │
  └────────────────────────┬────────────────────────────────────────┘
                           │
  ┌────────────────────────▼────────────────────────────────────────┐
  │                    DAO LAYER                                    │
  │                                                                 │
  │  HibernateOpenmrsDataDAO.getAll(includeVoided=false)            │
  │  → adds WHERE voided = false to every default query             │
  │                                                                 │
  │  ImmutableObsInterceptor  (Hibernate-level)                     │
  │  → blocks field edits on Obs; ALLOWS voided/voidReason changes  │
  │                                                                 │
  │  ImmutableOrderInterceptor (Hibernate-level)                    │
  │  → blocks field edits on Order; ALLOWS void fields              │
  │                                                                 │
  │  Hibernate Envers @Audited                                      │
  │  → every void/unvoid creates a revision in *_AUD tables         │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 23. The Difference Between Void and Retire — A Quick Comparison

```/dev/null/void-vs-retire.md#L1-30
┌─────────────────────┬──────────────────────────────┬───────────────────────────────┐
│  Feature            │  VOID                        │  RETIRE                       │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  Used for           │  Clinical data (patient-     │  Metadata / configuration     │
│                     │  specific records)           │  (shared definitions)         │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  Examples           │  Patient, Visit, Encounter,  │  Concept, EncounterType,      │
│                     │  Obs, Order, Cohort          │  Form, Drug, User, Role       │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  Cascades to        │  YES — voids all child data  │  NO — existing references     │
│  children?          │  (encounters→obs→orders)     │  remain valid                 │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  Invalidates        │  YES — references from other │  NO — old data using this     │
│  references?        │  non-voided data are invalid │  metadata stay valid          │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  Reason required?   │  YES (for Patient,           │  YES                          │
│                     │  Encounter, Obs, Order)      │                               │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  Reversible?        │  YES — unvoidXxx()           │  YES — unretireXxx()          │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  DB columns         │  voided, voided_by,          │  retired, retired_by,         │
│                     │  date_voided, void_reason    │  date_retired, retire_reason  │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  DB row deleted?    │  NO — row stays forever      │  NO — row stays forever       │
└─────────────────────┴──────────────────────────────┴───────────────────────────────┘
```

---

## 24. The Database Schema — What It Looks Like at Rest

For any clinical table like `encounter`:

```/dev/null/encounter-schema.sql#L1-20
CREATE TABLE encounter (
    encounter_id    INT(11) NOT NULL,
    encounter_type  INT(11) NOT NULL,
    patient_id      INT(11) NOT NULL,
    visit_id        INT(11),
    -- ... clinical fields ...
    creator         INT(11) NOT NULL,
    date_created    DATETIME NOT NULL,
    changed_by      INT(11),
    date_changed    DATETIME,

    -- ← THE VOID COLUMNS (present in every voidable table)
    voided          TINYINT(1) NOT NULL DEFAULT 0,
    voided_by       INT(11),        -- FK to users
    date_voided     DATETIME,
    void_reason     VARCHAR(255),

    uuid            VARCHAR(38) NOT NULL UNIQUE
);
```

After voiding an encounter, the row looks like:

```/dev/null/voided-row.sql#L1-8
-- BEFORE void:
encounter_id=42, patient_id=7, voided=0, voided_by=NULL, date_voided=NULL, void_reason=NULL

-- AFTER voidEncounter(encounter, "Wrong patient"):
encounter_id=42, patient_id=7,
  voided=1,
  voided_by=3,                          ← user ID who voided it
  date_voided=2024-01-15 10:30:00,
  void_reason="Wrong patient"
```

The row is **never deleted**. The `voided=1` flag is the sole difference. All normal queries add `WHERE voided = 0` automatically via `HibernateOpenmrsDataDAO`.

---

## 25. Key Design Principles — Why It Was Built This Way

1. **Auditability is non-negotiable** in healthcare. Laws in many countries require that clinical records be preserved for 7–20+ years. You cannot `DELETE` a blood pressure reading.

2. **Mistakes happen** — nurses enter data on the wrong patient, pharmacists order the wrong drug. The void-and-replace pattern ensures these errors are correctable *without erasing evidence* that the error happened.

3. **The timestamp matching rule** (only unvoid children with the same `dateVoided` as the parent) prevents "collateral unvoiding" — if an Obs was already voided *before* the parent Encounter was voided, unvoiding the Encounter must not accidentally restore that Obs.

4. **The immutability + void-and-replace for Obs** is the cleanest solution to medical record integrity: the record of what was believed to be true at a given time is never altered. Only new, corrected records are added, with the old marked void.

5. **The handler pattern** (AOP + `@Handler`) makes the system extensible: a module can add its own `VoidHandler` for a custom entity type, and it will automatically be picked up by `RequiredDataAdvice` without any changes to the core.