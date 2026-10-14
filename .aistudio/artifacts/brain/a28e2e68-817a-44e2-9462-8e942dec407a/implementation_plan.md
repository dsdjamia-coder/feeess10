# Student Academic Year History & Annual Bulk Promotion System

A unified student management architecture that preserves permanent student identities, maintains comprehensive year-by-year academic and fee history, provides a dedicated read-only historical archive table for past academic years, and introduces an intuitive bulk promotion tool for annual exam progression.

## User Review & Critical Decisions

> [!IMPORTANT]
> Based on your confirmed choices in Phase 1, the system is designed with the following core architectural rules:

- **Confirmed Decision 1: Single Student Profile with Multi-Year History**: Each student maintains a single permanent record (Admission Number, personal details, contact info) with a nested multi-year academic map (`academicYears[yearLabel] = { class, status, division, updatedAt }`). Personal details are never duplicated across years.
- **Confirmed Decision 2: Dedicated Read-Only Historical Archive Table**: When browsing past academic years, historical records are shown in a dedicated read-only archive view reflecting the exact class, division, and status the student had in that specific year, preventing accidental mutations of historical records.
- **Confirmed Decision 3: Bulk Promotion Tool with Pass/Retain Controls**: A dedicated annual promotion interface allows administrators to filter by current class, mark each student as Promoted (with automated next-grade suggestion) or Retained (stays in the same class), and commit the transition in a single atomic batch into the next academic year.

---

## 1. Overview & Core Concept

### What It Does
1. **Permanent Student Identity**: When a student is admitted, their core record is created once. Subsequent edits (e.g. phone number, address) update the master profile without altering their historical enrollment trajectory.
2. **Year-by-Year Academic Trajectory**: Every academic year records the student's specific class, division, enrollment status (Promoted, Retained, Graduated, Active), and concessions for that year.
3. **Annual Promotion Workflow**: At the conclusion of annual exams, administrators open the Bulk Promotion tool, select source class and target academic year, review auto-promoted classes (e.g., Class 1 -> Class 2), mark any retained students with a single click, and promote the entire batch.
4. **Historical Archive Viewing**: Selecting any previous academic year displays students in the exact grade and status they had during that specific period in a dedicated read-only table with export capabilities (Excel, CSV, PDF).

### Target Audience & Persona
- **School Administrators & Headmasters**: Seamlessly rollover school sessions at the end of each academic year without data loss or re-entering student profiles.
- **Fee Operators & Office Clerks**: Review historical dues, past payments, and past class rosters for any previous session without conflicting with current active enrollments.

---

## 2. User Experience & Visual Design

### Key User Flows

#### Flow 1: Annual Bulk Promotion (`Student Management -> Promote Students`)
1. Admin navigates to the new **"Promote Students"** tab in the sidebar under Student Management (or via a quick action in Manage Students).
2. Admin selects:
   - **Source Academic Year** (e.g., `2024-2025`)
   - **Source Class** (e.g., `Class 4` or `All Classes`)
   - **Target Academic Year** (e.g., `2025-2026`)
3. A clean, tabular roster populates showing:
   - Admission Number (`font-mono tabular-nums`)
   - Student Name
   - Current Class
   - Promotion Status Toggle: **Promote (Pass)** vs **Retain (Same Class)** vs **Graduate / Transfer**
   - Target Class Selector (defaults to next class, e.g., Class 5; editable dropdown)
4. Admin clicks **"Preview & Confirm Promotion"** modal, showing summary counts (e.g., "38 Promoted, 2 Retained, 0 Graduated").
5. On confirm, Firestore batch updates all records, updates the student's current year/class, and logs an audit log entry.

#### Flow 2: Viewing Historical Academic Years (`Manage Students & Archive`)
1. Admin or staff selects an academic year from the Academic Year selector (or opens the **"Historical Student Archive"**).
2. If the selected year is not the current active year, the system displays a clear banner: `"Viewing Archive: 2023-2024 (Read-Only Records)"`.
3. The student table displays each student's class and status *as they were in 2023-2024*.
4. Edit and delete action buttons are replaced with view and export actions (Excel/PDF) to preserve data integrity.

### Visual Identity & Theme
- **Color Discipline**: Clean white/neutral canvas (`#F8FAFC` to `#FFFFFF`), structural borders (`border-slate-200`), deep indigo accents for academic navigation (`#4338CA`), emerald accents for promotions (`#059669`), and amber for retentions (`#D97706`).
- **Typography**: Clean sans headers with tabular numerals (`tabular-nums font-mono`) for admission numbers, student roll counts, and statistics.
- **Zero-Pill Discipline**: Status labels rendered as clean text with typographic dividers (e.g., `Class 4 · Promoted to 5 · 2024-2025`), with interactive segmented tabs for filter controls.

---

## 3. Key Product Decisions & Trade-Offs

### Decision 1: Hybrid Data Model (Master Profile + Year History Map + Enrollment Index)
- **Chosen Approach**:
  - Master student record (`students/{studentId}`) contains personal info + `academicYear` (current year) + `class` (current class) + `academicYears: { [yearLabel]: { class, status, division, updatedAt } }`.
  - In addition, write lightweight enrollment documents to `academicYears/{yearDocId}/enrollments/{studentId}`.
- **Why**: Allows instant query of active students for the current year without expensive map lookups, while giving instant read-only lookups for past years.
- **Alternatives Considered**: Duplicating full student documents into each year collection (violates Single Source of Truth; editing student phone number would require updating multiple documents).

### Decision 2: Automated Class Progression Heuristic
- **Chosen Approach**: The promotion tool automatically maps standard numeric classes to the next level (e.g., `1 -> 2`, `9 -> 10`, `10 -> Graduated/Plus One`) while providing a fast per-student dropdown override for special divisions (e.g., moving from `4` to `5 A` or `5 B`).
- **Why**: Saves 95% of manual clicking for large classes while retaining complete flexibility for school-specific division assignments.

### Decision 3: Backward-Compatible Graceful Fallback
- **Chosen Approach**: Existing student documents without `academicYears` map are seamlessly backfilled on the fly when read by reading `student.academicYear` and `student.class`, ensuring zero loss or disruption for existing student data.

---

## 4. Technical Architecture & Data Strategy

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Master Student Profile                          │
│               Collection: /artifacts/.../students/{studentId}          │
├────────────────────────────────────────────────────────────────────────┤
│  adm: "1024"                       name: "Aisha Rahman"                │
│  gender: "Female"                  phone: "9876543210"                 │
│  status: "Active"                                                      │
│  currentAcademicYear: "2025-2026"  class: "5 A"                        │
│                                                                        │
│  academicYears: {                                                      │
│    "2024-2025": { class: "4 A", status: "Promoted", updatedAt: ... },  │
│    "2025-2026": { class: "5 A", status: "Active",   updatedAt: ... }   │
│  }                                                                     │
└─────────────────────────────────┬──────────────────────────────────────┘
                                  │
          ┌───────────────────────┴───────────────────────┐
          ▼                                               ▼
┌───────────────────────────────┐               ┌───────────────────────────────┐
│ Current Academic Year View    │               │ Historical Archive Table      │
│ (Active Roster & Edits)       │               │ (Read-Only Past Year View)    │
├───────────────────────────────┤               ├───────────────────────────────┤
│ • Filters by Current Year     │               │ • Filter by Selected Past Year│
│ • Full CRUD & Concessions     │               │ • Shows Class in THAT Year    │
│ • Feeds Fee Collection Hub    │               │ • Read-Only / Export Ready    │
└──────────────┬────────────────┘               └───────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Annual Bulk Promotion Workflow                       │
│              (Source Year: 2024-2025 ──► Target Year: 2025-2026)      │
├────────────────────────────────────────────────────────────────────────┤
│ • Renders class roster with Promoted / Retained radio or switch        │
│ • Auto-computes target class (e.g. 4 A -> 5 A)                         │
│ • Atomic writeBatch commits both master student and enrollment history │
└────────────────────────────────────────────────────────────────────────┘
```

### Component & File Strategy
1. **`admin.html`**:
   - Add **"Promote Students"** navigation link in sidebar under "Student Management".
   - Add `<section id="promote-students-page" class="page hidden">` with source year, source class, target year controls, and batch action table.
   - Add **"Historical Archive"** toggle or mode in `manage-students-page` when an archived or non-current academic year is selected.
2. **`scripts/pages/admin-page.js`**:
   - `renderPromotionTool()`: Loads students for source year, renders promotion table with auto-suggested next classes.
   - `executeBulkPromotion()`: Executes `runBatchedWrites` updating `student.academicYears[targetYear]`, `student.class`, `student.academicYear`, and enrollment indices.
   - `getStudentsForAcademicYear(yearLabel)`: Enhanced selector that checks `student.academicYears?.[yearLabel]` or `student.academicYear === yearLabel`.
   - `renderHistoricalArchiveTable(yearLabel)`: Read-only rendering with past status and class indicators.
3. **Firestore Security & Blueprint (`firebase-blueprint.json` & `firestore.rules`)**:
   - Ensure `academicYears` map property is documented in the student entity schema.
   - Validate batch update operations are fully supported within security rules.
