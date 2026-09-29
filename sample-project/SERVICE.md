# sample-project — Feature Index

One row per feature ever closed out for this service. Updated by
/business-analyst as part of each feature's own PR — see my_need.md
§2.1. Not a substitute for reading a feature's own spec.md — this is
an index to find the right folder, not a replacement for it.

| Feature | Summary | Touches | Status | Updated |
|---|---|---|---|---|
| [FEAT-001](feature/FEAT-001/spec.md) | Teacher CRUD — administrator-facing create, read, update and deactivate of teacher records, plus each teacher's owned education history | 8 HTTP endpoints (`/teachers` CRUD + `/teachers/{id}/deactivate` + education sub-resource add/edit/delete); 3 new tables — `teachers`, `teacher_education_histories`, `institution_levels` (seeded lookup); Go + `nexcommon` + Postgres, reusing nexcommon's `audit_helper` and regex constants; no auth or role model; field caps pinned — email/phone 50, institution/focused_subject 100, score 2 decimals, ASCII-only text (spec.md → Field constraints) | PR open | 2026-09-29 |
