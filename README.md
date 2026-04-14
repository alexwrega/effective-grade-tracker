# Reading HMG 2+ Tracker

Dashboard tracking Reading students with Highest Grade Mastered (HMG) 2 and above.

## View Online

**[https://alexwrega.github.io/effective-grade-tracker/Reading_HMG2plus_Tracker.html](https://alexwrega.github.io/effective-grade-tracker/Reading_HMG2plus_Tracker.html)**

## View Locally

### Option 1: Open directly

Double-click `Reading_HMG2plus_Tracker.html` or open it in any browser.

### Option 2: Local server (recommended)

```bash
cd "/Users/alexandra/Claude Code/Effective Grade Viewer"
python3 -m http.server 8080
```

Then open [http://localhost:8080/Reading_HMG2plus_Tracker.html](http://localhost:8080/Reading_HMG2plus_Tracker.html)

## Features

- **Campus multi-select filter** at the top
- **Hide Early Literacy** toggle (students without a grade 3+ course)
- **Exclude 100 for 100** toggle (removes 100-for-100 courses from course progress)
- **Summary stats** cards: Total, On Track, 1-2 Behind, 3+ Behind
- **Three collapsible sections** grouped by grades ahead status
- **Sortable columns** (click any header)
- **Expandable student detail panels** with three tabs:
  - **Course Progress** with active/inactive toggle and progress bars
  - **Recent Activity (10d)** with lesson names, XP, minutes, accuracy
  - **MAP Scores** with RIT, grade level estimate, percentile

## Key Calculations

- **R90**: Latest MAP `rit_grade_level_estimate` (max within most recent 2-week window)
- **Effective Grade**: `floor(R90)` (e.g. R90 of 3.7 = grade 3)
- **Grades Ahead**: `working_grade_level - effective_grade`

## Data Refresh

Data is embedded in the HTML file. To refresh, re-run the queries via the `timeback-reporting` MCP tool and regenerate the file. See `CLAUDE.md` for query details.
