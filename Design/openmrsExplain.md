🏥 What is EMR & How OpenMRS is Used in the Real World

## 🌍 What is an EMR System?

**EMR = Electronic Medical Record** system. It's a **digital version of a patient's paper chart** inside a hospital or clinic. It replaces paper registers, paper prescriptions, and paper lab result folders with a centralized digital system.

**OpenMRS specifically** is an open-source EMR platform built for **resource-constrained environments** — primarily used in:

| Country / Region | Real Deployments |
|---|---|
| 🇰🇪 Kenya | AMPATH (Academic Model Providing Access to Healthcare) — 150,000+ HIV patients |
| 🇿🇦 South Africa | Multiple HIV/TB clinics |
| 🇺🇬 Uganda | IDI (Infectious Diseases Institute) |
| 🇷🇼 Rwanda | Partners In Health |
| 🇮🇳 India | Community health programs |
| 🇭🇹 Haiti | Partners In Health post-earthquake |
| 🇵🇭 Philippines | TB/HIV programs |
| 🇹🇿 Tanzania | HIV treatment programs |

**What kind of hospitals use it?**
- District hospitals with limited IT budget
- HIV/AIDS treatment centers
- TB treatment clinics
- Community health centers in rural areas
- NGO-run clinics in developing countries

**What does it replace?**
- Paper patient registers (name books at reception)
- Paper medical record folders stored in filing cabinets
- Paper lab request slips
- Paper drug prescription books
- Paper tally sheets for statistics

---

# 👨‍👩‍👧 Real-World Hospital Flow — Step by Step

Let's follow a real patient **"John Doe"** from the moment he walks into the hospital gate all the way through treatment. I'll show you exactly which OpenMRS code handles each step.

---

## 🚪 STEP 1: Patient Walks Into the Hospital

**Real World:** John Doe walks to the **reception/registration desk** (called "OPD Registration" — Outpatient Department).

**OpenMRS Role at this point:** Nothing yet. The receptionist is about to open the OpenMRS O3 frontend (a React web app) in their browser.

---

## 🔐 STEP 2: Receptionist Logs In

**Real World:** The receptionist types their username and password.

**OpenMRS Code Flow:**

```
POST /openmrs/ws/rest/v1/session
    ↓
HibernateContextDAO.authenticate(credentials)
    ↓
Context.authenticate("receptionist_user", "password")
    ↓
UserContext created → stored in ThreadLocal per HTTP session
    ↓
User object loaded with their Roles ("Registration Clerk" role)
    ↓
Roles have Privileges:  ADD_PATIENTS, GET_PATIENTS, ADD_VISITS, ADD_ENCOUNTERS
```

The `UserContext` class holds this per-thread:

```openmrs-core/api/src/main/java/org/openmrs/api/context/UserContext.java#L56-75
private User user = null;
private final List<String> proxies = Collections.synchronizedList(new ArrayList<>());
private Locale locale = null;
private Role authenticatedRole = null;
private Role anonymousRole = null;
```

---

## 🔍 STEP 3: Receptionist Searches for Existing Patient

**Real World:** Before registering a new patient, the receptionist types the patient's name to check if they already exist in the system (to avoid duplicates).

**OpenMRS Code Flow:**

```
GET /openmrs/ws/rest/v1/patient?q=John+Doe
    ↓
PatientService.getPatients("John Doe", null, null, false)
    ↓
AOP: AuthorizationAdvice checks @Authorized({"Get Patients"})
    ↓
HibernatePatientDAO.getPatients()
    ↓
Hibernate Search (Lucene) does full-text search on:
    - PersonName.givenName
    - PersonName.familyName
    - PersonName.middleName
    using SOUNDEX, PHRASE, START, ANYWHERE analyzers
    ↓
Returns List<Patient> matching "John Doe"
```

The `PersonName` class has these full-text search indexes:

```openmrs-core/api/src/main/java/org/openmrs/PersonName.java#L62-80
@FullTextField(name = "givenNameExact", analyzer = SearchAnalysis.EXACT_ANALYZER)
@FullTextField(name = "givenNameStart", analyzer = SearchAnalysis.START_ANALYZER, ...)
@FullTextField(name = "givenNameAnywhere", analyzer = SearchAnalysis.ANYWHERE_ANALYZER, ...)
@FullTextField(name = "givenNameSoundex", analyzer = SearchAnalysis.SOUNDEX_ANALYZER)
private String givenName;
```

> **Why SOUNDEX?** If the receptionist types "Jon Do" instead of "John Doe", SOUNDEX-based search still finds the correct patient because it matches by phonetic similarity. This is critical in African/Asian clinics where names may be spelled inconsistently.

---

## 📝 STEP 4: New Patient Registration

**Real World:** John is a first-time patient. The receptionist fills in:
- Full name (Given name, Middle name, Family name)
- Date of birth (or estimated age)
- Gender
- Address (village, district, county)
- Phone number
- Next of kin
- National ID or hospital number

**OpenMRS Data Objects Created:**

### 4a. The `Person` object (demographic base)

```openmrs-core/api/src/main/java/org/openmrs/Person.java#L59-103
protected Integer personId;
private Set<PersonAddress> addresses;   // address1-14 fields
private Set<PersonName> names;          // givenName, middleName, familyName
private Set<PersonAttribute> attributes; // phone, next of kin, etc.
private String gender;
private Date birthdate;
private Boolean birthdateEstimated;     // true if receptionist guessed age
private Boolean dead;
private Date deathDate;
```

### 4b. The `Patient` object (extends `Person`)

```openmrs-core/api/src/main/java/org/openmrs/Patient.java#L33-42
public class Patient extends Person {
    private Integer patientId;
    private String allergyStatus = Allergies.UNKNOWN;
    private Set<PatientIdentifier> identifiers;  // THE HOSPITAL NUMBER
```

### 4c. The `PatientIdentifier` — the Hospital Number

This is critical. Every patient gets a unique ID. The **type** of ID depends on the hospital's configuration:

```openmrs-core/api/src/main/java/org/openmrs/PatientIdentifier.java#L75-100
private String identifier;          // e.g. "AMRS-00012345" or "KNH-2024-00456"
private PatientIdentifierType identifierType;  // e.g. "OpenMRS ID", "National ID", "NHIF Number"
private Location location;          // which clinic issued this ID
private Boolean preferred = false;  // the primary identifier
```

The `PatientIdentifierType` defines the rules:

```openmrs-core/api/src/main/java/org/openmrs/PatientIdentifierType.java#L90-100
private String format;              // regex validation pattern, e.g. "\\d{5}"
private Boolean required;           // is this ID mandatory?
private String validator;           // e.g. "LuhnIdentifierValidator" (check digit)
private UniquenessBehavior uniquenessBehavior; // UNIQUE, NON_UNIQUE, or LOCATION-based
```

> **Real World Example:** In AMPATH Kenya, each patient gets an **OpenMRS ID** like `AMRS-00012345`. The Luhn check digit algorithm validates it (same algorithm used in credit card numbers). This prevents typos — if you mistype the number, the system rejects it.

### 4d. The `PersonAddress` — where the patient lives

```openmrs-core/api/src/main/java/org/openmrs/PersonAddress.java#L56-100
private String address1;   // house/street
private String address2;   // village
private String address3;   // sub-location
private String address4;   // location
private String address5;   // sub-district
private String cityVillage; // city or village name
private String stateProvince;
private String country;
private String postalCode;
```

> **14 address fields** exist because address structures vary dramatically between Kenya (Village → Sub-location → Location → District), India (House → Street → Ward → City → State), and Haiti. Each deployment configures which fields matter.

### 4e. The API Call that saves everything

```
POST /openmrs/ws/rest/v1/patient
{
  "person": {
    "names": [{"givenName":"John","familyName":"Doe"}],
    "gender": "M",
    "birthdate": "1985-06-15",
    "addresses": [{"address1":"Plot 12","cityVillage":"Eldoret","country":"Kenya"}]
  },
  "identifiers": [{
    "identifier": "AMRS-00012345",
    "identifierType": "OpenMRS ID",
    "location": "Eldoret Clinic",
    "preferred": true
  }]
}
```

**Backend execution flow:**

```
PatientService.savePatient(patient)
    ↓
AOP: AuthorizationAdvice → checks "Add Patients" privilege
    ↓
AOP: RequiredDataAdvice → sets creator=receptionist, dateCreated=now, uuid=generated
    ↓
PatientServiceImpl.savePatient()
    ├── requireAppropriatePatientModificationPrivilege(patient)
    ├── checkPatientIdentifiers(patient)    ← validates format + Luhn check
    ├── setPreferredPatientIdentifier(patient)
    ├── setPreferredPatientName(patient)
    ├── setPreferredPatientAddress(patient)
    └── dao.savePatient(patient)
            ↓
        HibernatePatientDAO
            ↓
        Hibernate → INSERT INTO person (...) VALUES (...)
        Hibernate → INSERT INTO patient (...) VALUES (...)
        Hibernate → INSERT INTO person_name (...) VALUES (...)
        Hibernate → INSERT INTO person_address (...) VALUES (...)
        Hibernate → INSERT INTO patient_identifier (...) VALUES (...)
        Hibernate Envers → INSERT INTO person_aud, patient_aud (audit tables)
            ↓
        @Transactional COMMIT
```

The `savePatient` validation in `PatientServiceImpl`:

```openmrs-core/api/src/main/java/org/openmrs/api/impl/PatientServiceImpl.java#L131-150
public Patient savePatient(Patient patient) throws APIException {
    requireAppropriatePatientModificationPrivilege(patient);
    if (!patient.getVoided() && patient.getIdentifiers().size() == 1) {
        patient.getPatientIdentifier().setPreferred(true);
    }
    if (!patient.getVoided()) {
        checkPatientIdentifiers(patient);  // validates ALL identifiers
    }
    setPreferredPatientIdentifier(patient);
    setPreferredPatientName(patient);
    setPreferredPatientAddress(patient);
    return dao.savePatient(patient);
}
```

**Result:** John Doe now has a **patient record** in the database. He gets a printed card/wristband with his Hospital Number `AMRS-00012345`.

---

## 🏥 STEP 5: Patient is Sent to the Waiting Area — Visit Created

**Real World:** After registration, the receptionist "checks in" John for today's visit. This is like opening a new file for today's appointment.

**OpenMRS Object Created: `Visit`**

```openmrs-core/api/src/main/java/org/openmrs/Visit.java#L42-72
@Entity
@Table(name = "visit")
public class Visit {
    private Patient patient;             // → John Doe
    private VisitType visitType;         // e.g. "Outpatient", "Inpatient", "Emergency"
    private Location location;           // → "Eldoret Clinic, OPD"
    private Date startDatetime;          // → 2024-01-15 08:30:00
    private Date stopDatetime;           // null until patient leaves
    private Set<Encounter> encounters;   // all interactions this visit
```

```
POST /openmrs/ws/rest/v1/visit
{
  "patient": "patient-uuid",
  "visitType": "Outpatient",
  "location": "Eldoret Clinic",
  "startDatetime": "2024-01-15T08:30:00"
}
    ↓
VisitService.saveVisit(visit)
    ↓
AOP: checks "Add Visits" privilege
    ↓
HibernateVisitDAO → INSERT INTO visit (...)
```

> **Think of Visit as:** Opening a new folder/file for today. All the papers (encounters, observations, orders) from today go inside this folder.

---

## 🩺 STEP 6: Triage — Nurse Takes Vitals

**Real World:** A triage nurse takes John's:
- Weight: 72 kg
- Height: 175 cm
- Blood pressure: 120/80
- Temperature: 37°C
- Pulse: 72 bpm
- Oxygen saturation: 98%

**OpenMRS Objects Created:**

### 6a. `Encounter` (the triage interaction)

```openmrs-core/api/src/main/java/org/openmrs/Encounter.java#L59-112
@Entity
@Table(name = "encounter")
public class Encounter {
    private Date encounterDatetime;       // → 2024-01-15 08:45:00
    private Patient patient;              // → John Doe
    private Location location;            // → "Triage Room, Eldoret Clinic"
    private EncounterType encounterType;  // → "Triage"
    private Visit visit;                  // → today's Visit object
    private Set<Obs> obs;                 // → weight, BP, temp etc.
    private Set<EncounterProvider> encounterProviders; // → Nurse Jane
```

### 6b. `Obs` objects — each vital sign is ONE Obs

This is the EAV model at work. Each measurement is a separate row:

```
Obs 1: concept="WEIGHT (kg)"     → obsDatetime=08:45  → valueNumeric=72
Obs 2: concept="HEIGHT (cm)"     → obsDatetime=08:45  → valueNumeric=175
Obs 3: concept="SYSTOLIC BP"     → obsDatetime=08:45  → valueNumeric=120
Obs 4: concept="DIASTOLIC BP"    → obsDatetime=08:45  → valueNumeric=80
Obs 5: concept="TEMPERATURE (C)" → obsDatetime=08:45  → valueNumeric=37
Obs 6: concept="PULSE"           → obsDatetime=08:45  → valueNumeric=72
Obs 7: concept="O2 SAT"          → obsDatetime=08:45  → valueNumeric=98
```

The `Obs` object:

```openmrs-core/api/src/main/java/org/openmrs/Obs.java#L90-110
protected Integer obsId;
protected Concept concept;     // WHAT was measured (e.g. "WEIGHT")
protected Date obsDatetime;
protected Encounter encounter; // WHEN/WHERE (the triage encounter)
protected Person person;       // WHO (John Doe)
protected Location location;
// Only ONE of these value fields is populated:
protected Double valueNumeric;    // for numbers (weight, BP)
protected String valueText;       // for free text (notes)
protected Concept valueCoded;     // for coded answers (Yes/No, diagnosis codes)
protected Date valueDatetime;     // for date values
protected String valueComplex;    // for files (images, audio)
```

> **The `ImmutableObsInterceptor`** means once an Obs is saved, you **cannot edit it**. If the nurse entered weight as 72 but it was actually 27 (a typo), the system **voids** the old Obs and creates a **new** one with `previousVersion` pointing to the old one. This preserves the full audit trail — critical for medical records.

```openmrs-core/api/src/main/java/org/openmrs/api/db/hibernate/ImmutableObsInterceptor.java#L26-28
private static final String[] MUTABLE_PROPERTY_NAMES = new String[] { 
    "voided", "dateVoided", "voidedBy", "voidReason", "groupMembers" 
};
```

---

## 👨‍⚕️ STEP 7: Doctor Consultation — Encounter + Diagnosis + Orders

**Real World:** John sees Dr. Smith. Dr. Smith:
1. Reviews John's history (reads previous Encounters/Obs)
2. Records the chief complaint
3. Does a physical examination
4. Makes a diagnosis: "Hypertension"
5. Orders a blood test: "CBC + Metabolic Panel"
6. Prescribes medication: "Amlodipine 5mg once daily for 30 days"

**OpenMRS Objects Created:**

### 7a. A new `Encounter` of type "Consultation"

```
encounterType = "Consultation"
encounterProvider = Dr. Smith (role = "Attending Physician")
visit = today's Visit
```

### 7b. `Obs` for chief complaint and examination findings

```
Obs: concept="CHIEF COMPLAINT"  → valueText = "Headache and dizziness for 3 days"
Obs: concept="CLINICAL NOTES"   → valueText = "BP elevated, fundoscopy normal..."
```

### 7c. `Diagnosis` object

```openmrs-core/api/src/main/java/org/openmrs/Diagnosis.java
diagnosis.encounter = consultation encounter
diagnosis.condition = Hypertension (coded as a Concept from ICD-10)
diagnosis.certainty = CONFIRMED
diagnosis.rank = 1 (primary diagnosis)
```

### 7d. `DrugOrder` — the prescription

```openmrs-core/api/src/main/java/org/openmrs/DrugOrder.java
orderType = DRUG
drug = Amlodipine 5mg (a Drug entity linked to its Concept)
dose = 5
doseUnits = mg
frequency = "Once daily"
duration = 30
durationUnits = Days
route = ORAL
dateActivated = 2024-01-15
autoExpireDate = 2024-02-14  (calculated automatically by OrderServiceImpl)
careSetting = OUTPATIENT
```

The `Order` base class hierarchy:

```
Order (base: patient, encounter, orderer, dateActivated, urgency)
    └── DrugOrder  (drug, dose, frequency, duration, route)
    └── TestOrder  (laboratoryConcept, specimenSource)
    └── ReferralOrder (specialist, referralReason)
    └── ServiceOrder (service concept)
```

### 7e. `TestOrder` — the lab test

```
orderType = TEST
concept = "Complete Blood Count"
specimenSource = "BLOOD" (a Concept)
urgency = ROUTINE
```

> **The `ImmutableOrderInterceptor`** works the same as for Obs. You cannot edit an order once placed. To change it, you create a new order with `Action=REVISE` and the old order is automatically discontinued.

---

## 🧪 STEP 8: Lab Processes the Test

**Real World:** The lab technician receives the printed lab request, collects blood, runs the CBC test, and enters results.

**OpenMRS Objects Created:** More `Obs` linked to a "Lab Results" `Encounter`:

```
Obs: concept="WBC"          → valueNumeric=7.2  (reference range: 4.5-11.0)
Obs: concept="RBC"          → valueNumeric=4.8
Obs: concept="HEMOGLOBIN"   → valueNumeric=14.2
Obs: concept="PLATELETS"    → valueNumeric=250
```

Each `Obs` can have an `ObsReferenceRange` (normal range) for automatic flagging:

```openmrs-core/api/src/main/java/org/openmrs/ObsReferenceRange.java
private Double hiNormal;    // upper normal limit
private Double lowNormal;   // lower normal limit
private Double hiCritical;  // critical high (alert!)
private Double lowCritical; // critical low (alert!)
```

---

## 💊 STEP 9: Pharmacy Dispenses Medication

**Real World:** John goes to the pharmacy with his prescription. The pharmacist dispenses Amlodipine.

**OpenMRS Object Created: `MedicationDispense`**

```openmrs-core/api/src/main/java/org/openmrs/MedicationDispense.java
patient = John Doe
drug = Amlodipine 5mg
quantity = 30 tablets
quantityUnits = tablets
datesPrepared = 2024-01-15
wasSubstituted = false
dispensingLocation = pharmacy
```

---

## 🎗️ STEP 10: Patient Enrolled in a Care Program

**Real World:** Since John has hypertension, he's enrolled in the "Hypertension Management Program" so that he gets scheduled follow-up appointments and his progress is tracked.

**OpenMRS Objects: `PatientProgram` + `PatientState`**

```openmrs-core/api/src/main/java/org/openmrs/PatientProgram.java#L53-76
private Patient patient;          // John Doe
private Program program;          // "Hypertension Management"
private Location location;        // Eldoret Clinic
private Date dateEnrolled;        // 2024-01-15
private Date dateCompleted;       // null (still active)
private Set<PatientState> states; // current workflow state
```

A `Program` has a `ProgramWorkflow` with `ProgramWorkflowState`s:

```
Program: "Hypertension Management"
    Workflow: "Treatment Status"
        States:
            → "Initial Assessment" (current)
            → "On Treatment"
            → "Controlled" 
            → "Uncontrolled - Refer"
            → "Lost to Follow-up"
            → "Transferred Out"
            → "Deceased"
```

---

## 🔄 STEP 11: Patient Comes Back — Follow-up Visit

**Real World:** John returns 2 weeks later. The receptionist scans his patient card number.

```
GET /openmrs/ws/rest/v1/patient?identifier=AMRS-00012345
    ↓
PatientService.getPatients(null, "AMRS-00012345", null, true)
    ↓
PatientIdentifier search (exact match on identifier field)
    ↓
Returns John Doe's full patient object instantly
    ↓
Doctor sees:
    - Previous visit (Jan 15)
    - Vitals recorded (BP 120/80)
    - Diagnosis (Hypertension)
    - Active medications (Amlodipine 5mg)
    - Lab results (CBC normal)
```

A new `Visit` is created, a new `Encounter`, and new `Obs` for today's BP reading. The system now has a **longitudinal record** — all of John's visits, encounters, observations, and orders over time.

---

## 📊 The Complete Data Relationship Map

```
Patient (John Doe)  ←── Person (demographics)
    │   UUID: abc-123
    │   Identifier: AMRS-00012345
    │
    ├── Visit (Jan 15, Outpatient)
    │       │
    │       ├── Encounter: Triage (08:45)
    │       │       ├── Obs: Weight = 72kg
    │       │       ├── Obs: BP Systolic = 120
    │       │       └── Obs: BP Diastolic = 80
    │       │
    │       ├── Encounter: Consultation (09:15)
    │       │       ├── Obs: Chief Complaint = "Headache"
    │       │       ├── Diagnosis: Hypertension (CONFIRMED, rank=1)
    │       │       ├── Order: CBC Test (TestOrder)
    │       │       └── Order: Amlodipine 5mg x30 (DrugOrder)
    │       │
    │       └── Encounter: Lab Results (11:00)
    │               ├── Obs: WBC = 7.2
    │               └── Obs: Hemoglobin = 14.2
    │
    ├── Visit (Jan 29, Follow-up)
    │       └── Encounter: Consultation
    │               ├── Obs: BP Systolic = 115 (improved!)
    │               └── Order: Amlodipine 5mg RENEWED
    │
    ├── PatientProgram: Hypertension Program
    │       └── PatientState: "On Treatment" (since Jan 15)
    │
    ├── MedicationDispense: Amlodipine Jan 15
    └── Allergies: UNKNOWN (not yet assessed)
```

---

## 🧩 Why OpenMRS is Designed This Way

| Design Decision | Real World Reason |
|---|---|
| **`Person` separate from `Patient`** | A healthcare worker, a user, and a patient can all be the same physical person. Separating `Person` avoids duplicating demographic data. |
| **EAV model for `Obs`** | No one knows in advance what 10,000 different clinics will want to record. EAV lets you add new data types (new `Concept`s) without changing the database schema. |
| **`Concept` dictionary** | Standardizes medical terminology. "Blood Pressure" maps to LOINC code `55284-4`, ICD-10, SNOMED CT. This enables data sharing between clinics and countries. |
| **`Visit` wraps `Encounter`s** | A patient may see 3 providers in one day (triage → doctor → pharmacy). All three encounters belong to the same Visit (same "trip to hospital"). |
| **Immutable `Obs` + `Order`** | Medical records are legally protected documents. You can't just edit them. The void-and-replace pattern gives you full audit history — who changed what, when, and why. |
| **`PatientProgram` + workflows** | Chronic disease management (HIV, TB, diabetes) requires tracking patients across months/years with defined care stages. Programs model this care pathway. |
| **Module system** | Every country and clinic has different needs. Kenya needs HIV tracking. India needs TB protocols. The module system lets each deployment add what they need without forking the core. |
| **Role → Privilege security** | A lab technician should not be able to prescribe drugs. A nurse should not be able to delete patient records. Fine-grained privilege control enforces this.