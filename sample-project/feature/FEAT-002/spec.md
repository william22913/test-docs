---
feature_code: FEAT-002
service: sample-project
version: 1
status: draft
completeness:
  problem: covered
  target_user: covered
  success_criteria: covered
  non_goals: covered
  primary_flow: covered
  edge_cases: covered
  dependencies: covered
  priority: covered
sources:
  - path: knowledge/student_entity_spec.md
    source_version: unversioned
    note: The STUDENT entity, supplied by the user in conversation — the SRD defines no student entity. Holds the user's table verbatim plus the five clarifications that followed it; every student field in this spec traces here.
architect_feedback: []
pr_url: null
last_updated: 2026-09-30
---

# FEAT-002 — Subject catalogue, Student records, Teacher data completion

Service: `sample-project` · Part of the School Database Management System (SDMS).

Sourcing convention for this document — claims are marked:

- `(source: SRD §X)` — from `knowledge/system_requirement_document.md`, the Teacher
  Management SRD. **That file lives under FEAT-001's `knowledge/` folder**, not this
  one — see Open questions.
- `(source: FEAT-001)` — from FEAT-001's `spec.md`, as a deliberate recorded handoff.
  FEAT-001's own docs are **not** edited by this feature.
- `(source: user, this session)` — stated by the user in conversation, with no
  document behind it. **The `STUDENT` entity is in this category** — its table is
  recorded in `knowledge/student_entity_spec.md`, but that file is itself a record of
  what the user supplied, not an external document.

Anything marked **(unresolved)** is an open decision, not a settled requirement.

## Problem

FEAT-001 built teacher CRUD and deliberately dropped every Subject / Class / Student
dependency, plus the teacher fields that depended on them. Five requirements were cut
from Create, Update, Deactivate and the detail view, and seven success criteria were
deferred on authentication (source: FEAT-001 → Confirmed scope reduction; Success
criteria → Deferred).

Two separate debts came out of that:

1. **Teacher data is incomplete.** A teacher record cannot record which subject the
   teacher principally teaches, which subjects the teacher teaches at all, or show
   those on the detail view — because `SUBJECT` does not exist (source: FEAT-001).
2. **No student exists anywhere in the system.** SDMS cannot hold a student record at
   all, even though student records are the stated reason teacher hard-delete is
   prohibited — they protect student historical grade logs (source: SRD §4.4, §7).

FEAT-002 closes the part of that debt that does **not** require a Class entity. Class
is deliberately left to a later feature (source: user, this session).

## What FEAT-001 handed over

Every item FEAT-001 left unfinished, and where each one lands. This is the callback
the user asked for this session.

### Scope-reduction table (the five dropped requirements)

| # | FEAT-001 dropped | Lands in FEAT-002? | Why |
| :--- | :--- | :--- | :--- |
| 1 | `Primary_Subject` required on Create | **Yes** | `SUBJECT` now exists |
| 2 | Update reassigns classes when status changes | **No — deferred** | needs `CLASS` |
| 3 | Deactivate validates no active classes depend on the teacher | **No — deferred** | needs `CLASS` |
| 4 | Update edits a teacher's assigned subjects | **Yes** | `SUBJECT` now exists |
| 5a | Detail view: Taught Subjects | **Yes** | `SUBJECT` now exists |
| 5b | Detail view: Active Student Lists | **No — deferred** | needs a student↔teacher link, which `CLASS` should own |
| 5c | Detail view: Assigned Classes | **No — deferred** | needs `CLASS` |

Two further items FEAT-001 dropped from its primary flow are also unblocked and come
back: **search by `Subject`** on Read (source: SRD §4.2) and the **subject selector**
on Create (source: SRD §4.1).

### Deferred success criteria — all still deferred

FEAT-001's criteria 8, 20, 22, 23, 24 and the authentication halves of 33 and 34 stay
deferred. Every one of them is blocked on **authentication**, not on subject or
student, and authentication is out of scope here (source: user, this session). They
remain recorded in FEAT-001 for whichever feature adds a login.

### FEAT-001's open questions — resolution

| # | Open question (FEAT-001) | Outcome |
| :--- | :--- | :--- |
| 1 | Audit approval for `Teacher_ID` / `Hire_Date` edits | **Resolved — no approval workflow is built.** `Hire_Date` is a plain stored value: set on Create and saved. FEAT-001's criterion 15 is unchanged. (source: user, this session) |
| 2 | Phone regex pattern ownership | **`/architect` to source from the implementation**, unchanged from FEAT-001. Not a feature decision. (source: user, this session) |
| 3 | Teacher self-service role and all access control | Still deferred — authentication is out of scope. The nexcommon auth whitelist posture is retained, so `created_by` / `updated_by` remain NULL. (source: user, this session) |
| 4 | `On Leave` vs the SRD's class-reassignment prompt | Still deferred — moot until `CLASS` returns. (source: user, this session) |
| 5 | Concurrency mechanism | **Resolved by FEAT-001** — the architect shipped an optimistic lock (`version` column), PR #7. |

## Target user

Same as FEAT-001: the **School Administrator**, acting on teacher, subject and student
records (source: SRD §3). As in FEAT-001, no role model and no authentication exist in
this phase, so "administrator" describes the human doing the job, not an enforced
system role (source: user, this session).

Students and teachers remain **subjects of the data**, not users of the system — no
accounts, no logins.

## In scope

- **`STUDENT_SUBJECTS`** — a fixed, seeded lookup of four values (`LANGUAGE`, `ART`,
  `SCIENCE`, `SOCIAL`), in use this phase. Seeded only, no CRUD surface.
- **`SUBJECTS`** — the subject catalogue, seeded, with a **read-only API** (list/read
  only — no create, update or delete surface). Built now even though `CLASS` is its
  eventual consumer (source: user, this session).
- **`STUDENT`** — the entity below, with its own CRUD surface.
- **Teacher data completion** — `Primary_Subject` on Create, assigned-subjects editing
  on Update, search by `Subject`, and Taught Subjects on the detail view (items 1, 4,
  5a above).

## Non-goals

Explicitly out of scope for this phase:

- **`CLASS`** — deferred to a later feature in full (source: user, this session). This
  is why items 2, 3, 5b and 5c above stay deferred: each needs a class to point at.
- **Authentication and authorization** — no accounts, no roles, no login. FEAT-001's
  seven deferred criteria stay deferred, and FEAT-001's standing risk note applies
  unchanged: this service must not be deployed anywhere real before authentication
  lands (source: user, this session; source: FEAT-001).
- **Grading / score entry** — the interface for teachers entering student scores is a
  separate module (source: SRD §2.2).
- **Any link from `STUDENT` to `SUBJECTS` or `TEACHER`** — the student references only
  the seeded `student_subjects` lookup. No student↔subject-catalogue and no
  student↔teacher relationship is created this phase; those arrive with `CLASS`
  (source: user, this session). Consequence: "Active Student Lists" on the teacher
  detail view stays deferred (item 5b).
- **Payroll and salary management** (source: SRD §2.2).

## `STUDENT_SUBJECTS` and `SUBJECTS` — two tables, two levels

The user confirmed this session that these are **two distinct entities, each with its
own table** (source: user, this session):

| Table | What it is | Values | In use |
| :--- | :--- | :--- | :--- |
| `student_subjects` | The four fixed groupings | `LANGUAGE`, `ART`, `SCIENCE`, `SOCIAL` | **Now — this phase** |
| `subjects` | The subject catalogue (Mathematics, Physics, …) | open set | Consumed by **`CLASS`**, a later feature |

Both are **seeded**. The subject API is built now even though `CLASS` is its eventual
consumer (source: user, this session).

**Naming.** The user calls the four-value grouping a "student subject" and the
catalogue a "subject". Those names are used here. The analyst's earlier suggestion to
rename the grouping to "stream" / `penjurusan` was **not** adopted — the near-collision
between the two names is a known readability hazard, flagged for `/architect`.

### `student_subjects` — seeded lookup

Follows FEAT-001's `institution_levels` precedent exactly: seeded only with no CRUD
surface, `name` holds a **stable code rather than a display label**, and the
human-readable labels are i18n bundle content keyed by the code — never stored in the
database (source: FEAT-001 → database.md; source: user, this session).

| Column | Type | Notes |
| :--- | :--- | :--- |
| `id` | BIGINT | PK, sequence-backed |
| `uuid_key` | UUID | unique, house convention |
| `name` | VARCHAR(50) | unique; stable code — `LANGUAGE`, `ART`, `SCIENCE`, `SOCIAL` |
| `created_at` / `updated_at` | TIMESTAMP | house convention |

**(unresolved)** Whether those four strings are the exact seeded codes, and whether
en/id labels are in scope for this feature — `institution_levels` only gained them in
PR #8, after its own spec closed. See Open questions.

Referenced by **two** mandatory foreign keys: `STUDENT.student_subject_id` and
`SUBJECTS.student_subject_id`.

### `subjects` — the catalogue

Built now with a **read-only API** (read/list only — no create, update or delete
surface), consumed by `CLASS` later (source: user, this session). Each subject belongs
to exactly one `student_subject` — `subjects.student_subject_id` is mandatory (source:
user, this session).

Field-level design follows the `institution_levels` precedent for a seeded lookup:
stable code held in `name`, human-readable labels from the i18n bundle, no CRUD
surface. The concrete seed contents are `/architect`'s to propose and the user's to
confirm (source: user, this session).

## `STUDENT`

**(All of this section is `(source: user, this session)` — the SRD defines no student
entity, so none of these fields trace to a document.)**

Provided by the user this session in the same SRD §5 shape as `TEACHER`:

| Attribute | Data Type | Key Type | Nullable | Description |
| :--- | :--- | :--- | :--- | :--- |
| `student_code` | VARCHAR(20) | Business key | No | Unique system-generated identifier, e.g. `STU-2026-0001` |
| `student_subject_id` | FK | — | **No** | The student's grouping — one of the four seeded `student_subjects`. **Mandatory.** Not in the user's original table; confirmed separately |
| `first_name` | VARCHAR(50) | — | No | Student's legal first name |
| `last_name` | VARCHAR(50) | — | No | Student's legal last name |
| `date_of_birth` | DATE | — | No | Must put the student at least **5 years old**; not matched against `grade_level` |
| `gender` | Enum | — | Yes | `Male`, `Female`, `Other`, `Prefer not to say` |
| `email` | VARCHAR(50) | Unique | Yes | Student's school-issued email address |
| `phone_number` | VARCHAR(50) | — | Yes | Student or guardian contact number |
| `enrollment_date` | DATE | — | No | Date the student enrolled in the school |
| `grade_level` | Integer | — | No | Class grade, **1–12**: 1–6 primary, 7–9 middle, 10–12 senior |
| `status` | Enum | — | No | Enrollment status: `Active`, `Graduated`, `Suspended`, `Dropped Out` — **four values, no deletion value** |
| deletion flag | Boolean | — | No | Soft-delete state, its own field. `false` on create; set `true` by Deactivate. Named `is_deleted` by the user; FEAT-001's existing `deleted` column is the candidate. See `### Soft delete` |
| `created_at` | Timestamp | — | No | Record creation timestamp |
| `updated_at` | Timestamp | — | No | Last modification timestamp |

Corrections applied to the user's original table, all confirmed this session:

- **The original said `student_id` (`INT / VARCHAR(20)`) Primary Key.** The user
  clarified this is the **student code** — a unique business identifier. It therefore
  follows FEAT-001's resolved pattern: a surrogate primary key plus a
  `student_code VARCHAR(20)` business key with uniqueness enforced separately
  (source: FEAT-001; source: user, this session).
- **The original said `email VARCHAR(100)` and `phone_number VARCHAR(20)`.** These are
  the **same concepts** as the teacher's `Email` / `Phone_Number`, and the user
  confirmed students follow the same **50-character** cap. The two must not diverge —
  one concept, one rule (source: FEAT-001 → Field constraints; source: user, this
  session).
- **The original said `grade_level` was `INT / VARCHAR(10)` with examples `9`, `10`,
  `Grade 11`.** The user clarified it is a **class grade** on the Indonesian scale,
  **1 to 12**, with bands: 1–6 primary, 7–9 middle, 10–12 senior. The type is therefore
  integer, not a mixed string (source: user, this session).

### `STUDENT.status` is not the teacher's status

The user confirmed this session that a student's status is **enrollment status only**
— a different concept from `TEACHER.status`, where `INACTIVE` doubles as the
soft-delete state. `Graduated`, `Suspended` and `Dropped Out` all describe enrolment,
not deletion (source: user, this session).

The field therefore holds **exactly four values** — `Active`, `Graduated`,
`Suspended`, `Dropped Out` — and no deletion value is added to them. Deletion is
carried on its own field; see `### Soft delete`.

**The value list is a fixed constant** — not a database lookup, and nothing in the API
can add, rename or remove a value (source: user, this session). **Status is set through
the ordinary Update path**: there is no separate status-change endpoint (source: user,
this session). Implementation follows the teacher's, which stores status as a
width-limited string with a `CHECK` rather than a Postgres `ENUM`, so adding a value
later is a constraint change and not a type migration (source: FEAT-001 → database.md).

**No status is terminal.** Asked which values are one-way, the user replied that they
want to update a student through the API update — so **any value may be set from any
other**, including `Graduated` back to `Active`, and including on a soft-deleted
student's status field, which stays editable while the deletion flag is what keeps the
student out of active lists (source: user, this session). This follows from deletion
having left the `status` column: with no deletion value in the list, no value in the
list needs protecting as one-way.

### `grade_level` is stored and updatable, not derived

The user confirmed this session that `grade_level` is a **plain stored, updatable**
field, **not derived from `date_of_birth`**. The rationale: a student does not always
advance a grade each year — a student may repeat a level — so birth date is not a
reliable source (source: user, this session).

`date_of_birth` is therefore **not cross-checked against `grade_level`**; the two may
legitimately disagree. It carries one validation of its own: a student **under 5 years
old is rejected** (source: user, this session). A future date of birth fails the same
rule — a negative age is under 5 — so one check covers both.

`grade_level` is an integer **1–12**. A value outside that range is rejected, not
clamped — extending FEAT-001's reject-don't-coerce rule (source: FEAT-001 → Field
constraints; source: user, this session).

**The student-subject grouping applies to every grade level.** Asked whether a stream
belongs only to senior grades, the user confirmed it applies to **every** grade
(source: user, this session). A grade-1 student may therefore carry any of the four
values, and `student_subject` is independent of `grade_level` — no cross-check between
them.

### Soft delete

Students are **soft-deleted, following the teacher**: the record is retained in the
database and never hard-deleted (source: user, this session). This answers the
hard-delete question — the prohibition that protects teacher records applies to
students too.

**Soft delete is its own field, not a status value.** The user rejected overloading
`status` — one column serving both enrolment and deletion — and asked why not carry a
separate deletion flag instead (source: user, this session). FEAT-002 therefore departs
from the teacher's *mechanism* while keeping the teacher's *flow*: the deactivate
action, the retained row, the removal from active lists and selection dropdowns, and
the absence of any hard-delete path are identical to the teacher's (source: FEAT-001
§4.4; source: user, this session).

**Which column.** FEAT-001 already carries a `deleted` boolean on every table for
nexcommon's `audit_helper` filter, which FEAT-001 deliberately left **always false**
(source: FEAT-001 → database.md). Making that existing column the soft-delete flag is
the smaller change than adding a second boolean beside it. The user named the field
`is_deleted`; the column already present is `deleted`. Either way the spec requires only
that deletion is **not** carried on `status` — the column choice is `/architect`'s.
Note the consequence for FEAT-001: setting it true flips an invariant that feature
declared, so `/architect` must be told. See Open questions.

## Field constraints

FEAT-002 inherits FEAT-001's field rules rather than restating them. Where a student
field shares a concept with a teacher field, the same rule applies (source: FEAT-001 →
Field constraints; source: user, this session):

- **Max 50** for `email` and `phone_number`, matching the teacher caps.
- **ASCII only**, rejected with a field-level error.
- **`email` stored lowercased**, with case-insensitive uniqueness.
- Education-style **reject-don't-coerce** applies to `grade_level`: a value outside
  1–12 is rejected, not clamped.
- **Minimum age 5.** A `date_of_birth` that makes the student younger than 5 is
  rejected with a field-level error on `Date_Of_Birth`. **(unresolved)** The 5-year
  floor is measured against `enrollment_date` — the school-relevant reference, and
  stable, since a stored pair does not drift out of validity. Measuring it against the
  current date at request time is the alternative; see Open questions.

Lengths for `first_name` / `last_name` (2–50) carry over from FEAT-001 as well.

**(unresolved)** Two carry-overs are assumed rather than separately confirmed: that
`phone_number` takes FEAT-001's regex validation — which the user assigned to
`/architect` to source from the implementation (see `## What FEAT-001 handed over`) —
and that the student email uniqueness index is case-insensitive in the same way as the
teacher's.

## Dependencies

- **`STUDENT_SUBJECTS`** — new seeded lookup, four values, referenced by a **mandatory**
  foreign key from `STUDENT` (source: user, this session). Whether the four strings are
  the exact seeded codes, and whether en/id labels ship with this feature, is
  `/architect`'s; see Open questions.
- **`SUBJECTS`** — new seeded catalogue. Needed by: `Primary_Subject` on Create,
  assigned-subjects on Update, search by `Subject`, Taught Subjects on the detail view
  (source: user, this session). Consumed by `CLASS` when that feature arrives.
- **`STUDENT`** — new entity, defined above. Its only relationship this phase is a
  **mandatory** reference to `student_subjects`; it links to nothing else.
- **`TEACHER`** — gains a relationship to `SUBJECT`. FEAT-001's spec records that
  `TEACHER` "links to no other data except its own education history" — this feature
  deliberately changes that, and the change is a direct consequence of items 1 and 4
  above (source: FEAT-001; source: user, this session).
- **SRD §7 assumption confirmed:** "a teacher can teach multiple subjects across
  multiple classes (Many-to-Many relationship)" — so teacher↔subject is many-to-many,
  and the reference does **not** live as a single column on either table (source:
  SRD §7).
- **`institution_levels`** — FEAT-001's seeded lookup with en/id labels already
  exists. The student grade bands (primary / middle / senior) may overlap it
  conceptually; whether to reuse it or keep them separate is **(unresolved)** and is
  `/architect`'s call (source: user, this session).

## Success criteria

Each line is an action with an observable result, checkable by someone who was not
involved in this conversation. `[FEAT-001]` = a requirement FEAT-001 dropped, now
delivered. `[SRD]` = required by the source document. `[new]` = **derived by the
analyst, not stated line-by-line by the user.** The user confirmed the student CRUD
surface as a whole this session; each line below remains the analyst's rendering of it,
so any line that surprises you is a question, not a settled requirement.

### Reference data

1. **[new]** After seeding, the four student-subject values are present and the read
   endpoint returns exactly those four. The set is a fixed constant — nothing in the
   API can add, rename or remove a value.
2. **[new]** The student-subject route surface is read-only: no create, update or
   delete route exists for it.
3. **[new]** The subject catalogue is readable through a read-only API, each subject
   returned with its parent student-subject. No write route exists.

### Teacher completion — FEAT-001's dropped requirements, delivered

4. **[FEAT-001 #1]** Create accepts `Primary_Subject`; it persists and is returned.
   Omitting it is rejected, and the error names the field. (source: SRD §4.1)
5. **[FEAT-001 #4]** Update assigns and unassigns taught subjects, and the change
   persists. (source: SRD §4.3)
6. **[SRD]** Search by `Subject` returns the teachers who teach that subject.
   (source: SRD §4.2)
7. **[FEAT-001 #5a]** The teacher detail view includes Taught Subjects.
   (source: SRD §4.2)
8. **[new]** A subject may be taught by several teachers and a teacher may teach
   several subjects — the relationship is many-to-many, and no subject carries a
   single owning teacher. (source: SRD §7)

### Student

9. **[new]** Creating a student with all required fields persists the record, and it is
   retrievable by its `Student_Code`.
10. **[new]** `Student_Code` matches the agreed format and is unique — creating N
    students yields N distinct codes, with no reuse.
11. **[new]** `student_subject_id` is mandatory. Omitting it, or supplying an unknown
    value, is rejected with a field-level error.
12. **[new]** `grade_level` below 1 or above 12 is rejected, not clamped; 1 and 12 are
    both accepted.
13. **[new]** A `date_of_birth` putting the student at 5 or older at `enrollment_date`
    is accepted even when it disagrees with `grade_level`; one putting them under 5 is
    rejected with a field-level error on `Date_Of_Birth`.
14. **[new]** Creating a student whose email already exists, differing only in case, is
    rejected with a field-level error on `Email`, and zero rows are written.
15. **[new]** `Email` is stored lowercased and returned lowercased.
16. **[new]** A non-ASCII value in any text field is rejected, naming the field.
17. **[new]** A value exceeding its field's max length is rejected, naming the field; a
    value exactly at the max is accepted.
18. **[new]** List search by name and by `Student_Code` each return the expected
    student; search is case-insensitive and matches substrings.
19. **[new]** The list filters by `grade_level`, by `student_subject` and by status,
    and each filter narrows correctly.
20. **[new]** Pagination defaults to 20 per page; page 2 returns the next 20 with no
    duplicates and no gaps; the final page may be partial.
21. **[new]** Attempting to edit `student_code` or `enrollment_date` is rejected, and
    the stored value is unchanged.
22. **[new]** Update sets status through the normal update path — there is no separate
    status-change endpoint. The stored list is exactly the four enrollment codes; no
    deletion value appears among them.
23. **[new]** Any status value may be set from any other — `Graduated` back to `Active`
    included — with no transition refused as one-way.
24. **[new]** Deactivate sets the deletion flag: the row still exists, remains readable,
    is absent from the default list, and no hard-delete path is reachable. The flag and
    `status` are independent — deactivating does not change `status`, and no `status`
    update changes the flag.
25. **[new]** Each successful update writes exactly one audit entry recording the
    before and after values.
26. **[new]** A write built on a stale read is rejected, and the current record is
    returned so the administrator must re-fetch before retrying.

### Cross-cutting

27. **[new]** Email uniqueness holds under concurrency — two simultaneous creates with
    the same email produce exactly one row, enforced in the database.
28. **[new]** No authentication or role model exists this phase; `created_by` and
    `updated_by` remain NULL on every row written. (source: user, this session)

## Primary flow

### Reference data (read-only)

1. Administrator opens the subject list and sees the catalogue, each subject grouped
   under its student-subject.
2. Administrator opens the student-subject list and sees exactly four values.
3. Neither list offers a create, edit or delete action — the data is seeded.

### Teacher — completing FEAT-001

- **Create:** as FEAT-001, plus `Primary_Subject` is selected and required.
- **Update:** as FEAT-001, plus assigned subjects can be added and removed.
- **Read:** as FEAT-001, plus search by `Subject`, and Taught Subjects on the detail
  view.

### Student

**Create**

1. Administrator opens Student Management and selects Create.
2. Enters `first_name`, `last_name`, `date_of_birth`, `enrollment_date` and
   `grade_level`, and selects a `student_subject`. Optionally `gender`, `email`,
   `phone_number`.
3. Validation runs: required fields present, `grade_level` 1–12, `date_of_birth` puts
   the student at 5 or older, ASCII-only text, max lengths, email uniqueness.
4. On success `Student_Code` is generated and the record is saved with status
   `Active`.

**Read**

1. Administrator searches by name or `Student_Code`, or opens the list.
2. Optionally filters by `grade_level`, `student_subject` or status.
3. The list displays paginated, 20 records per page.
4. The detail view shows the student's fields; `student_subject` renders as a label
   resolved from the i18n bundle, not the stored code.

**Update**

1. Administrator edits the mutable fields, including `grade_level` and status.
2. `student_code` and `enrollment_date` are rejected as edits.
3. Any status value may be set, from any other value.
4. Save, and write an audit-log entry.

**Deactivate**

1. Administrator selects Deactivate.
2. The deletion flag is set true. `status` is untouched.
3. The row is retained, and the student leaves the default list.
4. No hard-delete path exists.

## Edge cases

- **`grade_level` boundaries** — 1 and 12 accepted; 0 and 13 rejected. The bands (1–6
  primary, 7–9 middle, 10–12 senior) are display groupings only: the user confirmed the
  `student_subject` applies to **every** grade, so no band restricts which value may be
  chosen, and `grade_level` and `student_subject` are never cross-checked.
- **`date_of_birth`** — a birth date putting the student under 5 at `enrollment_date` is
  rejected, which also rejects a future date. No cross-check against `grade_level`,
  because a repeating student is valid. **(unresolved)** The 5-year floor's reference
  point — `enrollment_date` or the request date — is the analyst's assumption, not the
  user's words.
- **Duplicate email** — rejected case-insensitively with zero rows written, and safe
  under concurrency.
- **Unknown `student_subject_id`** — rejected as a field-level error, not surfaced as a
  foreign-key crash.
- **Soft-deleted student** — remains readable by id, absent from the default list. Its
  `status` stays editable, since the flag and `status` are independent fields.
- **Deactivated then reactivated** — clearing the flag returns the student to the
  default list, since no status transition is one-way.
- **Empty states** — a grade level with no students returns an empty list, not an
  error; a subject taught by nobody returns an empty teacher list on search.
- **Teacher with no taught subjects** — valid. `Primary_Subject` is required, but the
  taught-subject list may be empty.

## Priority

Build order — **the analyst's proposal, awaiting the user's confirmation**:

1. **Reference data first** — the `student_subjects` seed and the `subjects` catalogue
   with its read-only API. Nothing else can be validated without them, and both
   student and teacher records point at them.
2. **Teacher completion** — `Primary_Subject`, taught-subject assignment, search by
   Subject, Taught Subjects on the detail view. The smallest slice that retires
   FEAT-001's longest-standing debt.
3. **Student records** — create, read, update, soft-delete. The largest surface, and
   the only part with a genuinely new entity.

- **This phase:** subject catalogue (read-only), the student-subject lookup, student
  CRUD, and the four FEAT-001 callback items.
- **Deferred:** everything needing `CLASS` (callback items 2, 3, 5b, 5c), everything
  needing authentication, and grading.

## Open questions

The four decisions that were blocking this spec — status count, status transitions,
grade-level scope of the stream, and date-of-birth validation — were all answered by
the user on 2026-09-30 and are now written into the body. What remains: item 1 is a
loose end created by those answers, items 2–4 are `/architect`'s to settle, and item 5
is a sourcing choice.

1. **How is the soft-delete flag carried?** The user settled the *shape* — deletion is
   its own field, never overloaded onto `status` (spec → `### Soft delete`) — and named
   it `is_deleted`. Two loose ends follow, both `/architect`'s:
   - **Reuse or add.** FEAT-001 already has a `deleted` boolean on every table for
     nexcommon's `audit_helper`, deliberately always false. Reusing it is the smaller
     change; adding `is_deleted` keeps that invariant intact at the cost of a second
     boolean. The spec permits either — only `status` is barred from carrying deletion.
   - **Tell `/architect` about the invariant.** Setting either column true contradicts
     a guarantee FEAT-001's `database.md` states, so the FEAT-002 migration is where
     that guarantee is formally amended. This should not be discovered mid-build.
2. **`SUBJECTS` seed contents.** Which subjects exist, and which `student_subject` each
   belongs to, is `/architect`'s to propose and the user's to confirm — the user
   deferred this to design (source: user, this session).
3. **Seeding mechanism for both lookups** — how `student_subjects` and `subjects` are
   seeded, and whether either carries en/id labels like `institution_levels` — is
   `/architect`'s to design (source: user, this session).
4. **`institution_levels` overlap.** FEAT-001's seeded lookup already carries
   primary/middle/senior concepts. Whether the student grade bands reuse it or stay
   separate is `/architect`'s call (source: user, this session).
5. **Which SRD copy does this feature cite?** The SRD lives under FEAT-001's
   `knowledge/`, so this feature's `sources` lists only `student_entity_spec.md`.
   Either copy the SRD into `knowledge/` here, or leave the cross-folder reference in
   prose.
