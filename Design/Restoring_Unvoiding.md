Unvoiding vs. Restoring — Why They Are Different Problems

This is a really important distinction that needs to be understood from first principles. Let me break it down completely.

---

## Part 1: First Understand WHY We Even Unvoid Data

Before getting into the mechanics, you need to understand the **real-world clinical situations** that cause someone to call unvoid:

```/dev/null/why-unvoid.md#L1-60
THE FIVE REAL REASONS AN UNVOID IS TRIGGERED:

┌────┬────────────────────────────────────┬──────────────────────────────────────────┐
│ #  │ What Happened                      │ What Needs Unvoiding                     │
├────┼────────────────────────────────────┼──────────────────────────────────────────┤
│ 1  │ ACCIDENTAL VOID                    │ The exact entity that was accidentally   │
│    │ Nurse accidentally clicks "void"   │ voided (an obs, an encounter).           │
│    │ on the wrong encounter or obs.     │ Happens minutes after the mistake.       │
│    │ Realizes immediately.              │ TIME WINDOW: minutes to hours            │
├────┼────────────────────────────────────┼──────────────────────────────────────────┤
│ 2  │ BAD PATIENT MERGE                  │ The entire non-preferred patient:        │
│    │ Two patients were merged but the   │ all their encounters, obs, orders,       │
│    │ merge was wrong (different people).│ visits that were voided by the merge.    │
│    │                                    │ TIME WINDOW: days to weeks               │
├────┼────────────────────────────────────┼──────────────────────────────────────────┤
│ 3  │ ADMIN ERROR                        │ A bulk-voided set of records that were   │
│    │ An admin script voided records it  │ incorrectly voided together.             │
│    │ shouldn't have.                    │ TIME WINDOW: hours to days               │
├────┼────────────────────────────────────┼──────────────────────────────────────────┤
│ 4  │ BUSINESS RULE CHANGE               │ Records voided under an old rule that    │
│    │ A previously valid void reason is  │ no longer applies. Rare.                 │
│    │ no longer considered valid.        │ TIME WINDOW: months to years             │
├────┼────────────────────────────────────┼──────────────────────────────────────────┤
│ 5  │ OBS CORRECTION UNVOID              │ A specific voided obs (the old version   │
│    │ A nurse corrected an obs but the   │ of a correction). This is unusual —      │
│    │ original value was actually right  │ you'd usually make a new correction      │
│    │ and the correction was wrong.      │ rather than unvoiding the old version.   │
│    │                                    │ TIME WINDOW: hours to days               │
└────┴────────────────────────────────────┴──────────────────────────────────────────┘

KEY OBSERVATION:
  Reasons 1, 2, 3 are the most common unvoid scenarios.
  They almost always happen SOON after the void (minutes → days).
  Reason 4 is rare and academic.
  Reason 5 is unusual because correcting-a-correction is easier.
```

---

## Part 2: How Unvoiding Currently Works (The Code Path)

Now let's trace exactly what happens in the code when `unvoidObs()` or `unvoidEncounter()` is called:

```/dev/null/unvoid-code-flow.md#L1-35
UNVOID FLOW (current system, no archiving):

obsService.unvoidObs(obs)
  │
  ├─► calls Context.getObsService().saveObs(obs, "unvoid obs")
  │
  ├─► saveObs() sees obs.getVoided() == true → saveNewOrVoidedObs()
  │       OR sees obs.isDirty() == false → saveObsNotDirty()
  │       (the AOP BaseUnvoidHandler has already cleared voided=false
  │        BEFORE saveObs is called, via RequiredDataAdvice.before())
  │
  └─► The AOP chain (RequiredDataAdvice) fires BEFORE the real method:
        detects "unvoid" prefix → runs UnvoidHandler chain:
          BaseUnvoidHandler.handle(obs):
            if obs.getVoided() == true
              AND (origParentVoidedDate == null
                   OR origParentVoidedDate == obs.getDateVoided()):
              obs.setVoided(false)      ← clears the void flag
              obs.setVoidedBy(null)     ← clears who voided it
              obs.setDateVoided(null)   ← clears when
              obs.setVoidReason(null)   ← clears why
        Saves to DB: obs row updated → voided=0, all void cols = NULL
```

```/dev/null/unvoid-encounter-flow.md#L1-40
UNVOID ENCOUNTER FLOW (with cascading):

encounterService.unvoidEncounter(encounter)
  │
  ├─► AOP fires: BaseUnvoidHandler clears encounter's void fields
  │
  ├─► Real method runs (EncounterServiceImpl.unvoidEncounter):
  │     voidReason = encounter.getVoidReason()  ← saved before clearing
  │     
  │     for each obs in encounter.getObsAtTopLevel(includeVoided=TRUE):
  │       if obs.getVoidReason() == encounter's original voidReason:
  │           obsService.unvoidObs(obs)  ← only unvoids obs that were
  │                                         voided WITH this encounter
  │                                         (same reason = same event)
  │     
  │     for each order in encounter.getOrders():
  │       if order.getVoidReason() == encounter's original voidReason:
  │           orderService.unvoidOrder(order)
  │     
  │     encounter.setVoided(false)
  │     ...save encounter...
  │
  └─► RESULT: Encounter + its obs + its orders are all unvoided
              BUT ONLY the ones voided WITH this encounter
              (obs previously voided for other reasons stay voided)
```

```/dev/null/unvoid-patient-flow.md#L1-30
UNVOID PATIENT FLOW (largest cascade):

patientService.unvoidPatient(patient)
  │
  ├─► AOP: BaseUnvoidHandler clears patient's void fields
  ├─► AOP: PersonUnvoidHandler clears personVoided fields
  ├─► PatientDataUnvoidHandler.handle(patient):
  │     originalVoidingUser = patient.getVoidedBy()  (saved before clear)
  │     origParentVoidedDate = patient.getDateVoided() (saved before clear)
  │     
  │     for each encounter of this patient (includeVoided=TRUE):
  │       if encounter.getVoided()
  │          AND encounter.getDateVoided() == origParentVoidedDate
  │          AND encounter.getVoidedBy() == originalVoidingUser:
  │             encounterService.unvoidEncounter(encounter)
  │             → cascades to obs + orders
  │     
  │     for each order of this patient:
  │       same timestamp + user matching check
  │       orderService.unvoidOrder(order)
  │     
  │     cohortService.notifyPatientUnvoided(patient, ...)
  │
  └─► RESULT: Patient + matching encounters + their obs + orders unvoided
```

---

## Part 3: The Critical Matching Rules in Unvoiding

This is the most important thing to understand about the current unvoid system — it uses **exact timestamp and user matching** to know what to unvoid together:

```/dev/null/matching-rules.md#L1-55
THE MATCHING RULES (from BaseUnvoidHandler and PatientDataUnvoidHandler):

RULE 1 — Timestamp Match (BaseUnvoidHandler):
  An obs/encounter child is unvoided only if:
    child.dateVoided == parent.dateVoided (the origParentVoidedDate)

  WHY: If an obs was voided BEFORE the encounter was voided,
       it had its own separate reason. When the encounter is unvoided,
       that pre-existing voided obs should NOT be unvoided — it was
       voided independently.

  EXAMPLE:
    obs101 voided on Day5 (correction, different reason)
    obs102 voided on Day10 (encounter was voided, same timestamp)
    encounter voided on Day10

    unvoidEncounter():
      obs101: dateVoided=Day5 ≠ encounter.dateVoided=Day10 → SKIP
      obs102: dateVoided=Day10 == encounter.dateVoided=Day10 → UNVOID ✓

RULE 2 — Void Reason Match (EncounterServiceImpl):
  An obs is unvoided from its encounter only if:
    obs.voidReason == encounter.voidReason

  WHY: Same timestamp is not enough (concurrent voids could coincide).
       The reason confirms this obs was voided as part of the same action.

RULE 3 — User Match (PatientDataUnvoidHandler):
  An encounter is unvoided from its patient only if:
    encounter.voidedBy == patient.voidedBy (originalVoidingUser)

  WHY: Prevents unvoiding encounters that happened to be voided
       at the same time by a different user action.

COMBINED EFFECT:
  These three rules together ensure that only the exact set of
  records voided together in one atomic clinical action get
  unvoided together. Records with independent void histories
  are left alone.
```

---

## Part 4: Now — The Core Problem with Archiving + Unvoiding

Here is where your question becomes extremely sharp. Once we introduce archiving, the unvoid operation **breaks silently** in the current system:

```/dev/null/the-break.md#L1-60
CURRENT UNVOID FLOW (without archiving):
  unvoidEncounter(encounter)
    → looks for obs with matching dateVoided in obs table
    → finds them (they are still in obs table, voided=1)
    → clears void fields, saves
    → ✓ Works

AFTER ARCHIVING IS INTRODUCED:
  Scenario: encounter voided on Day10, obs archived after Day40

  Day 50: Admin tries to unvoidEncounter(encounter)
    → looks for obs in encounter.getObsAtTopLevel(includeVoided=TRUE)
    → Hibernate loads encounter's obs collection from obs table
    → obs table has NO rows for this encounter (they were archived!)
    → getObsAtTopLevel(true) returns EMPTY SET
    → unvoidEncounter succeeds... but encounter has no obs restored
    → The encounter is now unvoided but EMPTY — all its obs are gone
    → Clinical data integrity: BROKEN ✗

This is the silent failure that makes restore so critical.
Without a restore mechanism, archiving would make unvoiding
permanently destructive.
```

---

## Part 5: Unvoiding vs. Restoring — They Are Two Different Operations

This is the key conceptual distinction you asked about:

```/dev/null/unvoid-vs-restore.md#L1-60
┌─────────────────────────────────────────────────────────────────┐
│                        UNVOIDING                               │
│                                                                 │
│  WHAT IT IS:                                                    │
│    A CLINICAL/BUSINESS OPERATION.                               │
│    "This data was marked as invalid. We are saying it IS valid."│
│                                                                 │
│  WHAT IT CHANGES:                                               │
│    Sets voided=0, clears voidedBy, dateVoided, voidReason.     │
│    The data becomes ACTIVE and visible in clinical queries.     │
│                                                                 │
│  WHO TRIGGERS IT:                                               │
│    A clinical user (nurse, doctor, admin) who realizes a        │
│    void was a mistake.                                          │
│                                                                 │
│  PRECONDITION:                                                  │
│    The entity must be in the PRIMARY TABLE (obs table) with    │
│    voided=1. It cannot act on an archived entity.               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        RESTORING                               │
│                                                                 │
│  WHAT IT IS:                                                    │
│    AN INFRASTRUCTURE/STORAGE OPERATION.                         │
│    "This data was physically moved to cold storage.             │
│    We are physically moving it back to hot storage."            │
│                                                                 │
│  WHAT IT CHANGES:                                               │
│    Moves the row from *_archive table back to primary table.   │
│    The row comes back as voided=1 (still voided after restore). │
│    Clinical visibility does NOT change from restore alone.      │
│                                                                 │
│  WHO TRIGGERS IT:                                               │
│    The system (automatically, when unvoid is called on an       │
│    archived entity) OR an admin (manually, to inspect/audit     │
│    data that was archived).                                     │
│                                                                 │
│  PRECONDITION:                                                  │
│    The entity must be in the *_archive table.                  │
└─────────────────────────────────────────────────────────────────┘

THE RELATIONSHIP:
  Restore is a PREREQUISITE for Unvoid when data is archived.
  Restore alone does NOT make data clinically active.
  Unvoid alone FAILS if data is in archive (not in primary table).
  Restore + Unvoid TOGETHER = full recovery of archived data.

  RESTORE   →   data moves from obs_archive → obs (still voided=1)
  UNVOID    →   data changes from voided=1  → voided=0 (clinically active)
```

---

## Part 6: The Complete Restore + Unvoid Scenarios

Now let's trace through every real-world restore scenario end to end.

### Scenario A: Accidental Void — Caught Within 30 Days (No Archive Yet)

```/dev/null/scenario-a.md#L1-55
TIMELINE:
  Day 1:  Nurse accidentally voids encounter (wrong encounter clicked)
          → encounter.voided=1, obs102.voided=1, obs103.voided=1
          → All rows still in primary table (within retention window)

  Day 1 (minutes later): Admin calls unvoidEncounter(encounter)
  
WHAT HAPPENS (current system, works perfectly):
  Step 1: AOP BaseUnvoidHandler clears encounter's void fields
  Step 2: EncounterServiceImpl.unvoidEncounter():
            getObsAtTopLevel(includeVoided=TRUE):
              → finds obs102 (dateVoided=Day1, voidReason="accidental")
              → finds obs103 (dateVoided=Day1, voidReason="accidental")
            both match encounter's voidReason → unvoidObs(obs102)
                                              → unvoidObs(obs103)
  
DOES ARCHIVING AFFECT THIS? NO.
  30-day retention window means no archiving happened yet.
  Everything is still in primary table.
  This scenario works identically with or without archiving.
  
  The 30-day retention window exists PRECISELY to protect
  this most common use case from ever needing a restore.

DB STATE AFTER UNVOID:
  encounter: voided=0, voidedBy=NULL, dateVoided=NULL ✓
  obs102:    voided=0, voidedBy=NULL, dateVoided=NULL ✓
  obs103:    voided=0, voidedBy=NULL, dateVoided=NULL ✓
```

### Scenario B: Bad Patient Merge — Caught Within 30 Days (No Archive Yet)

```/dev/null/scenario-b.md#L1-55
TIMELINE:
  Day 1:  Patient A merged into Patient B (wrong merge).
          → PatientA.voided=1
          → All PatientA's encounters voided (same timestamp)
          → All PatientA's obs voided (same timestamp)
          → All PatientA's orders voided (same timestamp)

  Day 15: Doctor realizes the merge was wrong.
          Admin calls unvoidPatient(patientA)

WHAT HAPPENS (current system, within 30-day window):
  Step 1: AOP clears patientA.voided fields
  Step 2: PatientDataUnvoidHandler:
            for each encounter (includeVoided=TRUE):
              encounter.dateVoided == Day1 AND voidedBy==mergeUser?
              → YES → unvoidEncounter(encounter)
                        → unvoidObs(all obs with same dateVoided)
                        → unvoidOrder(all orders with same dateVoided)
            cohortService.notifyPatientUnvoided(patientA)

DOES ARCHIVING AFFECT THIS? NO.
  15 days < 30-day retention window.
  Nothing has been archived yet.
  Full unvoid cascade works perfectly.

DB STATE AFTER UNVOID:
  PatientA: voided=0 ✓
  All encounters: voided=0 ✓
  All obs: voided=0 ✓
  All orders: voided=0 ✓
  PatientA fully restored to clinical active status ✓
```

### Scenario C: Bad Patient Merge — Caught After 45 Days (ARCHIVING HAS HAPPENED)

This is the scenario where restoring becomes essential:

```/dev/null/scenario-c.md#L1-100
TIMELINE:
  Day 1:   Patient A merged into Patient B (wrong merge).
           → PatientA.voided=1 (dateVoided=Day1)
           → 300 encounters voided (dateVoided=Day1)
           → 3000 obs voided (dateVoided=Day1)
           → 500 orders voided (dateVoided=Day1)

  Day 31:  Archival task runs (retention=30 days).
           All 3000 obs from Day1 eligible → moved to obs_archive.
           All 500 orders → moved to orders_archive.
           All 300 encounters → moved to encounter_archive.
           PatientA row → moved to patient_archive.

  Day 45:  Doctor realizes the merge was wrong.
           Admin tries: unvoidPatient(patientA)

WHAT HAPPENS WITHOUT A RESTORE MECHANISM:
  Step 1: AOP tries to load patientA from DB
            → patientA NOT in patient table (archived!)
            → NullPointerException or entity not found ✗ CRASH

  EVEN IF PATIENT ROW WAS FOUND:
  Step 2: PatientDataUnvoidHandler searches for encounters:
            es.getEncounters(patient, includeVoided=TRUE)
            → searches encounter table (including voided=1 rows)
            → encounter table has ZERO rows for patientA (all archived)
            → returns empty list
            → no encounters unvoided
            → patient "unvoided" but all clinical data gone ✗

WHAT MUST HAPPEN WITH THE RESTORE MECHANISM:

PHASE 1 — RESTORE (infrastructure operation, moves rows back):
  
  ArchiveService.restoreForUnvoid(patientA_uuid):
  
  Step 1: Find patientA in patient_archive by UUID
          → row found: voided=1, dateVoided=Day1

  Step 2: Restore patient first (parents before children)
          → atomic: INSERT into patient SELECT FROM patient_archive
                    DELETE FROM patient_archive
          → patientA is now back in patient table (still voided=1)

  Step 3: Find all encounters in encounter_archive where:
            patient_id = patientA.id
            dateVoided = Day1 (same timestamp as patient)
            voidedBy = mergeUser (same user)
          → 300 encounters found in encounter_archive

  Step 4: Restore each encounter (batch, atomic per batch)
          → 300 encounters back in encounter table (still voided=1)

  Step 5: Find all obs in obs_archive where:
            encounter_id IN (patientA's 300 encounter IDs)
            dateVoided = Day1
          → 3000 obs found in obs_archive

  Step 6: Restore each obs (batch, atomic)
          → 3000 obs back in obs table (still voided=1)

  Step 7: Find all orders in orders_archive:
            patient_id = patientA.id
            dateVoided = Day1
          → 500 orders found

  Step 8: Restore orders (batch, atomic)
          → 500 orders back in orders table (still voided=1)

  → After PHASE 1: Everything is back in primary tables. Still voided.
    DB looks exactly like it did on Day 2 (just after the merge).

PHASE 2 — UNVOID (clinical operation, changes voided=0):
  
  Now the existing unvoidPatient() works exactly as before:
  unvoidPatient(patientA)
    → PatientDataUnvoidHandler finds encounters (now back in table)
    → Timestamp matches Day1 → unvoidEncounter() for all 300
      → EncounterServiceImpl finds obs (now back in obs table)
      → Timestamp + reason match → unvoidObs() for all 3000
    → All 500 orders unvoided
    → CohortService notified

  → After PHASE 2: Everything voided=0. PatientA clinically active. ✓

THE RESTORE MUST BE TRANSPARENT:
  Ideally, the admin just calls unvoidPatient(patientA).
  The system detects "patientA is in archive" and auto-triggers
  the restore BEFORE proceeding with the unvoid.
  The admin sees one operation, not two.
```

### Scenario D: Accidental Void of a Single Obs — Caught After 45 Days

```/dev/null/scenario-d.md#L1-60
TIMELINE:
  Day 1:  Nurse makes a correction to obs (BP: 140→120)
          → obs101 (value=140) voided=1
          → obs102 (value=120) active, previous_version=101

  Day 31: Archival task runs.
          obs101 eligible? Under Option A+C: YES (30 days old, voided)
          → obs101 moved to obs_archive
          → obs102 updated: previous_version FK cleared
                            previous_version_uuid = obs101.uuid stored

  Day 45: Nurse realizes the original value 140 was correct.
          She wants the correction UNDONE.
          
TWO CHOICES FOR THE NURSE:

CHOICE 1: Make a new correction (simpler, preferred)
  Create new obs103 with value=140
  obs102 (value=120) becomes voided (previous version of 103)
  obs101 stays in archive (historical curiosity, not needed)
  
  → NO RESTORE NEEDED at all.
  → The clinical record now shows: 140 (newest), with history
    showing 120 was entered and corrected back to 140.
  → This is the RECOMMENDED path in OpenMRS:
    "The recommended way to update an obs is to create a new obs
     and void the old one" — never unvoid an old version.

CHOICE 2: Actually unvoid obs101 from archive (unusual)
  Why would you want this? You want the original obs101 as the
  definitive record, NOT a new obs103.
  Rare case: maybe for audit/legal reasons the chain must show
  obs101 as the continuous valid record.
  
  RESTORE FLOW:
  Step 1: ArchiveService.restore(obs101_uuid)
          → obs101 moved from obs_archive → obs table (voided=1)
          → obs102.previous_version FK restored to obs101.obs_id
  
  Step 2: unvoidObs(obs101)
          → obs101.voided = 0
          → obs101 is now active again
  
  BUT NOW WE HAVE TWO ACTIVE OBS FOR THE SAME CONCEPT:
    obs101 (value=140) voided=0 (just unvoided)
    obs102 (value=120) voided=0 (was the corrected version)
  
  CONFLICT! Both active, both for the same patient/concept/date.
  → Must also void obs102 to complete the undo.
  
  CONCLUSION: This is messy. Choice 1 is always better.
  Restoring a predecessor obs is an edge case that requires
  careful admin handling, not a routine clinical operation.
```

### Scenario E: Accidental Void of an Entire Visit — Caught After 45 Days

```/dev/null/scenario-e.md#L1-60
TIMELINE:
  Day 1:  Encounter transferred to another patient
          → original encounter voided (void reason: "transfer")
          → visit had only this encounter → visit also voided

  Day 31: Archival task runs
          → obs from encounter archived (obs_archive)
          → orders archived (orders_archive)
          → encounter archived (encounter_archive)
          → visit archived (visit_archive)

  Day 45: Admin realizes the transfer was to the wrong patient.
          The original encounter should be restored.

RESTORE + UNVOID FLOW:

PHASE 1 — RESTORE (bottom-up: children before parents in restore):
  Step 1: Restore visit from visit_archive → visit table (voided=1)
  Step 2: Restore encounter from encounter_archive → encounter table (voided=1)
  Step 3: Restore obs from obs_archive → obs table (voided=1)
  Step 4: Restore orders from orders_archive → orders table (voided=1)

  WHY BOTTOM-UP HERE? (opposite of archive order which is top-down)
  Because in the primary table, FK relationships must be satisfied:
    obs.encounter_id must point to a valid encounter_id in encounter table
    encounter.visit_id must point to a valid visit_id in visit table
  So we restore from the root down:
    visit first → then encounter (can reference valid visit) 
    → then obs (can reference valid encounter)

PHASE 2 — UNVOID:
  unvoidVisit(visit)
    → VisitUnvoidHandler: finds encounters with matching dateVoided+voidReason
    → unvoidEncounter(encounter)
      → unvoids obs, orders with matching voidReason
  
  → visit, encounter, obs, orders all active again ✓

ADDITIONAL CLEANUP:
  The incorrect transfer encounter (at wrong patient) still exists.
  It must be voided/deleted separately.
  The transferred obs copies at the wrong patient must be voided.
```

---

## Part 7: The Combined System — How Restore Integrates Transparently

```/dev/null/integrated-system.md#L1-90
THE SEAMLESS INTEGRATION DESIGN:

When unvoidXxx() is called, the system should TRANSPARENTLY handle
whether the target is in the primary table or in the archive.
The calling code should not need to know about archiving at all.

CURRENT CALL (without archiving awareness):
  admin: unvoidEncounter(encounter)
  → works if encounter is in primary table
  → silently corrupts if encounter is in archive

NEW CALL (with transparent restore):
  admin: unvoidEncounter(encounter)   ← SAME API, no change
         ↓
  EncounterServiceImpl.unvoidEncounter():
    [NEW STEP]: Check if encounter is archived:
      if ArchiveService.isArchived(encounter.uuid):
          ArchiveService.restoreForUnvoid(encounter)
          → restores encounter + its obs + orders from archive tables
          → everything back in primary tables (still voided=1)
    [EXISTING STEPS proceed unchanged]:
      for each obs: if voidReason matches → unvoidObs()
      for each order: if voidReason matches → unvoidOrder()
      encounter.setVoided(false)
      save encounter

  RESULT: Admin called ONE method. Got full recovery. ✓

HOW isArchived() WORKS:
  Fast lookup: SELECT COUNT(*) FROM encounter_archive WHERE uuid=?
  Returns true if row found in archive table.
  This adds one DB query per unvoid call — negligible cost.

HOW restoreForUnvoid() WORKS:
  It does NOT fully restore and then unvoid.
  It ONLY restores to primary table (voided=1 state).
  The existing unvoid logic then does the clinical unvoid.

  ORDER OF RESTORE (visit→encounter→obs→orders):
    1. Check if visit (of this encounter) is in visit_archive → restore
    2. Restore encounter from encounter_archive → encounter table
    3. Find all obs in obs_archive where encounter_id = encounter.id
       AND dateVoided = encounter.dateVoided → restore to obs table
    4. Find all orders in orders_archive → restore to orders table

  All restores are atomic (INSERT+DELETE per batch, transactional).
```

---

## Part 8: The Complete Picture — Void → Archive → Restore → Unvoid

Here is the full lifecycle in one diagram:

```/dev/null/full-lifecycle.md#L1-100
════════════════════════════════════════════════════════════════════
     COMPLETE DATA LIFECYCLE: VOID → ARCHIVE → RESTORE → UNVOID
════════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────────┐
  │  STATE 1: ACTIVE (normal clinical data)                      │
  │  obs table: obs102 voided=0, value=120                       │
  │             obs101 voided=0, value=140 (before correction)   │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                     voidEncounter() called
                     (wrong patient scenario)
                                 │
  ┌──────────────────────────────▼───────────────────────────────┐
  │  STATE 2: VOIDED (soft-deleted, still in primary table)      │
  │  obs table: obs102 voided=1, dateVoided=Day5, reason="Wrong" │
  │             obs101 voided=1, dateVoided=Day2, reason="Correc"│
  │                                                              │
  │  DURING THIS WINDOW (0 → 30 days):                          │
  │  → unvoidEncounter() works directly, no restore needed       │
  │  → Data is immediately recoverable                           │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                    30-day retention expires
                    Archival task runs (weekly)
                                 │
  ┌──────────────────────────────▼───────────────────────────────┐
  │  STATE 3: ARCHIVED (moved to cold storage)                   │
  │  obs table:        EMPTY for this encounter                  │
  │  obs_archive:  obs102 voided=1, date_archived=Day35          │
  │                obs101 voided=1, date_archived=Day35          │
  │  encounter_archive: encounter voided=1                       │
  │                                                              │
  │  DURING THIS STATE:                                          │
  │  → Clinical queries: data invisible (correct behavior)       │
  │  → Audit queries: data queryable via ArchiveService          │
  │  → Direct unvoidEncounter(): would fail (encounter in archive)│
  └──────────────────────────────┬───────────────────────────────  └──────────────────────────────┬───────────────────────────────
                                 │
                    Admin calls unvoidEncounter()
                    (realizes wrong-patient void was a mistake)
                                 │
  ┌──────────────────────────────▼───────────────────────────────┐
  │  STATE 4: RESTORE TRIGGERED (automatic, transparent)         │
  │                                                              │
  │  ArchiveService detects encounter is in encounter_archive    │
  │  Restores: encounter_archive → encounter (voided=1)          │
  │  Restores: obs_archive → obs (voided=1)                      │
  │  Restores: orders_archive → orders (voided=1)                │
  │                                                              │
  │  obs table:     obs102 voided=1 ← back here                 │
  │                 obs101 voided=1 ← back here                  │
  │  obs_archive:   EMPTY (rows moved out)                       │
  │                                                              │
  │  DB now looks exactly like STATE 2                           │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                    Existing unvoid logic runs
                    (unchanged, works on primary table)
                                 │
  ┌──────────────────────────────▼───────────────────────────────┐
  │  STATE 5: UNVOIDED (back to clinical active)                 │
  │  obs table: obs102 voided=0, value=120 ← active again        │
  │             obs101 voided=1  ← stays voided (was pre-existing│
  │                                 correction, different reason) │
  │  encounter: voided=0 ← active again                          │
  └──────────────────────────────────────────────────────────────┘
```

---

## Part 9: Why Restoring Is NOT the Same as Unvoiding — The Final Clarity

```/dev/null/final-clarity.md#L1-80
THE MOST IMPORTANT DISTINCTION:

  RESTORE changes WHERE data lives (archive table → primary table)
  UNVOID  changes WHAT the data says (voided=1 → voided=0)

  These are completely orthogonal operations.

  ┌─────────────────┬───────────────┬──────────────────────────┐
  │  Operation      │ Where is data │  What does voided say?   │
  ├─────────────────┼───────────────┼──────────────────────────┤
  │  Original void  │ obs table     │  voided = 1              │
  │  Archive        │ obs_archive   │  voided = 1 (unchanged)  │
  │  Restore        │ obs table     │  voided = 1 (unchanged)  │
  │  Unvoid         │ obs table     │  voided = 0              │
  └─────────────────┴───────────────┴──────────────────────────┘

WHY WE RESTORE WITHOUT UNVOIDING:
  Case 1 — Audit/Inspection:
    An admin wants to LOOK at an archived obs for a legal audit.
    They restore it to see the full Hibernate entity with all fields.
    They do NOT want to unvoid it (it was correctly voided,
    they just want to read it).
    After inspection: re-archive it.

  Case 2 — Prerequisite for Unvoid:
    Admin wants to unvoid an archived encounter.
    System auto-restores it first (so unvoid logic can find it).
    Then unvoid proceeds normally.
    The restore is a MEANS, not an END.

  Case 3 — Partial Restore for Investigation:
    Admin restores just the patient record (not encounters)
    to check the patient's void metadata without full restoration.

WHY WE UNVOID WITHOUT RESTORING:
  The data was never archived (still within retention window).
  The accidental void happened today, the mistake caught today.
  unvoidEncounter() works directly on the primary table.
  No restore needed.

THE RETENTION WINDOW IS THE DIVIDING LINE:
  ┌──────────────────────────────────────────────────────────┐
  │  Void happened < 30 days ago:                           │
  │    unvoid() directly → works ✓                          │
  │    No restore needed                                     │
  ├──────────────────────────────────────────────────────────┤
  │  Void happened > 30 days ago AND archiving has run:     │
  │    unvoid() → system detects archived → auto-restore    │
  │    → then unvoid proceeds → works ✓                     │
  │    Restore is transparent to the caller                  │
  └──────────────────────────────────────────────────────────┘

THE 30-DAY WINDOW DESIGN INSIGHT:
  The retention window is NOT arbitrary.
  It is deliberately sized to cover the most common unvoid need:
  accidental voids are always caught quickly (hours to days).
  Systematic errors (bad merges) are caught within weeks.
  30 days means: for all practical unvoid use cases, the data
  is still in the primary table and restore is never needed.
  The restore path exists for the rare case, not the common case.
```

---

## Part 10: Summary — The Complete Mental Model

```/dev/null/mental-model.md#L1-70
══════════════════════════════════════════════════════════════════
      COMPLETE MENTAL MODEL: VOID, ARCHIVE, RESTORE, UNVOID
══════════════════════════════════════════════════════════════════

VOID = "mark as invalid"
  → voided=1 in primary table
  → data invisible to clinical queries
  → data still physically present

ARCHIVE = "move to cold storage"
  → row moves from primary table to *_archive table
  → data still voided=1 (archive does not change clinical state)
  → data invisible to primary table queries
  → data queryable only via ArchiveService
  → happens AUTOMATICALLY after retention window (30 days)
  → triggered by background scheduled task

RESTORE = "move back from cold storage"
  → row moves from *_archive table back to primary table
  → data is still voided=1 after restore (no clinical change yet)
  → triggered AUTOMATICALLY when unvoid is called on archived entity
  → can also be triggered MANUALLY by admin for audit/inspection
  → the restore operation itself is never user-visible in normal flow

UNVOID = "mark as valid again"
  → voided=0 in primary table
  → data becomes visible to clinical queries
  → cascades to children (obs, orders) with matching timestamp
  → REQUIRES data to be in primary table first
  → if data is in archive: system auto-restores before unvoiding

══════════════════════════════════════════════════════════════════

  THE SEQUENCE THAT ALWAYS WORKS:

  VOID ──────────► (within 30 days) ──────────► UNVOID
                                                 ↑ direct, no restore needed

  VOID ──► ARCHIVE ──► (any time after) ──► RESTORE ──► UNVOID
                                             ↑ auto-triggered, transparent

══════════════════════════════════════════════════════════════════

  WHY THE RESTORE PATH IS SAFE:

  1. No data loss possible:
     Restore uses atomic INSERT+DELETE in one transaction.
     If INSERT fails: rollback, data stays in archive.
     If DELETE fails: rollback, data stays in archive.
     Primary table is always the source of truth once committed.

  2. No data duplication possible:
     Restore checks: "Is this UUID already in primary table?"
     If yes: skip restore (already there). Prevent duplicates.

  3. The unvoid logic is completely unchanged:
     After restore, the primary table looks exactly like State 2
     (immediately after the void). The existing unvoid code
     runs with zero modification. No new bugs introduced.

  4. The retention window protects the common case:
     99% of unvoids happen within hours or days of the void.
     The 30-day window guarantees these never touch the archive.
     The restore path is only for exceptional situations.

  5. Restore is reversible:
     If an admin accidentally restores something they shouldn't,
     they can re-archive it manually (archive_exclusion_list aside).
     Nothing is permanently committed by a restore.
══════════════════════════════════════════════════════════════════
```

---

To give you the **direct, complete answer** to your question:

**Why do we unvoid?** Because a clinical user made a mistake — either they voided the wrong thing, or a cascade (patient merge, encounter transfer) voided things that shouldn't have been voided. Unvoiding is a clinical correction that brings data back to active status.

**Why do we restore?** Because archiving physically removed the data from the primary table. The existing `unvoidXxx()` methods all look for data in the primary table only — they cannot act on rows that are sitting in `obs_archive`. Restoring is the step that puts the rows back into the primary table (still voided=1) so that the unchanged unvoid logic can then find and unvoid them normally.

**The relationship between them**: Restore is purely a **storage-layer prerequisite** for unvoiding archived data. It is triggered automatically and transparently — the admin just calls `unvoidPatient()` or `unvoidEncounter()` as they always have, and the system handles the restore behind the scenes. From the admin's perspective, there is only one operation. From the system's perspective, there are two: first move the data back from cold to hot storage (restore), then flip the clinical flag from invalid to valid (unvoid).