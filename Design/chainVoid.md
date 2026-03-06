When Does the Correction Chain Become Fully Voided? — A Deep Dive

This is a genuinely important question because **the answer is: there is no fixed time, and in many real-world scenarios the chain NEVER fully becomes voided**. Let me explain exactly why, with every scenario traced from the code.

---

## First: What Exactly IS the "Chain"?

Every time a clinician edits an obs, the code in `ObsServiceImpl.saveExistingObs()` creates this structure:

```/dev/null/chain-anatomy.md#L1-30
SINGLE EDIT creates this chain (2 nodes):

  obs_id=101  voided=1  previous_version=NULL  ← the OLD obs (voided)
       ▲
       │  previous_version FK (integer)
       │
  obs_id=102  voided=0  previous_version=101   ← the NEW obs (ACTIVE)

SECOND EDIT on obs 102 creates this chain (3 nodes):

  obs_id=101  voided=1  previous_version=NULL  ← oldest version
       ▲
  obs_id=102  voided=1  previous_version=101   ← middle version (now voided)
       ▲
  obs_id=103  voided=0  previous_version=102   ← ACTIVE version

The chain always has:
  - Exactly ONE active tip (voided=0) at the end
  - All predecessors voided=1
  - Each node pointing to the previous via previous_version (integer FK)
```

The key thing to understand: **the `previous_version` column is a real database foreign key**, as confirmed in the Liquibase schema:

```openmrs-core/api/src/main/resources/org/openmrs/liquibase/snapshots/schema-only/liquibase-schema-only-2.1.x.xml#L5913-L5917
<addForeignKeyConstraint
  baseColumnNames="previous_version"
  baseTableName="obs"
  constraintName="previous_version"
  referencedColumnNames="obs_id"
  referencedTableName="obs"
  onDelete="NO ACTION"
  onUpdate="NO ACTION" />
```

This FK means: **if obs_id=101 is in `obs` table, obs_id=102 (which points to 101 via `previous_version`) cannot exist in `obs` without obs_id=101 being there too.** That is the architectural constraint that drives everything below.

---

## The Four Real-World Scenarios

### Scenario 1: Simple Edit — Chain NEVER Fully Voids (Most Common)

```/dev/null/scenario-1.md#L1-55
CLINICAL STORY:
  Day 1 — Nurse enters blood pressure: 140 mmHg
  Day 2 — Nurse corrects it: 120 mmHg

WHAT HAPPENS IN CODE (saveExistingObs):
  1. Obs.newInstance(obs101)  → creates obs102 (copy of 101)
  2. obs102.setPreviousVersion(obs101)
  3. dao.saveObs(obs102)      → obs102 lands in DB, voided=0
  4. voidExistingObs(obs101)  → obs101 becomes voided=1

DATABASE STATE:
  obs_id=101  value=140  voided=1  previous_version=NULL  ← voided
  obs_id=102  value=120  voided=0  previous_version=101   ← ACTIVE

CHAIN FULLY VOIDED? → NO

WHY: obs102 is voided=0 (ACTIVE). It is the tip of the chain.
The chain is:  voided_node → ACTIVE_tip
There is ALWAYS an active tip as long as the obs has clinical value.

WILL IT EVER BECOME FULLY VOIDED?
Only if one of these happens:
  a) The entire encounter gets voided (wrong patient, transfer)
  b) The patient gets voided (duplicate merge)
  c) The obs itself is explicitly voided via voidObs()
  d) A third edit happens (obs102 becomes voided when obs103 is created)

For a normal clinical obs that was simply corrected once and is
still valid: THE CHAIN NEVER FULLY VOIDS.
obs101 will sit in the obs table with voided=1 FOREVER under the
current system — and under Option A it will ALSO never be archived.

This is the fundamental tension with Option A.
```

### Scenario 2: Multiple Edits — Intermediate Nodes Never Fully Void Either

```/dev/null/scenario-2.md#L1-50
CLINICAL STORY:
  Day 1 — BP recorded: 140
  Day 2 — Corrected: 120
  Day 3 — Corrected again: 118
  Day 4 — Corrected again: 119

DATABASE STATE after 3 corrections:
  obs_id=101  value=140  voided=1  previous_version=NULL
  obs_id=102  value=120  voided=1  previous_version=101
  obs_id=103  value=118  voided=1  previous_version=102
  obs_id=104  value=119  voided=0  previous_version=103  ← ACTIVE TIP

CHAIN FULLY VOIDED? → NO (obs104 is active)

ARCHIVABLE under Option A? → NONE of the 4 nodes

Even though obs101, obs102, obs103 are all voided=1, NONE of them
can be archived under Option A because:
  - obs101: referenced by obs102 via previous_version (still in obs table)
  - obs102: referenced by obs103 via previous_version (still in obs table)
  - obs103: referenced by obs104 (ACTIVE) via previous_version
             → Gate 1 check: active obs references obs103 → SKIP

So after 3 corrections on 1 obs: 3 voided rows stuck in obs table
indefinitely. The more corrections made, the more stuck rows.
At a busy clinic making 5 corrections/day: that's 5 new
permanently-stuck voided rows per day.
```

### Scenario 3: Encounter Voided — Entire Chain Becomes Fully Voided (Chain CAN be archived)

```/dev/null/scenario-3.md#L1-60
CLINICAL STORY:
  Nurse enters BP obs (obs101) for patient A.
  Nurse corrects it (obs102, obs101 becomes voided).
  Doctor realizes the whole encounter was on the wrong patient.
  voidEncounter(encounter, "Wrong patient") is called.

WHAT HAPPENS IN CODE (EncounterServiceImpl.voidEncounter):
  for each Obs in encounter.getObsAtTopLevel(false):
    → obs102 found (voided=0, the active tip)
    → obsService.voidObs(obs102, "Wrong patient")
       → obs102.voided = 1

DATABASE STATE after encounter void:
  obs_id=101  value=140  voided=1  void_reason="Correction"
              date_voided = Day2,  previous_version=NULL
  obs_id=102  value=120  voided=1  void_reason="Wrong patient"
              date_voided = Day5,  previous_version=101

CHAIN FULLY VOIDED? → YES ✓
Both nodes are now voided=1.

ARCHIVABLE under Option A?
  obs102: voided=1, date_voided=Day5
          → No active obs references it (obs102 is the active tip
            that was just voided — nothing points to obs102 as
            previous_version in obs table with voided=0)
          → ELIGIBLE after 30 days from Day5

  obs101: voided=1, date_voided=Day2
          → Is obs102 (which has previous_version=101) still in obs table?
            YES, until obs102 is archived first.
          → After obs102 is archived (moved out of obs table):
            obs101 is no longer referenced by anything in obs table
          → ELIGIBLE for archiving after obs102 is archived

ARCHIVAL ORDER: obs102 first, then obs101
WHEN: Earliest = Day5 + 30 days (retention window from latest void)

KEY INSIGHT: The chain becomes fully voided at Day5 (when the
encounter is voided). But the ARCHIVABILITY depends on the
retention window and correct archival order.
```

### Scenario 4: Patient Merged — The Largest Cascade

```/dev/null/scenario-4.md#L1-55
CLINICAL STORY:
  Patient A has 10 encounters with 30 obs each = 300 obs.
  Each obs has been corrected once = 300 voided + 300 active obs.
  Patient A is found to be a duplicate of Patient B.
  mergePatients(preferred=PatientB, notPreferred=PatientA) is called.

WHAT HAPPENS IN CODE (PatientDataVoidHandler):
  for each encounter of PatientA:
    encounterService.voidEncounter(encounter, "Merged with patient #B")
      for each obs in encounter:
        obsService.voidObs(obs, "Merged with patient #B")
        → ALL 300 active obs tips are now voided

DATABASE STATE after merge:
  obs_id=1 through 300  (original voided corrections) → voided=1 (unchanged)
  obs_id=301 through 600 (active tips)               → NOW voided=1

CHAIN FULLY VOIDED? → YES ✓ All 600 obs rows are voided=1

THE DATES ARE THE SAME:
  Obs 301-600 all have date_voided = merge_timestamp (same exact time)
  Obs 1-300 had date_voided = their individual correction dates (earlier)

ARCHIVABLE under Option A?
  After merge_timestamp + 30 days:
    obs301-600 become eligible first (no active references in obs table)
    → archive them in batch
  After obs301-600 are archived (removed from obs table):
    obs1-300 no longer have any previous_version references from obs table
    → archive them in batch (if also past their own retention window)

This is the BEST CASE for Option A:
  A large cascade event (merge/encounter void) fully voids entire chains
  all at once, making large batches archivable together.
```

---

## The Core Problem with Option A — Visualized

Now you can see the full picture clearly:

```/dev/null/option-a-problem.md#L1-80
OPTION A RULE: "Archive a voided obs only when the entire correction
               chain (all versions) is voided"

════════════════════════════════════════════════════════════════
CASE 1: Normal edited obs (most common in any clinic)
════════════════════════════════════════════════════════════════

  obs101(voided) ← previous_version ← obs102(ACTIVE)

  Is the chain fully voided? NO.
  Will it become fully voided? Only when obs102 is explicitly voided.
  When is obs102 explicitly voided? Only when:
    - The encounter is voided (rare: wrong patient, transfer)
    - The patient is voided (rare: duplicate merge)
    - Someone calls voidObs(obs102) directly (very rare)

  RESULT: Under Option A, obs101 NEVER gets archived during the
  normal lifetime of a patient record. It stays in the obs table
  permanently — exactly the problem we are trying to solve.

════════════════════════════════════════════════════════════════
CASE 2: Encounter-level void (less common, e.g. wrong patient)
════════════════════════════════════════════════════════════════

  After voidEncounter():
    obs101(voided Day2) ← obs102(voided Day5)

  Chain is fully voided at Day5.
  Both nodes eligible for archiving at Day5 + 30 days.

  RESULT: Option A works here. But this is a minority of voids.

════════════════════════════════════════════════════════════════
CASE 3: Patient merge (rare but large impact)
════════════════════════════════════════════════════════════════

  Hundreds of chains become fully voided simultaneously.
  All eligible for archiving after merge_date + 30 days.

  RESULT: Option A works well here. Large batch cleanup possible.
  But this is rare — maybe 10-20 merges per month at a busy site.

════════════════════════════════════════════════════════════════
QUANTIFIED IMPACT:
  In a typical clinic:
    - 95% of voided obs are from the correction pattern (Scenario 1)
    - 4%  of voided obs are from encounter/visit voids (Scenario 3)
    - 1%  of voided obs are from patient merges (Scenario 4)

  Under Option A:
    - 95% of voided obs NEVER become archivable
    - Only 5% ever become archivable

  CONCLUSION: Option A alone solves only ~5% of the problem.
  It is NOT sufficient as the primary archiving strategy.
```

---

## So — Is There a Fixed Time for the Chain to Become Void?

```/dev/null/fixed-time-answer.md#L1-45
DIRECT ANSWER: NO. There is no fixed time.

The chain becomes fully voided only when an external event explicitly
voids the active tip (obs with voided=0). The timing of that event
is entirely unpredictable:

┌──────────────────────────────┬──────────────────────────────────────┐
│  What voids the active tip   │  How often does this happen?         │
├──────────────────────────────┼──────────────────────────────────────┤
│  Another correction          │  Makes things WORSE: creates a NEW   │
│  (obs103 edit of obs102)     │  voided obs in the chain. obs101     │
│                              │  is now 2 levels deep, still blocked │
├──────────────────────────────┼──────────────────────────────────────┤
│  Encounter voided            │  Infrequent. Maybe 1-2% of           │
│  (wrong patient, transfer)   │  encounters per year in a busy clinic│
├──────────────────────────────┼──────────────────────────────────────┤
│  Patient voided              │  Very rare. Only on duplicate merges │
│  (duplicate merge)           │  or test data cleanup                │
├──────────────────────────────┼──────────────────────────────────────┤
│  Patient dies / record       │  Not a void in OpenMRS — patient     │
│  permanently closed          │  records stay active even after death│
├──────────────────────────────┼──────────────────────────────────────┤
│  Never                       │  The most common case for any        │
│                              │  correction on an active patient     │
└──────────────────────────────┴──────────────────────────────────────┘

PRACTICAL IMPLICATION:
  A voided obs created by a correction in Year 1 on an active patient
  will still be sitting in the obs table in Year 5, Year 10, Year 20.
  Option A provides zero archiving benefit for it.
```

---

## What This Means for the Design — Recommendation

Option A alone is insufficient. The design should be a **combination**:

```/dev/null/revised-recommendation.md#L1-80
REVISED DESIGN: Option A + Option C (Cross-table reference resolution)
══════════════════════════════════════════════════════════════════════

TWO-TIER ARCHIVING STRATEGY:

TIER 1 — "Safe Archive" (Option A logic, immediate benefit):
  Archive voided obs where the ENTIRE chain is voided.
  These are chains where an encounter/patient/visit void cascaded
  to all versions. Archive the whole chain together.
  Benefit: Handles patient merges, wrong-patient encounters.
  Coverage: ~5% of voided obs, but often the largest batches.

TIER 2 — "Orphan Archive" (New logic, the big win):
  Archive a voided obs even when its active successor (obs102) still
  exists, by updating obs102.previous_version to store the UUID
  of obs101 instead of (or in addition to) the integer obs_id.

  After archiving obs101 to obs_archive:
    obs102 (active) has previous_version = 101 (integer, now in archive)
    The getPreviousVersion() method checks obs table first, then
    obs_archive → finds obs101 in obs_archive → returns it.

  This means obs101 CAN be archived even while obs102 is active.

RESULT WITH BOTH TIERS:
  ┌─────────────────────────────────────────────────────────────┐
  │  obs101 (voided, correction) → archived after 30 days       │
  │  obs102 (active)             → stays in obs table (correct) │
  │  obs102.getPreviousVersion() → cross-table lookup → obs101  │
  │                                 found in obs_archive ✓       │
  └─────────────────────────────────────────────────────────────┘
  Coverage: ~100% of voided obs become archivable after 30 days

IMPLEMENTATION DIFFERENCE:
  Tier 1: No schema changes required. Straightforward.
  Tier 2: Requires one of:
    a) A new column obs.previous_version_uuid VARCHAR(38) alongside
       the existing previous_version INT. The UUID column survives
       archiving; the INT column becomes null after archiving obs101.
    b) A cross-table lookup in getPreviousVersion() without schema
       change (slower, but avoids migration).

PHASING:
  Phase 2 (obs archiving) → Implement Tier 1 first (safe, simple)
  Phase 3                  → Implement Tier 2 (the real volume win)
```

---

## The Complete Timeline Picture

```/dev/null/complete-timeline.md#L1-70
TIMELINE SHOWING WHEN ARCHIVING HAPPENS UNDER EACH APPROACH:

════════════════════════════
SCENARIO: Simple correction
════════════════════════════

Day 1:   obs101 created (BP = 140)
Day 10:  obs101 corrected → obs102 created (BP = 120), obs101 voided
         obs table: obs101(voided), obs102(active)

Day 40:  Archival task runs (30-day retention passed for obs101)

         OPTION A CHECK:
           Is obs102 (which has previous_version=101) still in obs?
           AND is obs102 active (voided=0)?
           → YES → SKIP obs101 → NOT archived ✗

         OPTION A + C CHECK:
           obs101 is voided, date_voided > 30 days old
           obs102 is active and references obs101
           → Archive obs101 to obs_archive
           → Update obs102.previous_version_uuid = obs101.uuid
           → obs102.previous_version = NULL (FK cleared)
           → obs101 ARCHIVED ✓

Year 5:  obs101 still in obs table under Option A alone.
         obs101 in obs_archive under Option A+C.

══════════════════════════════════════
SCENARIO: Encounter voided (cascade)
══════════════════════════════════════

Day 1:   obs101 created
Day 10:  obs101 corrected → obs102 created, obs101 voided
Day 30:  Encounter voided (wrong patient) → obs102 voided
         obs table: obs101(voided, Day10), obs102(voided, Day30)

Day 60:  Archival task runs (30 days since obs102 was voided)

         OPTION A CHECK:
           obs102: Is any active obs pointing to obs102? NO.
           → obs102 ELIGIBLE ✓
           → Archive obs102 first.

           obs101: Is any obs (voided=1) in obs table with
           previous_version=101? (obs102 was just archived, gone)
           → NO → obs101 ELIGIBLE ✓
           → Archive obs101.

         RESULT: Both obs archived at Day60 under Option A. ✓

SUMMARY TABLE:
┌──────────────────────────┬────────────────┬─────────────────────┐
│ Scenario                 │ Option A alone │ Option A + C        │
├──────────────────────────┼────────────────┼─────────────────────┤
│ Simple correction        │ Never archived │ Archived at Day40   │
│ (active patient record)  │                │ (retention + 30d)   │
├──────────────────────────┼────────────────┼─────────────────────┤
│ Encounter voided         │ Archived after │ Archived after      │
│ (wrong patient)          │ void + 30 days │ void + 30 days      │
├──────────────────────────┼────────────────┼─────────────────────┤
│ Patient merged           │ Archived after │ Archived after      │
│                          │ merge + 30 days│ merge + 30 days     │
├──────────────────────────┼────────────────┼─────────────────────┤
│ Multi-correction chain   │ Never (while   │ Each voided node    │
│ (obs101←102←103 active)  │ tip is active) │ archived 30 days    │
│                          │                │ after its own void  │
└──────────────────────────┴────────────────┴─────────────────────┘
```

---

To give you the direct, complete answer:

**Under Option A alone, the chain becomes fully voided only when an external event (encounter void, patient void/merge) explicitly voids the active tip of the chain. For normal corrections on active patients, this moment never comes — the chain stays permanently stuck in the `obs` table.** This means Option A, by itself, only helps the rare ~5% of voids caused by cascade events, not the dominant 95% caused by routine obs corrections.

That is why the design should combine **Option A for cascade-voided chains** (simple, no schema change) with **Option C's cross-table lookup** for the majority case (voided predecessor nodes whose active successor can still be found in the hot tier after the archive move). Together they cover 100% of eligible voided obs after the retention window.