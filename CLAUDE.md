# CLAUDE.md — Effective Grade Tracker (HMG 2+ Reading Focus)

## Project Overview

This project builds an interactive HTML dashboard that tracks **Reading** students whose **Highest Grade Mastered (HMG) is 2 or above**. It is inspired by [Nick's MAP K–2 Reading Progress tracker](https://nicka-nto.github.io/Effective-Grade-Tracker/MAP_K2_Reading_Progress.html) but differs in scope and data source:

- **Data source**: Live queries via the `timeback-reporting` MCP tool (SQL against the reporting database), **not** static/hardcoded data.
- **Scope**: Students where `rpt2_mastery.highest_grade_mastered >= 2` AND `rpt2_mastery.subject = 'Reading'` AND student is active (not withdrawn).
- **Campus filter**: Multi-select campus filter at the top that applies globally to all tables/sections.
- **Student detail panels**: Each student row has a toggle (▶/▼) that expands an inline sub-panel with detailed information.

The output is a **single self-contained HTML file** with embedded CSS and JS — no external frameworks, no build step.

---

## Data Source: timeback-reporting MCP

### Authentication

The project uses the `timeback-reporting` MCP tool connected via Claude. Credentials (for direct API access outside MCP, e.g. via Claude Code):

```json
{
  "SSO": {
    "AWS_COGNITO_REGION": "us-east-1",
    "AWS_COGNITO_APP_CLIENT_ID": "REDACTED",
    "AWS_COGNITO_ISSUER": "https://cognito-idp.us-east-1.amazonaws.com/REDACTED",
    "AWS_COGNITO_JWKS_URI": "https://cognito-idp.us-east-1.amazonaws.com/REDACTED/.well-known/jwks.json",
    "AWS_COGNITO_CLIENT_SECRET": "REDACTED"
  },
  "API": {
    "AWS_COGNITO_APP_CLIENT_ID": "REDACTED",
    "AWS_COGNITO_CLIENT_SECRET": "REDACTED"
  }
}
```

API documentation (Scalar/OpenAPI): `https://api.alpha-1edtech.ai/scalar` and `https://qti.alpha-1edtech.ai/scalar`

### Available Database Views

| View | Purpose | Key Columns |
|------|---------|-------------|
| `rpt2_student` | Student roster | `student_id`, `name`, `campus`, `age_grade_level`, `age_grade_level_numeric`, `enrollment_date`, `withdrawn_date`, `guide`, `guide_id` |
| `rpt2_mastery` | HMG per subject | `student_id`, `subject`, `highest_grade_mastered`, `highest_grade_mastered_date`, `working_grade_level`, `age_grade_level`, `grades_ahead`, `onboarding_complete_date` |
| `rpt2_map_scores` | MAP test results | `student_id`, `subject`, `test_date`, `test_season`, `rit_score`, `rit_grade_level_estimate`, `achievement_percentile`, `growth_multiple`, `map_test_type` |
| `rpt2_enrollment` | Course enrollments | `student_id`, `course_id`, `subject`, `begin_date`, `end_date`, `course_type` |
| `rpt2_student_course_progress` | Progress per course | `student_id`, `course_id`, `subject`, `grade_level`, `course_name`, `total_xp`, `course_earned_xp`, `course_remaining_xp`, `pct_complete`, `is_enrolled`, `course_type` (base/hole-filling) |
| `rpt2_daily_activity` | Daily XP/minutes | `student_id`, `course_id`, `course_name`, `subject`, `app_name`, `calendar_date`, `earned_xp`, `active_minutes`, `correct_questions`, `total_questions` |
| `rpt2_activity_log` | Granular activity events | `student_id`, `course_id`, `course_name`, `subject`, `app_name`, `event_time`, `activity_name`, `earned_xp`, `correct_questions`, `total_questions` |
| `rpt2_course` | Course metadata | `course_id`, `course_name`, `subject`, `app_name`, `primary_grade_level`, `total_xp`, `total_lessons` |
| `rpt2_course_sequence` | Sequence/stage structure | `sequence_id`, `subject`, `stage_position`, `stage_type` (course/assessment/hole-filling), `grade_level`, `course_id` |
| `rpt2_question_outcomes` | Individual question results | `student_id`, `subject`, `grade_level`, `score_date`, `test_name`, `is_correct`, `score`, `attempt` |
| `rpt2_subject` | Subject metadata | `subject_name`, `is_core_subject`, `typical_xp_per_grade_level` |
| `rpt2_calendar` | School calendar | — |
| `rpt2_course_content` | Course content details | — |
| `rpt2_app` | App metadata | — |

### Core Query — HMG ≥ 2 Reading Students

```sql
SELECT
  s.student_id,
  s.name,
  s.campus,
  s.age_grade_level,
  s.age_grade_level_numeric,
  s.guide,
  m.highest_grade_mastered,
  m.highest_grade_mastered_date,
  m.working_grade_level,
  m.grades_ahead,
  m.onboarding_complete_date
FROM rpt2_student s
JOIN rpt2_mastery m ON s.student_id = m.student_id
WHERE m.subject = 'Reading'
  AND m.highest_grade_mastered >= 2
  AND (s.withdrawn_date IS NULL OR s.withdrawn_date = '')
ORDER BY s.campus, s.name
```

### Detail Sub-Panel Queries (per student)

**Current Reading Course Progress:**
```sql
SELECT
  scp.course_name,
  scp.grade_level,
  scp.course_type,
  scp.is_enrolled,
  scp.total_xp,
  scp.course_earned_xp,
  scp.course_remaining_xp,
  ROUND(scp.pct_complete::numeric * 100, 1) AS pct_complete
FROM rpt2_student_course_progress scp
WHERE scp.student_id = '{student_id}'
  AND scp.subject = 'Reading'
  AND scp.is_enrolled = true
ORDER BY scp.grade_level
```

**Recent Activity (last 14 days):**
```sql
SELECT
  da.calendar_date,
  da.course_name,
  da.app_name,
  da.earned_xp,
  da.active_minutes,
  da.correct_questions,
  da.total_questions
FROM rpt2_daily_activity da
WHERE da.student_id = '{student_id}'
  AND da.subject = 'Reading'
  AND da.calendar_date >= CURRENT_DATE - INTERVAL '14 days'
ORDER BY da.calendar_date DESC
```

**MAP Scores:**
```sql
SELECT
  ms.test_date,
  ms.test_season,
  ms.rit_score,
  ms.rit_grade_level_estimate,
  ms.achievement_percentile,
  ms.growth_multiple,
  ms.map_test_type
FROM rpt2_map_scores ms
WHERE ms.student_id = '{student_id}'
  AND ms.subject = 'Reading'
ORDER BY ms.test_date DESC
```

---

## Dashboard Layout Specification

### 1. Header & Global Controls

- **Title**: "Reading HMG 2+ Tracker" with subtitle "Students with Highest Mastered Grade 2 and above in Reading"
- **Timestamp**: "Data as of: {query execution time}"
- **Campus Multi-Select Filter**: Dropdown/checkbox list of all campuses present in the data. Defaults to ALL selected. Filtering applies to every table below. Show count: "Showing X of Y students"

### 2. Summary Stats Bar

Four stat cards in a horizontal row:

| Card | Value | Description |
|------|-------|-------------|
| Total HMG 2+ Students | Count | All active students with Reading HMG ≥ 2 |
| On Track or Ahead | Count where `grades_ahead >= 0` | HMG ≥ age grade level |
| Behind (1–2 grades) | Count where `grades_ahead` is -1 or -2 | Need monitoring |
| Significantly Behind (3+) | Count where `grades_ahead <= -3` | Need intervention |

Color coding: green (on track), yellow (1–2 behind), red (3+ behind).

### 3. Student Table — Grouped by Status

Group students into collapsible sections by `grades_ahead` status:

#### Section A: "✅ On Track or Ahead" (grades_ahead ≥ 0)

| Column | Source |
|--------|--------|
| ▶ (toggle) | Expand/collapse detail panel |
| Student | `s.name` |
| Campus | `s.campus` |
| HMG | `m.highest_grade_mastered` (the grade level: 2, 3, 4, etc.) |
| Age Grade | `s.age_grade_level` |
| Grades Ahead | `m.grades_ahead` |
| HMG Date | `m.highest_grade_mastered_date` (formatted) |
| Working Level | `m.working_grade_level` |

#### Section B: "⚡ 1–2 Grades Behind" (grades_ahead is -1 or -2)

Same columns as above, plus:

| Additional Column | Source |
|-------------------|--------|
| Active Course | From `rpt2_student_course_progress` where enrolled & Reading |
| Course Type | `scp.course_type` (base / hole-filling) |
| XP Remaining | `scp.course_remaining_xp` |

#### Section C: "🚨 3+ Grades Behind" (grades_ahead ≤ -3)

Same as Section B (these students need the most attention).

### 4. Student Detail Sub-Panel (toggle)

When the ▶ toggle is clicked, expand an inline panel below the student row with three tabs/sections:

#### Tab 1: Course Progress
- Table of all currently enrolled Reading courses
- Columns: Course Name, Grade Level, Type, XP Earned / Total, % Complete (progress bar)

#### Tab 2: Recent Activity (14 days)
- Table of daily activity
- Columns: Date, Course, App, XP Earned, Minutes, Accuracy (correct/total)
- Summary row: Total XP, Total Minutes, Avg Accuracy over the period

#### Tab 3: MAP Scores
- Table of all MAP Reading scores
- Columns: Date, Season, RIT Score, Grade Level Estimate, Percentile, Growth Multiple, Test Type
- If no MAP data, show "No MAP scores available"

### 5. Table Features

- **Sortable columns**: Click column header to sort asc/desc (▲/▼ indicators)
- **Campus filter**: The multi-select at the top filters ALL sections simultaneously
- **Collapsible sections**: Each status group (A/B/C) can be expanded/collapsed; show student count in header
- **Sticky header**: Table headers stick on scroll within each section

---

## Styling Guidelines

- Clean, professional look — similar to Nick's report
- Font: system-ui / -apple-system / Segoe UI fallback stack
- Status colors: `#22c55e` (green/on-track), `#f59e0b` (amber/behind), `#ef4444` (red/significantly behind)
- Table rows: alternating `#f9fafb` / `#ffffff` backgrounds
- Detail sub-panel: slightly indented, light blue-gray background (`#f0f4f8`)
- Responsive: should work on desktop and tablet widths
- Progress bars in detail panel: use inline `<div>` bars with percentage width

---

## Build Process

1. **Query data** via `timeback-reporting:getData` MCP tool (or direct API with credentials above)
2. **Aggregate** all query results into JSON
3. **Generate** a single self-contained HTML file with:
   - Embedded `<style>` block
   - Embedded `<script>` block with data as `const DATA = {...}`
   - All rendering logic in vanilla JS (no React/Vue/etc.)
4. **Output** to `/mnt/user-data/outputs/Reading_HMG2plus_Tracker.html`

### Data Refresh Workflow

To refresh the dashboard with current data:
1. Re-run the core query and detail queries via MCP
2. Regenerate the HTML file
3. The HTML file is fully self-contained and can be opened in any browser or hosted on GitHub Pages

---

## Key Differences from Nick's Version

| Aspect | Nick's Version | This Version |
|--------|---------------|--------------|
| Scope | All K–2 Reading students grouped by MAP target gap | Students with HMG ≥ 2 in Reading |
| Data source | Hardcoded/static HTML | Live queries via timeback-reporting MCP |
| Grouping | By MAP target achievement status | By `grades_ahead` status (on track / behind / significantly behind) |
| Student details | No expandable detail | Toggle sub-panel with course progress, recent activity, MAP scores |
| Campus filter | None | Multi-select campus filter affecting all tables |
| Columns | MAP Target, Current HMG, Activity, XP to Test, Doom Loop | HMG, Age Grade, Grades Ahead, HMG Date, Working Level, Active Course info |

---

## SQL Pitfalls to Avoid

- Always parenthesize OR conditions: `(s.withdrawn_date IS NULL OR s.withdrawn_date = '')`
- Use `ROUND(value::numeric, 2)` — Postgres requires the `::numeric` cast
- Use `NULLIF(divisor, 0)` for safe division
- Column names are `snake_case` (e.g., `age_grade_level`, `grades_ahead`, `earned_xp`)
- String values in `withdrawn_date` can be empty string `''` rather than NULL — always check both
- The `rpt2_mastery` view has **one row per student per subject** — no deduplication needed
- `highest_grade_mastered` is numeric: Pre-K = -1, K = 0, 1st = 1, 2nd = 2, etc.
- Test/sandbox accounts are automatically excluded from all views

---

## File Structure

```
project/
├── CLAUDE.md                          # This file — project spec
├── Reading_HMG2plus_Tracker.html       # Generated dashboard (output)
└── queries/                           # (Optional) saved SQL queries
    ├── core_hmg2plus_students.sql
    ├── student_course_progress.sql
    ├── student_recent_activity.sql
    └── student_map_scores.sql
```
