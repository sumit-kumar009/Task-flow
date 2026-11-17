# TaskFlow

A mobile-first task planner built with the MERN stack. You plan tasks for a specific date and time, and TaskFlow groups them into Monday–Sunday weeks so you can see at a glance how much is done and how much is still open.

- **Live app:** https://taskflow-sumit.netlify.app
- **API:** https://taskflow-api-vxry.onrender.com/api (health check: [`/api/health`](https://taskflow-api-vxry.onrender.com/api/health))

> The API runs on Render's free tier, so the first request after a period of inactivity can take up to a minute while the server wakes up.

## Why I built it

Most to-do apps show one long list, which makes it hard to see how a week is actually going. I wanted something closer to a weekly planner: pick a day, see that day's tasks, and get a simple "done vs. pending" picture for the whole week. It was also a chance to build a complete full-stack app end to end – a REST API with validation and tests, a database design, a responsive React UI with touch gestures, and a real deployment.

## Features

- **Create, edit and delete tasks** – title (required), description, date, start/end time and priority (Low / Medium / High).
- **Mark tasks as Completed or In Progress** with the checkbox – the week's counts and progress bar update straight away.
- **Home screen** – week day picker (with previous / next week), Task Complete / Task Pending cards, weekly progress bar and the tasks of the selected day.
- **All Weeks** (`View All`) – one card per week showing open and completed tasks; tap a card to expand it and see its tasks.
- **Swipe left** on a task to reveal **Delete** (or use the trash icon, which asks for confirmation).
- **Search** tasks by keywords in the title or description, and view / edit / delete them from the results.
- Tap a task to see its details (description, time, status).

## Tech stack

| Part     | Tech                                             |
| -------- | ------------------------------------------------ |
| Frontend | React 19, Vite, Tailwind CSS v4, React Router    |
| Backend  | Node.js, Express 5, Mongoose                     |
| Database | MongoDB (Atlas)                                  |
| Tests    | Jest + Supertest                                 |

## How it works

```
React app (Netlify)  ──HTTP/JSON──▶  Express API (Render)  ──Mongoose──▶  MongoDB Atlas
```

1. The React app calls the REST API through a small API client (`client/src/api/tasks.js`).
2. Express validates every request (`server/src/utils/validate.js`) and returns clear, per-field errors.
3. Mongoose stores tasks in MongoDB. An index on `dueDate` keeps date-range queries fast, and search is a case-insensitive match on the title and description (user input is escaped before it is turned into a regex).
4. For the weekly view, MongoDB groups tasks into Monday–Sunday weeks itself (aggregation with `$dateTrunc`), using the user's time zone sent from the browser.

### Design decisions

- **Weeks are calculated on the server, in the user's time zone.** A task at 2 AM on Monday in India is still Sunday night in UTC, so grouping in UTC would put it in the previous week. Grouping in the database with the browser's time zone keeps the counts correct for everyone.
- **Status has its own endpoint** (`PATCH /tasks/:id/status`) so ticking a checkbox is a small, safe update that can't accidentally change other fields.
- **Validation lives on the server** (the client also checks the form, but the API never trusts the client).
- **Mobile-first UI**: a bottom sheet for adding/editing, swipe-left to delete, and a confirm dialog for the trash icon so nothing is deleted by accident.

## Project structure

```
client/                 React app
  src/api/              API client
  src/components/       TaskItem, TaskSheet, WeekStrip, SummaryCards …
  src/context/          App-wide state (sheet, confirm dialog, toasts)
  src/pages/            Onboarding, Home, Search, Weeks
  src/utils/            Date helpers
server/                 Express API
  src/models/Task.js    Mongoose schema
  src/controllers/      Route handlers
  src/utils/validate.js Request validation
  tests/                API tests
```

## Running locally

Requirements: Node.js 18+ and a MongoDB connection string (local MongoDB or a free Atlas cluster).

**1. API**

```bash
cd server
cp .env.example .env    # then set MONGODB_URI
npm install
npm run dev             # http://localhost:5000
```

**2. Frontend** (in a second terminal)

```bash
cd client
npm install
npm run dev             # http://localhost:5173
```

In development the Vite dev server forwards `/api` requests to `localhost:5000`, so no extra config is needed.

**Tests**

```bash
cd server
npm test
```

## API

Base URL: `/api`

| Method | Endpoint                 | Description                                                      |
| ------ | ------------------------ | ---------------------------------------------------------------- |
| GET    | `/tasks`                 | List tasks. Query: `search`, `from`, `to` (ISO dates), `status`  |
| GET    | `/tasks/weeks`           | Open / completed counts per Monday–Sunday week. Query: `tz`      |
| GET    | `/tasks/:id`             | Get one task                                                     |
| POST   | `/tasks`                 | Create a task                                                    |
| PUT    | `/tasks/:id`             | Update title, description, dueDate, endDate or priority          |
| PATCH  | `/tasks/:id/status`      | Set status to `In Progress` or `Completed`                       |
| DELETE | `/tasks/:id`             | Delete a task                                                    |

Example task:

```json
{
  "id": "6736a1f0c2a4b1e5d8f0a123",
  "title": "Doing Homework",
  "description": "Maths chapter 4",
  "dueDate": "2026-11-14T11:00:00.000Z",
  "endDate": "2026-11-14T11:45:00.000Z",
  "priority": "High",
  "status": "In Progress",
  "createdAt": "2026-11-14T08:12:00.000Z",
  "updatedAt": "2026-11-14T08:12:00.000Z"
}
```

Validation errors return `400` with the problem fields:

```json
{ "message": "Validation failed", "errors": { "title": "Title is required" } }
```

### Weeks and time zones

Weeks start on Monday. Because "which week is this task in" depends on where the user is, the client sends its time zone (`tz`, e.g. `Asia/Kolkata`) and MongoDB groups tasks with `$dateTrunc` in that time zone.

## Deployment

- **API → Render** (`render.yaml` is included, so it can be created as a Blueprint; or manually as a web service with root directory `server`, build `npm install`, start `npm start`). Environment variables: `MONGODB_URI`, `CLIENT_URL` (the Netlify URL, used for CORS).
- **Frontend → Netlify** (base directory `client`, build `npm run build`, publish `client/dist`). Environment variable: `VITE_API_URL` = `https://<render-app>.onrender.com/api`. `public/_redirects` makes page refreshes work on client-side routes.
