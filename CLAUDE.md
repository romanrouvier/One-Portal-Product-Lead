# CLAUDE.md — One Portal Product Lead

This file is the single source of truth for Claude Code across all sessions working on this repository.

---

## 1. What We Are Building

A **Product Lead operational dashboard** for the BTDP (Beauty Tech Data Platform) team at
L'Oréal. It replaces manual ServiceNow browsing with a single-page app that surfaces sprint
health, team capacity, incidents, and release status.

**Target users:** Product Leads (e.g. Alex Laurent) managing one or more BTDP assignment groups,
running daily stand-ups, sprint reviews, and capacity planning sessions.

**Five views (tabs):**

| Tab | Purpose |
|---|---|
| Product Lead Portal | Sprint health overview: capacity, incidents/defects, activity feed, priority weighting, burndown |
| Daily Standup | Per-person story board for the current sprint with timer |
| Sprint Review | Completed / deferred / cancelled stories + release notes generation |
| Availability | Per-person per-day availability calendar across sprints (0 / 0.5 / 1 MD) |
| Capacity Planner | MD totals, sprint story table, profile breakdown, capacity per epic |

---

## 2. Repository Layout

```
One-Portal-Product-Lead/
├── backend/                  Express API — port 3001
│   ├── .env                  (gitignored) — env vars, BQ config, SN credentials
│   ├── data/mockData.js      Single source of truth for ALL mock data
│   ├── lib/
│   │   ├── bigquery.js       BQ singleton + query helper
│   │   └── servicenow.js     ServiceNow fetch helper
│   ├── routes/               One file per endpoint
│   │   ├── sprint.js
│   │   ├── stories.js
│   │   ├── team.js
│   │   ├── projects.js       (epics)
│   │   ├── stateChanges.js
│   │   ├── incidents.js
│   │   ├── availability.js
│   │   ├── demands.js
│   │   ├── nextSprint.js
│   │   ├── releases.js       ← new
│   │   ├── burndown.js       ← new
│   │   └── assignmentGroups.js ← new
│   └── server.js
└── frontend/                 React 18 + Vite SPA — port 5173
    └── src/
        ├── App.jsx            Navigation: useState string, no router
        ├── context/DataContext.jsx   Fetches all endpoints in parallel → useData()
        ├── components/AIAssistant.jsx  Sidebar chatbot (keyword-only, no LLM)
        └── views/
            ├── ProductLeadPortal.jsx   ✅
            ├── CapacityPlanner.jsx     ✅
            ├── Availability.jsx        ✅
            ├── DailyStandup.jsx        ← to build (Phase 2)
            └── SprintReview.jsx        ← to build (Phase 2)
```

---

## 3. Dev Commands

All commands run from the repo root (or `backend/` / `frontend/` for package-specific ops):

```bash
npm run install:all   # Install backend + frontend dependencies
npm run dev           # Run backend (3001) and frontend (5173) concurrently
npm run dev:back      # Backend only
npm run dev:front     # Frontend only
npm run build         # Vite production build → frontend/dist/
```

No test suite is currently configured.

---

## 4. Architecture Decisions (DO NOT CHANGE)

- **No router** — navigation is a `useState` string in `App.jsx`. Tabs: `'portal' | 'daily' | 'sprint-review' | 'capacity' | 'availability'`
- **Single data hook** — `DataContext.jsx` fetches all endpoints in parallel on mount; components consume `useData()` only, never fetch directly.
- **Mock fallback** — every route checks `process.env.USE_MOCK_FALLBACK`. When `'true'`, it returns the mock data. Flip to `'false'` per route as real data is validated.
- **Design system is fixed** — do not introduce new UI libraries. Stack:
  - Tailwind with custom palettes: `loreal` blue `#2649B2`, `payne` gray `#4A5568`
  - Reusable CSS classes in `frontend/src/index.css` under `@layer components`: `.widget-card`, `.widget-title`, `.badge`, `.btn-primary`, `.btn-ghost`
  - Charts: `recharts` (BarChart, PieChart, LineChart)
  - Icons: `lucide-react`

---

## 5. Environment Variables (`backend/.env`)

```
PORT=3001
USE_MOCK_FALLBACK=true

BQ_BILLING_PROJECT=oa-data-claude-np
BQ_PUBLISHED_PROJECT=itg-btdppublished-gbl-ww-dv
BQ_DATASET_AGILE=btdp_ds_c1_054_agile_eu_dv
BQ_DATASET_ITBM=btdp_ds_c1_055_itbm_eu_dv
BQ_DATASET_ITSM=btdp_ds_c1_056_itsm_eu_dv
BQ_DATASET_IDENTITY=btdp_ds_c1_058_identity_eu_dv

SN_BASE_URL=https://<instance>.service-now.com
SN_USER=
SN_PASS=
```

---

## 6. Data Architecture

### 6.1 GCP BigQuery (primary read layer)

- **Published project:** `itg-btdppublished-gbl-ww-dv`
- **Billing project for all queries:** `oa-data-claude-np` — always pass this as `projectId` to the BQ client
- **Auth:** Application Default Credentials (`gcloud auth application-default login`)
- **Node.js client:** `@google-cloud/bigquery`
- **All tables are READ-ONLY BigQuery views** with OA Pass row-level filtering. Never attempt INSERT/UPDATE.
- **Data freshness:** daily batch — ~24h old. Do not claim real-time accuracy.

```js
// BQ query pattern
const { BigQuery } = require('@google-cloud/bigquery');
const bq = new BigQuery({ projectId: 'oa-data-claude-np' });
const [rows] = await bq.query({
  query: `SELECT ... FROM \`itg-btdppublished-gbl-ww-dv.btdp_ds_c1_054_agile_eu_dv.sprint_v1\`
          WHERE assignment_group_sk = @groupSk`,
  params: { groupSk: '...' }
});
```

#### Accessible tables (confirmed rows > 0)

| Dataset | Table | Maps to entity |
|---|---|---|
| `btdp_ds_c1_054_agile_eu_dv` | `sprint_v1` | Sprints |
| `btdp_ds_c1_054_agile_eu_dv` | `story_v1` | Stories |
| `btdp_ds_c1_054_agile_eu_dv` | `epic_v1` | Epics / Projects |
| `btdp_ds_c1_054_agile_eu_dv` | `defect_v1` | Defects (DEF) |
| `btdp_ds_c1_054_agile_eu_dv` | `release_v1` | Releases |
| `btdp_ds_c1_054_agile_eu_dv` | `scrum_task_v1` | Sub-tasks / estimates |
| `btdp_ds_c1_054_agile_eu_dv` | `scrum_theme_v1` | Themes |
| `btdp_ds_c1_055_itbm_eu_dv` | `time_card_day_v1` | Timesheet actuals |
| `btdp_ds_c1_056_itsm_eu_dv` | `problem_v1` | Problems |
| `btdp_ds_c1_056_itsm_eu_dv` | `change_request_v1` | Change requests |

#### Blocked tables (0 rows — OA Pass enrollment pending)

| Table | Unblocks |
|---|---|
| `btdp_ds_c1_056_itsm_eu_dv.incident_v1` | INC count + critical list |
| `btdp_ds_c1_058_identity_eu_dv.identity_azure_users_v1` | Team member profiles |
| `btdp_ds_c1_055_itbm_eu_dv.resource_allocation_daily_v1` | Forward availability (0/0.5/1 MD) |
| `btdp_ds_c1_055_itbm_eu_dv.resource_allocation_v1` | Capacity MD totals |
| `btdp_ds_c1_055_itbm_eu_dv.resource_assignments_v1` | Per-person allocated MD |
| `btdp_ds_c1_055_itbm_eu_dv.resource_aggregate_monthly_v1` | Capacity distribution per epic |

**Strategy for blocked tables:** keep mock data as fallback via `USE_MOCK_FALLBACK=true` per entity.

#### Key schemas

**sprint_v1**
```
sprint_sk, number, short_description, state, planned_start_date, planned_end_date,
assignment_group, assignment_group_sk, group_capacity_points, current_scope_points,
completed_points, committed_points, percent_complete, update_date
```
States: `Complete | Current | Draft | Planning | Cancelled`

**story_v1**
```
story_sk, number, short_description, type, prioity (⚠️ typo in schema), state, points,
sprint_sk, assigned_to, assigned_to_sk, epic_sk, assignment_group_sk,
blocked, blocked_reason, substate, rag_status, update_date, open_date, close_date
```
States: `Complete | Draft | Testing | Work in progress | Ready | Ready for testing | Cancelled | Rejected | Incomplete`
Types: `Story | Evolution request | Design | Enabler | Technical | Development | Hot fixing | Defect Fix | Problem fixing | ...`

**epic_v1**
```
epic_sk, number, short_description, assignment_group_sk, state, rag_status,
percent_complete, total_story_count, planned_start_date, planned_end_date,
completed_count, total_estimate, update_date
```

**defect_v1**
```
defect_sk, number, short_description, priority, state, assignment_group_sk,
assigned_to, open_date, update_date, rag_status, environment, release_sk
```

**release_v1**
```
release_sk, number, short_description, state, assignment_group_sk,
planned_start_date, planned_end_date, actual_start_date, actual_end_date
```

**incident_v1** (0 rows until OA Pass — use mock fallback)
```
incident_sk, number, short_description, priority, priority_code, state, state_code,
assignment_group_sk, creation_date, last_modif_date, type_of_issue
```

**resource_allocation_daily_v1** (0 rows until OA Pass — use mock fallback)
```
resource_allocation_daily_sk, user_sk, user, date, person_days, hours, fte, task_sk
```

### 6.2 ServiceNow REST API

**Base URL:** `https://<instance>.service-now.com/api/now/table/`
**Auth:** Basic auth — credentials in `backend/.env`

| Entity | ServiceNow table | When to call |
|---|---|---|
| State changes / activity feed | `sys_history_line` | Always (not in GCP) |
| Team member type | `sys_user` | Until 058_identity unblocked |
| Incident details (INC) | `incident` | Until 056_itsm unblocked |
| Forward availability | `rm_resource_alloc_daily` | Until 055_itbm unblocked |
| Assignment group members | `sys_user_grmember` | Always |

```js
// ServiceNow call pattern
const snFetch = (table, params) =>
  fetch(`${process.env.SN_BASE_URL}/api/now/table/${table}?${new URLSearchParams(params)}`, {
    headers: {
      Authorization: `Basic ${Buffer.from(`${process.env.SN_USER}:${process.env.SN_PASS}`).toString('base64')}`,
      Accept: 'application/json'
    }
  }).then(r => r.json());
```

### 6.3 Computed / derived data

| Metric | How to derive |
|---|---|
| Time progress % | `(today - sprint.planned_start_date) / (sprint.planned_end_date - sprint.planned_start_date)` |
| Days remaining | `sprint.planned_end_date - today` |
| Burndown ideal line | Linear: `group_capacity_points → 0` over sprint working days |
| Burndown actual | `SUM(story.points WHERE state != Complete)` per day — snapshot only |
| Capacity by epic | `SUM(story.points) GROUP BY epic_sk` until resource tables unblocked |
| Priority score per epic | Map `rag_status`: R=100, A=70, G=40; blend with `percent_complete` |
| Story estimate days | `scrum_task.planned_hours / 8` summed per `story_sk` |
| Story elapsed days | `time_card_day.value_timesheet / 8` summed per task |
| Team member initials | First + last initial from `assigned_to` string |

---

## 7. API Endpoints

### Existing routes (all currently mock data)

| Method | Path | Data |
|---|---|---|
| GET | `/api/sprint` | Current sprint metadata + velocity-per-day |
| GET | `/api/team` | Team members with capacity/allocation |
| GET | `/api/projects` | Epics with priority scores and story counts |
| GET | `/api/stories` | Stories; also `/:id` and `/team-load` sub-routes |
| GET | `/api/state-changes` | Recent story status-change events |
| GET | `/api/incidents` | Incidents/defects summary + critical list |
| GET | `/api/availability` | Per-developer per-day availability + sprint calendar |

### New routes to add

| Method | Path | Source | Purpose |
|---|---|---|---|
| GET | `/api/releases` | `release_v1` + `story_v1` | Sprint Review tab |
| GET | `/api/burndown` | `story_v1` (computed) | Burndown chart |
| GET | `/api/assignment-groups` | `sprint_v1 DISTINCT` | Filter dropdowns |
| GET | `/api/stories/team-standup` | `story_v1` grouped by assignee | Daily Standup view |

---

## 8. Key Constraints

- **GCP views are READ-ONLY.** Never INSERT/UPDATE BigQuery published views.
- **Billing project is `oa-data-claude-np`.** Always pass it as `projectId` to the BQ client — not the published project.
- **Only 5 datasets are in scope:** `054_agile`, `055_itbm`, `056_itsm`, `058_identity`. Do not query any other dataset.
- **Never query `itg-btdpfront-gbl-ww-dv` or `itg-btdpback-gbl-ww-dv`** — only the published project.
- **`prioity` is a typo** in the GCP `story_v1` schema. Use the misspelled column name when querying.
- **No router** — navigation stays as `useState` string in `App.jsx`.
- **Design system is fixed** — no new UI libraries. Tailwind + recharts + lucide-react only.
- **Daily batch freshness** — BQ data is ~24h old. State changes require ServiceNow for current activity.

---

## 9. Implementation Phases

### Phase 1 — Real data for existing views
1. Create `backend/.env`
2. Install `@google-cloud/bigquery node-fetch dotenv` in backend
3. Create `backend/lib/bigquery.js` — BQ singleton + `query(sql, params)` helper
4. Create `backend/lib/servicenow.js` — SN fetch helper
5. Migrate `/api/sprint` → `sprint_v1`
6. Migrate `/api/stories` → `story_v1` JOIN `epic_v1` for epic name
7. Migrate `/api/projects` → `epic_v1` with computed priority score
8. Migrate `/api/incidents` → `defect_v1` (DEF) + mock fallback for INC
9. Add `/api/releases` → `release_v1`
10. Add `/api/assignment-groups` → DISTINCT groups from `sprint_v1`
11. Add `/api/burndown` → computed from `story_v1`
12. Mount new routes in `server.js`
13. Add assignment group filter dropdown to all views

### Phase 2 — New views (build against mock data first)
1. Create `frontend/src/views/DailyStandup.jsx`
   - `StandupTimer` — countdown 15:00, Start/Pause/Reset
   - `PersonColumn` — avatar + name + type badge + story cards
   - `StoryCard` — title, type tag ([FRONT]/[TECH]/[BACK]), status badge, Est/Expend footer, blocked highlight
2. Create `frontend/src/views/SprintReview.jsx`
   - Sprint Recap: group + sprint selectors, Completed/Deferred/Cancelled sections
   - Release Notes: group + release selectors, story list, Generate button
3. Add both tabs to navigation in `App.jsx`
4. Add `/api/stories/team-standup` endpoint
5. Add fetch calls to `DataContext.jsx` for new endpoints

### Phase 3 — State changes + AI
1. Wire `/api/state-changes` to ServiceNow `sys_history_line`
2. Upgrade `AIAssistant.jsx` to call backend `/api/ai-query`
3. Backend `/api/ai-query` routes to Product Owner Agent via MCP

### Phase 4 — Unlock blocked data (after OA Pass)
1. Incidents (INC) via `incident_v1`
2. Team member profiles via `identity_azure_users_v1`
3. Availability grid via `resource_allocation_daily_v1`
4. Capacity MD totals via `resource_allocation_v1` + `resource_assignments_v1`

---

## 10. Useful Query Patterns

```sql
-- Current sprint for a group
SELECT * FROM `itg-btdppublished-gbl-ww-dv.btdp_ds_c1_054_agile_eu_dv.sprint_v1`
WHERE assignment_group_sk = @groupSk AND state = 'Current'
LIMIT 1;

-- Stories for a sprint with epic name
SELECT s.*, e.short_description AS epic_name
FROM `...agile...story_v1` s
LEFT JOIN `...agile...epic_v1` e ON s.epic_sk = e.epic_sk
WHERE s.sprint_sk = @sprintSk;

-- Open defects for a group
SELECT defect_sk, number, short_description, priority, state, open_date, update_date
FROM `...agile...defect_v1`
WHERE assignment_group_sk = @groupSk AND state NOT IN ('Closed', 'Cancelled')
ORDER BY priority, open_date;

-- Capacity distribution by epic (proxy via story points)
SELECT e.short_description AS epic_name, e.epic_sk,
       SUM(s.points) AS total_points,
       ROUND(SUM(s.points) / SUM(SUM(s.points)) OVER () * 100, 1) AS pct
FROM `...agile...story_v1` s
JOIN `...agile...epic_v1` e ON s.epic_sk = e.epic_sk
WHERE s.sprint_sk = @sprintSk
GROUP BY e.epic_sk, e.short_description
ORDER BY total_points DESC;

-- Stories grouped by assignee (Daily Standup)
SELECT assigned_to, assigned_to_sk,
       ARRAY_AGG(STRUCT(story_sk, number, short_description, state, type,
                        points, blocked, blocked_reason, epic_sk)) AS stories
FROM `...agile...story_v1`
WHERE sprint_sk = @sprintSk
GROUP BY assigned_to, assigned_to_sk;
```
