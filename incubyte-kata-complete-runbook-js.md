# Complete Runbook — Incubyte TDD Kata (Car Dealership Inventory System)
### Every step, folder-creation to form-submission. Check off as you go.

**Deadline: Thursday 23 July, 10:00 PM IST.** Work top to bottom — nothing later depends on something you skipped earlier being "good enough," it depends on it being **done**.

**Database: MongoDB (via Mongoose). Language: JavaScript (no TypeScript).**

---

## PHASE 0 — Before You Open a Single File (15 min)

- [ ] Re-read the assessment email and Job Description PDF once, fully, no skimming.
- [ ] Open the assessment Google Form in a browser tab now (don't wait until you're done) — glance at every field it asks for (usually: name, enrollment number, GitHub repo link, deployed link if any, maybe a text reflection). **Knowing the form's fields up front tells you exactly what you must produce.** Screenshot the empty form for your own reference.
- [ ] Create a GitHub account / confirm you're logged in, and confirm you can create a **public** repo.
- [ ] Confirm Antigravity is installed and signed in.
- [ ] Confirm you have Node.js (v18+) installed, and either:
  - MongoDB installed locally (`mongod` running on `localhost:27017`), **or**
  - A free MongoDB Atlas cluster set up (faster to get working reliably if local install is a hassle — takes ~5 min at [mongodb.com/atlas](https://www.mongodb.com/atlas)).
- [ ] Open a plain text file somewhere (Notes app, `scratch.md`) — this is where you'll paste every Antigravity prompt as you send it, live, for `PROMPTS.md` later. Don't skip this — reconstructing it later is the #1 way people run out of time.

---

## PHASE 1 — Repo & Folder Creation (10 min)

```bash
mkdir car-dealership-inventory
cd car-dealership-inventory
git init
git branch -M main
```

Create the skeleton folders now, empty, so the structure exists before any code does:

```bash
mkdir backend frontend
touch README.md PROMPTS.md .gitignore
```

`.gitignore` — create this file with:
```
node_modules/
dist/
build/
.env
.env.local
*.log
.DS_Store
coverage/
```

```bash
git add .gitignore README.md PROMPTS.md
git commit -m "chore: initialize repository structure"
```

- [ ] Go to GitHub → New Repository → name it `car-dealership-inventory` (or similar) → **Public** → do **not** initialize with a README (you already have one) → Create.
- [ ] Link and push:
```bash
git remote add origin https://github.com/<your-username>/car-dealership-inventory.git
git push -u origin main
```
- [ ] Confirm on GitHub.com that the empty repo with your first commit is actually there.

---

## PHASE 2 — Backend Scaffold (30–45 min)

Open the `car-dealership-inventory` folder in **Antigravity**.

### 2.1 Manual scaffold (do this yourself first — faster and more reliable than prompting for boilerplate)

```bash
cd backend
npm init -y
npm install express cors dotenv bcrypt jsonwebtoken zod mongoose
npm install -D nodemon jest supertest mongodb-memory-server
```

Add `"type": "module"` to `backend/package.json` so you can use `import`/`export` syntax in plain JavaScript (cleaner than `require` for this kind of app — optional, but recommended; if you'd rather use CommonJS `require`/`module.exports` throughout, that's fine too, just be consistent).

- [ ] Create `.env` in `backend/` with:
```
MONGODB_URI=mongodb://localhost:27017/dealership
JWT_SECRET=<any-long-random-string>
PORT=4000
```
  (If using Atlas instead of local Mongo, replace `MONGODB_URI` with the connection string Atlas gives you, e.g. `mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/dealership`.)

- [ ] Create `.env.example` with the same keys, no real secrets — commit this one (never commit `.env` itself).

> **Note on "no in-memory database":** the brief requires a real, persistent database — MongoDB running locally or on Atlas satisfies this. `mongodb-memory-server` above is a **test-only** dependency (spins up a throwaway Mongo instance purely for Jest runs so your test suite doesn't touch real data) — it is not what the running application uses, so it doesn't violate the "no in-memory database" rule. Keep that distinction straight if it comes up in the interview.

### 2.2 Folder structure inside `backend/src`

```bash
mkdir -p src/models src/routes src/controllers src/middleware src/lib
```

### 2.3 Mongoose models

`src/models/User.js`:
```js
import { Schema, model } from 'mongoose';

const userSchema = new Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['USER', 'ADMIN'], default: 'USER' },
});

export const User = model('User', userSchema);
```

`src/models/Vehicle.js`:
```js
import { Schema, model } from 'mongoose';

const vehicleSchema = new Schema({
  make: { type: String, required: true },
  model: { type: String, required: true },
  category: { type: String, required: true },
  price: { type: Number, required: true },
  quantity: { type: Number, required: true, default: 0 },
});

export const Vehicle = model('Vehicle', vehicleSchema);
```

### 2.4 DB connection

`src/lib/db.js`:
```js
import mongoose from 'mongoose';

export async function connectDB() {
  const uri = process.env.MONGODB_URI;
  await mongoose.connect(uri);
  console.log('MongoDB connected');
}
```

### 2.5 `package.json` scripts — add these

```json
"scripts": {
  "dev": "nodemon src/app.js",
  "test": "node --experimental-vm-modules node_modules/.bin/jest --coverage",
  "start": "node src/app.js"
}
```
(The `--experimental-vm-modules` flag is only needed if you keep `"type": "module"` + Jest with ES module `import` syntax. If you switch to CommonJS `require`, drop that flag and just use `"test": "jest --coverage"`.)

### 2.6 Minimal boot file

`src/app.js`:
```js
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import { connectDB } from './lib/db.js';
dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

app.get('/health', (_req, res) => res.json({ status: 'ok' }));

const PORT = process.env.PORT || 4000;

connectDB().then(() => {
  app.listen(PORT, () => console.log(`Server running on ${PORT}`));
});

export default app;
```

- [ ] `npm run dev` — confirm you see `MongoDB connected` and `Server running on 4000` with no errors.
- [ ] Hit `http://localhost:4000/health`, confirm `{"status":"ok"}`.
- [ ] `git add . && git commit -m "chore: scaffold backend, mongoose models, db connection, health check"`
- [ ] `git push`

---

## PHASE 3 — Backend TDD, Endpoint by Endpoint (4–6 hrs)

**The loop, every single time, no shortcuts:**
1. Write/ask Antigravity for a **failing test** for one behavior (use `mongodb-memory-server` in your Jest setup so tests run against a real, throwaway Mongo instance, not mocks).
2. Run `npm test` — confirm it fails (Red). If it doesn't fail, your test is wrong — fix the test first.
3. Ask Antigravity (or write yourself) the **minimum code** to pass.
4. Run `npm test` — confirm it passes (Green).
5. Refactor if needed, re-run tests.
6. **Paste the exact prompt you used into your scratch PROMPTS file, right now**, with 1 line on the outcome.
7. `git add . && git commit -m "..."` with the AI co-author trailer (template below) if AI wrote/materially shaped this step.
8. `git push`.

**Commit template (copy-paste and fill in):**
```
git commit -m "feat: <what this commit does>

<1-3 sentences: what test you wrote, what failed, what you asked
the AI for, what you changed by hand>

Co-authored-by: Antigravity <antigravity@users.noreply.github.com>"
```

### Checklist — go in this order, do NOT skip ahead:

- [ ] **Test:** `POST /api/auth/register` — 201 + user id on valid input
- [ ] **Test:** `POST /api/auth/register` — 400 on duplicate email
- [ ] **Test:** `POST /api/auth/register` — 400 on weak/missing password
- [ ] **Implement** register endpoint (hash password with bcrypt, save via Mongoose) → tests green
- [ ] **Test:** `POST /api/auth/login` — 200 + JWT on correct credentials
- [ ] **Test:** `POST /api/auth/login` — 401 on wrong password / unknown user
- [ ] **Implement** login endpoint → tests green
- [ ] **Test:** auth middleware rejects requests with missing/invalid/expired token (401)
- [ ] **Implement** `requireAuth` middleware → test green
- [ ] **Test:** admin middleware rejects non-admin users (403) on an admin-only route
- [ ] **Implement** `requireAdmin` middleware → test green
- [ ] **Test:** `POST /api/vehicles` — 201 on valid protected request; 401 unauthenticated; 400 on missing fields
- [ ] **Implement** create vehicle → tests green
- [ ] **Test:** `GET /api/vehicles` — 200, returns list
- [ ] **Implement** list vehicles → test green
- [ ] **Test:** `GET /api/vehicles/search` — filter by `make` alone
- [ ] **Test:** `GET /api/vehicles/search` — filter by `model` alone
- [ ] **Test:** `GET /api/vehicles/search` — filter by `category` alone
- [ ] **Test:** `GET /api/vehicles/search` — filter by `minPrice`/`maxPrice`
- [ ] **Test:** `GET /api/vehicles/search` — combined filters
- [ ] **Implement** search endpoint incrementally until all above are green (build the Mongoose filter object dynamically from query params)
- [ ] **Test:** `PUT /api/vehicles/:id` — 200 on valid update; 404 on unknown id; 401 unauthenticated
- [ ] **Implement** update vehicle → tests green
- [ ] **Test:** `DELETE /api/vehicles/:id` — 200 for admin; 403 for non-admin; 404 unknown id
- [ ] **Implement** delete vehicle → tests green
- [ ] **Test:** `POST /api/vehicles/:id/purchase` — 200, quantity decrements by 1
- [ ] **Test:** `POST /api/vehicles/:id/purchase` — 400 when quantity is already 0
- [ ] **Test:** `POST /api/vehicles/:id/purchase` — 404 unknown id
- [ ] **Implement** purchase endpoint → tests green
- [ ] **Test:** `POST /api/vehicles/:id/restock` — 200 for admin, quantity increments
- [ ] **Test:** `POST /api/vehicles/:id/restock` — 403 for non-admin
- [ ] **Implement** restock endpoint → tests green

- [ ] Run `npm test` one final time — everything green.
- [ ] Run `npm test -- --coverage`, screenshot or copy the summary table (you need this for the README later).
- [ ] `git push`

---

## PHASE 4 — Frontend Scaffold (20–30 min)

```bash
cd ../frontend
npm create vite@latest . -- --template react
npm install
npm install -D tailwindcss postcss autoprefixer vitest @testing-library/react @testing-library/jest-dom jsdom
npx tailwindcss init -p
```

Note the template is **`react`**, not `react-ts` — plain JavaScript, `.jsx` files.

- [ ] Configure `tailwind.config.js` content paths to `./index.html` and `./src/**/*.{js,jsx}`.
- [ ] Add the three Tailwind directives (`@tailwind base; @tailwind components; @tailwind utilities;`) to `src/index.css`.
- [ ] Add a `test` script to `package.json`: `"test": "vitest run"`.
- [ ] `npm run dev` — confirm the default Vite page loads with Tailwind working (add a `bg-blue-500` test div, confirm it's blue, then remove it).
- [ ] `git add . && git commit -m "chore: scaffold Vite+React+Tailwind frontend, configure Vitest"`
- [ ] `git push`

---

## PHASE 5 — Frontend TDD, Piece by Piece (4–6 hrs)

Same Red-Green-Refactor loop and commit template as Phase 3. Keep logging prompts. All files are `.jsx`/`.js`, no type annotations.

- [ ] **Test:** Register form — renders fields, shows validation error on empty submit
- [ ] **Implement** register form → test green
- [ ] **Test:** Login form — same pattern
- [ ] **Implement** login form → test green
- [ ] **Implement** Auth context — stores JWT (in memory / sessionStorage), exposes `login`, `logout`, `user`
- [ ] **Test:** protected route redirects unauthenticated users to `/login`
- [ ] **Implement** protected route wrapper → test green
- [ ] **Test:** Dashboard renders a list of vehicles given mock API data
- [ ] **Implement** dashboard/vehicle list → test green
- [ ] **Test:** Search/filter bar calls the API with correct query params on submit
- [ ] **Implement** search/filter bar → test green
- [ ] **Test:** Purchase button is disabled when `quantity === 0`, enabled otherwise
- [ ] **Test:** Clicking purchase calls the API and updates displayed quantity on success
- [ ] **Implement** purchase button behavior → tests green
- [ ] **Test:** Admin-only add/edit/delete forms only render for `role === 'ADMIN'`
- [ ] **Implement** admin CRUD UI → tests green
- [ ] Style pass: consistent spacing/color scale across all pages (Tailwind) — no default-looking, unstyled screens.
- [ ] Run `npm test` — all frontend tests green.
- [ ] `git push`

---

## PHASE 6 — End-to-End Verification (1 hr)

Do this like a real QA pass, manually, both servers running:

- [ ] Register a new user → succeeds, redirects appropriately.
- [ ] Log out, log back in with same credentials → succeeds.
- [ ] Try logging in with wrong password → clear error shown, no crash.
- [ ] Dashboard loads and shows vehicles (seed a few directly in MongoDB via `mongosh` or MongoDB Compass if the list is empty).
- [ ] Search/filter by make, model, category, price range — each works, and combined.
- [ ] Purchase a vehicle with quantity > 0 → quantity decreases in UI.
- [ ] Try purchasing a vehicle at quantity 0 → button is disabled, no request fires.
- [ ] Log in as an admin (manually flip a user's `role` to `ADMIN` in MongoDB Compass/`mongosh` if you don't have a promote-to-admin flow) → add, edit, delete a vehicle from the UI, confirm all three work.
- [ ] Log in as a non-admin → confirm admin controls are hidden/blocked.
- [ ] Open the browser dev console throughout — zero uncaught errors. (Let Antigravity's browser agent drive this pass and flag anything it catches.)
- [ ] `git push` any fixes from this pass, each as its own small commit.

---

## PHASE 7 — README.md (45 min)

Fill in every section below — don't leave placeholders:

```markdown
# Car Dealership Inventory System

## Overview
[2-4 sentences: what the app does, stack used]

## Tech Stack
- Backend: Node.js, Express, Mongoose, MongoDB (JavaScript)
- Auth: JWT, bcrypt
- Frontend: React, Vite, Tailwind CSS (JavaScript)
- Testing: Jest + Supertest + mongodb-memory-server (backend),
  Vitest + React Testing Library (frontend)

## Setup & Run Locally

### Backend
1. cd backend
2. npm install
3. Create a `.env` file (see `.env.example`) with MONGODB_URI and JWT_SECRET
   - Local Mongo: `mongodb://localhost:27017/dealership`
   - Or use a free MongoDB Atlas connection string
4. npm run dev   (runs on http://localhost:4000)

### Frontend
1. cd frontend
2. npm install
3. npm run dev   (runs on http://localhost:5173)

## API Reference
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | /api/auth/register | none | Register a new user |
| POST | /api/auth/login | none | Log in, returns JWT |
| GET | /api/vehicles | required | List all vehicles |
| GET | /api/vehicles/search | required | Filter by make/model/category/price |
| POST | /api/vehicles | required | Add a vehicle |
| PUT | /api/vehicles/:id | required | Update a vehicle |
| DELETE | /api/vehicles/:id | admin | Delete a vehicle |
| POST | /api/vehicles/:id/purchase | required | Purchase (decrement qty) |
| POST | /api/vehicles/:id/restock | admin | Restock (increment qty) |

## Screenshots
[Insert: login page, dashboard, search/filter in use, purchase flow
before/after, admin add/edit/delete panel]

## Test Report
[Paste backend `npm test -- --coverage` summary table]
[Paste frontend `npm test` output]

## My AI Usage
**Tools used:** Google Antigravity (Gemini 3)

**How I used it:**
[Be specific and honest, e.g.:
- "Used Antigravity to scaffold the Express + Mongoose project structure
  and generate initial failing tests for each endpoint before writing
  implementations myself."
- "Used the browser-in-the-loop agent to catch a bug where the purchase
  button didn't re-render after a successful purchase."
- "Wrote the purchase/restock quantity-guard logic and the admin
  middleware by hand after AI's first draft missed the zero-quantity
  edge case."]

**Reflection:**
[1-2 honest paragraphs: where it sped you up, where you had to correct
or override it, what you'd do differently.]

## Live Demo
[link, or "Not deployed" if you skip the optional step]
```

- [ ] `.env.example` file created (same keys as `.env`, no real secrets) and committed.
- [ ] README committed and pushed.

---

## PHASE 8 — PROMPTS.md (20 min if you logged as you went; hours if you didn't — don't skip Phase 0's scratch file)

Format, chronological, one entry per meaningful Antigravity interaction:

```markdown
# AI Prompt History

## Task: Scaffold backend
**Prompt:** "..."
**Outcome:** ...

## Task: Auth register endpoint — failing test
**Prompt:** "..."
**Outcome:** ...

## Task: Auth register endpoint — implementation
**Prompt:** "..."
**Outcome:** ...

[... continue for every task, in the order you actually did them]
```

- [ ] Every prompt you actually sent is in here, in order, nothing invented, nothing missing.
- [ ] Committed at repo root (not inside `backend/` or `frontend/`).
- [ ] Pushed.

---

## PHASE 9 — Optional: Deploy (skip if time is short — this is brownie points, not required)

- [ ] Backend → Render/Railway/Fly.io, with `MONGODB_URI` pointed at a **MongoDB Atlas** cluster (free tier is enough — a hosted app can't reach your `localhost` database, so Atlas is required here even if you developed against local Mongo)
- [ ] Frontend → Vercel/Netlify, pointing its API calls at the deployed backend URL via an env var
- [ ] Confirm the live link actually works end-to-end (register → login → purchase) before putting it in the README
- [ ] Add the link to README's "Live Demo" section, commit, push

---

## PHASE 10 — Final Pre-Submission Audit (30 min, do NOT skip)

Go through this as if you were the reviewer, cold:

- [ ] Clone your repo fresh into a throwaway folder and follow your own README's setup steps exactly, word for word. If it doesn't work from scratch, fix it now.
- [ ] `git log --oneline` — does it read like a real Red-Green-Refactor journey, not one giant dump?
- [ ] Spot-check 3 commits with `Co-authored-by:` — can you explain out loud, right now, what the AI did vs. what you changed in each?
- [ ] README has: overview, setup instructions (verified working), screenshots, API reference, test report, **My AI Usage** section, deliverables all present.
- [ ] `PROMPTS.md` is in the repo root and complete.
- [ ] Repo is **public** — check in an incognito window that you can view it while logged out.
- [ ] No `.env` or secrets committed (`git log --all --full-history -- backend/.env` should show nothing).
- [ ] No code copy-pasted from another repo/tutorial you can't personally explain.

---

## PHASE 11 — Fill Out and Submit the Google Form (10 min)

- [ ] Open the assessment Google Form link from the email.
- [ ] Enter your name and **Enrollment Number exactly as issued**: `230280116064` (double-check for typos — mismatches can cause shortlisting issues).
- [ ] Paste your **public GitHub repo URL** — click it yourself in a fresh tab first to confirm it's not a typo'd/private link.
- [ ] If the form has a deployed-app link field and you did Phase 9, paste that link too (test it once more first). If not deployed, leave it blank or follow the form's own instructions for that field.
- [ ] If the form asks for a short reflection/AI-usage text field, keep it consistent with your README's "My AI Usage" section — don't contradict yourself between the two.
- [ ] Review every field once before hitting submit.
- [ ] **Submit before Thursday, 23 July, 10:00 PM IST** — don't submit at 9:58 PM; leave buffer for form-loading issues, slow uploads, or a bad connection.
- [ ] Screenshot the form's "response recorded" confirmation page, and/or the confirmation email if one is sent. Keep this — it's your proof of on-time submission if anything about receipt is ever disputed.

---

## Done

If every checkbox above is checked, you have satisfied every explicit requirement in the assessment brief: RESTful API with real DB (MongoDB) and JWT auth, all specified endpoints, a React+Tailwind SPA with all specified functionality, a visible TDD commit history, AI co-authorship where genuine, a complete README with the mandatory AI Usage section and test report, a complete PROMPTS.md, and a submitted form before the deadline.
