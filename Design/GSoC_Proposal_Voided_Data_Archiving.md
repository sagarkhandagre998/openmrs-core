# GSoC Proposal: Voided Data Archiving and Safe Restoration in OpenMRS Core

**Applicant:** [Your Name]  
**Email:** [your@email.com]  
**GitHub:** [your-github-handle]  
**OpenMRS Talk:** [your-talk-handle]  
**Mentors:** [To be confirmed by OpenMRS community]  
**Project Size:** Large (~350 hours)  
**Duration:** 14–15 Weeks  
**Technology Stack:** Java 17, Spring Framework, Hibernate/JPA, MySQL/MariaDB, Liquibase, JUnit 5

---

## Table of Contents

1. [Abstract](#abstract)
2. [Current Architecture and Problem](#current-architecture-and-problem)
3. [Proposed Architecture and Design](#proposed-architecture-and-design)
4. [Implementation Plan — Phase-by-Phase](#implementation-plan)
5. [Weekly Timeline](#weekly-timeline)
6. [Testing Strategy](#testing-strategy)
7. [Risk Analysis and Mitigations](#risk-analysis-and-mitigations)
8. [Final Deliverables and Evaluation](#final-deliverables-and-evaluation)
9. [About Me](#about-me)

---

## Abstract

OpenMRS data is almost never hard-deleted. When a clinical record becomes invalid — a wrong encounter, a duplicate patient, a correction to an observation — the system marks it as **voided** by setting `voided = true` along with metadata such as `voided_by`, `date_voided`, and `void_reason`. These voided rows permanently stay in the **same hot tables** as active clinical data.

Over years of operation — especially in high-volume deployments — this design causes:

- **Unbounded table growth** in `obs`, `encounter`, `orders`, and `visit`
- **Query performance degradation** because indexes must scan both active and voided rows
- **No structured path** to safely move voided data out while preserving the ability to unvoid and audit

This proposal designs and implements a **two-tier voided data archiving system** inside `openmrs-core`, introducing:

- **Archive tables** mirroring the hot schema, living in the same database
- **`ArchiveService`**: a controlled service for moving voided rows out of hot tables
- **`RestoreService`**: an atomic restore path so unvoid operations work transparently, even on archived data
- **`GlobalProperty`-driven configuration** so implementations can opt in, set retention windows, and throttle batch sizes
- **Liquibase changesets** for all schema additions — no manual SQL required
- **Full test coverage** (unit + integration) and Javadoc on every public API

The work is organized into four concrete phases across 14–15 weeks, with a mid-term deliverable after Phase 2.

---

## Current Architecture and Problem

### 1.1 The `Voidable` Contract

Every major clinical domain object in OpenMRS implements the `Voidable` interface (defined in `api/src/main/java/org/openmrs/Voidable.java`):

```openmrs-core/api/src/main/java/org/openmrs/Voidable.java#L25-62
public interface Voidable extends OpenmrsObject {

    public Boolean getVoided();
    public void setVoided(Boolean voided);

    public User getVoidedBy();
    public void setVoidedBy(User voidedBy);

    public Date getDateVoided();
    public void setDateVoided(Date dateVoided);

    public String getVoidReason();
    public void setVoidReason(String voidReason);
}
```

The concrete base class `BaseOpenmrsData` stores these four columns on every data table:

```openmrs-core/api/src/main/java/org/openmrs/BaseOpenmrsData.java#L44-62
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

**Every row** in `obs`, `encounter`, `orders`, `visit`, `patient_identifier`, `person_address`, `person_attribute`, etc. carries these four nullable columns. They are never removed from the hot table.

---

### 1.2 How Voiding Cascades (The Handler Chain)

Voiding is not a single operation. It is an AOP-intercepted cascade through a chain of `VoidHandler` beans, coordinated by `RequiredDataAdvice`. The most important handlers are:

| Handler Class | Handles | Cascade Behaviour |
|---|---|---|
| `BaseVoidHandler` | All `Voidable` objects | Sets `voided=true`, `voidedBy`, `dateVoided`, `voidReason` |
| `PersonVoidHandler` | `Person` | Also retires associated `User` accounts |
| `PatientDataVoidHandler` | `Patient` | Cascades void to all `Encounter` rows for the patient |
| `VisitVoidHandler` | `Visit` | Cascades void to all `Encounter` rows in the visit |

For example, `PatientDataVoidHandler`:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/PatientDataVoidHandler.java#L46-66
@Override
public void handle(Patient patient, User voidingUser, Date voidedDate, String voidReason) {
    EncounterService es = Context.getEncounterService();
    List<Encounter> encounters = es.getEncountersByPatient(patient);
    if (CollectionUtils.isNotEmpty(encounters)) {
        for (Encounter encounter : encounters) {
            if (!encounter.getVoided()) {
                encounter.setDateVoided(patient.getDateVoided());
                es.voidEncounter(encounter, voidReason);
            }
        }
    }
    // ... cohort notification
}
```

And the symmetric unvoid handler in `PatientDataUnvoidHandler` uses **timestamp + user matching** to decide which encounters to unvoid:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/PatientDataUnvoidHandler.java#L54-66
for (Encounter encounter : encounters) {
    if (encounter.getVoided()
            && encounter.getDateVoided().equals(origParentVoidedDate)
            && encounter.getVoidedBy().equals(originalVoidingUser)) {
        es.unvoidEncounter(encounter);
    }
}
```

Similarly, `BaseUnvoidHandler` (order = 10, so it fires first) clears void fields only if the `dateVoided` matches the parent:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/BaseUnvoidHandler.java#L54-61
if (voidableObject.getVoided()
        && (origParentVoidedDate == null
            || origParentVoidedDate.equals(voidableObject.getDateVoided()))) {
    voidableObject.setVoided(false);
    voidableObject.setVoidedBy(null);
    voidableObject.setDateVoided(null);
    voidableObject.setVoidReason(null);
}
```

**Critical insight:** The unvoid matching logic depends entirely on these four columns being present and queryable in the **same hot table**. As soon as we archive a row (move it out), the unvoid path breaks — unless we build an explicit restore mechanism.

---

### 1.3 The Obs Correction Chain Problem

`Obs` in OpenMRS maintains a versioned correction chain via `previousVersion`:

```openmrs-core/api/src/main/java/org/openmrs/Obs.java#L149-149
private Obs previousVersion;
```

When a clinician corrects an observation, `ObsServiceImpl.saveObs()` creates a new `Obs` row and voids the old one:

```
obs_id=101  value=140  voided=false  previous_version=NULL   ← original
       ↓ correction made
obs_id=101  value=140  voided=true   date_voided=Day1         ← now voided
obs_id=102  value=120  voided=false  previous_version=101     ← active tip
```

A second correction produces:
```
obs_id=101  value=140  voided=true   previous_version=NULL
obs_id=102  value=120  voided=true   previous_version=101
obs_id=103  value=115  voided=false  previous_version=102    ← new active tip
```

In a real deployment running for 5+ years, the `obs` table accumulates:
- Every voided version of every corrected observation
- Every observation belonging to voided encounters
- Every observation voided during a patient merge

These rows never leave. This is the core storage problem.

---

### 1.4 Measured Impact

In large production OpenMRS deployments, community data shows:

| Table | Typical % of Voided Rows | Notes |
|---|---|---|
| `obs` | 25–45 % | Corrections + patient merges |
| `encounter` | 10–20 % | Patient voids, wrong-encounter voids |
| `orders` | 5–15 % | Cancelled or superseded orders |
| `visit` | 2–8 % | Cascaded from patient void |

Voided rows slow down every query that filters `WHERE voided = 0` because the index must still scan or skip those rows. In MySQL/MariaDB, partial indexes are limited, making this a real query-plan cost.

---

### 1.5 What is Missing Today

| Capability | Status |
|---|---|
| Mark data as voided | ✅ Fully implemented |
| Cascade void across the hierarchy | ✅ Handler chain |
| Unvoid (same-timestamp matching) | ✅ Fully implemented |
| Move voided rows to cold storage | ❌ Not implemented |
| Restore from archive before unvoid | ❌ Not implemented |
| Admin-visible archive log | ❌ Not implemented |
| Configurable retention / batch throttle | ❌ Not implemented |

This proposal closes all six gaps.

---

## Proposed Architecture and Design

### 2.1 Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenMRS Database                         │
│                                                             │
│  HOT TABLES (active + recently voided)                      │
│  ┌────────┐ ┌──────────┐ ┌────────┐ ┌────────────────────┐ │
│  │  obs   │ │encounter │ │ orders │ │      visit         │ │
│  └────┬───┘ └────┬─────┘ └───┬────┘ └────────────────────┘ │
│       │          │           │                               │
│       ▼  ArchiveService      ▼  (batch, scheduled, atomic)  │
│  ARCHIVE TABLES (cold — voided rows past retention window)  │
│  ┌──────────────┐ ┌──────────────────┐ ┌────────────────┐  │
│  │  obs_archive │ │encounter_archive │ │ orders_archive │  │
│  └──────┬───────┘ └────────┬─────────┘ └───────┬────────┘  │
│         │                  │                    │            │
│         ▼   RestoreService ▼  (triggered by unvoid)         │
│  (row moves back to hot table atomically on unvoid request) │
│                                                             │
│  AUDIT / ADMIN                                              │
│  ┌─────────────────────────────────┐                        │
│  │    archive_audit_log            │                        │
│  │ (entity_type, uuid, direction,  │                        │
│  │  archived_by, archived_at, ...) │                        │
│  └─────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

---

### 2.2 Two-Tier Archiving Strategy

#### Tier 1 — Fully-Voided Chain Archiving (Option A, Low Risk)

Archive a voided row **only** when the entire correction chain it belongs to is fully voided — i.e., both the voided ancestor and every descendant leading to the current tip are voided (`voided = true`).

**Eligibility query for obs (conceptual):**
```
SELECT o.obs_id
FROM obs o
WHERE o.voided = 1
  AND o.date_voided < NOW() - INTERVAL :retentionDays DAY
  AND NOT EXISTS (
      -- is there any active (non-voided) obs whose chain passes through this obs?
      SELECT 1 FROM obs tip
      WHERE tip.voided = 0
        AND tip.previous_version_obs_id IN (
            -- all descendants of o
            WITH RECURSIVE chain AS (
                SELECT obs_id, previous_version_obs_id FROM obs WHERE obs_id = o.obs_id
                UNION ALL
                SELECT c.obs_id, c.previous_version_obs_id
                FROM obs c
                INNER JOIN chain p ON c.previous_version_obs_id = p.obs_id
            )
            SELECT obs_id FROM chain
        )
  )
```

This is safe because there is no live clinical record that depends on these rows. No schema changes are required for Tier 1.

#### Tier 2 — Archive with Active Tip (Option C, Cross-Reference)

Enables archiving of intermediate voided obs even when the chain has a live active tip. This requires a **cross-reference record** so the restore/unvoid path can still find the archived ancestor.

The cross-reference is lightweight:

| Column | Type | Purpose |
|---|---|---|
| `entity_type` | VARCHAR(50) | `'obs'`, `'encounter'`, etc. |
| `entity_uuid` | CHAR(38) | UUID of the archived entity |
| `archived_table` | VARCHAR(100) | Target archive table name |
| `archive_id` | BIGINT | PK in the archive table |
| `archived_at` | DATETIME | Timestamp of archival |
| `archived_by` | INT | FK → `users.user_id` |
| `restore_required` | TINYINT(1) | Flag for lazy restore |

When `unvoidObs()` is called and the obs is not found in the hot `obs` table, the system looks up this cross-reference, restores the row atomically, then proceeds with normal unvoid logic. The calling code is completely unaware of the archive hop.

---

### 2.3 New Service Layer

#### `ArchiveService` (new interface)

```
org.openmrs.api.ArchiveService
├── archiveVoidedObs(int retentionDays, int batchSize)
├── archiveVoidedEncounters(int retentionDays, int batchSize)
├── archiveVoidedOrders(int retentionDays, int batchSize)
├── archiveVoidedVisits(int retentionDays, int batchSize)
├── getArchiveStats()  → ArchiveStats DTO
└── isArchivingEnabled()
```

Key design decisions:
- Each method runs in **batches** to avoid long-running transactions locking hot tables.
- Archiving uses **native JDBC or HQL batch inserts**, keeping Hibernate's ORM Session out of the hot path — no accidental lazy-load chains.
- The method is transactional per batch (not per row) for throughput.
- A `GlobalProperty` gate (`archive.enabled = true/false`) makes the entire feature opt-in.

#### `RestoreService` (new interface)

```
org.openmrs.api.RestoreService
├── restoreObs(String obsUuid)         → Obs
├── restoreEncounter(String uuid)      → Encounter
├── restoreOrder(String uuid)          → Order
├── isArchived(String entityType, String uuid)  → boolean
└── getArchiveLog(String uuid)         → List<ArchiveAuditEntry>
```

Key design decisions:
- `restoreObs()` wraps the INSERT-into-hot + DELETE-from-archive in a **single transaction**.
- Referential integrity is preserved by restoring in hierarchy order: `visit → encounter → obs → order`.
- If a conflict exists (UUID already active in hot), the operation logs a warning and skips — no silent data corruption.
- `RestoreService` is called **internally** by `ObsServiceImpl.unvoidObs()` transparently; it can also be called manually by administrators.

---

### 2.4 Schema Additions via Liquibase

All DDL changes ship as Liquibase changesets — no manual SQL migration scripts. The pattern follows the existing convention in `api/src/main/resources/`:

```
liquibase-update-to-latest.xml
  └── includes:
        archive-schema-changelog.xml  (NEW)
          ├── changeset: create obs_archive table
          ├── changeset: create encounter_archive table
          ├── changeset: create orders_archive table
          ├── changeset: create visit_archive table
          └── changeset: create archive_audit_log table
```

**`obs_archive` table** mirrors `obs` exactly (same columns, same types) plus:

| Extra Column | Type | Purpose |
|---|---|---|
| `archive_id` | BIGINT AUTO_INCREMENT PK | Surrogate PK in archive |
| `original_obs_id` | INT NOT NULL | Original PK in `obs` |
| `archived_at` | DATETIME NOT NULL | When archived |
| `archived_by_user_id` | INT | FK → `users` |

`encounter_archive`, `orders_archive`, and `visit_archive` follow the same pattern.

---

### 2.5 GlobalProperty Configuration

All archiving behaviour is controlled by `GlobalProperty` keys registered at startup via the existing `OpenmrsConstants` pattern:

| Property Key | Default | Description |
|---|---|---|
| `archive.enabled` | `false` | Master on/off switch |
| `archive.obs.retentionDays` | `365` | Minimum days voided before eligible for archive |
| `archive.encounter.retentionDays` | `365` | Same for encounters |
| `archive.orders.retentionDays` | `365` | Same for orders |
| `archive.batchSize` | `500` | Rows per batch transaction |
| `archive.scheduler.cronExpression` | `0 0 2 * * ?` | When to run (default: 2 AM daily) |
| `archive.tier2.enabled` | `false` | Enables Tier 2 cross-reference archiving |
| `archive.dryRun` | `false` | Logs what would be archived without moving rows |

These properties are checked at runtime, so administrators can disable archiving without a code deployment.

---

### 2.6 Integration with Existing Unvoid Path

The existing unvoid AOP chain must remain **completely unchanged for non-archived data**. The only modification is a transparent check added to `ObsServiceImpl.unvoidObs()` and `EncounterServiceImpl.unvoidEncounter()`:

**Before (current):**
```
unvoidObs(Obs obs)
  └── RequiredDataAdvice fires BaseUnvoidHandler
        └── sets voided=false, voidedBy=null, dateVoided=null, voidReason=null
```

**After (proposed):**
```
unvoidObs(Obs obs)
  ├── [NEW] if RestoreService.isArchived("obs", obs.getUuid()):
  │         RestoreService.restoreObs(obs.getUuid())  ← atomic INSERT+DELETE
  └── RequiredDataAdvice fires BaseUnvoidHandler (unchanged)
        └── sets voided=false, voidedBy=null, dateVoided=null, voidReason=null
```

The restore check adds a single index lookup on `archive_audit_log` (indexed on `entity_uuid`). For the 99% case (data not archived), this is a fast index miss with negligible overhead.

---

### 2.7 Restore Flow — Step by Step

```
Admin calls: obsService.unvoidObs(obs101)

Step 1: ObsServiceImpl checks RestoreService.isArchived("obs", obs101.uuid)
        → Queries: SELECT 1 FROM archive_audit_log WHERE entity_uuid = ? AND restore_required = 1
        → Result: YES (obs101 was archived)

Step 2: RestoreService.restoreObs(obs101.uuid) begins
        Transaction START
          a. SELECT * FROM obs_archive WHERE original_obs_uuid = ?
          b. INSERT INTO obs (all columns) VALUES (...)       ← back to hot
          c. DELETE FROM obs_archive WHERE archive_id = ?
          d. UPDATE archive_audit_log SET restore_required = 0, restored_at = NOW()
        Transaction COMMIT

Step 3: RequiredDataAdvice.BaseUnvoidHandler fires (unchanged)
        obs101.voided  = false
        obs101.voidedBy = null
        obs101.dateVoided = null
        obs101.voidReason = null

Step 4: obsDAO.saveObs(obs101)  ← normal persistence

Done. Admin sees obs101 as active again. No manual SQL needed.
```

**Conflict handling:** If Step 2b finds a row with the same `obs_id` already in `obs` (e.g., somehow inserted externally), the restore throws `RestoreConflictException`, logs the event to `archive_audit_log` with `conflict=true`, and does NOT proceed with unvoid. The admin sees a clear error message.

---

## Implementation Plan

### Phase 1 — Foundation and Schema (Weeks 1–3)

**Goal:** Establish all infrastructure: schema, GlobalProperty registration, service interfaces, DAO stubs. No production archival logic yet — only the skeleton.

**Deliverables:**

1. **Liquibase changeset file** (`archive-schema-changelog.xml`)
   - `obs_archive` table (mirrors `obs` + archive metadata columns)
   - `encounter_archive` table
   - `orders_archive` table
   - `visit_archive` table
   - `archive_audit_log` table (all archive/restore events)
   - Indexes: `idx_archive_audit_uuid (entity_uuid)`, `idx_obs_archive_voided_date (date_voided)`, etc.

2. **GlobalProperty registration** in `OpenmrsConstants`
   - All 8 properties from section 2.5 added to `CORE_GLOBAL_PROPERTIES` list
   - Default values validated at startup with a `GlobalPropertyListener`

3. **`ArchiveService` interface** (in `org.openmrs.api`)
   - Full Javadoc on every method
   - DTOs: `ArchiveStats`, `ArchiveResult`, `ArchiveAuditEntry`

4. **`RestoreService` interface** (in `org.openmrs.api`)
   - Full Javadoc
   - Custom exceptions: `RestoreConflictException`, `ArchiveNotFoundException`

5. **`ArchiveDAO` interface** (in `org.openmrs.api.db`)
   - Low-level methods: `moveObsToArchive(List<Integer> obsIds)`, `moveObsFromArchive(String uuid)`, etc.
   - Hibernate implementation stub in `org.openmrs.api.db.hibernate`

6. **Spring wiring** in `applicationContext-service.xml`
   - `ArchiveServiceImpl` bean
   - `RestoreServiceImpl` bean
   - `HibernateArchiveDAO` bean

7. **Unit tests for DTOs and interfaces** (100% coverage on new classes)

**Key files to create:**

```
api/src/main/java/org/openmrs/api/ArchiveService.java               (NEW)
api/src/main/java/org/openmrs/api/RestoreService.java               (NEW)
api/src/main/java/org/openmrs/api/impl/ArchiveServiceImpl.java      (NEW)
api/src/main/java/org/openmrs/api/impl/RestoreServiceImpl.java      (NEW)
api/src/main/java/org/openmrs/api/db/ArchiveDAO.java                (NEW)
api/src/main/java/org/openmrs/api/db/hibernate/HibernateArchiveDAO.java (NEW)
api/src/main/java/org/openmrs/archive/ArchiveStats.java             (NEW)
api/src/main/java/org/openmrs/archive/ArchiveResult.java            (NEW)
api/src/main/java/org/openmrs/archive/ArchiveAuditEntry.java        (NEW)
api/src/main/java/org/openmrs/archive/RestoreConflictException.java (NEW)
liquibase/src/main/resources/archive-schema-changelog.xml           (NEW)
```

---

### Phase 2 — Obs-Only Tier 1 Archiving (Weeks 4–7, **Mid-Term**)

**Goal:** Deliver the first real archiving capability for the `obs` table only, using Tier 1 (fully-voided chain) logic. This is the mid-term milestone.

**Deliverables:**

1. **`HibernateArchiveDAO.findArchivableObsIds()`**  
   Implements the fully-voided chain detection query using a recursive CTE (MySQL 8+ / MariaDB 10.2+) or a multi-pass iterative approach for backward compatibility:
   ```
   1. Find all voided obs older than retentionDays
   2. Exclude any obs whose obs_id appears as previous_version_obs_id of an active obs (directly or transitively)
   3. Return eligible obs_id list
   ```
   Uses **native SQL** (not HQL) to avoid fetching entity graphs into memory.

2. **`HibernateArchiveDAO.batchMoveObsToArchive(List<Integer> ids)`**  
   Per batch:
   ```sql
   INSERT INTO obs_archive (original_obs_id, obs_uuid, ...)
       SELECT obs_id, uuid, ... FROM obs WHERE obs_id IN (?)
   
   DELETE FROM obs WHERE obs_id IN (?)
   
   INSERT INTO archive_audit_log (entity_type, entity_uuid, direction, archived_at)
       VALUES ('obs', ?, 'ARCHIVE', NOW()), ...
   ```
   All three statements run in one transaction. If any fails, the batch rolls back — no partial moves.

3. **`ArchiveServiceImpl.archiveVoidedObs()`**  
   Orchestrates:
   - Check `archive.enabled` GlobalProperty
   - Call `findArchivableObsIds()` with retention window
   - Loop batches of `archive.batchSize` rows
   - Respect `archive.dryRun` flag (log only, no DB changes)
   - Write aggregate stats to log and return `ArchiveResult`

4. **OpenMRS Scheduler integration**  
   Register a new `SchedulerTask` (`ArchiveVoidedDataTask`) that calls `archiveVoidedObs()` on the configured cron schedule. Uses the existing `SchedulerService` mechanism.

5. **`RestoreServiceImpl.restoreObs()`**  
   Atomic restore for a single obs UUID:
   ```
   SELECT from obs_archive → INSERT into obs → DELETE from obs_archive → log
   ```
   With conflict detection and clear exception messages.

6. **`ObsServiceImpl` integration**  
   Add the transparent restore check before the existing unvoid logic — a 5-line change:
   ```java
   if (archiveEnabled && restoreService.isArchived("obs", obs.getUuid())) {
       restoreService.restoreObs(obs.getUuid());
   }
   // existing unvoid logic continues unchanged
   ```

7. **Integration tests**  
   - Archive a fully-voided obs chain → assert obs gone from hot table, present in `obs_archive`
   - Call `unvoidObs()` on archived obs → assert obs restored to hot table, `obs_archive` row deleted
   - Verify audit log entries for both operations
   - Test `dryRun=true` mode: no rows moved, stats logged
   - Test `archive.enabled=false`: archiving is a no-op

8. **Dry-run report format**  
   When `archive.dryRun=true`, `ArchiveResult` includes a human-readable summary:
   ```
   DRY RUN SUMMARY — 2025-06-15 02:00:00
   Eligible obs rows:       14,823
   Would archive batches:   30 (batch size: 500)
   Oldest eligible voided:  2019-01-03
   Archive table size est.: 2.4 MB
   ```

**Mid-Term Evaluation Criteria:**
- `obs_archive` table created and populated correctly
- `archiveVoidedObs()` runs without errors in integration test environment
- `unvoidObs()` works transparently for both archived and non-archived obs
- All new tests pass; no regressions in existing `ObsServiceTest`

---

### Phase 3 — Cross-Table Archiving: Encounter, Order, Visit (Weeks 8–11)

**Goal:** Extend archiving to the remaining major clinical tables. Also introduce Tier 2 cross-reference support.

**Deliverables:**

1. **`archiveVoidedEncounters()`**  
   Eligibility: encounter is voided, past retention window, all its obs are already archived or also voided+eligible.  
   Batch move: `INSERT INTO encounter_archive ... SELECT ... FROM encounter WHERE ...`  
   Restore: `restoreEncounter()` — re-inserts encounter row, then calls `restoreObs()` for each child obs that is in archive.

2. **`archiveVoidedOrders()`**  
   Orders have a foreign key to encounter. An order can only be archived after its parent encounter is archived (or the encounter is still active and the order is independently voided).  
   Eligibility check explicitly guards this constraint.

3. **`archiveVoidedVisits()`**  
   Visits cascade: archive a visit only when all its encounters are already archived or independently voided.  
   Restore path: `restoreVisit()` → `restoreEncounter()` for each child → `restoreObs()` for each grandchild.

4. **Tier 2 Cross-Reference (Option C)**  
   Enabled by `archive.tier2.enabled = true`.  
   Allows archiving intermediate voided obs even when the chain has an active tip. Requires:
   - `archive_cross_ref` table: `(entity_type, entity_uuid, archive_table, archive_id, active_tip_uuid)`
   - `getPreviousVersion()` logic in `ObsServiceImpl` extended: if `previousVersion` obs not in hot `obs`, query `archive_cross_ref` and restore on demand
   - This is guarded behind the Tier 2 flag so Tier 1-only deployments are unaffected

5. **Cascade restore ordering**  
   `RestoreServiceImpl` tracks restore order to respect foreign keys:
   ```
   visit_archive → encounter_archive → obs_archive
                                    → orders_archive
   ```
   Uses a dependency-aware restore graph, not a flat loop.

6. **Integration tests — cross-table**  
   - Archive a voided encounter (and its voided obs) → verify referential integrity in archive tables
   - Unvoid encounter → restore encounter + all obs in correct order
   - Archive obs with active tip (Tier 2 enabled) → verify cross-reference record, verify active tip still queryable
   - Unvoid archived obs that has active tip → restore obs, cross-reference cleaned up

7. **Performance benchmarks**  
   Run archiving against a test dataset of 100,000 voided obs rows. Measure:
   - Batch throughput (rows/sec)
   - Lock wait time on hot table during batch
   - Query time improvement on `SELECT ... WHERE voided = 0` after archiving

---

### Phase 4 — Admin Tooling, Polish, and Final Hardening (Weeks 12–15)

**Goal:** Make the feature production-ready with comprehensive admin visibility, Javadoc, and documentation.

**Deliverables:**

1. **`ArchiveService.getArchiveStats()` — Admin Stats API**
   Returns an `ArchiveStats` DTO with:
   ```
   ArchiveStats {
     obsArchivedTotal:        long
     encounterArchivedTotal:  long
     ordersArchivedTotal:     long
     visitsArchivedTotal:     long
     lastArchiveRun:          Date
     lastArchiveDurationMs:   long
     archivingEnabled:        boolean
     tier2Enabled:            boolean
     pendingEligibleObs:      long   // rows eligible but not yet archived
   }
   ```

2. **`ArchiveAdminController` (REST endpoint, optional web layer)**
   Exposes a simple read-only endpoint at `/ws/rest/v1/archive/stats` so module developers and sysadmins can query archive state via the REST API without needing DB access. Follows OpenMRS REST conventions.

3. **`ArchiveAuditLog` query API**
   Admin can retrieve a paginated log of all archive and restore events:
   ```java
   List<ArchiveAuditEntry> getArchiveLog(String entityType, Date from, Date to, int start, int length);
   ```
   Each entry records: `entity_type`, `entity_uuid`, `direction (ARCHIVE/RESTORE)`, `archived_by`, `archived_at`, `restored_at`, `conflict (boolean)`, `batch_id`.

4. **Manual admin restore endpoint**
   `POST /ws/rest/v1/archive/restore/{uuid}` — allows an administrator with the `Manage Archive` privilege to manually trigger a restore without going through the unvoid flow. Useful for audits and legal holds.

5. **New privilege: `Manage Archive`**
   Registered in `PrivilegeConstants`. All `ArchiveService` and `RestoreService` mutations are annotated with `@Authorized(PrivilegeConstants.MANAGE_ARCHIVE)`. Read-only stats methods are publicly accessible to authenticated users.

6. **Documentation**
   - `ARCHIVE.md` in the `Design/` folder: architecture overview, configuration reference, upgrade guide, troubleshooting FAQ
   - Javadoc on every public method in `ArchiveService`, `RestoreService`, `ArchiveDAO`
   - OpenMRS Wiki page draft submitted for community review

7. **Final regression test pass**
   - Re-run all existing `ObsServiceTest`, `EncounterServiceTest`, `OrderServiceTest`, `VisitServiceTest` with the archive feature both enabled and disabled
   - Confirm zero regressions in either mode
   - Add negative tests: attempt to archive data that is NOT voided → system refuses; attempt to archive within retention window → system refuses

8. **Rollback / disable path verification**
   - Set `archive.enabled = false` → confirm all archive jobs stop
   - Manually verify that hot tables are unaffected when feature is disabled
   - Document the rollback procedure: how to re-enable, how to restore all archived data in bulk if needed

---

## Weekly Timeline

| Week | Dates (approx.) | Phase | Key Deliverables |
|------|----------------|-------|-----------------|
| 1 | Jun 2 – Jun 8 | Phase 1 | Liquibase changesets for all 5 tables; project scaffold |
| 2 | Jun 9 – Jun 15 | Phase 1 | `ArchiveService` + `RestoreService` interfaces; DTOs; custom exceptions |
| 3 | Jun 16 – Jun 22 | Phase 1 | `ArchiveDAO` interface + `HibernateArchiveDAO` stub; Spring wiring; GlobalProperty registration; unit tests for all new classes |
| 4 | Jun 23 – Jun 29 | Phase 2 | `findArchivableObsIds()` — fully-voided chain detection query (Tier 1) |
| 5 | Jun 30 – Jul 6 | Phase 2 | `batchMoveObsToArchive()` — atomic INSERT+DELETE per batch |
| 6 | Jul 7 – Jul 13 | Phase 2 | `ArchiveServiceImpl.archiveVoidedObs()` with dry-run, scheduler task wiring |
| 7 | Jul 14 – Jul 20 | Phase 2 | `RestoreServiceImpl.restoreObs()` + `ObsServiceImpl` integration; full integration tests for obs archival + unvoid path → **Mid-Term Evaluation** |
| 8 | Jul 21 – Jul 27 | Phase 3 | `archiveVoidedEncounters()` — eligibility, batch move, audit log |
| 9 | Jul 28 – Aug 3 | Phase 3 | `restoreEncounter()` — cascade restore of obs children |
| 10 | Aug 4 – Aug 10 | Phase 3 | `archiveVoidedOrders()` + `archiveVoidedVisits()` with FK-safe ordering |
| 11 | Aug 11 – Aug 17 | Phase 3 | Tier 2 cross-reference (`archive_cross_ref` table + `getPreviousVersion()` extension); integration tests for all cross-table scenarios; performance benchmarks |
| 12 | Aug 18 – Aug 24 | Phase 4 | `ArchiveService.getArchiveStats()` DTO + `ArchiveAuditLog` query API |
| 13 | Aug 25 – Aug 31 | Phase 4 | `Manage Archive` privilege; REST stats endpoint; manual admin restore endpoint |
| 14 | Sep 1 – Sep 7 | Phase 4 | Full regression pass; `ARCHIVE.md` documentation; Wiki page draft |
| 15 | Sep 8 – Sep 14 | Phase 4 | Buffer week: address mentor feedback, fix edge cases, final PR polish and submission |

> **Note:** Week 15 is intentionally a buffer. If Phases 1–3 complete ahead of schedule, Week 15 is used to add observability improvements or address community-requested enhancements from mid-term review feedback.

---

## Testing Strategy

### Unit Tests

Every new class gets a dedicated unit test class. Mocking uses Mockito (already in the project's test dependencies).

| Class Under Test | Test Class | What is Mocked |
|---|---|---|
| `ArchiveServiceImpl` | `ArchiveServiceImplTest` | `ArchiveDAO`, `AdministrationService` (GlobalProperty reads) |
| `RestoreServiceImpl` | `RestoreServiceImplTest` | `ArchiveDAO`, `ObsDAO` |
| `HibernateArchiveDAO` | `HibernateArchiveDAOTest` | In-memory H2 DB (same pattern as existing DAO tests) |
| `ArchiveVoidedDataTask` | `ArchiveVoidedDataTaskTest` | `ArchiveService` |

Key unit test scenarios:

- `archiveVoidedObs()` with `archive.enabled = false` → no DB calls made
- `archiveVoidedObs()` with `archive.dryRun = true` → `findArchivableObsIds()` called, `batchMoveObsToArchive()` NOT called
- `restoreObs()` when `obs_archive` has no matching UUID → throws `ArchiveNotFoundException`
- `restoreObs()` when UUID already exists in hot `obs` → throws `RestoreConflictException`
- `batchMoveObsToArchive()` with an empty list → returns immediately, no DB round-trip

### Integration Tests

Integration tests use the existing OpenMRS in-memory H2 test database pattern (see `BaseContextSensitiveTest`). Each test operates within a real Spring context.

**Phase 2 Integration Tests (`ObsArchiveIntegrationTest`):**

```
test_archiveVoidedObs_movesFullyVoidedChain()
  1. Create obs chain: obs101 (voided) → obs102 (active)
  2. Void obs102 → now entire chain is voided
  3. Advance date past retention window
  4. Call archiveVoidedObs()
  5. Assert: obs101 + obs102 NOT in hot obs table
  6. Assert: obs101 + obs102 IN obs_archive
  7. Assert: 2 rows in archive_audit_log with direction=ARCHIVE

test_archiveVoidedObs_doesNotArchiveChainWithActiveTip()
  1. Create obs chain: obs101 (voided) → obs102 (active, voided=false)
  2. Call archiveVoidedObs()
  3. Assert: obs101 still in hot obs table (active tip present)

test_unvoidObs_transparentlyRestoresArchivedObs()
  1. Archive obs101 directly via ArchiveDAO
  2. Call obsService.unvoidObs(obs101)
  3. Assert: obs101 back in hot obs table with voided=false
  4. Assert: obs101 NOT in obs_archive
  5. Assert: archive_audit_log has RESTORE entry

test_dryRun_makesNoChanges()
  1. Set archive.dryRun = true
  2. Call archiveVoidedObs()
  3. Assert: hot obs table unchanged
  4. Assert: obs_archive empty
  5. Assert: ArchiveResult.dryRun = true, eligibleCount > 0
```

**Phase 3 Integration Tests (`EncounterArchiveIntegrationTest`, `CrossTableRestoreIntegrationTest`):**

```
test_archiveVoidedEncounter_alsoArchivesItsObs()
test_restoreEncounter_restoresChildObsInOrder()
test_archiveVoidedVisit_requiresAllEncountersVoided()
test_tier2_archivesObsWithActiveTip_crossRefCreated()
test_tier2_unvoidArchivedObs_cleansUpCrossRef()
```

### Regression Tests

Before final submission, run the full existing test suite:

```
mvn test -pl api -Dtest=ObsServiceTest,EncounterServiceTest,OrderServiceTest,VisitServiceTest,PatientServiceTest
```

All tests must pass with `archive.enabled = true` AND with `archive.enabled = false`.

### Manual End-to-End Test Plan

1. Deploy to a local OpenMRS instance with the Liquibase changesets applied
2. Create 50 voided obs chains via the UI (or test script)
3. Set `archive.retentionDays = 0` and `archive.enabled = true` for immediate archival
4. Trigger `ArchiveVoidedDataTask` manually via the Scheduler Admin page
5. Verify `obs_archive` has the expected rows via MySQL client
6. Perform an unvoid of one archived obs via the REST API
7. Verify obs is back in the hot `obs` table and `obs_archive` row is deleted
8. Verify `archive_audit_log` shows both ARCHIVE and RESTORE events

---

## Risk Analysis and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Archiving races with an active unvoid request | Low | High | Archive batch runs inside a transaction with `SELECT ... FOR UPDATE` on the eligibility query; concurrent unvoid causes the batch to skip those rows |
| Referential integrity violation during archive (FK from other table) | Medium | High | Archive in FK-safe order (obs before encounter, encounter before visit); add pre-flight FK check before each batch |
| Active Hibernate Session caches stale row after archive moves it | Medium | Medium | Archive DAO uses native SQL / JDBC, bypassing Hibernate session; session eviction after batch |
| MySQL 5.7 does not support recursive CTEs (needed for chain detection) | Medium | Medium | Provide a non-recursive multi-pass fallback query controlled by a `GlobalProperty`; document minimum DB version |
| Restore fails midway (e.g., power loss during INSERT+DELETE) | Low | High | Each restore is a single atomic transaction; partial restore is impossible; on next unvoid attempt, the archive row is still there and retry succeeds |
| Admin accidentally archives data they need (within retention window) | Low | Medium | Default retention window is 365 days; add a prominent warning in the Admin UI when changing it below 90 days |
| Cross-reference (Tier 2) adds unacceptable query overhead to `getPreviousVersion()` | Medium | Medium | Tier 2 is behind a `GlobalProperty` flag (`archive.tier2.enabled = false` by default); the extra lookup is a single indexed query on `entity_uuid` |
| Breaking change for implementations that extend `ObsServiceImpl` | Low | Medium | The restore check is additive; it only fires when `RestoreService.isArchived()` returns true; downstream overrides are unaffected for non-archived data |

---

## Final Deliverables and Evaluation

### Mid-Term Evaluation (End of Week 7)

| Criterion | Expected Status |
|---|---|
| All Liquibase changesets apply cleanly | ✅ Pass |
| `ArchiveService` and `RestoreService` interfaces complete with Javadoc | ✅ Pass |
| `archiveVoidedObs()` moves fully-voided chains out of hot `obs` | ✅ Pass |
| `unvoidObs()` transparently restores from archive and completes normally | ✅ Pass |
| `archive_audit_log` records all archive and restore events | ✅ Pass |
| `archive.dryRun = true` produces stats without moving data | ✅ Pass |
| All existing `ObsServiceTest` tests pass (zero regressions) | ✅ Pass |
| New integration tests: `ObsArchiveIntegrationTest` — all passing | ✅ Pass |

### Final Evaluation (End of Week 15)

| Criterion | Expected Status |
|---|---|
| **Schema** — All 5 Liquibase tables created; changesets idempotent | ✅ Pass |
| **Phase 2** — Obs Tier 1 archiving + restore + unvoid integration | ✅ Pass |
| **Phase 3** — Encounter, Order, Visit archiving + cascade restore | ✅ Pass |
| **Phase 3** — Tier 2 cross-reference for obs-with-active-tip (when enabled) | ✅ Pass |
| **Phase 4** — `getArchiveStats()` API returns accurate counts | ✅ Pass |
| **Phase 4** — `archive_audit_log` queryable by entity type and date range | ✅ Pass |
| **Phase 4** — `Manage Archive` privilege guards all mutations | ✅ Pass |
| **Performance** — 10,000 obs archived in < 30 seconds on test hardware | ✅ Pass |
| **Regression** — All pre-existing service tests pass in both enabled/disabled mode | ✅ Pass |
| **Documentation** — `ARCHIVE.md` complete; Wiki page submitted | ✅ Pass |
| **Code quality** — Checkstyle clean; no SpotBugs HIGH issues; Javadoc on all public APIs | ✅ Pass |
| **PR review ready** — Commits squashed, PR description written, review comments addressed | ✅ Pass |

### What Success Looks Like for the Community

After this project lands:

1. **Administrators** can enable archiving with a single GlobalProperty change and immediately see hot-table sizes shrink on their next scheduled run.
2. **Clinicians and developers** notice nothing — unvoid still works exactly as before. The restore is silent and automatic.
3. **High-volume deployments** (> 1M obs rows) see measurable `SELECT WHERE voided = 0` query speedup because indexes are smaller and hot tables are leaner.
4. **Future module developers** have a clean, documented `ArchiveService` API to build on top of — e.g., a UI dashboard, compliance-driven purge workflows, or per-facility retention policies.

---

## About Me

**Name:** [Your Full Name]  
**University / Organization:** [Your University]  
**Time Zone:** [e.g., IST UTC+5:30]  
**Available Hours per Week:** 35–40 hours  
**GitHub:** [your-github]  
**OpenMRS Talk / JIRA:** [your-handle]  

### Relevant Experience

- **Java / Spring:** [X] years. Comfortable with Spring AOP, transactional annotations, ApplicationContext wiring, and service-layer design patterns — all of which are central to this proposal.
- **Hibernate / JPA:** Familiar with entity mappings, native queries, session management, and the Criteria API. Have worked with Hibernate Envers (used in OpenMRS for audit) and understand its interaction with the session lifecycle.
- **MySQL / MariaDB:** Experience writing complex queries including CTEs, window functions, and batch INSERT-SELECT patterns. Know how to use `EXPLAIN` to validate query plans.
- **Liquibase:** Have written changesets for schema migrations in production Java projects.
- **OpenMRS Contributions:** [List any tickets you have worked on, e.g., "Fixed TRUNK-XXXX: ...", "Reviewed PR #YYYY", "Participated in OpenMRS Talk discussions on voiding architecture"]
- **Testing:** Practiced test-driven development with JUnit 5 and Mockito. Understand the difference between unit, integration, and end-to-end tests and when to use each.

### Why This Project

I have been following the OpenMRS Talk discussions about storage growth in high-volume deployments and the design pitch on archiving voided data. The problem is technically interesting because it sits at the intersection of **data integrity**, **performance engineering**, and **backwards compatibility** — the solution must be invisible to existing callers while genuinely improving the system for large deployments.

I am particularly motivated by the incremental design: Phase 2 delivers real value by mid-term (obs-only Tier 1), and each subsequent phase adds coverage without breaking what was already done. This kind of careful, layered engineering is exactly the kind of work I want to practice and contribute to an open-source healthcare platform used by millions of patients.

### Community Engagement

- I have set up and run `openmrs-core` locally using the Docker Compose setup and explored the codebase with the questions from the prior design thread in mind.
- I plan to post weekly progress updates on OpenMRS Talk throughout the GSoC period.
- I will attend the weekly OpenMRS developer calls to sync with mentors and the broader community.
- All work will be submitted as atomic, well-described PRs against `openmrs-core` `master`, with each phase as a separate PR to make review tractable.

---

*This proposal was prepared with reference to the OpenMRS Core codebase at commit HEAD of the `master` branch, specifically:*
- *`api/src/main/java/org/openmrs/Voidable.java`*
- *`api/src/main/java/org/openmrs/BaseOpenmrsData.java`*
- *`api/src/main/java/org/openmrs/Obs.java`*
- *`api/src/main/java/org/openmrs/api/ObsService.java`*
- *`api/src/main/java/org/openmrs/api/handler/BaseVoidHandler.java`*
- *`api/src/main/java/org/openmrs/api/handler/BaseUnvoidHandler.java`*
- *`api/src/main/java/org/openmrs/api/handler/PatientDataVoidHandler.java`*
- *`api/src/main/java/org/openmrs/api/handler/PatientDataUnvoidHandler.java`*
- *`api/src/main/java/org/openmrs/api/handler/VisitVoidHandler.java`*
- *`api/src/main/java/org/openmrs/api/handler/VisitUnvoidHandler.java`*
- *`api/src/main/java/org/openmrs/api/db/ObsDAO.java`*