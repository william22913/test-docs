# STUDENT Entity Specification

**Provenance:** supplied by the user in conversation on 2026-09-30, in the same
`Attribute / Data Type / Key Type / Nullable / Description` shape as SRD §5's
`TEACHER` entity. There is **no external document** behind it — the SRD covering this
service (`knowledge/system_requirement_document.md`, held under FEAT-001) defines only
`TEACHER`. This file exists so every `STUDENT` field in `spec.md` traces to a recorded
source rather than to chat.

**Document version:** none. The user supplied no version, date-in-document or
change history, so this is recorded as `unversioned`.

---

## As supplied

| Attribute | Data Type | Key Type | Nullable | Description |
| :--- | :--- | :--- | :--- | :--- |
| `student_id` | INT / VARCHAR(20) | Primary Key | No | Unique system-generated student identifier (e.g., `STU-2026-0001`) |
| `first_name` | VARCHAR(50) | - | No | Student's legal first name |
| `last_name` | VARCHAR(50) | - | No | Student's legal last name |
| `date_of_birth` | DATE | - | No | Used for age verification and grade-level matching |
| `gender` | ENUM | - | Yes | `Male`, `Female`, `Other`, `Prefer not to say` |
| `email` | VARCHAR(100) | Unique | Yes | Student's school-issued email address |
| `phone_number` | VARCHAR(20) | - | Yes | Student or guardian contact number |
| `enrollment_date` | DATE | - | No | Date the student enrolled in the school |
| `grade_level` | INT / VARCHAR(10) | - | No | Current grade/year level (e.g., 9, 10, Grade 11) |
| `status` | ENUM | - | No | Enrollment status: `Active`, `Graduated`, `Suspended`, `Dropped Out` |
| `created_at` | TIMESTAMP | - | No | Timestamp of record creation |
| `updated_at` | TIMESTAMP | - | No | Timestamp of last record modification |

---

## Clarifications given after the table

Recorded here so this file and `spec.md` do not appear to contradict each other. Each
was confirmed by the user on 2026-09-30, after the table above was supplied.

1. **`student_id` is the student code, not the primary key.** The user clarified it is
   the unique business identifier. It therefore becomes `student_code VARCHAR(20)`
   alongside a surrogate primary key, following FEAT-001's resolved pattern for
   `teacher_code`.
2. **`email` and `phone_number` cap at 50, not 100 / 20.** These are the same concepts
   as the teacher's `Email` / `Phone_Number`, which FEAT-001 pinned to 50 (spec →
   Field constraints). One concept, one rule.
3. **`grade_level` is an integer 1–12**, on the Indonesian class-grade scale: 1–6
   primary, 7–9 middle, 10–12 senior. The user clarified the mixed `9` / `10` /
   `Grade 11` examples were loose; the type is integer, not a mixed string.
4. **`status` is enrollment status only** — a different concept from
   `TEACHER.status`, where `INACTIVE` doubles as the soft-delete state. `Graduated`,
   `Suspended` and `Dropped Out` all describe enrolment, not deletion.
5. **`student_subject_id` — mandatory.** The user confirmed the student carries a
   reference to one of the four seeded `student_subjects` values, and that it is
   **required**. It is not present in the table above; it was confirmed separately.
