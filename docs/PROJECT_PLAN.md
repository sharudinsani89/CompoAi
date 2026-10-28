# CompoAI: Project Plan

A progressive web app (PWA) that lets authorised teachers photograph a composition question paper and a batch of student scripts. Claude then marks each script against an agreed rubric and files one report per student into a project folder.

---

## 1. Screen map

```
┌──────────────┐     ┌────────────────────────────┐     ┌──────────────────────────────┐
│  /login      │ ──▶ │  /  (Projects)             │ ──▶ │  /projects/[id]              │
│  email +     │     │  [ + New ]  (top right)    │     │  list of student reports     │
│  password /  │     │  ┌──────────┐ ┌──────────┐ │     │  status: queued / marking /  │
│  magic link  │     │  │ Title    │ │ Title    │ │     │  done / failed               │
└──────────────┘     │  │ 24 comps │ │ 31 comps │ │     │  tap → report view / PDF     │
                     │  └──────────┘ └──────────┘ │     └──────────────────────────────┘
                     └─────────────┬──────────────┘
                                   │ New
                                   ▼
                     ┌────────────────────────────────────────┐
                     │  /new  (Input page)                    │
                     │  Project: (•) existing [dropdown]      │
                     │           ( ) new → Title [_______]    │
                     │  Question paper: [📷 Camera] [🖼 Library]│
                     │  Student scripts: [📷 Camera] [🖼 Library]│
                     │     → grouped per student (see §4)     │
                     │  [ Submit ]                            │
                     └─────────────┬──────────────────────────┘
                                   ▼
                     redirect to /projects/[id], with live progress
```

| # | Requirement | Where it lives |
|---|---|---|
| 1 | Login, authorised users only | `/login`, Supabase Auth, invite-only (public sign-up disabled), middleware guards every route |
| 2 | Mark compositions | Background marking job → Claude API (vision) |
| 3 | Landing page: project title + composition count, **New** button | `/`, a `projects` query with count of `scripts` |
| 4 | Input page: existing/new project, title, question paper upload/camera, scripts upload/camera | `/new`, using `<input type="file" accept="image/*" capture="environment">` plus a library picker |
| 5 | Claude processes on Submit using the agreed rubric | `/api/submissions` → queue → worker calls Claude using a **rubric prompt you supply** (see §6) |
| 6 | Create folder if needed, place individual reports in it | `projects` row = folder; each `report` stored as DB row + PDF in `reports/{project_id}/{student}.pdf` |

---

## 2. Recommended tech stack

| Layer | Choice | Why |
|---|---|---|
| Frontend + API | **Next.js (App Router, TypeScript)** | One codebase for UI and server routes; easy to make a PWA |
| PWA | `manifest.webmanifest` + service worker (Serwist) | Installable on phones; camera access via file input |
| Styling | Tailwind CSS + shadcn/ui | Fast, accessible, mobile-first |
| Auth, DB, file storage | **Supabase** (Postgres + Auth + Storage) | Login, row-level security so teachers only see their own projects, image/PDF storage |
| AI | **Claude API** (Anthropic SDK), vision input | Reads handwriting from photos and marks against the rubric |
| Background jobs | Supabase Edge Function + DB queue, **or** Inngest | Marking 30 scripts takes minutes, which is too long for one web request |
| Reports | Stored as structured JSON → rendered as web page + PDF export | Viewable in the app, printable/shareable |
| Hosting | Vercel (app) + Supabase (backend) | Free tiers are enough to start |

---

## 3. Data model

```
teachers        (id, email, name, created_at)                  ← Supabase auth.users
projects        (id, teacher_id, title, created_at)            ← the "folder"
submissions     (id, project_id, question_paper_paths[], status, created_at)
scripts         (id, submission_id, project_id, student_label,
                 image_paths[], status: queued|marking|done|failed, error)
reports         (id, script_id, project_id, student_label,
                 transcription, scores_json, total, feedback_md,
                 pdf_path, model, created_at)
```

- **Composition count** on the landing page = `count(scripts) where project_id = …`.
- Storage buckets (private): `question-papers/`, `scripts/`, `reports/{project_id}/`.
- Row-level security: every table filtered by `teacher_id = auth.uid()`.

---

## 4. Key design decision: one student = many photos

A composition is often 2–4 pages. On the input page, scripts are entered **per student**:

```
Student 1  [name/ID optional]  [📷 add page] [🖼 add pages]   p1 p2 p3
Student 2  …
[ + Add student ]
```

Alternatives (pick one, see Inputs §6):
- **A. Per-student grouping in the UI** (recommended, most reliable)
- B. Upload everything in order and insert a "next student" divider
- C. Let Claude detect where each script starts (convenient but error-prone)

Images are compressed client-side (max ~1600px long edge, JPEG) before upload, so they upload quickly on school Wi-Fi and stay within API limits.

---

## 5. Marking pipeline (backend)

```
Submit
  │
  ├─ 1. Create project (if "new") ─────────────── projects row
  ├─ 2. Upload question paper + script images ─── Storage
  ├─ 3. Insert submission + one scripts row per student (status = queued)
  └─ 4. Enqueue one job per script; redirect to project page
                     │
         Worker (per script, run in parallel with a concurrency limit)
                     │
  ├─ a. Load rubric prompt (versioned) + question paper images + script images
  ├─ b. Call Claude:
  │       system = rubric + marking instructions + output JSON schema
  │       user   = [question paper images] + [script page images]
  │       → returns: transcription, criterion scores, total, strengths,
  │         areas to improve, annotated errors, teacher-facing comments
  ├─ c. Validate JSON (retry once on failure)
  ├─ d. Save report row + render PDF → reports/{project_id}/{student}.pdf
  └─ e. Set script status = done (or failed + error); UI updates live
```

Notes:
- Use **prompt caching** on the rubric + question paper so each student's call is cheaper and faster.
- Use **structured output / tool-use JSON schema** so every report has the same shape.
- Keep the teacher in control: reports are marked "AI draft" and the teacher can edit scores/comments before exporting.

---

## 6. Inputs needed from you

### Must-have (I can't build the marking step without these)
1. **The marking rubric / instructions from your Cowork chats.** This session cannot see your Cowork conversations. Paste the final agreed version here (or export it as a file). It should cover:
   - criteria and band descriptors (e.g. Content, Organisation, Language/Vocabulary, Grammar, Mechanics)
   - marks per criterion and total
   - level/syllabus (e.g. PSLE / O-Level / UPSR / SPM / IGCSE…) and language
   - the exact report layout you want per student (sections, tone, length)
   - anything special: word-count penalties, off-topic rules, error-code annotations
2. **1–3 sample scripts with your own marks** (photos + your score/comments). Used to check that Claude's marking matches yours.
3. **Anthropic API key** (console.anthropic.com), stored as an environment secret, never in the repo.
4. **Supabase project** (free): URL + anon key + service-role key.

### Decisions
5. **Who can log in?** Only you, a fixed list of teachers, or a whole school? (Determines invite flow / admin page.)
6. **Script grouping**: option A, B or C from §4.
7. **Student identity**: type the name, read it from the script, or anonymous "Student 1…"?
8. **Report format**: in-app view only, or also PDF / Word download? Should one project export as a ZIP?
9. **Can teachers edit** AI marks before finalising? (Recommended: yes.)
10. **Data/privacy**: student work is personal data. Confirm where data may be hosted (region) and how long images are kept (e.g. delete photos after 30 days, keep reports).

### Nice-to-have
11. App name, logo, colours (otherwise I'll design a clean default).
12. Expected volume (teachers × scripts per month), which helps estimate API cost.

---

## 7. Build workflow (how I'd build it)

| Phase | Deliverable | Done when |
|---|---|---|
| **0. Setup** | Next.js + TS + Tailwind scaffold, Supabase project, env secrets, CI (lint/typecheck/test) | App deploys to a preview URL |
| **1. Auth** | `/login`, invite-only users, route protection, logout | Unknown emails can't get in |
| **2. Data + storage** | SQL migrations for tables above, RLS policies, private buckets | Teacher A can't see Teacher B's data (tested) |
| **3. Landing page** | Projects grid with title + composition count, **New** button, empty state | Matches §1 on phone and desktop |
| **4. Input page** | Existing/new project selector, title field, question-paper + per-student script capture, client compression, upload progress, validation | Can submit a 30-student batch from a phone |
| **5. Marking engine** | Rubric prompt (from your input), Claude call with images, JSON schema, retries, job queue, status updates | Sample scripts marked within agreed tolerance of your marks |
| **6. Reports** | Project page with live statuses, report view, teacher edit, PDF export, folder storage | Each student has a report in the project folder |
| **7. PWA polish** | Manifest, icons, service worker, install prompt, offline shell, "upload resumes when back online" | Installs on iOS/Android home screen |
| **8. Hardening** | Error states, rate limits, cost guard, data-retention job, accessibility pass, end-to-end tests (Playwright) | Full flow passes E2E test |
| **9. Pilot** | You mark one real class; compare with your own marks; tune rubric prompt | You're happy with accuracy |

Each phase goes in on its own branch/PR so you can review and test as it lands.

---

## 8. Proposed folder structure

```
compoai/
├─ app/
│  ├─ login/page.tsx
│  ├─ (app)/page.tsx                 # projects landing
│  ├─ (app)/new/page.tsx             # input page
│  ├─ (app)/projects/[id]/page.tsx   # reports list
│  ├─ (app)/reports/[id]/page.tsx    # single report
│  └─ api/submissions/route.ts
├─ lib/
│  ├─ supabase/                      # client/server helpers
│  ├─ marking/
│  │  ├─ rubric.md                   # ← your Cowork rubric goes here
│  │  ├─ prompt.ts                   # builds Claude request
│  │  ├─ schema.ts                   # report JSON schema
│  │  └─ mark-script.ts              # worker logic
│  └─ pdf/render-report.tsx
├─ supabase/migrations/*.sql
├─ public/manifest.webmanifest, icons/
└─ tests/ (unit + e2e)
```
