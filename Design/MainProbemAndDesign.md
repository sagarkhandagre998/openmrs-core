Part 1: Understanding the Problem in Depth

### 1.1 What Is Actually Happening in the Database Right Now

To understand the problem, you first need to understand **how the `obs` table grows in OpenMRS's immutable data model**. Let's trace what happens when a nurse corrects a single blood pressure reading:

```/dev/null/obs-growth.md#L1-30
BEFORE correction:
  obs table:
  ┌────────┬──────────┬────────────┬────────┬─────────────────┐
  │ obs_id │ concept  │ value_text │ voided │ previous_version│
  ├────────┼──────────┼────────────┼────────┼─────────────────┤
  │  101   │ BP_SYS   │  "140"     │   0    │      NULL       │
  └────────┴──────────┴────────────┴────────┴─────────────────┘

AFTER nurse corrects BP to 120 (saveObs is called):
  obs table:
  ┌────────┬──────────┬────────────┬────────┬─────────────────┐
  │ obs_id │ concept  │ value_text │ voided │ previous_version│
  ├────────┼──────────┼────────────┼────────┼─────────────────┤
  │  101   │ BP_SYS   │  "140"     │   1    │      NULL       │  ← VOIDED (old)
  │  102   │ BP_SYS   │  "120"     │   0    │       101       │  ← ACTIVE (new)
  └────────┴──────────┴────────────┴────────┴─────────────────┘

After a year of 10 corrections per day on 500 patients:
  Active obs:  ~1,825,000  (500 × 10/day × 365 = active clinical data)
  Voided obs:  ~1,825,000  (one voided for every correction)
  Ratio:       50% of the table is dead weight that can never be queried clinically
```

This is the **immutable obs pattern** from `ObsServiceImpl.saveExistingObs()` — every edit creates a new row AND keeps the old one voided. Over years, the voided accumulation compounds:

```/dev/null/growth-rate.md#L1-25
Sources of voided row accumulation:

1. Obs edits (most common):
   Every correction to a clinical observation = 1 new void row permanently
   A busy clinic doing 1000 obs/day with 5% correction rate =
   50 new voided obs rows EVERY DAY, forever

2. Patient merges:
   Merging 2 duplicate patients = void ALL encounters + obs of the
   non-preferred patient. A hospital merging 10 duplicates/month =
   potentially thousands of voided obs per merge event

3. Encounter voiding (wrong patient charting):
   Voiding 1 encounter = void ALL its obs + orders.
   One wrong-patient incident on a busy form with 30 obs =
   30 voided rows added instantly

4. Visit voiding:
   Cascades to all encounters → all obs → all orders.
   One voided visit can generate hundreds of voided rows

5. Cohort membership voiding:
   When a patient is voided, their cohort memberships are voided
   (may be restored on unvoid, but void rows accumulate on re-voids)
```

### 1.2 Why This Is a Serious Problem

```/dev/null/problem-impact.md#L1-50
┌──────────────────────────────────────────────────────────────────────┐
│  PROBLEM 1: QUERY PERFORMANCE DEGRADATION                           │
│                                                                      │
│  Every single query on obs must include WHERE voided = 0.            │
│  As voided rows pile up:                                             │
│  - Full table scans get slower (more rows to filter)                │
│  - Index pages fill with dead rows — index fragmentation            │
│  - Even indexed queries must skip over millions of voided rows       │
│  - Buffer pool fills with pages containing mostly voided data        │
│                                                                      │
│  Example: obs table with 10M rows, 7M voided                        │
│  SELECT * FROM obs WHERE voided=0 AND concept_id=1234               │
│  → MySQL must examine/skip 7M voided rows before finding 3M active  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  PROBLEM 2: STORAGE BLOAT                                           │
│                                                                      │
│  The obs table has ~20 columns including text/blob fields.          │
│  A voided obs with value_complex (file reference) still holds the   │
│  metadata row (though complex files may be deleted).                │
│  At scale (millions of voided obs), disk usage grows unboundedly.   │
│  Backups take longer. Replication lag increases.                     │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  PROBLEM 3: HIBERNATE SESSION OVERHEAD                              │
│                                                                      │
│  Hibernate loads an Encounter's obs as a Set<Obs>.                  │
│  Even though the UI calls getObs() (non-voided only), Hibernate     │
│  still loads ALL obs (including voided) into the session cache       │
│  before filtering. The more voided obs on an encounter, the         │
│  larger the heap footprint for a simple encounter load.              │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  PROBLEM 4: AUDIT LOG TABLE EXPLOSION (Hibernate Envers)            │
│                                                                      │
│  Every void creates a revision in obs_AUD.                          │
│  Every subsequent unvoid creates another revision.                  │
│  The audit tables grow even faster than the base tables.            │
│  obs_AUD can grow to 3-5x the size of obs itself.                   │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  PROBLEM 5: LEGAL/OPERATIONAL TENSION                               │
│                                                                      │
│  Simply purging (deleting) voided data is NOT an option because:    │
│  - Healthcare regulations require audit trails                       │
│  - The unvoid operation must be possible (data restoration)         │
│  - previousVersion links between obs must remain navigable          │
│  - Some voided data may need to be queried for research/audit        │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.3 Why the Existing Approaches Are Insufficient

```/dev/null/existing-limits.md#L1-30
Current approaches and why they fall short:

1. purgePatient() / purgeObs() — HARD DELETE
   ✗ Loses data forever, no restore, not acceptable for clinical records

2. Hibernate Envers audit tables
   ✗ These are the READ-ONLY history tables, not a solution for the
     performance problem — in fact they make it WORSE (more bloat)

3. WHERE voided=0 filter in every DAO query
   ✗ Filters at query time but doesn't remove the data from disk.
     The dead rows still exist, still consume storage, still slow
     index scans and buffer pool

4. Database-level partitioning on voided column
   ✗ Some DBMSs support this (MySQL partition pruning), but it's
     database-specific, not portable, and requires DBA-level schema
     changes that OpenMRS's Liquibase migrations don't currently manage.
     Also doesn't solve the restore requirement.

5. Do nothing
   ✗ Works for small clinics, catastrophic for large implementations
     (national-scale deployments with millions of patients)
```

---

## Part 2: The Design Solution — Voided Data Archival

The solution must satisfy all of these constraints simultaneously:

```/dev/null/constraints.md#L1-20
✅ MUST preserve voided data (no permanent loss)
✅ MUST allow restore (unarchive → back to original table)
✅ MUST NOT break any existing service/API calls
✅ MUST improve query performance on primary tables
✅ MUST be transparent to clinical users and most developers
✅ MUST be configurable (not all deployments need aggressive archiving)
✅ MUST handle referential integrity (obs references encounter, etc.)
✅ MUST handle the previousVersion chain in obs
✅ MUST handle Hibernate Envers audit tables
✅ SHOULD be runnable as a background job (no downtime)
✅ SHOULD be reversible/idempotent
```

---

### 2.1 Core Concept: Archive Tables as a "Cold Storage" Tier

The fundamental idea is to introduce a **two-tier storage model** within the same database:

```/dev/null/two-tier.md#L1-35
 ┌──────────────────────────────────────────────────────────────────┐
 │                    HOT TIER (Primary Tables)                     │
 │                                                                  │
 │  obs, encounter, orders, visit, patient, ...                     │
 │                                                                  │
 │  Contains: ONLY active (voided=0) records                        │
 │  Used by: all clinical queries, the application, Hibernate ORM   │
 │  Size:    small, fast, well-indexed, cache-friendly              │
 └──────────────────────────────┬───────────────────────────────────┘
                                │   Archive moves rows here
                                │   Restore pulls rows back
                                ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                   COLD TIER (Archive Tables)                     │
 │                                                                  │
 │  obs_archive, encounter_archive, orders_archive, ...             │
 │                                                                  │
 │  Contains: voided records (moved from primary tables)            │
 │  Used by: admin/audit queries, restore operations only           │
 │  Size:    can be large; separate tablespace/disk if needed       │
 │  Access:  NOT loaded by Hibernate ORM for normal operations      │
 └──────────────────────────────────────────────────────────────────┘
```

### 2.2 Archive Table Structure

Each archive table is an **exact structural mirror** of its source table, with one additional metadata column:

```/dev/null/archive-table-schema.sql#L1-40
-- Example: obs_archive mirrors obs exactly
CREATE TABLE obs_archive (
    -- === All original obs columns (identical types) ===
    obs_id           INT(11) NOT NULL,        -- NOT auto-increment (preserves original PK)
    person_id        INT(11) NOT NULL,
    concept_id       INT(11) NOT NULL,
    encounter_id     INT(11),
    order_id         INT(11),
    obs_datetime     DATETIME NOT NULL,
    location_id      INT(11),
    obs_group_id     INT(11),
    accession_number VARCHAR(255),
    value_group_id   INT(11),
    value_coded      INT(11),
    value_numeric    DOUBLE,
    value_text       LONGTEXT,
    value_datetime   DATETIME,
    value_complex    VARCHAR(1000),
    comments         VARCHAR(255),
    previous_version INT(11),               -- FK no longer enforced (source row gone)
    creator          INT(11) NOT NULL,
    date_created     DATETIME NOT NULL,
    voided           TINYINT(1) NOT NULL,   -- always 1 in archive
    voided_by        INT(11),
    date_voided      DATETIME,
    void_reason      VARCHAR(255),
    uuid             VARCHAR(38) NOT NULL,
    status           VARCHAR(16),
    interpretation   VARCHAR(32),

    -- === Archive-specific metadata ===
    date_archived    DATETIME NOT NULL,     -- when it was moved to archive
    archived_by      INT(11),               -- user/daemon who triggered archiving
    archive_reason   VARCHAR(255),          -- e.g. "scheduled archival", "manual"

    PRIMARY KEY (obs_id),
    INDEX idx_obs_archive_uuid (uuid),
    INDEX idx_obs_archive_date_voided (date_voided),
    INDEX idx_obs_archive_person (person_id)
    -- NO foreign key constraints (source rows may also be archived)
);
```

**Critical design decision**: Archive tables have **no foreign key constraints**. This is intentional — when an encounter is archived, its obs may be archived too, and FK constraints would prevent independent archival. The integrity is maintained at the application layer instead, not the DB layer.

### 2.3 Which Tables Get Archive Counterparts?

Not every table needs archiving — only the ones that are both:
1. **Frequently voided** (accumulate large numbers of voided rows), and
2. **Voidable** (implement `Voidable`)

```/dev/null/archive-candidates.md#L1-30
TIER 1 — Must archive (highest voided accumulation):
┌────────────────┬──────────────────────────────────────────────┐
│  obs           │ Immutable pattern guarantees 1 void per edit  │
│                │ Most critical table for archiving             │
├────────────────┼──────────────────────────────────────────────┤
│  encounter     │ Voided on patient merge, wrong-patient        │
├────────────────┼──────────────────────────────────────────────┤
│  orders        │ Voided with encounter; DISCONTINUE/REVISE     │
│                │ chains accumulate void entries                │
├────────────────┼──────────────────────────────────────────────┤
│  visit         │ Voided on transfer, cascade from patient      │
└────────────────┴──────────────────────────────────────────────┘

TIER 2 — Should archive (moderate accumulation):
┌────────────────┬──────────────────────────────────────────────┐
│  patient       │ Voided on merge; rare but rows are large      │
│  person        │ Tied to patient voiding                       │
│  person_name   │ Updated via void-and-replace pattern          │
│  person_address│ Same as above                                 │
│  cohort_member │ Voided on patient void                        │
└────────────────┴──────────────────────────────────────────────┘

TIER 3 — Optional / Low priority:
┌────────────────┬──────────────────────────────────────────────┐
│  concept_name  │ Voided on concept edits; metadata not clinical│
│  patient_ident │ Voided on identifier correction              │
└────────────────┴──────────────────────────────────────────────┘
```

### 2.4 Archivable — The New Interface

Mirror of `Voidable`, a new marker interface identifies entities that support archiving:

```/dev/null/Archivable-interface.md#L1-25
/**
 * Implemented by entities whose voided rows can be moved to archive tables.
 * All Archivable entities must also be Voidable.
 *
 * An entity is only eligible for archiving when:
 *   1. voided = true
 *   2. date_voided is older than the configured retention period
 *   3. No non-archived, non-voided entity holds a live reference to it
 */
public interface Archivable extends Voidable {

    // When was this entity moved to the archive?
    Date getDateArchived();
    void setDateArchived(Date dateArchived);

    // Who/what triggered the archiving?
    User getArchivedBy();
    void setArchivedBy(User archivedBy);

    // Reason for archiving (e.g. "scheduled", "manual", "patient-merge-cleanup")
    String getArchiveReason();
    void setArchiveReason(String reason);
}
```

### 2.5 The Archival Eligibility Rules

Not every voided row can be immediately archived. There are **three eligibility gates**:

```/dev/null/eligibility-rules.md#L1-55
┌──────────────────────────────────────────────────────────────────────┐
│  GATE 1: Minimum Retention Period                                    │
│                                                                      │
│  A voided row cannot be archived until it has been voided for at     │
│  least N days (configurable via GlobalProperty).                     │
│                                                                      │
│  WHY: Administrators may need to quickly unvoid recently voided      │
│  data (e.g., "I accidentally voided the wrong encounter 10 minutes   │
│  ago"). If we immediately archive, the restore path is harder.       │
│  A 30-day window gives a comfortable unvoid window via UI.           │
│                                                                      │
│  Default: 30 days                                                    │
│  Configurable: openmrs.archive.retention.days = 30                  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  GATE 2: No Active Dependent References                              │
│                                                                      │
│  A voided row cannot be archived if a NON-VOIDED row references it. │
│                                                                      │
│  Examples:                                                           │
│  - obs.previous_version → points to a voided obs                    │
│    The ACTIVE obs (new version) references the voided obs.           │
│    We CANNOT archive the voided obs while the active obs exists,     │
│    because the previousVersion link would break.                     │
│                                                                      │
│  - encounter.voided=1 but obs.encounter_id still references it      │
│    (shouldn't happen due to cascade, but must be verified)           │
│                                                                      │
│  Solution: Archive in the correct ORDER (children before parents     │
│  is wrong here; we archive parents only when all children            │
│  that reference them are also voided/archived)                       │
│                                                                      │
│  Special case for obs.previous_version:                              │
│  Archive the voided obs ONLY when the active obs referencing it      │
│  via previous_version is itself voided (the whole chain is void).   │
│  OR: Update obs.previous_version to store the UUID (not obs_id)      │
│  so the link survives archiving (see section 2.7).                  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  GATE 3: Configurable Entity-Level Opt-Out                          │
│                                                                      │
│  Some void operations may be flagged as "do not archive" at void    │
│  time. For example, a patient record voided as part of a merge        │
│  that is still under review. A flag on the archive table itself      │
│  (or a separate "exclude from archiving" list) could mark these.    │
│                                                                      │
│  This is an optional advanced feature.                               │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.6 Archival Order — Respecting the Dependency Chain

The archival must happen in a **specific order** to respect the FK relationships that exist in the hot tier:

```/dev/null/archival-order.md#L1-45
The dependency graph (what references what):

visit ◄── encounter ◄── obs ◄── obs (obsGroup/previousVersion)
                    ◄── orders ◄── orders (previousOrder chain)

CORRECT ARCHIVAL ORDER (bottom-up, deepest children first):

Step 1: Archive voided OBS that are:
   - obs_group members (children of an obs group)
   - obs with previous_version pointing to another obs that is
     ALSO voided (the whole correction chain is voided)
   - Stand-alone obs not referenced by any active entity

Step 2: Archive voided OBS groups (parent obs)
   - Only after all their group members are already archived

Step 3: Archive voided ORDERS
   - Only after their encounter is also voided
   - Order chains: archive REVISE/DISCONTINUE orders before base orders

Step 4: Archive voided ENCOUNTERS
   - Only after all their obs and orders are archived

Step 5: Archive voided VISITS
   - Only after all their encounters are archived

Step 6: Archive voided PATIENTS (rare, complex case)
   - Only after all encounters, obs, orders, visits are archived

Why this order matters:
  If encounter is archived while obs still exists in obs table with
  obs.encounter_id pointing to the now-archived encounter,
  a JOIN query (encounter JOIN obs) would fail silently or return
  nulls. By archiving children first, we ensure no orphaned FK
  references remain in the hot tier.
```

### 2.7 The `previous_version` Problem — The Hardest Design Challenge

This is the most delicate problem in the whole design, and it requires an explicit solution:

```/dev/null/previous-version-problem.md#L1-55
Current state of obs correction chain:

  obs_id=101 (voided=1) ← previous_version — obs_id=102 (voided=0, ACTIVE)

Scenario: obs 101 is eligible for archiving (old, voided)
Problem:  obs 102 is ACTIVE and contains previous_version=101 (integer FK)
          If obs 101 is moved to obs_archive, obs 102.previous_version=101
          points to a row that no longer exists in the obs table.
          Hibernate will throw a LazyInitializationException or return null
          when loading obs 102's previousVersion.

Three design options to solve this:

OPTION A: Archive obs 101 only when obs 102 is also voided
  ┌─────────────────────────────────────────────────────┐
  │  Only archive the whole correction chain together   │
  │  (when the latest version is also voided)           │
  └─────────────────────────────────────────────────────┘
  PRO: Simple rule, no schema changes
  CON: If a patient's obs goes through 10 corrections,
       the oldest 9 voided rows cannot be archived until
       the 10th (active) obs is also voided. This delays
       archiving significantly for active records.

OPTION B: Change previous_version to store UUID instead of ID
  ┌─────────────────────────────────────────────────────┐
  │  obs.previous_version_uuid VARCHAR(38)              │
  │  (instead of / in addition to previous_version INT) │
  └─────────────────────────────────────────────────────┘
  PRO: UUID is stable across archive/restore;
       The active obs can still link to the voided archived obs
       via UUID lookup in obs_archive
  CON: Schema migration required; UUID lookup is slower than
       integer PK lookup; requires Hibernate mapping change

OPTION C: Cross-table reference resolution (recommended)
  ┌─────────────────────────────────────────────────────┐
  │  Keep previous_version as INT but:                  │
  │  1. When loading obs.getPreviousVersion(), check     │
  │     obs table first, then obs_archive if not found  │
  │  2. The ArchiveService maintains a                  │
  │     uuid→original_table→archive_table mapping       │
  └─────────────────────────────────────────────────────┘
  PRO: No schema change to primary table;
       Fully transparent to most code paths
  CON: getPreviousVersion() needs to be aware of archives;
       Two-table lookup overhead (minor)

RECOMMENDED: Option A as the base rule + Option C for edge cases
  Primary rule: Archive obs only when the entire correction chain
                (all versions) is voided
  Fallback:     Admin can force-archive with cross-table lookup
```

### 2.8 The ArchiveService — Core Service Design

```/dev/null/archive-service-design.md#L1-80
ArchiveService interface responsibilities:

1. ARCHIVE OPERATION
   archiveVoidedData(EntityType, criteria) → ArchivalReport
   ├── Finds all eligible voided rows for entity type
   ├── Validates eligibility (retention period, no active refs)
   ├── Moves rows from primary table to archive table (INSERT+DELETE)
   ├── Does this in a TRANSACTION (atomic move, not two operations)
   └── Logs the archival event (new archive_log table)

2. RESTORE OPERATION
   restoreArchivedData(EntityType, identifier) → RestoredEntity
   ├── Finds row in archive table (by uuid or archive ID)
   ├── Validates that restore is possible (no conflicts)
   ├── Moves row back from archive table to primary table
   ├── Restores parent entities too if needed (encounter, visit)
   └── Logs the restore event

3. QUERY OPERATIONS
   getArchivedObs(searchCriteria) → List<ArchivedObs>
   getArchivedEncounters(searchCriteria) → List<ArchivedEncounter>
   getArchivalLog(dateFrom, dateTo) → List<ArchiveLogEntry>

4. STATUS OPERATIONS
   getArchivalEligibleCount(EntityType) → long
   getArchiveSize(EntityType) → ArchiveSizeReport
   getLastArchivalRun() → ArchivalRun

Key design principle for the MOVE operation:
  The transfer MUST be atomic. Use a single transaction:
    BEGIN TRANSACTION
      INSERT INTO obs_archive SELECT * FROM obs WHERE obs_id=? AND voided=1
      DELETE FROM obs WHERE obs_id=? AND voided=1
    COMMIT
  If either step fails, the transaction rolls back — no data loss.
  The row stays in obs (source of truth).
```

### 2.9 The Scheduled Archival Task

Archiving should happen **automatically in the background** without operator intervention, using the existing `SchedulerService`:

```/dev/null/scheduled-task-design.md#L1-50
VoidedDataArchivalTask extends AbstractTask {

    execute():
    ├── Authenticate as Daemon user (existing pattern)
    ├── Compute cutoff date = NOW - retention.days
    ├── For each entity type in archival order:
    │     ├── obs (leaf first, then obs groups)
    │     ├── orders
    │     ├── encounters
    │     └── visits
    ├── For each entity type:
    │     ├── Fetch eligible IDs in BATCHES (e.g., 1000 at a time)
    │     │   (never load all millions into memory at once)
    │     ├── For each batch:
    │     │     ├── Validate eligibility
    │     │     ├── Perform atomic INSERT+DELETE in transaction
    │     │     └── Commit batch
    │     └── Log batch completion
    └── Write summary to archive_run_log table

Task configuration (via TaskDefinition in scheduler):
  Name:         "Voided Data Archival"
  Class:        org.openmrs.api.task.VoidedDataArchivalTask
  Interval:     Weekly (configurable)
  Start time:   Off-hours (e.g., Sunday 2:00 AM)
  Properties:
    batchSize = 1000        (rows per transaction batch)
    maxRunMinutes = 120     (safety timeout)

GlobalProperties controlling behavior:
  openmrs.archive.enabled          = true
  openmrs.archive.retention.days   = 30
  openmrs.archive.batch.size       = 1000
  openmrs.archive.entities         = obs,encounter,orders,visit
```

### 2.10 The Restore Path — Design for Recoverability

The restore path is just as important as the archive path:

```/dev/null/restore-design.md#L1-55
RESTORE OPERATION DESIGN

Trigger: Admin manually initiates a restore via API or UI
         (or it's triggered automatically by unvoidXxx if the
          entity is found in the archive table, not the primary)

Steps:

1. Locate the archived entity
   → Search obs_archive, encounter_archive, etc. by UUID or ID

2. Determine what else to restore (cascade upward)
   → If restoring a voided obs, check if its parent encounter
      is also in encounter_archive — restore the encounter first
   → If restoring an encounter, check if the visit is archived —
      restore the visit first
   → Restore parents before children (opposite of archive order)

3. Conflict check
   → Is there already a non-voided row with the same UUID in the
      primary table? (Shouldn't happen, but must check)
   → If UUID exists in primary table (different obs_id), flag conflict

4. Atomic INSERT + DELETE in transaction
    BEGIN
      INSERT INTO obs SELECT [all columns except archive metadata]
        FROM obs_archive WHERE obs_id = ?
      DELETE FROM obs_archive WHERE obs_id = ?
    COMMIT

5. Re-establish referential integrity
   → Update obs.previous_version if needed
   → Notify application cache to invalidate affected entities

6. Log the restore event
   → archive_restore_log: who, what, when, reason

Integration with the existing unvoid* service methods:
   When unvoidObs(obs) is called, the ObsService first checks
   if the obs exists in the hot tier (obs table) — if not found,
   checks obs_archive and triggers a restore automatically.
   The existing unvoid callers see no difference.
```

### 2.11 Manual Admin Operations

Beyond the scheduled job, admins need direct control:

```/dev/null/admin-operations.md#L1-35
Admin-facing operations (exposed via API + admin UI):

1. MANUAL ARCHIVE NOW
   → Force archival of a specific patient's voided data
     (useful after a patient merge to clean up immediately)
   ArchiveService.archivePatientVoidedData(patient, reason)

2. MANUAL RESTORE
   → Restore a specific voided entity from archive by UUID
   ArchiveService.restoreArchivedEntity(uuid, entityType)

3. VIEW ARCHIVE CONTENTS
   → Query obs_archive for a patient's history
   ArchiveService.getArchivedObsForPatient(patient)

4. ARCHIVE STATUS REPORT
   → Show: how many rows are eligible, how many archived,
     last run time, archive table sizes
   ArchiveService.getArchivalStatus()

5. DISABLE ARCHIVING FOR A VOID
   → When voiding a specific record, mark it as "do not archive"
   → Useful for voided data under legal hold / active investigation
   voidObs(obs, reason, doNotArchive=true)
   → Adds to a protected_from_archival table (ID + reason + hold_expiry)
```

---

## Part 3: How All the Pieces Fit Together

```/dev/null/complete-architecture.md#L1-80
══════════════════════════════════════════════════════════════════
              ARCHIVING FEATURE — COMPLETE ARCHITECTURE
══════════════════════════════════════════════════════════════════

  EXISTING SYSTEM (unchanged behavior):
  ┌──────────────────────────────────────────────────────────────┐
  │  voidEncounter(encounter, reason)                            │
  │    → AOP: RequiredDataAdvice sets voided=1 fields            │
  │    → EncounterServiceImpl cascades to obs + orders           │
  │    → Hibernate saves to obs table (voided=1 rows accumulate) │
  └──────────────────────────────────────────────────
  
  
```

Now let me continue the complete design in depth.

---

## Part 3: How All the Pieces Fit Together — Complete Architecture

```/dev/null/complete-architecture-full.md#L1-100
══════════════════════════════════════════════════════════════════════
          VOIDED DATA ARCHIVING — COMPLETE SYSTEM ARCHITECTURE
══════════════════════════════════════════════════════════════════════

LAYER 1: EXISTING VOIDING (no changes here)
┌──────────────────────────────────────────────────────────────────┐
│  Any service void* method                                        │
│    → AOP sets voided=1, voidedBy, dateVoided, voidReason         │
│    → Cascade handlers fire (VisitVoidHandler, etc.)              │
│    → Rows land in primary tables with voided=1                   │
│    → PRIMARY TABLE: obs, encounter, orders, visit                │
│      (voided rows accumulate here — the PROBLEM)                 │
└──────────────────────────────────┬───────────────────────────────┘
                                   │  rows sit here aging
                                   │  until retention period expires
                                   ▼
LAYER 2: ARCHIVAL ENGINE (new)
┌──────────────────────────────────────────────────────────────────┐
│  VoidedDataArchivalTask (Scheduler, runs weekly off-hours)       │
│    │                                                             │
│    ├─ Reads GlobalProperty: archive.enabled=true                 │
│    ├─ Reads GlobalProperty: archive.retention.days=30            │
│    ├─ Calculates cutoff = NOW - 30 days                          │
│    │                                                             │
│    └─ Calls ArchiveService.archiveEligibleData():                │
│         ├── Phase 1: Archive eligible OBS                        │
│         │     - WHERE voided=1 AND date_voided < cutoff          │
│         │     - AND no active obs.previous_version ref to it     │
│         │     - Batch INSERT→ obs_archive, DELETE← obs           │
│         ├── Phase 2: Archive eligible ORDERS                     │
│         ├── Phase 3: Archive eligible ENCOUNTERS                 │
│         └── Phase 4: Archive eligible VISITS                     │
└──────────────────────────────────┬───────────────────────────────┘
                                   │ atomic move per batch
                                   ▼
LAYER 3: ARCHIVE STORAGE (new cold tier)
┌──────────────────────────────────────────────────────────────────┐
│  obs_archive, encounter_archive, orders_archive, visit_archive   │
│    - Identical column structure to source tables                 │
│    - Extra columns: date_archived, archived_by, archive_reason   │
│    - No FK constraints (cross-table refs resolved via UUID)      │
│    - Can be placed on separate slower/cheaper storage            │
│    - Indexed for: uuid, date_voided, patient UUID                │
└──────────────────────────────────┬───────────────────────────────┘
                                   │ if restore needed
                                   ▼
LAYER 4: RESTORE ENGINE (new, mirrors archival)
┌──────────────────────────────────────────────────────────────────┐
│  ArchiveService.restoreArchivedEntity(uuid) OR                   │
│  Triggered automatically when unvoidObs() target is not in obs   │
│    │                                                             │
│    ├─ Look up entity in *_archive table by UUID                  │
│    ├─ Restore parents first (encounter before obs)               │
│    ├─ Atomic DELETE← *_archive, INSERT→ primary table            │
│    └─ Log restore event to archive_restore_log                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Detailed Component-by-Component Design

### 4.1 New Database Tables Required

```/dev/null/new-db-tables.md#L1-80
NEW TABLES NEEDED (all managed via Liquibase changesets):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TABLE 1: obs_archive
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Mirrors obs exactly + 3 archive metadata columns.
  PRIMARY KEY (obs_id)  ← same value as original, not auto-increment
  INDEX on (uuid)       ← for restore-by-uuid lookups
  INDEX on (person_id, date_voided)  ← for patient history queries
  INDEX on (date_archived)           ← for archive log/audit queries
  NO foreign keys       ← self-contained cold storage

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TABLE 2: encounter_archive
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Mirrors encounter exactly + 3 archive metadata columns.
  Same index strategy.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TABLE 3: orders_archive
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Mirrors orders exactly + 3 archive metadata columns.
  Note: drug_order and test_order (subclass tables) need
  their own archive tables too (drug_order_archive, etc.)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TABLE 4: visit_archive
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Mirrors visit exactly + 3 archive metadata columns.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TABLE 5: archive_run_log
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Tracks each archival run for auditability:

  archive_run_id   INT  PK auto-increment
  run_started_at   DATETIME
  run_completed_at DATETIME
  triggered_by     INT   FK→ users (NULL = scheduled daemon)
  status           VARCHAR(20)  [RUNNING, COMPLETED, FAILED, PARTIAL]
  cutoff_date      DATETIME     (date_voided < this date)
  obs_archived     INT          count of obs moved
  encounters_arch  INT          count of encounters moved
  orders_archived  INT          count of orders moved
  visits_archived  INT          count of visits moved
  error_message    TEXT         (if status=FAILED)
  notes            VARCHAR(500) (e.g., "manual run by admin")

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TABLE 6: archive_restore_log
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Tracks every restore operation:

  restore_id        INT  PK auto-increment
  restored_at       DATETIME
  restored_by       INT  FK→ users
  entity_type       VARCHAR(50)  (e.g., "obs", "encounter")
  entity_uuid       VARCHAR(38)
  entity_id         INT
  restore_reason    VARCHAR(500)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TABLE 7: archive_exclusion_list  (optional, for legal hold)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Records entities that must NOT be archived:

  exclusion_id      INT  PK auto-increment
  entity_type       VARCHAR(50)
  entity_uuid       VARCHAR(38)
  entity_id         INT
  excluded_by       INT  FK→ users
  excluded_at       DATETIME
  exclusion_reason  VARCHAR(500)  (e.g., "Legal hold until 2026-01-01")
  hold_expiry_date  DATETIME      (NULL = permanent hold)
```

### 4.2 GlobalProperties for Configuration

```/dev/null/global-properties.md#L1-40
New GlobalProperties added to OpenmrsConstants.CORE_GLOBAL_PROPERTIES():

┌──────────────────────────────────────────┬──────────┬────────────────────────────┐
│  Property Key                            │ Default  │ Description                │
├──────────────────────────────────────────┼──────────┼────────────────────────────┤
│ archive.voided.enabled                   │  false   │ Master on/off switch.      │
│                                          │          │ Default OFF for safety;    │
│                                          │          │ admin must opt-in          │
├──────────────────────────────────────────┼──────────┼────────────────────────────┤
│ archive.voided.retention.days            │  30      │ Voided rows must be this   │
│                                          │          │ many days old before       │
│                                          │          │ eligible for archiving     │
├──────────────────────────────────────────┼──────────┼────────────────────────────┤
│ archive.voided.entities                  │  obs     │ Comma-separated list of    │
│                                          │          │ entity types to archive.   │
│                                          │          │ Start conservative (obs    │
│                                          │          │ only), expand as proven    │
├──────────────────────────────────────────┼──────────┼────────────────────────────┤
│ archive.voided.batch.size                │  500     │ Rows per DB transaction    │
│                                          │          │ batch. Smaller = safer     │
│                                          │          │ (less lock contention)     │
├──────────────────────────────────────────┼──────────┼────────────────────────────┤
│ archive.voided.max.runtime.minutes       │  120     │ Safety cutoff for the      │
│                                          │          │ scheduled task             │
├──────────────────────────────────────────┼──────────┼────────────────────────────┤
│ archive.voided.allow.manual.restore      │  true    │ Allow admins to restore    │
│                                          │          │ archived data via API      │
├──────────────────────────────────────────┼──────────┼────────────────────────────┤
│ archive.voided.auto.restore.on.unvoid    │  true    │ Automatically restore when │
│                                          │          │ unvoidXxx() is called on   │
│                                          │          │ an archived entity         │
└──────────────────────────────────────────┴──────────┴────────────────────────────┘
```

### 4.3 The Atomic Move — The Single Most Important Operation

The core of the entire design is this single operation, which must be **atomic, transactional, and safe**:

```/dev/null/atomic-move-design.md#L1-60
THE ATOMIC MOVE OPERATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ARCHIVE MOVE (hot → cold):
┌─────────────────────────────────────────────────────────┐
│  BEGIN TRANSACTION (isolation: REPEATABLE READ)         │
│                                                         │
│  Step 1: Lock the source row                            │
│    SELECT * FROM obs WHERE obs_id=? FOR UPDATE          │
│    → Prevents concurrent modification during move       │
│                                                         │
│  Step 2: Re-validate eligibility under lock             │
│    Confirm: voided=1, date_voided < cutoff, no active   │
│    references, not in exclusion list                    │
│    → If any check fails: ROLLBACK (skip this row)       │
│                                                         │
│  Step 3: Insert into archive table                      │
│    INSERT INTO obs_archive                              │
│      SELECT *, NOW(), current_user_id, 'scheduled'      │
│      FROM obs WHERE obs_id=?                            │
│                                                         │
│  Step 4: Delete from primary table                      │
│    DELETE FROM obs WHERE obs_id=?                       │
│    → Row vanishes from primary table                    │
│                                                         │
│  Step 5: Insert into archive_run_log detail             │
│    (batch-level logging, not per-row for performance)   │
│                                                         │
│  COMMIT                                                 │
└─────────────────────────────────────────────────────────┘
  If COMMIT fails → automatic ROLLBACK
  Row stays in obs table (primary table = source of truth)
  No data loss is possible

RESTORE MOVE (cold → hot):
┌─────────────────────────────────────────────────────────┐
│  BEGIN TRANSACTION                                      │
│                                                         │
│  Step 1: Lock the archive row                           │
│    SELECT * FROM obs_archive WHERE uuid=? FOR UPDATE    │
│                                                         │
│  Step 2: Conflict check in primary table                │
│    SELECT COUNT(*) FROM obs WHERE obs_id = archived_id  │
│    → If row already exists: ROLLBACK (already restored) │
│                                                         │
│  Step 3: Insert back into primary table                 │
│    INSERT INTO obs                                      │
│      SELECT [all original columns, not archive columns] │
│      FROM obs_archive WHERE uuid=?                      │
│                                                         │
│  Step 4: Delete from archive table                      │
│    DELETE FROM obs_archive WHERE uuid=?                 │
│                                                         │
│  Step 5: Insert into archive_restore_log                │
│                                                         │
│  COMMIT                                                 │
└─────────────────────────────────────────────────────────┘
```

### 4.4 How the Existing Service Layer Changes (Minimally)

The guiding principle is **zero breakage of existing callers**. Only targeted, minimal changes:

```/dev/null/service-layer-changes.md#L1-70
CHANGE 1: ObsService.getObs(obsId)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BEFORE: Looks up only in obs table. Returns null if not found.
AFTER:  Looks up in obs table first.
        If not found AND archive.enabled=true:
          look up in obs_archive and return (marked as archived).
        Callers that needed to display obs for audit can now see it.
        Callers that expected null now get the archived entity back.

CHANGE 2: ObsService.unvoidObs(obs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BEFORE: Clears void fields, saves obs. Works only if obs is in obs table.
AFTER:  Before clearing void fields, check:
          Is this obs currently in obs_archive?
          If YES: restore it first (move back to obs table),
                  then proceed with normal unvoid logic.
          If NO: proceed normally (obs is in hot tier, not yet archived).
        Result: unvoidObs() works transparently whether archived or not.

CHANGE 3: EncounterService.unvoidEncounter(encounter)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BEFORE: Unvoids encounter + its obs/orders (must be in primary tables).
AFTER:  Before unvoiding obs/orders, check if they are in archive tables.
        Auto-restore from archive before applying unvoid logic.

CHANGE 4: PatientService.voidPatient() / mergePatients()
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NO CHANGE to the void operation itself.
After mergePatients() completes successfully, optionally schedule
an immediate archival of the now-voided notPreferred patient's data.
This is configurable: archive.voided.archive.after.merge=true

CHANGE 5: All DAO getAll(includeVoided=true) calls
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BEFORE: includeVoided=true includes all voided rows in primary table.
AFTER:  includeVoided=true means:
          primary table rows (all) + archive table rows (via UNION)
        This requires a new DAO method or enhanced query.
        BUT this is only needed for audit/admin contexts.
        The default includeVoided=false path is completely unchanged.

CRITICAL NON-CHANGE: All void* methods
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
voidObs(), voidEncounter(), voidPatient(), voidOrder(), voidVisit()
→ ZERO CHANGES. These continue to write voided rows into the
  primary table exactly as before. Archiving is a SEPARATE,
  DEFERRED, BACKGROUND operation. It does not change the void path.
```

### 4.5 Handling the `obs.previous_version` Chain at Archival Time

Here is the concrete decision tree the archive engine uses for each voided obs:

```/dev/null/previous-version-decision-tree.md#L1-60
For each candidate voided obs row:

  obs_candidate (voided=1, date_voided < cutoff)
         │
         ▼
  Q1: Is this obs referenced as previous_version
      by any ACTIVE (voided=0) obs in the obs table?
         │
    YES ─┼─► SKIP THIS ROW.
         │   It cannot be archived yet because the active
         │   version (obs_id=102) still holds a FK pointing
         │   to it (obs_id=101). Archiving 101 would leave
         │   102 with a dangling previous_version reference.
         │   Revisit this row in a future archival run.
         │
    NO ──┼─► Q2: Is this obs referenced as previous_version
         │       by any VOIDED obs in the obs table?
         │          │
         │     YES ─┼─► The referencing voided obs should be
         │          │   archived FIRST (it becomes eligible later).
         │          │   Mark both as a "correction chain group".
         │          │   Archive the chain together in the correct
         │          │   order: oldest voided first (obs_id=100),
         │          │   then the next (obs_id=101), etc.
         │          │   The previous_version FK remains internally
         │          │   consistent within obs_archive.
         │          │
         │     NO ──┼─► Q3: Is this obs in the archive_exclusion_list?
         │                   │
         │              YES ─┼─► SKIP. Legal/admin hold.
         │                   │
         │              NO ──┼─► ELIGIBLE FOR ARCHIVING ✓
         │                        Move to obs_archive atomically.
         ▼
  After archiving: no row remains in obs table with a dangling
  previous_version FK, because all referencing active rows were
  checked in Q1 and the chain structure in obs_archive is intact.
```

### 4.6 The Archive Engine's Eligibility Query (Conceptual SQL)

This is what the ArchiveService runs to find candidates for each batch:

```/dev/null/eligibility-query-design.sql#L1-55
-- Find obs eligible for archiving (conceptual, not final SQL)
-- Run in batches of ${archive.voided.batch.size} rows

SELECT o.obs_id
FROM obs o
WHERE
  -- Gate 1: Must be voided
  o.voided = 1

  -- Gate 2: Must be older than retention period
  AND o.date_voided < DATE_SUB(NOW(), INTERVAL :retentionDays DAY)

  -- Gate 3: Must NOT be referenced as previous_version by any
  --         ACTIVE (non-voided) obs in the hot tier
  AND NOT EXISTS (
    SELECT 1 FROM obs active_ref
    WHERE active_ref.previous_version = o.obs_id
      AND active_ref.voided = 0
  )

  -- Gate 4: Must NOT be in the exclusion list (legal hold)
  AND NOT EXISTS (
    SELECT 1 FROM archive_exclusion_list ex
    WHERE ex.entity_type = 'obs'
      AND ex.entity_id = o.obs_id
      AND (ex.hold_expiry_date IS NULL OR ex.hold_expiry_date > NOW())
  )

ORDER BY o.date_voided ASC   -- oldest first = most impactful
LIMIT :batchSize
FOR UPDATE SKIP LOCKED        -- skip rows locked by other transactions
                              -- (safe for concurrent runs or retries)
;

-- Similarly for encounters:
SELECT e.encounter_id FROM encounter e
WHERE e.voided = 1
  AND e.date_voided < DATE_SUB(NOW(), INTERVAL :retentionDays DAY)
  -- All obs in this encounter must already be archived (not in obs table)
  AND NOT EXISTS (
    SELECT 1 FROM obs o
    WHERE o.encounter_id = e.encounter_id
      AND o.voided = 1  -- voided obs still in hot tier not yet archived
  )
  -- No active (non-voided) obs either (shouldn't exist, but safety check)
  AND NOT EXISTS (
    SELECT 1 FROM obs o
    WHERE o.encounter_id = e.encounter_id AND o.voided = 0
  )
  AND NOT EXISTS (
    SELECT 1 FROM archive_exclusion_list ex
    WHERE ex.entity_type = 'encounter' AND ex.entity_id = e.encounter_id
  )
ORDER BY e.date_voided ASC
LIMIT :batchSize
FOR UPDATE SKIP LOCKED
;
```

### 4.7 Impact on Hibernate and the ORM Layer

```/dev/null/hibernate-impact.md#L1-55
HOW HIBERNATE IS AFFECTED (and how to isolate the impact):

PROBLEM: Hibernate's session and 2nd-level cache does not know
about archive tables. If a row is archived (removed from obs),
Hibernate may still have it cached. Subsequent loads could return
stale cached objects while the DB row is gone.

SOLUTION A: Cache Invalidation on Archive
  When a batch is archived, explicitly evict affected entities from
  the Hibernate 2nd-level cache (if enabled):
    sessionFactory.getCache().evict(Obs.class, obsId);
  The archival task has access to the Hibernate SessionFactory
  via the DAO layer and can perform targeted evictions per batch.

PROBLEM: Hibernate mappings for Obs, Encounter, etc. all point
to the primary tables. Archive tables have no Hibernate entity
class mapping — they are invisible to the ORM.

SOLUTION B: Archive tables accessed via Native SQL only
  The ArchiveService DAO uses plain JDBC / native SQL queries
  (not HQL/JPQL) to read from and write to archive tables.
  This is intentional — archive tables are NOT domain objects
  in the Hibernate model. They are infrastructure tables.
  The ArchiveService returns simple DTO objects (not Hibernate
  entities) when querying archive tables.

  Result:
    Normal clinical code: uses Hibernate entities (no change)
    Archive/restore code: uses raw JDBC for archive tables

PROBLEM: ImmutableObsInterceptor only allows voided/voidReason
fields to be changed on Obs. The DELETE operation (part of archive)
goes through JDBC, not Hibernate flush — so ImmutableObsInterceptor
does NOT fire for the archive DELETE. This is correct behavior:
the archive DELETE is an admin-level infrastructure operation,
not a Hibernate entity update.

PROBLEM: Hibernate Envers (@Audited) writes to obs_AUD.
When obs is deleted from obs table (archive move), Envers records
a REVTYPE=2 (DELETE) revision in obs_AUD.
When obs is restored (archive → obs), Envers records a new
INSERT revision.

This is actually DESIRABLE — the Envers audit trail faithfully
records: "this obs was deleted at T1 (archived), then re-inserted
at T2 (restored)". The full history is preserved across both tables.
```

### 4.8 Handling Inheritance Tables (Drug Orders, Test Orders)

Orders in OpenMRS use table-per-subclass inheritance (`orders`, `drug_order`, `test_order`):

```/dev/null/order-inheritance-archiving.md#L1-40
Current order inheritance structure:
  orders table  (base columns: order_id, patient_id, voided, ...)
    ↑ joined by order_id
  drug_order table  (drug-specific columns: drug_id, dose, ...)
  test_order table  (test-specific columns: concept_id, ...)

For archiving this means:
  When archiving a voided DrugOrder:
    Step 1: INSERT into drug_order_archive
            SELECT drug-specific columns FROM drug_order WHERE order_id=?
    Step 2: INSERT into orders_archive
            SELECT base columns FROM orders WHERE order_id=?
    Step 3: DELETE FROM drug_order WHERE order_id=?
    Step 4: DELETE FROM orders WHERE order_id=?
    All in ONE TRANSACTION

  Archive table structure:
    orders_archive    — mirrors orders base table
    drug_order_archive — mirrors drug_order subclass table
    test_order_archive — mirrors test_order subclass table

  Restore reverses the steps:
    Step 1: INSERT into orders SELECT FROM orders_archive
    Step 2: INSERT into drug_order SELECT FROM drug_order_archive
    Step 3: DELETE both from archive tables
    All in ONE TRANSACTION

  The archive DAO must know the order type (from orders.order_type)
  to determine which subclass archive table to use.
```

---

## Part 5: Risk Analysis and Mitigation

### 5.1 Risk Register

```/dev/null/risk-register.md#L1-80
┌──┬──────────────────────────────────┬──────────┬──────────────────────────────────┐
│  │ RISK                             │ SEVERITY │ MITIGATION                       │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│1 │ Data loss during archive move    │ CRITICAL │ Atomic transaction (INSERT then  │
│  │ (INSERT succeeds, DELETE fails,  │          │ DELETE in same txn). If DELETE   │
│  │ or vice versa)                   │          │ fails, INSERT rolls back.         │
│  │                                  │          │ Primary table = source of truth.  │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│2 │ Archiving a row that should not  │ HIGH     │ Retention period gate (30 days). │
│  │ have been archived (was going    │          │ Exclusion list for legal holds.   │
│  │ to be unvoided soon)             │          │ Restore path always available.    │
│  │                                  │          │ Admin UI shows what was archived.│
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│3 │ previous_version FK broken:      │ HIGH     │ Gate 1 in eligibility query      │
│  │ active obs points to archived obs│          │ (NOT EXISTS check). Only archive │
│  │ via previous_version=old_obs_id  │          │ when no active obs references it. │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│4 │ Archive task locks production    │ MEDIUM   │ SKIP LOCKED in eligibility query │
│  │ tables during clinical hours     │          │ avoids contention. Batch size    │
│  │                                  │          │ tunable. Run during off-hours.   │
│  │                                  │          │ Max runtime limit kills task.    │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│5 │ Restore conflicts: archived obs  │ MEDIUM   │ Conflict check in restore: verify│
│  │ has same obs_id as a new obs     │          │ no row with same obs_id in obs.  │
│  │ (obs_id was reused)              │          │ UUIDs are globally unique so     │
│  │                                  │          │ UUID-based restore is safe.      │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│6 │ Hibernate cache inconsistency:   │ MEDIUM   │ Explicit cache eviction per      │
│  │ stale entities after archive     │          │ archived entity UUID. Or disable │
│  │                                  │          │ 2nd-level cache for affected     │
│  │                                  │          │ regions during archival run.     │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│7 │ Envers audit tables (obs_AUD)    │ LOW-MED  │ Envers auto-records DELETE when  │
│  │ grow even without main table     │          │ obs is archived. obs_AUD rows    │
││  │ grow even without main table     │          │ obs is archived. obs_AUD rows    │
│  │ (AUD table keeps history of      │          │ themselves could be archived      │
│  │ the archived/deleted rows)       │          │ separately (future scope).        │
│  │                                  │          │ Phase 1: accept AUD table growth. │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│8 │ Module compatibility: a module   │ LOW-MED  │ ArchiveService is additive.      │
│  │ does its own obs queries without │          │ Modules querying obs table still │
│  │ knowing about obs_archive        │          │ work on the hot tier. They just  │
│  │                                  │          │ won't see archived rows (which   │
│  │                                  │          │ is the correct behavior anyway:  │
│  │                                  │          │ archived = voided = invisible).  │
├──┼──────────────────────────────────┼──────────┼──────────────────────────────────┤
│9 │ Database-specific SQL: SKIP      │ LOW      │ Abstract SKIP LOCKED behind a    │
│  │ LOCKED syntax not on all DBs     │          │ DAO strategy interface. MySQL,   │
│  │                                  │          │ PostgreSQL support it. H2 (test) │
│  │                                  │          │ uses fallback (no-lock mode).    │
└──┴──────────────────────────────────┴──────────┴──────────────────────────────────┘
```

---

## Part 6: Phased Rollout Plan

One of the most important aspects of this design is that it should be introduced **incrementally and safely**. Here is the phased strategy:

```/dev/null/phased-rollout.md#L1-110
════════════════════════════════════════════════════════════════
  PHASED ROLLOUT — VOIDED DATA ARCHIVING
════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────┐
│  PHASE 0 — MEASUREMENT (No code changes)                    │
│  Goal: Understand the problem before solving it             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Add diagnostic queries / admin report to show:            │
│  - Total rows in obs / encounters / orders / visits         │
│  - Voided rows vs. active rows per table                    │
│  - Oldest voided rows (date_voided histogram)               │
│  - Growth rate of voided rows (per week/month)              │
│  - Estimated archive-eligible rows at 30/60/90 day windows  │
│                                                             │
│  Deliverable: "Archive Readiness Report" that admins can    │
│  run to see the scale of the problem at their deployment.   │
│                                                             │
│  Value: Justifies the feature. Shows the exact benefit      │
│  before any risk is taken. Zero chance of data issues.      │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1 — INFRASTRUCTURE (Schema + Config, No Archiving)   │
│  Goal: Add the plumbing without flipping the switch         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Liquibase changesets: create all *_archive tables,      │
│     archive_run_log, archive_restore_log,                   │
│     archive_exclusion_list                                  │
│                                                             │
│  2. Add GlobalProperties (all defaulting to disabled):      │
│     archive.voided.enabled = false                          │
│                                                             │
│  3. Add ArchiveService interface + impl (skeleton)          │
│     - getArchivalStatus() works                             │
│     - archiveEligibleData() is a no-op when disabled        │
│                                                             │
│  4. Register the VoidedDataArchivalTask in scheduler        │
│     - Task exists but does nothing when enabled=false       │
│                                                             │
│  5. Add PrivilegeConstants:                                 │
│     ARCHIVE_VOIDED_DATA, RESTORE_ARCHIVED_DATA              │
│                                                             │
│  Risk: Zero. Nothing is archived. Tables exist but empty.   │
│  New schema is purely additive.                             │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2 — OBS ARCHIVING ONLY (Narrowest scope)             │
│  Goal: Archive only obs (highest impact, well-understood)   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Implement eligibility query for obs only                │
│     (with all Gates: retention, no active ref, exclusion)   │
│                                                             │
│  2. Implement atomic move: obs → obs_archive                │
│                                                             │
│  3. Implement obs restore: obs_archive → obs                │
│                                                             │
│  4. Add auto-restore hook in ObsService.unvoidObs()         │
│                                                             │
│  5. Update ObsService.getObs() to check obs_archive         │
│     if not found in primary obs table                       │
│                                                             │
│  6. Enable via GlobalProperty (opt-in):                     │
│     archive.voided.enabled = true                           │
│     archive.voided.entities = obs   ← only obs              │
│                                                             │
│  7. Run DRY-RUN mode first:                                 │
│     A mode where the task REPORTS what it would archive     │
│     but performs no actual database moves. Admins can       │
│     review the candidate list before committing.            │
│     archive.voided.dry.run = true                           │
│                                                             │
│  Testing requirement:                                       │
│  - Unit tests: atomic move, rollback on failure             │
│  - Integration tests: archive obs, unvoid archived obs      │
│  - Performance tests: obs table query speed before/after    │
│                                                             │
│  Risk: Low-Medium. Only affects voided obs rows.            │
│  Restore path fully tested. Dry-run available.              │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3 — ENCOUNTER + ORDER ARCHIVING                      │
│  Goal: Extend to encounters and orders                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Implement eligibility query for encounters              │
│     (children — obs, orders — must be in archive first)     │
│                                                             │
│  2. Implement eligibility query for orders                  │
│     (handle drug_order / test_order subclass tables)        │
│                                                             │
│  3. Implement atomic move for both                          │
│                                                             │
│  4. Add auto-restore hooks in EncounterService.unvoidEncounter│
│                                                             │
│  5. Extend archive.voided.entities GlobalProperty:          │
│     archive.voided.entities = obs,encounter,orders          │
│                                                             │
│  Risk: Medium. More complex FK relationships.               │
│  Gated behind the Phase 2 success.                          │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 4 — VISIT + PATIENT ARCHIVING (Full coverage)        │
│  Goal: Complete the archiving hierarchy                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Implement visit archiving                               │
│     (all encounters must already be in encounter_archive)   │
│                                                             │
│  2. Implement patient/person archiving (rare, complex)      │
│     (all visits, encounters, obs, orders archived first)    │
│                                                             │
│  3. Admin UI: Archive Explorer                              │
│     - View archived records by patient                      │
│     - Restore individual records                            │
│     - View archive_run_log history                          │
│     - View archive size statistics                          │
│                                                             │
│  Risk: Higher complexity, lower frequency of voids.         │
│  Most value already captured by Phases 2 and 3.             │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 5 — ADVANCED / FUTURE SCOPE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Envers (AUD table) archiving                            │
│     Archive obs_AUD rows for archived obs to obs_aud_archive│
│                                                             │
│  2. External archive storage                                │
│     Export archive tables to compressed files / separate DB │
│     for very large deployments (national scale)             │
│                                                             │
│  3. Archive analytics                                       │
│     Allow read-only queries across both hot+cold tiers      │
│     for research/audit purposes without restoring           │
│                                                             │
│  4. Legal hold management UI                                │
│     Manage the archive_exclusion_list via admin interface   │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 7: The Full System — Master Design Diagram

```/dev/null/master-design-diagram.md#L1-130
══════════════════════════════════════════════════════════════════════════
        VOIDED DATA ARCHIVING — MASTER DESIGN DIAGRAM
══════════════════════════════════════════════════════════════════════════

TIME AXIS →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→

                   [T0]              [T0+30days]         [T0+restore?]
                    │                    │                     │
                    │                    │                     │
  ┌─────────────────▼────────────────────▼─────────────────────▼──────┐
  │                      PRIMARY TABLE (obs)                          │
  │                                                                   │
  │  obs_id=101  voided=1  void_reason="Correction"  date_voided=T0  │─►archived at T0+30d
  │  obs_id=102  voided=0  (active, previous_version=101)            │  (stays; active ref)
  │                                                                   │
  │  obs_id=201  voided=1  void_reason="Wrong patient" date_voided=T0│─►archived at T0+30d
  │  obs_id=202  voided=1  void_reason="Wrong patient" date_voided=T0│─►archived at T0+30d
  │             (both voided together, no active references)          │
  │                                                                   │
  └───────────────────────────────────────────────────────────────────┘
                              │                          ▲
                    Archive Task                    Restore path
                    (atomic move)                  (on unvoid call)
                              │                          │
  ┌───────────────────────────▼──────────────────────────┴───────────┐
  │                    ARCHIVE TABLE (obs_archive)                    │
  │                                                                   │
  │  obs_id=201  voided=1  void_reason="Wrong patient"               │
  │              date_archived=T0+30d  archived_by=scheduler_daemon  │
  │                                                                   │
  │  obs_id=202  voided=1  void_reason="Wrong patient"               │
  │              date_archived=T0+30d  archived_by=scheduler_daemon  │
  │                                                                   │
  └───────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════
  HOW ALL COMPONENTS CONNECT
═══════════════════════════════════════════════════════════════════════

  ┌──────────────────┐      weekly       ┌──────────────────────────┐
  │  SchedulerService│──────trigger─────►│  VoidedDataArchivalTask  │
  │  (existing)      │                   │  (new, extends           │
  └──────────────────┘                   │   AbstractTask)          │
                                         └────────────┬─────────────┘
                                                      │ calls
                                                      ▼
  ┌──────────────────┐    reads    ┌──────────────────────────────────┐
  │  GlobalProperty  │◄───────────│  ArchiveService                  │
  │  (existing)      │            │  (new service interface + impl)   │
  │  archive.enabled │            │                                   │
  │  retention.days  │            │  archiveEligibleData()            │
  │  batch.size etc  │            │  restoreArchivedEntity(uuid)      │
  └──────────────────┘            │  getArchivedObs(patient)          │
                                  │  getArchivalStatus()              │
                                  │  dryRun()                        │
                                  └────────────┬─────────────────────┘
                                               │ uses
                                               ▼
                                  ┌──────────────────────────────────┐
                                  │  ArchiveDAO                      │
                                  │  (new, uses native JDBC/SQL)     │
                                  │  - NOT a Hibernate entity DAO    │
                                  │  - Direct JDBC for archive tables │
                                  │  - Uses same SessionFactory      │
                                  │    (to participate in txn)       │
                                  └───────────┬──────────────────────┘
                                              │ reads/writes
                              ┌───────────────┴───────────────┐
                              │                               │
                   ┌──────────▼──────────┐       ┌───────────▼──────────┐
                   │  Primary Tables     │       │  Archive Tables      │
                   │  (Hibernate ORM)    │       │  (JDBC native only)  │
                   │                     │       │                      │
                   │  obs                │◄─────►│  obs_archive         │
                   │  encounter          │       │  encounter_archive   │
                   │  orders             │       │  orders_archive      │
                   │  visit              │       │  visit_archive       │
                   └─────────────────────┘       │  archive_run_log     │
                                                 │  archive_restore_log │
                                                 │  archive_exclusion   │
                                                 └──────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │  INTEGRATION POINTS WITH EXISTING SERVICES                      │
  │                                                                 │
  │  ObsService.unvoidObs()                                         │
  │    → If obs not in primary table: auto-restore from obs_archive │
  │    → Then apply normal unvoid logic                             │
  │                                                                 │
  │  ObsService.getObs(id)                                          │
  │    → Check obs table first; fallback to obs_archive             │
  │    → Return DTO with isArchived=true flag                       │
  │                                                                 │
  │  EncounterService.unvoidEncounter()                             │
  │    → Auto-restore encounter from encounter_archive if needed    │
  │    → Then restore its obs/orders from their archives            │
  │                                                                 │
  │  PatientService.mergePatients()                                 │
  │    → After successful merge: optionally trigger immediate       │
  │      archival of voided notPreferred patient's data             │
  │      (if archive.voided.archive.after.merge=true)              │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Design Decision Summary — The Key Choices

```/dev/null/design-decisions.md#L1-80
KEY DESIGN DECISIONS AND WHY EACH WAS MADE:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION 1: Same database, separate tables (not a separate DB)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WHY: OpenMRS deployments range from a single laptop in a rural
  clinic to national-scale installations. A separate database
  would require connection pool management, distributed
  transactions, and deployment complexity that most sites cannot
  handle. Keeping archive tables in the same database:
    - Allows atomic transactions across primary + archive tables
    - Requires no new database connections or configuration
    - Works with the existing Liquibase migration system
    - Is reversible via a simple Liquibase rollback
  TRADE-OFF: Does not fully separate storage; a future Phase 5
  can optionally export to external storage.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION 2: Native JDBC for archive tables, not Hibernate entities
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WHY: Creating Hibernate entity classes for archive tables would
  mean doubling the domain model, confusing every developer, and
  risking accidental use of archive entities in clinical code.
  Archive tables are infrastructure, not domain objects.
  JDBC keeps them invisible to normal ORM code paths.
  TRADE-OFF: Less type safety, raw SQL must be maintained.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION 3: Deferred archiving (not immediate at void time)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WHY: Moving a row to archive at the exact moment of voiding
  would tightly couple the void operation to the archive
  infrastructure. If the archive table has a problem (disk full,
  schema migration in progress), voiding would fail. Voiding
  must remain simple, fast, and independent.
  Deferred archiving via background task decouples these concerns
  completely. Voiding always succeeds. Archiving catches up later.
  TRADE-OFF: Hot tier still contains recent voided rows during
  the retention window. This is acceptable — and intentional
  (the retention window exists precisely for easy unvoid).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION 4: Primary table is always the source of truth
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WHY: The atomic move is INSERT-then-DELETE. If the DELETE fails,
  the transaction rolls back — the row stays in the primary table.
  The archive table never becomes the authoritative copy until the
  DELETE from primary is committed. This makes crash recovery
  trivial: if the system crashes mid-archive, rows remain in the
  primary table (duplicated briefly in archive if INSERT committed
  before crash, but the transaction prevents this scenario).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION 5: Archiving is opt-in, disabled by default
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WHY: OpenMRS serves many contexts. A small 2-year-old clinic
  with 500 patients does not need archiving. Enabling by default
  would introduce background processes and complexity to every
  deployment regardless of need. Admins who have a scale problem
  know they have it (slow queries, large DB) and will opt-in.
  This also gives time for the feature to mature in production
  before being widely adopted.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION 6: Start with obs only (narrowest scope first)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WHY: The obs table grows fastest due to the immutable
  void-and-replace pattern. Every single obs edit creates a
  voided row. This is the biggest pain point. Solving obs first
  delivers 80% of the value with 40% of the complexity.
  Encounters and orders are voided far less frequently.
  Starting narrow means: faster delivery, lower risk, real
  production validation before expanding scope.
```

---

## Summary: The Problem and Solution in One View

```/dev/null/executive-summary.md#L1-60
THE PROBLEM IN ONE SENTENCE:
  Every obs edit (immutable pattern) permanently adds a voided row
  to the obs table. Over years, voided rows outnumber active rows,
  causing slow queries, storage bloat, and session overhead —
  with no current way to clean them up safely.

THE SOLUTION IN ONE SENTENCE:
  A background archival task periodically moves eligible voided rows
  from primary clinical tables into mirror archive tables within the
  same database, using atomic transactions, with a transparent restore
  path invoked automatically on unvoid operations.

WHAT CHANGES FOR CLINICAL USERS:
  Nothing. Voiding, unvoiding, and all clinical queries work
  exactly the same. The archival is completely invisible.

WHAT CHANGES FOR DEVELOPERS:
  Almost nothing for most developers. Only the ArchiveService
  team and service-layer maintainers need to understand the
  auto-restore hooks in unvoidObs() and getObs().

WHAT CHANGES FOR ADMINS:
  New GlobalProperties to configure the feature.
  New scheduled task visible in the scheduler admin UI.
  New Archive Explorer UI to query archived data and run restores.
  New diagnostic report to see archival statistics.

WHAT THE DATABASE LOOKS LIKE AFTER 2 YEARS WITH ARCHIVING:
  BEFORE (no archiving):
    obs table: 10,000,000 rows  (7M voided, 3M active)
    Queries: slow, index fragmented

  AFTER (with 30-day archiving):
    obs table:        3,100,000 rows  (100K recent voided, 3M active)
    obs_archive:      6,900,000 rows  (all old voided, rarely touched)
    Active queries:   fast, clean index on small hot table
    Audit queries:    still possible via obs_archive
    Restore:          available at any time via API or UI
```

---

That is the **complete design** — from the root cause all the way through every architectural decision, risk, and phased delivery plan. To recap the key pillars:

1. **Why it exists** — the immutable obs pattern guarantees permanent voided row growth; at scale this breaks query performance
2. **What it does** — moves eligible voided rows from primary tables to mirror archive tables inside the same database
3. **How it stays safe** — atomic INSERT+DELETE transactions, deferred execution, retention windows, eligibility gates, and a full restore path
4. **How it integrates** — background scheduler task, zero changes to void* methods, transparent auto-restore on unvoid*, native JDBC for archive tables to stay invisible to the ORM
5. **How it ships** — five phases from measurement-only to full coverage, each individually safe and reversible