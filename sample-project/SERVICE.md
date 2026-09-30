# sample-project — Feature Index

One row per feature ever closed out for this service. Updated by
/business-analyst as part of each feature's own PR — see my_need.md
§2.1. Not a substitute for reading a feature's own spec.md — this is
an index to find the right folder, not a replacement for it.

| Feature | Summary | Touches | Status | Updated |
|---|---|---|---|---|
| [FEAT-001](feature/FEAT-001/spec.md) | Teacher CRUD — administrator-facing create, read, update and deactivate of teacher records, plus each teacher's owned education history | 8 HTTP endpoints (`/teachers` CRUD + `/teachers/{id}/deactivate` + education sub-resource add/edit/delete); 3 new tables — `teachers`, `teacher_education_histories`, `institution_levels` (seeded lookup); Go + `nexcommon` + Postgres, reusing nexcommon's `audit_helper` and regex constants; no auth or role model; field caps pinned — email/phone 50, institution/focused_subject 100, score 2 decimals, ASCII-only text (spec.md → Field constraints) | PR open | 2026-09-29 |
| [FEAT-002](feature/FEAT-002/spec.md) | Subject catalogue and student records — the seeded four-value `student_subjects` lookup, the `subjects` catalogue with a read-only API, the `STUDENT` entity with its own CRUD surface, and the four requirements FEAT-001 dropped that do not need a `CLASS` entity | 3 new tables — `students`, `subjects`, `student_subjects` (both lookups seeded) — plus a many-to-many join for teacher↔subject; read-only subject API, student CRUD + deactivate, and teacher `Primary_Subject` / assigned subjects / search by Subject / Taught Subjects; no auth, `created_by`/`updated_by` stay NULL; field caps inherited from FEAT-001, min age 5, soft delete carried on its own column rather than `status`; `CLASS`, grading and authentication deferred (spec.md → What FEAT-001 handed over) | PR open | 2026-09-30 |
