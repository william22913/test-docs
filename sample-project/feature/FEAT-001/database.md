---
feature_code: FEAT-001
service: sample-project
version: 2
status: complete
spec_version: 9
sources:
  - path: knowledge/system_requirement_document.md
    source_version: "1.0"
    note: The SRD for the Teacher Management Module. Source of the TEACHER entity attribute table (§5), the status enum values, and the hard-delete prohibition (§4.4). Superseded in several places by spec.md — see spec.md's scope-reduction table.
  - path: knowledge/nexcommon-reuse-survey.md
    source_version: "1.0"
    note: Survey of the shared nexcommon library. Source of the house DDL/migration style (§5), the audit_helper before-snapshot query that mandates uuid_key and deleted (§1), and the regex constants (§3).
pr_url: https://github.com/william22913/test-docs/pull/2
last_updated: 2026-09-28
---

# FEAT-001 — Database Design

Three tables, one migration, one seed. No audit table.

## Scope

Designs the persisted shape for Teacher CRUD: the `teachers` table, its owned
`teacher_education_histories`, and the `institution_levels` lookup. Covers
columns, keys, constraints, indexes, `teacher_code` generation, and seed data.

Does **not** cover: the Go struct/DAO layer (see `architecture.md`), the HTTP
surface, or anything linked to Subject / Class / Student (dropped by spec
non-goals).

The data design answers four decisions taken with the user this session
(D1–D4); each is restated at the point it applies below.

## Conventions inherited

The DDL below follows the house migration style recorded in
`knowledge/nexcommon-reuse-survey.md` §5 — sequence-backed `BIGINT id`,
compact constraint names, `idx_<compact>_<col>` index names, `sql-migrate`
headers.

Three columns appear that the spec never asked for. Each exists for a
mechanical reason, not a preference:

| Column | On | Why it exists |
| :--- | :--- | :--- |
| `uuid_key` | `teachers`, `teacher_education_histories` | `audit_helper`'s before-snapshot query selects `uuid_key` by literal name (`SELECT a.id, uuid_key, row_to_json(a) FROM (...) a`). A table passed to the audit helper without it fails at runtime. Survey §1. |
| `deleted` | `teachers` | `GetDataForAuditByIDTx` filters `deleted = false` for every non-delete action. Survey §1. |
| `version` | `teachers` | The concurrency mechanism (criteria 35–36) — decision D-concurrency below. |

`uuid_key` is `UUID NOT NULL DEFAULT gen_random_uuid()`. This needs
`pgcrypto` on PostgreSQL < 13; on 13+ it is built in. Confirm the target
Postgres version before the migration runs.

**Where `deleted` is deliberately absent:** `teacher_education_histories` is
**hard-deleted**, not soft-deleted — spec criterion 27 requires that deleting an
education row "removes that row only", and the Education history section says
rows are "freely editable and deletable after creation". There is no soft-delete
state for these rows, so the column would be permanently `false`. It is omitted
rather than added for symmetry.

Note the asymmetry this creates, which is intentional: `teachers` has a
`deleted` column it will never set (criterion 19 — no hard delete is
reachable), purely so `audit_helper` can filter on it. `teachers` soft-deletes
via `status = 'INACTIVE'`, not via `deleted`.

---

## Table: `institution_levels`

Lookup table for the `institution_level` field the spec mandates on each
education row. Decision D3 (user, this session): the field is a reference to
this table rather than a free-text column — *"i plan to make it on another table
named institution levels"* — and the table is **seeded only**, with no CRUD
surface of its own.

It hangs off `teacher_education_histories`, **not** off `teachers`: a teacher
does not have one institution level, a teacher with three schools has three.
The spec is explicit that the level is per-row — "Institution level — high
school, university, etc. An explicit field on the row, not inferred from the
institution name."

Not audited, so it carries no `uuid_key` and no `deleted`.

```sql
-- +migrate Up

-- +migrate StatementBegin

CREATE SEQUENCE IF NOT EXISTS institution_levels_pkey_seq;
CREATE TABLE IF NOT EXISTS "institution_levels" (
    id                      BIGINT NOT NULL     DEFAULT nextval('institution_levels_pkey_seq'::regclass),
    uuid_key                UUID NOT NULL       DEFAULT gen_random_uuid(),
    name                    VARCHAR(50) NOT NULL,
    created_at              TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT pk_institutionlevels_id PRIMARY KEY (id),
    CONSTRAINT uq_institutionlevels_uuidkey UNIQUE (uuid_key),
    CONSTRAINT uq_institutionlevels_name UNIQUE (name)
);
```

---

## Table: `teachers`

```sql
CREATE SEQUENCE IF NOT EXISTS teachers_pkey_seq;
CREATE TABLE IF NOT EXISTS "teachers" (
    id                      BIGINT NOT NULL     DEFAULT nextval('teachers_pkey_seq'::regclass),
    uuid_key                UUID NOT NULL       DEFAULT gen_random_uuid(),
    teacher_code            VARCHAR(20) NOT NULL,
    first_name              VARCHAR(50) NOT NULL,
    last_name               VARCHAR(50) NOT NULL,
    email                   VARCHAR(100) NOT NULL,
    phone                   VARCHAR(20) NULL,
    hire_date               DATE NOT NULL,
    status                  VARCHAR(10) NOT NULL DEFAULT 'ACTIVE',
    version                 INTEGER NOT NULL DEFAULT 1,
    deleted                 BOOLEAN NOT NULL DEFAULT FALSE,
    created_by              BIGINT NULL,
    updated_by              BIGINT NULL,
    created_at              TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT pk_teachers_id PRIMARY KEY (id),
    CONSTRAINT uq_teachers_uuidkey UNIQUE (uuid_key),
    CONSTRAINT uq_teachers_teachercode UNIQUE (teacher_code),
    CONSTRAINT uq_teachers_email UNIQUE (email),
    CONSTRAINT ck_teachers_status CHECK (status IN ('ACTIVE', 'ON_LEAVE', 'INACTIVE'))
);

CREATE INDEX IF NOT EXISTS idx_teachers_status ON teachers (status);
CREATE INDEX IF NOT EXISTS idx_teachers_deleted ON teachers (deleted);
```

### Column notes

**`teacher_code`** — the spec's `Teacher_ID` (`TCH-2026-001`). Named
`teacher_code` so the surrogate primary key can be plain `id` and foreign keys
read unambiguously. Generation is covered below.

**`status`** — the spec's three-value set. Note it is **not** the SRD's: SRD §5
specifies `ACTIVE, INACTIVE, ON_LEAVE` and §4.1 says
`Active, On Leave, Terminated`. The spec supersedes both — `Terminated` does not
exist, and `Inactive` is the soft-delete state. `VARCHAR(10)` + `CHECK` rather
than a Postgres `ENUM` type: adding a status later is an `ALTER TABLE ... DROP
CONSTRAINT / ADD CONSTRAINT`, no type migration, and it matches the house
`VARCHAR` style seen in the library's own migrations.

**`email` UNIQUE is load-bearing.** Success criterion 30 requires that two
simultaneous creates with the same email produce exactly one row, and states it
is "enforced in the database, not only in application code". A plain `UNIQUE`
constraint is what delivers that — the application-level pre-check in the create
flow (criterion 3) is a UX nicety for the error message, not the guarantee.
`/go-dev` must not treat the pre-check as the enforcement point, and must map
the resulting `23505` unique-violation to the field-level `Email` error.

**`version`** — optimistic concurrency (criteria 35–36). Integer, starts at 1,
incremented on every real write. Chosen over an ETag or a conditional
`updated_at` compare because it is monotonic, needs no clock agreement, and
cannot be bypassed by a client that simply omits a token — the value is read
from the row the UPDATE targets. Enforcement shape:

```sql
UPDATE teachers
   SET <fields>, version = version + 1, updated_at = CURRENT_TIMESTAMP
 WHERE id = $1 AND version = $expected_version;
```

Zero rows affected means a stale write. The service must read back the current
record and return it alongside the conflict so the losing administrator can
re-fetch (criterion 35). The check is server-side by construction — a client
that omits the version cannot match a row (criterion 36).

**`updated_at` on real change only** (criterion 17). This is *not* enforceable
by the UPDATE above, which sets it unconditionally. The service must diff the
incoming values against the current row and **skip the UPDATE entirely** when
nothing changed — a no-op save writes no row, changes no timestamp, and
therefore (per below) writes no audit entry either. Postgres triggers are not
used here: the diff has to happen in Go regardless, because the audit entry
depends on it.

**`created_by` / `updated_by`** — house audit columns, present for the schema
convention. With no authentication they will always be `NULL`; see the audit
section.

---

## Table: `teacher_education_histories`

```sql
CREATE SEQUENCE IF NOT EXISTS teacher_education_histories_pkey_seq;
CREATE TABLE IF NOT EXISTS "teacher_education_histories" (
    id                      BIGINT NOT NULL DEFAULT nextval('teacher_education_histories_pkey_seq'::regclass),
    uuid_key                UUID NOT NULL DEFAULT gen_random_uuid(),
    teacher_id              BIGINT NOT NULL,
    institution_level_id    BIGINT NOT NULL,
    institution             VARCHAR(150) NOT NULL,
    study_start_date        DATE NOT NULL,
    study_end_date          DATE NOT NULL,
    score                   NUMERIC(5,2) NOT NULL,
    focused_subject         VARCHAR(150) NOT NULL,
    created_at              TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT pk_teachereducationhistories_id PRIMARY KEY (id),
    CONSTRAINT uq_teachereducationhistories_uuidkey UNIQUE (uuid_key),
    CONSTRAINT fk_teachereducationhistories_teacher FOREIGN KEY (teacher_id)
        REFERENCES teachers (id),
    CONSTRAINT fk_teachereducationhistories_level FOREIGN KEY (institution_level_id)
        REFERENCES institution_levels (id),
    CONSTRAINT ck_teachereducationhistories_score CHECK (score >= 0 AND score <= 100),
    CONSTRAINT ck_teachereducationhistories_daterange CHECK (study_end_date >= study_start_date)
);

CREATE INDEX IF NOT EXISTS idx_teachereducationhistories_teacherid
    ON teacher_education_histories (teacher_id);
CREATE INDEX IF NOT EXISTS idx_teachereducationhistories_teacherid_enddate
    ON teacher_education_histories (teacher_id, study_end_date DESC);
```

### Column notes

**`score` is `NUMERIC(5,2)`, and that is not stylistic.** Success criterion 25
requires a score to round-trip exactly — "3.75 stored reads back as 3.75, with
no floating-point drift". `REAL`/`DOUBLE PRECISION` cannot guarantee that.
`NUMERIC` can. Note that in Go this maps to a decimal type, **not** `float64` —
a `float64` round-trip through the driver reintroduces exactly the drift the
criterion forbids. `/go-dev` must pick a decimal representation (or scan into
`string`) for this field.

**The score range is enforced twice, deliberately.** The `CHECK` constraint
(criterion 28) is the guarantee; the DTO validator is what produces the
field-level error message the criterion asks for. A bare `23514` violation is
not a usable error.

**`study_end_date` not in the future** (criterion 29) has **no constraint
here**, and cannot have one. Postgres will not accept `now()` or
`CURRENT_DATE` in a `CHECK` — the expression must be immutable, and both are
stable, not immutable. This is enforced in the service layer only. It is the one
education rule with no database backstop; recorded so the gap is a known
decision rather than an oversight.

**Overlapping ranges are permitted** (spec, Education history): deliberately
*no* exclusion constraint. Two rows at the same institution may overlap — the
user confirmed same-institution overlap is acceptable. An `EXCLUDE USING gist`
would be the opposite design; it is not used.

**`institution_level_id` is `NOT NULL`.** The spec calls institution level an
explicit per-row field with no "unset" state described; a seeded lookup means
there is always a valid value. If a teacher's schooling predates any seeded
level, a row must still name one — this is a seed-content concern, not a schema
one.

**`focused_subject`** is free text by spec — explicitly "not a reference to the
school's subject data and not linked to any subject the teacher teaches here".

---

## `teacher_code` generation

Decision D2 (user, this session): derive from the primary key. **No separate
sequence, no counter table.**

```sql
CREATE OR REPLACE FUNCTION set_teacher_code() RETURNS trigger AS $$
BEGIN
    IF NEW.teacher_code IS NULL OR NEW.teacher_code = '' THEN
        NEW.teacher_code := 'TCH-'
            || to_char(NEW.hire_date, 'YYYY')
            || '-'
            || lpad(NEW.id::text, 3, '0');
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_teachers_setcode
    BEFORE INSERT ON teachers
    FOR EACH ROW EXECUTE FUNCTION set_teacher_code();
```

A `BEFORE INSERT` trigger works here because PostgreSQL applies the column
`DEFAULT nextval(...)` *before* firing `BEFORE INSERT` triggers, so `NEW.id` is
already populated.

Why this shape:

- **Uniqueness and no-reuse are free.** The PK is unique and a sequence never
  hands out the same value twice, which is exactly what criterion 2 asks — "N
  distinct IDs, with no reuse".
- **The year component is `hire_date`'s year**, not the creation year — a
  business-meaningful date, and the one an administrator would expect to see.
- **Gaps are possible and acceptable.** A failed create still consumed a
  sequence value, so codes will skip. Criterion 2 forbids *reuse*, not gaps.
- **There is no per-year reset.** Code 1 and code 400 in 2026 are followed by
  code 401 in 2027, not a return to 001. The spec's example `TCH-2026-001`
  implies but never states a reset, and criterion 2 requires only format,
  uniqueness and no-reuse. A per-year reset would require a counter table with
  row-level locking on every create — real machinery for a property no criterion
  asks for. If a reset *is* wanted, this is the decision to revisit, and the
  replacement is an `ON CONFLICT DO UPDATE ... RETURNING` counter keyed by year.

`teacher_code` is written once and never updated. Criterion 15 requires edits to
`Teacher_ID` to be rejected — the service must not expose it as a writable
field, and the trigger's `IF NEW.teacher_code IS NULL` guard means a stray
update carrying the existing value is a no-op rather than a regeneration.

---

## Seed data

```sql
INSERT INTO institution_levels (name) VALUES
    ('High School'),
    ('Vocational High School'),
    ('University')
ON CONFLICT (name) DO NOTHING;
```

**This list is provisional content, not design.** The spec gives "high school,
university, etc." and no authoritative enumeration exists in the SRD. The three
rows above are a starting set so the `NOT NULL` foreign key is satisfiable on
day one. Amend before deploy — the list is data, and changing it is an ordinary
insertion, not a schema migration.

Seeding happens **in the same migration as the tables**, not a separate one:
this is a brand-new project's starting schema, and the whole initial schema
belongs in one reviewed change.

No other seed data. There is no default teacher, and no bootstrap user — there
is no user table.

---

## Indexes and query paths

| Query | Path | Index |
| :--- | :--- | :--- |
| Fetch by surrogate key | `WHERE id = $1` | PK |
| Fetch by `Teacher_ID` | `WHERE teacher_code = $1` | `uq_teachers_teachercode` |
| Email uniqueness pre-check | `WHERE email = $1` | `uq_teachers_email` |
| List, filter `Active` | `WHERE status <> 'INACTIVE'` | `idx_teachers_status` |
| List, filter `Inactive` | `WHERE status = 'INACTIVE'` | `idx_teachers_status` |
| Education rows for a teacher, ordered | `WHERE teacher_id = $1 ORDER BY study_end_date DESC` | `idx_teachereducationhistories_teacherid_enddate` |

### The one deliberate ceiling: name search

Success criteria 9–10 require search by **name** that is **case-insensitive and
matches substrings, not only exact values**. The `lk` filter operator in
nexcommon's get-list validator maps to `LIKE`, which covers this — but
`ILIKE '%foo%'` **cannot use a B-tree index**, so a name search is a sequential
scan of `teachers`.

That is accepted at this scale, and it collides with one criterion: **criterion
32 asks for p95 < 500ms at 20 records per page.** The spec itself flags that
number as invented — "Invented number — replace with the real target or drop
the line." No target is being designed against.

The upgrade path, when the table is large enough for it to matter, is a
`pg_trgm` GIN index:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_teachers_name_trgm
    ON teachers USING GIN ((first_name || ' ' || last_name) gin_trgm_ops);
```

Not created now. It requires an extension, adds write cost to every teacher
insert and update, and buys nothing until the table is large. Recorded so the
decision is visible: **substring search is unindexed today, by choice.**

**Pagination** uses the get-list DAO's standard offset/limit (20 default, per
criterion 12). Offset pagination is correct here — rows are ordinal and the
dataset is small. Keyset pagination would be the fix if deep pages ever become
slow, and would key on `(last_name, first_name, id)`.

---

## Audit: why there is no audit table

Decision D1 (user, this session): **reuse**, not build.

Criterion 16 requires exactly one audit entry with before and after values per
successful update, and the spec's Dependencies section records that "no log
schema is specified". A `teacher_audit_logs` table was the obvious design. It is
not needed — nexcommon's `services/audit_helper` already does this, and adding a
table would duplicate it. Full mechanics in
`knowledge/nexcommon-reuse-survey.md` §1.

Shape of it: the service function is invoked with an open `sql.Tx`; the helper
captures the before-state with `SELECT a.id, uuid_key, row_to_json(a) FROM
(SELECT * FROM <schema>.<table> WHERE <filters> FOR UPDATE) a`, and after commit
publishes the resulting `AuditSystemModel` list to a **NATS subject**.

**Only `teachers` is registered with the audit helper.** This is forced by
criterion 16's word "exactly one": an update that also adds or removes education
rows would, if education were audited too, emit one entry per changed row — two,
three, four entries for a single save. Registering one table keeps the count at
exactly one per successful update.

The cost is that education-row changes are **not** independently audited; they
are only visible as part of whatever teacher-record change accompanied them, and
an education change that touches no teacher field at all (a pure row add) leaves
no audit trace. This tension between criterion 16's "exactly one" and the
broader goal of auditing personnel data is real and is flagged rather than
silently resolved — it is worth a `/business-analyst` read if education changes
need their own trail.

Three consequences to accept, all recorded:

1. **At-most-once, not exactly-once.** The helper publishes in a goroutine
   (`go ab.pushAuditToMessageBroker(...)`) with a plain `nats.Publish`, and a
   failure is logged, not retried and not rolled back. A publish that fails
   loses the entry while the write succeeds. Criterion 16 is satisfied in the
   happy path only.
2. **The audit trail is not durable until something consumes it.** Nothing is
   written to this database. FEAT-001 requires a NATS JetStream connection and
   a downstream consumer that persists the subject, or the audit data exists
   only as an unrouted message.
3. **No actor.** With `WhitelistValidator` being a no-op, nothing populates
   `ctx.AuthAccessTokenModel.ResourceUserID`, so `CreatedBy`/`CreatedClient`
   are empty. Entries record *what* changed, not *who*. `created_by` /
   `updated_by` on `teachers` stay `NULL` for the same reason.

`NewAuditHelper(enable, db, nats, subject, defaultSchema)` takes the JetStream
context — **if this project has no NATS, D1 flips**, and the fallback is a local
audit table written inside the same transaction (which also restores
exactly-once). Confirm NATS availability before implementation.

---

## Spec invariant → database enforcement

Every row is a spec requirement that lands on the schema. Requirements enforced
only in application code are listed too — that is where the risk is.

| Spec | Requirement | Enforced by |
| :--- | :--- | :--- |
| C2 | `Teacher_ID` unique, no reuse | `uq_teachers_teachercode` + PK-derived generation |
| C3 | Duplicate email rejected | `uq_teachers_email` |
| C4 | Name length 2–50 | App only (`VARCHAR(50)` caps the upper bound; the 2-char floor is app) |
| C5 | Required fields present | `NOT NULL` + DTO validator |
| C6 | Status defaults `Active` | `DEFAULT 'ACTIVE'` |
| C7 | Create + education rows atomic | Single `sql.Tx` covering both inserts |
| C25 | Score round-trips exactly | `NUMERIC(5,2)` |
| C27 | Education row deletion removes only that row | `ON DELETE` not cascading from teacher; explicit delete by id |
| C28 | Score outside 0–100 rejected | `ck_teachereducationhistories_score` |
| C29 | Future end date rejected | **App only** — `CHECK` cannot use `CURRENT_DATE` |
| C30 | Email uniqueness under concurrency | `uq_teachers_email` (DB-enforced by design) |
| C35/C36 | Stale write rejected, server-side | `version` column + conditional `UPDATE` |
| Date rule | Start precedes end | `ck_teachereducationhistories_daterange` |
| — | Status is one of three values | `ck_teachers_status` |

## Open items carried into implementation

1. **NATS availability** — decides whether the audit design stands as written
   (see above). Blocking for criterion 16.
2. **Postgres version** — `gen_random_uuid()` needs `pgcrypto` below 13.
3. **Seed list contents** — provisional, owned by the user.
4. **`institution_levels` editability** — seeded-only this phase; a CRUD surface
   for it would be a separate feature.
