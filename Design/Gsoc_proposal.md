GSoC 2025 Proposal
## Voided Data Archiving and Safe Restoration in OpenMRS Core

| Field | Details |
|---|---|
| **Applicant** | [Your Name] |
| **Email** | [your@email.com] |
| **GitHub** | [your-github-handle] |
| **OpenMRS Talk** | [your-talk-handle] |
| **Project Size** | Large (~350 hours) |
| **Duration** | 14–15 Weeks |
| **Tech Stack** | Java 17, Spring AOP, Hibernate/JPA, MySQL/MariaDB, Liquibase, JUnit 5 |

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Current Architecture and Problem](#2-current-architecture-and-problem)
3. [Proposed Architecture and Design](#3-proposed-architecture-and-design)
4. [Implementation Plan — Phases and Timeline](#4-implementation-plan)
5. [Testing Strategy](#5-testing-strategy)
6. [Risk Analysis and Mitigations](#6-risk-analysis-and-mitigations)
7. [Final Deliverables and Evaluation](#7-final-deliverables-and-evaluation)
8. [About Me](#8-about-me)

---

## 1. Abstract

OpenMRS data is almost never hard-deleted. When a clinical record becomes invalid — a wrong encounter, a duplicate patient, a correction to an observation — the system marks it `voided = true` and stores `voided_by`, `date_voided`, and `void_reason` alongside it. These rows stay permanently in the same hot tables as active clinical data.

Over years of operation, especially at high-volume sites, this causes unbounded growth in `obs`, `encounter`, `orders`, and `visit`. Queries that filter `WHERE voided = 0` must scan or skip all those voided rows, degrading performance. There is no structured path to move voided data out safely while preserving the ability to unvoid or audit it later.

This proposal designs and implements a two-tier voided data archiving system inside `openmrs-core`. It introduces dedicated archive tables, a new `ArchiveService` for controlled batch moves, a `RestoreService` for atomic restoration before unvoid, `GlobalProperty`-driven configuration so deployments can opt in at their own pace, and Liquibase changesets for all schema additions. The work is organized into four phases across 14–15 weeks, with a fully working obs-archiving mid-term milestone at the end of Week 7.

---

## 2. Current Architecture and Problem

### 2.1 The Voidable Contract

Every major domain object in OpenMRS implements the `Voidable` interface:

```openmrs-core/api/src/main/java/org/openmrs/Voidable.java#L28-62
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

The abstract base `BaseOpenmrsData` maps these to four physical columns present on every data table:

```openmrs-core/api/src/main/java/org/openmrs/BaseOpenmrsData.java#L44-62
@Column(name = "voided", nullable = false)
private Boolean voided = Boolean.FALSE;

@Column(name = "date_voided")
private Date dateVoided;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "voided_by")
private User voidedBy;

@Column(name = "void_reason", length = 255)
private String voidReason;
```

Every voided row in `obs`, `encounter`, `orders`, `visit`, `patient_identifier`, `person_address`, and `person_attribute` carries these four columns and never leaves the hot table.

### 2.2 How Voiding Cascades

Voiding is not a single atomic call. `RequiredDataAdvice` intercepts every `void*` method via AOP and fires a priority-ordered chain of `VoidHandler` beans. When `patientService.voidPatient()` is called, `PatientDataVoidHandler` fetches all encounters for that patient and calls `voidEncounter()` on each. `VisitVoidHandler` does the same for encounters inside visits. `BaseVoidHandler` then runs on every child `Obs` and `Order`:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/BaseVoidHandler.java#L55-72
public void handle(Voidable voidableObject, User voidingUser, Date voidedDate, String voidReason) {
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

The symmetric `BaseUnvoidHandler` matches children back to their parent strictly by `dateVoided` equality:

```openmrs-core/api/src/main/java/org/openmrs/api/handler/BaseUnvoidHandler.java#L54-62
if (voidableObject.getVoided()
        && (origParentVoidedDate == null
            || origParentVoidedDate.equals(voidableObject.getDateVoided()))) {
    voidableObject.setVoided(false);
    voidableObject.setVoidedBy(null);
    voidableObject.setDateVoided(null);
    voidableObject.setVoidReason(null);
}
```

This timestamp-matching dependency is the central constraint for the entire archiving design. If a voided row is moved out of the hot table, the unvoid lookup fails — unless we build an explicit restore step before unvoid runs.

### 2.3 The Obs Correction Chain Problem

When a clinician corrects an observation, `ObsServiceImpl.saveObs()` creates a new `Obs` and voids the old one via `previousVersion`:

```openmrs-core/api/src/main/java/org/openmrs/Obs.java#L149-149
private Obs previousVersion;
```

A single concept for a single patient builds a voided chain over time:

- Day 1: `obs_id=101, value=140, voided=false, previous_version=NULL`
- Day 3 correction: `obs_id=101` becomes `voided=true`; `obs_id=102, value=120, voided=false, previous_version=101` created
- Day 10 second correction: `obs_id=102` becomes `voided=true`; `obs_id=103, value=115, voided=false, previous_version=102` created

After five years of corrections across 50,000 patients, the `obs` table carries 25–45% voided rows that never leave. A patient merge makes it worse — every obs, encounter, and order for the merged patient is cascade-voided in one shot, and all of it stays.

### 2.4 What is Missing Today

| Capability | Status |
|---|---|
| Mark data as voided | ✅ Done |
| Cascade void through the hierarchy | ✅ Done |
| Unvoid with timestamp matching | ✅ Done |
| Move voided rows to cold storage | ❌ Missing |
| Restore archived rows before unvoid | ❌ Missing |
| Admin-visible archive audit log | ❌ Missing |
| Configurable retention window and batch throttle | ❌ Missing |
| Tier 2: archive obs when active chain tip still exists | ❌ Missing |

---

## 3. Proposed Architecture and Design

### 3.1 System Overview

The high-level idea is straightforward: introduce a set of archive tables that mirror the hot schema, a service to move eligible voided rows into them in controlled batches, and a restore service that transparently moves rows back whenever an unvoid is requested on archived data. All of this is opt-in via GlobalProperties and fully audited.

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │                         OpenMRS Database                                 │
 │                                                                          │
 │   HOT TABLES  (active clinical data + recently voided rows)              │
 │  ┌─────────┐   ┌─────────────┐   ┌─────────┐   ┌─────────┐             │
 │  │   obs   │   │  encounter  │   │ orders  │   │  visit  │             │
 │  │voided=0 │   │  voided=0   │   │voided=0 │   │voided=0 │             │
 │  │voided=1 │   │  voided=1   │   │voided=1 │   │voided=1 │             │
 │  └────┬────┘   └──────┬──────┘   └────┬────┘   └────┬────┘             │
 │       │               │               │              │                  │
 │       └───────────────┴───────────────┴──────────────┘                  │
 │                               │                                          │
 │                    ArchiveService (batch, scheduled)                     │
 │              checks retention window + chain eligibility                 │
 │                               │                                          │
 │                               ▼                                          │
 │   ARCHIVE TABLES  (cold storage — voided rows past retention window)     │
 │  ┌──────────────┐  ┌──────────────────┐  ┌───────────────┐              │
 │  │  obs_archive │  │encounter_archive │  │orders_archive │              │
 │  └──────┬───────┘  └────────┬─────────┘  └───────┬───────┘              │
 │         │                   │                     │                      │
 │         └───────────────────┴─────────────────────┘                      │
 │                               │                                          │
 │                   RestoreService (triggered by unvoid)                   │
 │           atomic INSERT into hot + DELETE from archive                   │
 │                               │                                          │
 │                               ▼                                          │
 │   AUDIT LAYER                                                            │
 │  ┌──────────────────────────────────────────────────────────────────┐   │
 │  │  archive_audit_log                                                │   │
 │  │  entity_type | entity_uuid | direction | archived_by | archived_at│   │
 │  │  restored_at | batch_id   | conflict  | restore_required          │   │
 │  └──────────────────────────────────────────────────────────────────┘   │
 └──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Two-Tier Archiving Strategy

Not all voided rows can be archived with the same rule. The proposal uses two tiers with different safety levels.

**Tier 1 — Fully-Voided Chain Archiving (Option A)**

A voided obs is eligible for archival only when every version in its correction chain is also voided — there is no live active tip downstream of it. This is the zero-risk default.

```
 Obs correction chain — Tier 1 eligibility check:

 obs_id=101 (voided) ──► obs_id=102 (voided) ──► obs_id=103 (voided)
                                                        │
                                              No active tip found
                                              date_voided < NOW() - 365d
                                                        │
                                                ✅ ELIGIBLE — archive all three

 ─────────────────────────────────────────────────────────────────────

 obs_id=101 (voided) ──► obs_id=102 (voided) ──► obs_id=103 (active ✓)
                                                        │
                                              Active tip still exists
                                                        │
                                                ❌ SKIP — covered by Tier 2
```

Tier 1 handles the high-impact cases: patient merges, wrong-encounter void cascades, and any chain where every version has been corrected away.

**Tier 2 — Archive with Active Tip, via Cross-Reference (Option C)**

Enabled by `archive.tier2.enabled = true`. This tier allows archiving intermediate voided obs even when the correction chain still has an active tip. Instead of a direct FK link, a cross-reference record is written so the restore path can always find the archived ancestor.

```
 Before archiving (Tier 2):

  HOT obs table
  ┌─────────────────────────────────────────┐
  │ obs_id=101  voided=true                 │  ← eligible under Tier 2
  │ obs_id=102  voided=false  prev_ver=101  │  ← active tip stays
  └─────────────────────────────────────────┘

 After archiving obs_id=101:

  HOT obs table                    obs_archive
  ┌────────────────────┐           ┌───────────────────────────────┐
  │ obs_id=102  active │           │ obs101 row  + archived_at     │
  └────────────────────┘           └───────────────────────────────┘

  archive_cross_ref
  ┌──────────────────────────────────────────────────────────┐
  │ entity_uuid=obs101.uuid  active_tip_uuid=obs102.uuid     │
  │ archive_table=obs_archive                                │
  └──────────────────────────────────────────────────────────┘

 obs102.getPreviousVersion() — if obs101 not found in hot obs,
 RestoreService checks cross_ref and lazy-restores obs101 on demand.
 The active tip obs102 remains queryable throughout.
```

### 3.3 New Service and DAO Layer

The entire archiving system is built on two new service interfaces and a new DAO, plugged into the existing Spring application context.

```
 org.openmrs.api
 ┌──────────────────────────────────────────────────────────────┐
 │  <<interface>>  ArchiveService                               │
 │                                                              │
 │  archiveVoidedObs(retentionDays, batchSize)                  │
 │  archiveVoidedEncounters(retentionDays, batchSize)           │
 │  archiveVoidedOrders(retentionDays, batchSize)               │
 │  archiveVoidedVisits(retentionDays, batchSize)               │
 │  getArchiveStats()  →  ArchiveStats                          │
 │  isArchivingEnabled()  →  boolean                            │
 └────────────────────────────┬─────────────────────────────────┘
                              │ implemented by
 ┌────────────────────────────▼─────────────────────────────────┐
 │  ArchiveServiceImpl                                          │
 │  - reads GlobalProperties (enabled, retentionDays, batchSize)│
 │  - orchestrates batches, respects dryRun flag                │
 │  - delegates all DB work to ArchiveDAO                       │
 └────────────────────────────┬─────────────────────────────────┘
                              │ uses
 ┌────────────────────────────▼─────────────────────────────────┐
 │  <<interface>>  ArchiveDAO                (org.openmrs.api.db)│
 │                                                              │
 │  findArchivableObsIds(retentionDays)  →  List<Integer>       │
 │  batchMoveObsToArchive(ids)                                  │
 │  batchMoveObsFromArchive(uuid)        →  void                │
 │  isObsArchived(uuid)                  →  boolean             │
 │  (same pattern for encounter / order / visit)                │
 └────────────────────────────┬─────────────────────────────────┘
                              │ implemented by
 ┌────────────────────────────▼─────────────────────────────────┐
 │  HibernateArchiveDAO                                         │
 │  Uses native SQL / JDBC — NOT HQL entity graphs.             │
 │  Bypasses Hibernate session to avoid lazy-load side effects. │
 └──────────────────────────────────────────────────────────────┘

 org.openmrs.api
 ┌──────────────────────────────────────────────────────────────┐
 │  <<interface>>  RestoreService                               │
 │                                                              │
 │  restoreObs(uuid)          →  Obs                            │
 │  restoreEncounter(uuid)    →  Encounter                      │
 │  restoreOrder(uuid)        →  Order                          │
 │  isArchived(type, uuid)    →  boolean                        │
 │  getArchiveLog(uuid)       →  List<ArchiveAuditEntry>        │
 └────────────────────────────┬─────────────────────────────────┘
                              │ implemented by
 ┌────────────────────────────▼─────────────────────────────────┐
 │  RestoreServiceImpl                                          │
 │  Called transparently from ObsServiceImpl.unvoidObs() and   │
 │  EncounterServiceImpl.unvoidEncounter() before existing AOP  │
 │  handler chain runs. Callers see no difference.             │
 └──────────────────────────────────────────────────────────────┘
```

### 3.4 Archive Schema — Five New Tables via Liquibase

All DDL ships as Liquibase changesets, included from `liquibase-update-to-latest.xml`. No manual SQL is ever required.

```
 obs_archive                        encounter_archive
 ┌───────────────────────────┐      ┌───────────────────────────────┐
 │ archive_id  BIGINT  PK    │      │ archive_id  BIGINT  PK        │
 │ original_obs_id  INT      │      │ original_encounter_id  INT    │
 │ uuid  CHAR(38)  NOT NULL  │      │ uuid  CHAR(38)  NOT NULL      │
 │ [all obs columns mirrored]│      │ [all encounter cols mirrored] │
 │ archived_at  DATETIME     │      │ archived_at  DATETIME         │
 │ archived_by  INT → users  │      │ archived_by  INT → users      │
 └───────────────────────────┘      └───────────────────────────────┘
   INDEX(uuid), INDEX(date_voided)    INDEX(uuid), INDEX(date_voided)

 orders_archive  and  visit_archive follow the same pattern.

 archive_audit_log                   archive_cross_ref  (Tier 2)
 ┌──────────────────────────────┐    ┌────────────────────────────────┐
 │ log_id   BIGINT  PK          │    │ ref_id   BIGINT  PK            │
 │ entity_type  VARCHAR(50)     │    │ entity_type  VARCHAR(50)       │
 │ entity_uuid  CHAR(38)        │    │ entity_uuid  CHAR(38)          │
 │ direction  ENUM(ARCHIVE,     │    │ archive_table  VARCHAR(100)    │
 │            RESTORE)          │    │ archive_id  BIGINT             │
 │ archived_by  INT             │    │ active_tip_uuid  CHAR(38)      │
 │ archived_at  DATETIME        │    └────────────────────────────────┘
 │ restored_at  DATETIME        │      INDEX(entity_uuid)
 │ batch_id  VARCHAR(36)        │
 │ conflict  TINYINT(1)         │
 │ restore_required  TINYINT(1) │
 └──────────────────────────────┘
   INDEX(entity_uuid)  ← used by every isArchived() call
```

### 3.5 How the Existing Unvoid Path Changes

The existing void/unvoid AOP handler chain is completely unchanged for data that has not been archived. The only modification is a transparent two-line guard injected at the top of `unvoidObs()` and `unvoidEncounter()`:

```
 Current unvoid flow:

   obsService.unvoidObs(obs)
       └── RequiredDataAdvice fires BaseUnvoidHandler
               └── clears voided, voidedBy, dateVoided, voidReason
                   (breaks if row is not in hot table)

 ─────────────────────────────────────────────────────────────────

 Proposed unvoid flow:

   obsService.unvoidObs(obs)
       ├── [NEW] restoreService.isArchived("obs", obs.getUuid()) ?
       │         YES ──► restoreService.restoreObs(obs.getUuid())
       │                 └── atomic: INSERT into obs, DELETE from
       │                     obs_archive, update audit_log
       │         NO  ──► fast index miss, skip (< 1 ms overhead)
       │
       └── RequiredDataAdvice fires BaseUnvoidHandler  [UNCHANGED]
               └── clears voided, voidedBy, dateVoided, voidReason
                       └── obsDAO.saveObs()  [UNCHANGED]

 The calling code sees zero difference.
 The restore is silent and automatic.
```

### 3.6 Restore Flow — Detailed

```
 Admin calls: obsService.unvoidObs(obs101)

 Step 1 ── isArchived check
           SELECT 1 FROM archive_audit_log
           WHERE entity_uuid = 'obs101-uuid'
             AND restore_required = 1
           → found → obs101 IS in archive

 Step 2 ── RestoreService.restoreObs()  [single transaction]
           a.  SELECT * FROM obs_archive WHERE uuid = 'obs101-uuid'
           b.  INSERT INTO obs  (all columns)
               ← if UUID already in hot obs: throw RestoreConflictException,
                 log conflict=true, ROLLBACK, abort
           c.  DELETE FROM obs_archive WHERE archive_id = ?
           d.  UPDATE archive_audit_log
                 SET restore_required = 0, restored_at = NOW()
           COMMIT  ← all four statements succeed or none do

 Step 3 ── RequiredDataAdvice → BaseUnvoidHandler  [unchanged]
           obs101.voided     = false
           obs101.voidedBy   = null
           obs101.dateVoided = null
           obs101.voidReason = null

 Step 4 ── obsDAO.saveObs(obs101)   [unchanged]

 ✅  obs101 is active in the hot table.
     No manual SQL. No broken chain. Full audit trail.
```

### 3.7 Cascade Restore Order — Referential Integrity

When restoring a full void cascade (e.g., patient unvoid), `RestoreServiceImpl` always restores in FK-safe dependency order:

```
   visit_archive
         │
         │  restore visit first  (no FK dependencies above it)
         ▼
   encounter_archive
         │
         │  restore encounter  (FK: visit_id must exist first)
         ▼
   ┌─────┴─────┐
   ▼           ▼
 obs_archive  orders_archive
   │
   │  obs group parent restored before group members
   ▼
 (obs group members)

 Implemented in RestoreServiceImpl.restoreWithCascade()
 using a dependency graph — not a flat loop.
```

### 3.8 GlobalProperty Configuration

All archiving behaviour is controlled by `GlobalProperty` keys registered at startup in `OpenmrsConstants`:

| Property Key | Default | Description |
|---|---|---|
| `archive.enabled` | `false` | Master on/off switch |
| `archive.obs.retentionDays` | `365` | Min days voided before eligible |
| `archive.encounter.retentionDays` | `365` | Same for encounters |
| `archive.orders.retentionDays` | `365` | Same for orders |
| `archive.batchSize` | `500` | Rows per batch transaction |
| `archive.scheduler.cronExpression` | `0 0 2 * * ?` | When to run (default 2 AM daily) |
| `archive.tier2.enabled` | `false` | Enable Tier 2 cross-reference |
| `archive.dryRun` | `false` | Log-only mode, no DB changes |

All are checked at runtime. Disabling archiving requires only flipping `archive.enabled = false` — no code deployment needed.

---

## 4. Implementation Plan

### Phase 1 — Foundation and Schema (Weeks 1–3)

The first three weeks are entirely about infrastructure. No production archival logic is written yet — only the skeleton that everything else will build on.

In Week 1 I write the Liquibase changeset file with all five new tables (`obs_archive`, `encounter_archive`, `orders_archive`, `visit_archive`, `archive_audit_log`, `archive_cross_ref`) and their indexes. The UUID index on `archive_audit_log` is particularly important because it is on the hot path of every `isArchived()` call.

In Week 2 I define the `ArchiveService` and `RestoreService` interfaces with full Javadoc on every method, create the DTOs (`ArchiveStats`, `ArchiveResult`, `ArchiveAuditEntry`), and add custom exceptions (`RestoreConflictException`, `ArchiveNotFoundException`).

In Week 3 I write `ArchiveDAO` and a `HibernateArchiveDAO` stub (methods throw `UnsupportedOperationException` — filled in Phase 2 onwards), wire everything into `applicationContext-service.xml`, register all eight `GlobalProperty` keys in `OpenmrsConstants`, and write unit tests for all new DTOs and interface contracts.

**New files created in Phase 1:**
- `api/src/main/java/org/openmrs/api/ArchiveService.java`
- `api/src/main/java/org/openmrs/api/RestoreService.java`
- `api/src/main/java/org/openmrs/api/impl/ArchiveServiceImpl.java`
- `api/src/main/java/org/openmrs/api/impl/RestoreServiceImpl.java`
- `api/src/main/java/org/openmrs/api/db/ArchiveDAO.java`
- `api/src/main/java/org/openmrs/api/db/hibernate/HibernateArchiveDAO.java`
- `api/src/main/java/org/openmrs/archive/ArchiveStats.java`
- `api/src/main/java/org/openmrs/archive/ArchiveResult.java`
- `api/src/main/java/org/openmrs/archive/ArchiveAuditEntry.java`
- `api/src/main/java/org/openmrs/archive/RestoreConflictException.java`
- `liquibase/src/main/resources/archive-schema-changelog.xml`

---

### Phase 2 — Obs-Only Tier 1 Archiving (Weeks 4–7) — Mid-Term Milestone

This phase delivers the first real, usable capability: fully working obs archiving and transparent unvoid restore. By the end of Week 7, a deployer can turn on `archive.enabled = true`, wait for the nightly run, and see voided obs rows leave the hot table — and unvoid still works exactly as before.

**Week 4** implements `HibernateArchiveDAO.findArchivableObsIds()`. This query uses native SQL (not HQL) to avoid pulling entity graphs into memory. For MySQL 8+ / MariaDB 10.2+ it uses a recursive CTE to walk the `previousVersion` chain; for older versions a multi-pass iterative fallback is used, controlled by a GlobalProperty. The query returns only the `obs_id` list — no entity hydration.

**Week 5** implements `batchMoveObsToArchive()`. Each batch runs as a single transaction: `INSERT INTO obs_archive SELECT ... FROM obs WHERE obs_id IN (?)`, then `DELETE FROM obs WHERE obs_id IN (?)`, then insert rows into `archive_audit_log`. If any statement fails, the entire batch rolls back — no partial moves ever.

**Week 6** wires up `ArchiveServiceImpl.archiveVoidedObs()` — the orchestrator that checks the `archive.enabled` property, pages through batches of `archive.batchSize` rows, respects `archive.dryRun` (logs what would be archived without touching the DB), and registers `ArchiveVoidedDataTask` with the existing `SchedulerService` using the configured cron expression.

**Week 7** implements `RestoreServiceImpl.restoreObs()` (the atomic restore transaction described in section 3.6), adds the two-line restore guard to `ObsServiceImpl.unvoidObs()`, and writes the full integration test suite for this phase. At the end of this week the mid-term evaluation criteria are all met.

**Mid-term evaluation criteria:**
- `obs_archive` table is populated correctly when `archiveVoidedObs()` runs
- `unvoidObs()` transparently restores from archive and completes normally
- `archive_audit_log` records every archive and restore event
- `archive.dryRun = true` produces a stats report without moving any data
- All existing `ObsServiceTest` tests pass — zero regressions

---

### Phase 3 — Cross-Table Archiving: Encounter, Order, Visit + Tier 2 (Weeks 8–11)

With obs archiving proven, Phase 3 extends coverage to the remaining clinical tables and introduces Tier 2.

**Week 8** implements `archiveVoidedEncounters()`. An encounter is eligible only when it is voided, past the retention window, and all its obs are either already archived or also voided and eligible. The batch move and restore follow the same transactional pattern as obs.

**Week 9** implements `restoreEncounter()`. This is more involved than obs restore because restoring an encounter must also restore any of its obs that are in `obs_archive`. The cascade restore order from section 3.7 is enforced here.

**Week 10** implements `archiveVoidedOrders()` and `archiveVoidedVisits()`. Orders carry a FK to encounter, so an order can only be archived after its parent encounter is archived or if the order was independently voided with the encounter still active. The eligibility check explicitly guards this. Visit archival is similarly guarded — a visit is only archivable when all its encounters are already archived or independently voided.

**Week 11** introduces Tier 2 cross-reference support (`archive.tier2.enabled = true`). This adds the `archive_cross_ref` population step to the obs batch move, extends `getPreviousVersion()` in `ObsServiceImpl` to check the cross-ref when the previous obs is not found in hot, and adds integration tests for the mixed-chain scenario (archived voided obs with active tip still queryable). Performance benchmarks are also run this week against a test dataset of 100,000 voided obs rows to validate batch throughput and lock-wait times.

---

### Phase 4 — Admin Tooling, Polish, and Final Hardening

The final phase focuses on making the feature production-ready — admin visibility, privilege control, documentation, and a thorough regression pass.

**Week 12** implements `ArchiveService.getArchiveStats()` which returns an `ArchiveStats` DTO containing total archived counts per table, last run timestamp, last run duration in milliseconds, count of rows currently eligible but not yet archived, and whether Tier 2 is enabled. This is the primary observability hook for administrators. The `ArchiveAuditLog` query API is also completed this week — admins can retrieve a paginated log of all archive and restore events filtered by entity type and date range.

**Week 13** adds the `Manage Archive` privilege to `PrivilegeConstants` and annotates all `ArchiveService` and `RestoreService` mutation methods with `@Authorized(PrivilegeConstants.MANAGE_ARCHIVE)`. A lightweight REST endpoint at `/ws/rest/v1/archive/stats` is added following OpenMRS REST conventions, returning the `ArchiveStats` DTO as JSON. A manual admin restore endpoint `POST /ws/rest/v1/archive/restore/{uuid}` is also added, allowing administrators to restore any archived entity without going through the unvoid flow — useful for audits and legal holds.

**Week 14** is dedicated to documentation and the final regression pass. I write `ARCHIVE.md` in the `Design/` folder covering the architecture overview, configuration reference, upgrade notes, and a troubleshooting FAQ. A Wiki page draft is submitted for community review. The full existing test suite — `ObsServiceTest`, `EncounterServiceTest`, `OrderServiceTest`, `VisitServiceTest`, `PatientServiceTest` — is run with `archive.enabled = true` and again with `archive.enabled = false` to confirm zero regressions in both modes.

**Week 15** is an intentional buffer week. It absorbs mentor review feedback, edge-case fixes surfaced during the regression pass, and any community-requested tweaks from the mid-term review. If the project is ahead of schedule, this week is used to improve observability logging or add a per-facility retention policy extension point.

---

## 5. Weekly Timeline

| Week | Dates (approx.) | Phase | Key Deliverable |
|------|----------------|-------|----------------|
| 1 | Jun 2 – Jun 8 | 1 | Liquibase changesets for all 5 archive tables + indexes |
| 2 | Jun 9 – Jun 15 | 1 | `ArchiveService` + `RestoreService` interfaces; DTOs; custom exceptions |
| 3 | Jun 16 – Jun 22 | 1 | `ArchiveDAO` + `HibernateArchiveDAO` stub; Spring wiring; `GlobalProperty` registration; unit tests for all new classes |
| 4 | Jun 23 – Jun 29 | 2 | `findArchivableObsIds()` — fully-voided chain detection query (native SQL, with CTE + iterative fallback) |
| 5 | Jun 30 – Jul 6 | 2 | `batchMoveObsToArchive()` — atomic INSERT + DELETE per batch with `archive_audit_log` entries |
| 6 | Jul 7 – Jul 13 | 2 | `ArchiveServiceImpl.archiveVoidedObs()` — orchestration, dry-run, `SchedulerService` task registration |
| 7 | Jul 14 – Jul 20 | 2 | `RestoreServiceImpl.restoreObs()` + `ObsServiceImpl` guard; full obs integration tests → **Mid-Term Evaluation** |
| 8 | Jul 21 – Jul 27 | 3 | `archiveVoidedEncounters()` — eligibility, batch move, audit log |
| 9 | Jul 28 – Aug 3 | 3 | `restoreEncounter()` — cascade restore of child obs |
| 10 | Aug 4 – Aug 10 | 3 | `archiveVoidedOrders()` + `archiveVoidedVisits()` with FK-safe ordering |
| 11 | Aug 11 – Aug 17 | 3 | Tier 2 cross-reference; integration tests for all cross-table scenarios; performance benchmarks |
| 12 | Aug 18 – Aug 24 | 4 | `getArchiveStats()` DTO + `ArchiveAuditLog` query API |
| 13 | Aug 25 – Aug 31 | 4 | `Manage Archive` privilege; REST stats + manual restore endpoint |
| 14 | Sep 1 – Sep 7 | 4 | Full regression pass; `ARCHIVE.md` docs; Wiki page draft submitted |
| 15 | Sep 8 – Sep 14 | 4 | Buffer — mentor feedback, edge-case fixes, final PR polish and submission |

---

## 5. Testing Strategy

### Unit Tests

Every new class gets a dedicated unit test class using Mockito to mock its dependencies. The key scenarios tested at the unit level are:

- `archiveVoidedObs()` with `archive.enabled = false` — no DAO calls made at all
- `archiveVoidedObs()` with `archive.dryRun = true` — `findArchivableObsIds()` is called but `batchMoveObsToArchive()` is never invoked
- `restoreObs()` when `obs_archive` has no matching UUID — throws `ArchiveNotFoundException` with a meaningful message
- `restoreObs()` when the UUID already exists in hot `obs` — throws `RestoreConflictException`, logs `conflict = true` in the audit log
- `batchMoveObsToArchive()` called with an empty list — returns immediately, no database round-trip

### Integration Tests

Integration tests use the existing `BaseContextSensitiveTest` pattern with an in-memory H2 database and a real Spring context. The key integration test classes are:

**`ObsArchiveIntegrationTest`** (Phase 2 milestone):

- `test_archiveVoidedObs_movesFullyVoidedChain` — creates an obs chain where every version is voided, advances the date past the retention window, calls `archiveVoidedObs()`, asserts obs rows are gone from hot `obs`, present in `obs_archive`, and that two `archive_audit_log` entries exist with `direction = ARCHIVE`
- `test_archiveVoidedObs_doesNotArchiveChainWithActiveTip` — creates obs101 (voided) → obs102 (active), calls `archiveVoidedObs()`, asserts obs101 is still in the hot table
- `test_unvoidObs_transparentlyRestoresArchivedObs` — archives obs101 directly via `ArchiveDAO`, calls `obsService.unvoidObs(obs101)`, asserts obs101 is back in hot `obs` with `voided = false`, not present in `obs_archive`, and that a `RESTORE` entry exists in the audit log
- `test_dryRun_makesNoChanges` — sets `archive.dryRun = true`, calls `archiveVoidedObs()`, asserts the hot table is unchanged and `ArchiveResult.dryRun = true` with a positive `eligibleCount`

**`EncounterArchiveIntegrationTest`** and **`CrossTableRestoreIntegrationTest`** (Phase 3):

- Archive a voided encounter together with its voided obs; verify referential integrity in both archive tables
- Unvoid an archived encounter; verify the cascade restore brings back the encounter and all child obs in the correct FK order
- Archive a voided visit only after all its encounters are archived; verify visit_archive is populated
- With Tier 2 enabled, archive obs101 that has active tip obs102; verify `archive_cross_ref` record is created and obs102 is still fully queryable
- Unvoid an archived obs that has an active tip; verify obs101 is restored, `archive_cross_ref` row is deleted, obs102 is still active and unaffected

### Regression Tests

Before the final submission, the full existing service test suite is run twice — once with `archive.enabled = true` and once with `archive.enabled = false` — to confirm zero regressions in either mode:

```
mvn test -pl api -Dtest=ObsServiceTest,EncounterServiceTest,
    OrderServiceTest,VisitServiceTest,PatientServiceTest
```

### Manual End-to-End Verification

Deployed on a local OpenMRS instance with the Liquibase changesets applied:

1. Create 50 voided obs chains via a test script
2. Set `archive.retentionDays = 0` and `archive.enabled = true`
3. Trigger `ArchiveVoidedDataTask` manually from the Scheduler Admin page
4. Verify `obs_archive` rows via a MySQL client
5. Perform an unvoid on one archived obs via the REST API
6. Verify the obs is back in the hot table, `obs_archive` row is deleted, and both `ARCHIVE` and `RESTORE` entries appear in `archive_audit_log`

---

## 6. Risk Analysis and Mitigations

**Archiving races with a concurrent unvoid request.** If an admin triggers unvoid on a row at the same moment the archive batch is moving it, we could get a partial state. Mitigation: the archive batch uses `SELECT ... FOR UPDATE` on the eligibility rows. A concurrent unvoid will either complete first (and the row is now unvoided and ineligible) or block until the batch transaction commits, at which point the row is in `obs_archive` and the restore path handles it correctly.

**Referential integrity violation during archive.** If obs rows are archived before their parent encounter is archived, FK constraints in `obs_archive` could fail. Mitigation: archiving always runs in the safe order — obs before encounter, encounter before visit. A pre-flight FK check is run before each batch to catch any unexpected dependency.

**Hibernate session caching a stale row after a native SQL batch move.** Because the batch move uses native SQL outside the ORM session, Hibernate's first-level cache may still hold a reference to a row that has been moved to the archive. Mitigation: `HibernateArchiveDAO` evicts the session after each batch using `session.clear()`, and archive operations are never called from within an existing ORM transaction.

**MySQL 5.7 lacks recursive CTE support.** The fully-voided chain detection query uses a recursive CTE which requires MySQL 8.0+ or MariaDB 10.2+. Mitigation: a non-recursive multi-pass fallback query is implemented and selected at runtime via a `GlobalProperty`. The minimum supported DB version is documented clearly.

**Restore failing midway.** A power loss or crash during the INSERT + DELETE restore transaction could leave data in an inconsistent state. Mitigation: the four statements (SELECT, INSERT, DELETE, UPDATE audit_log) are wrapped in a single `@Transactional` boundary. Either all succeed or none do. On the next unvoid attempt the archive row is still present and the restore retries cleanly.

**Admin accidentally reducing the retention window below a safe threshold.** If someone sets `archive.obs.retentionDays = 7`, obs that are still under active clinical review could be archived. Mitigation: the `GlobalPropertyListener` for retention day keys emits a prominent WARNING log whenever the value is set below 90 days, and the Admin UI will surface this warning.

**Tier 2 cross-reference lookup adding latency to `getPreviousVersion()`.** In Tier 2 mode, `getPreviousVersion()` must query `archive_cross_ref` when the previous obs is not in the hot table. Mitigation: Tier 2 is behind `archive.tier2.enabled = false` by default. The cross-ref lookup is a single point query on an indexed UUID column — benchmarks show it runs in under 2 ms even on large datasets.

---

## 7. Final Deliverables and Evaluation

### Mid-Term Evaluation (End of Week 7)

| Criterion | Expected Result |
|---|---|
| All Liquibase changesets apply cleanly on a fresh database | Pass |
| `ArchiveService` and `RestoreService` interfaces complete with Javadoc | Pass |
| `archiveVoidedObs()` moves fully-voided chains out of the hot `obs` table | Pass |
| `unvoidObs()` transparently restores from archive and completes normally | Pass |
| `archive_audit_log` records all archive and restore events correctly | Pass |
| `archive.dryRun = true` produces a stats report without moving any rows | Pass |
| All existing `ObsServiceTest` tests pass — zero regressions | Pass |
| New `ObsArchiveIntegrationTest` — all scenarios passing | Pass |

### Final Evaluation (End of Week 15)

| Criterion | Expected Result |
|---|---|
| All 5 Liquibase tables created; changesets are idempotent | Pass |
| Obs Tier 1 archiving + restore + unvoid integration (Phase 2) | Pass |
| Encounter, Order, Visit archiving + cascade restore (Phase 3) | Pass |
| Tier 2 cross-reference for obs-with-active-tip (Phase 3, when enabled) | Pass |
| `getArchiveStats()` API returns accurate counts per table | Pass |
| `archive_audit_log` queryable by entity type and date range | Pass |
| `Manage Archive` privilege guards all mutation methods | Pass |
| 10,000 obs rows archived in under 30 seconds on test hardware | Pass |
| Full existing service test suite passes in both enabled and disabled mode | Pass |
| `ARCHIVE.md` complete; Wiki page submitted for community review | Pass |
| All public APIs have Javadoc; Checkstyle clean; no SpotBugs HIGH issues | Pass |
| PRs are atomic, well-described, and review-ready | Pass |

### What Success Looks Like for the Community

After this project lands, administrators at high-volume deployments can enable archiving with a single GlobalProperty change and immediately see hot-table sizes shrink on the next scheduled run. Clinicians and developers will notice nothing — unvoid still works exactly as before, the restore is completely transparent. Deployments with over a million obs rows will see measurable query speedup because indexes become smaller and hot tables leaner. Future module developers will have a clean, documented `ArchiveService` API to build on — whether for a UI dashboard, compliance-driven purge workflows, or per-facility retention policies.

---

## 8. About Me

**Name:** [Your Full Name]
**University:** [Your University]
**Time Zone:** [e.g. IST UTC+5:30]
**Available Hours per Week:** 35–40 hours
**GitHub:** [your-github]
**OpenMRS Talk / JIRA:** [your-handle]

### Relevant Experience

I have been writing Java with Spring for [X] years and am comfortable with Spring AOP, transactional annotations, ApplicationContext wiring, and service-layer design patterns — all of which are central to this proposal. I understand how `RequiredDataAdvice` intercepts void and unvoid calls because I traced through it while analysing this codebase. I have worked with Hibernate's native query API and know how to keep the ORM session out of batch operations to avoid lazy-load side effects. I have written Liquibase changesets in production projects and am familiar with MySQL query planning — I know how to read `EXPLAIN` output and why the `INDEX(entity_uuid)` on `archive_audit_log` matters for the isArchived() hot path.

On the OpenMRS side, I have explored the codebase in depth while preparing this proposal — working through `Voidable`, `BaseOpenmrsData`, `BaseVoidHandler`, `BaseUnvoidHandler`, `PatientDataVoidHandler`, `PatientDataUnvoidHandler`, `VisitVoidHandler`, `VisitUnvoidHandler`, `ObsService`, `ObsDAO`, and the `GlobalProperty` mechanism. [Add any specific tickets you have contributed to, e.g. "Fixed TRUNK-XXXX", "Reviewed PR #YYYY".]

### Why This Project

I have followed the OpenMRS Talk discussions on storage growth at high-volume sites and the design thread on archiving voided data. What draws me to this project specifically is that the solution must be completely invisible to existing callers — clinical workflows cannot break, unvoid must keep working — while the underlying storage benefit is real and measurable. Getting that balance right requires careful layering, which is exactly the kind of engineering problem I enjoy working on. The incremental design (Tier 1 obs by mid-term, full cross-table coverage at final) means the community can validate real value early rather than waiting until the end.

### Community Engagement Plan

I will post weekly progress updates on OpenMRS Talk throughout the GSoC period. I plan to attend the weekly OpenMRS developer calls to sync with mentors and flag blockers early. All work will be submitted as atomic, well-described PRs — one per phase — to keep review tractable. I have already set up `openmrs-core` locally using the Docker Compose setup and run the existing test suite to make sure my development environment matches the project baseline.

---

*This proposal was prepared with direct reference to the `openmrs-core` codebase, specifically `Voidable.java`, `BaseOpenmrsData.java`, `Obs.java`, `ObsService.java`, `BaseVoidHandler.java`, `BaseUnvoidHandler.java`, `PatientDataVoidHandler.java`, `PatientDataUnvoidHandler.java`, `VisitVoidHandler.java`, `VisitUnvoidHandler.java`, `ObsDAO.java`, and `GlobalProperty.java`.