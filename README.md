# 🏃 Endurance Intelligence — Project README (Interview Study Guide)

> A Strava-connected web app that reads your real running history and generates a
> **science-backed, week-by-week training plan** for a target race, plus a
> **fitness-loss forecast** that predicts how much fitness you lose when you take rest days.
>
> Built as an **MSc Big Data Analytics** student project (SJU / St Joseph's University, Bangalore).

This README is written so that **reading it once refreshes the whole project in your head** before an interview. It explains *what* was built, *how* it works, *why* each choice was made, and ends with a **Q&A cheat sheet** of the questions you're most likely to be asked.

---

## 1. The 30-Second Elevator Pitch

> "Endurance Intelligence is a full-stack web app. A runner connects their Strava account with one click, and the app pulls their last 60 runs. From that real data it auto-detects their fitness level, then generates a personalised training plan for a race they pick — with the right weekly mileage, long runs, and pace zones for every session. It follows real coaching science: the 10% weekly-growth rule, 4-week build/recover cycles, an 80/20 easy-hard split, and a taper before race day. It also has a second model that forecasts how much fitness you lose during rest days using an exponential decay curve calibrated on a running dataset. The frontend is React, the backend is serverless functions on Vercel, and tokens are stored in a Supabase Postgres database."

If you remember only that paragraph, you can carry a conversation. The rest of this document fills in the details.

---

## 2. What Problem Does It Solve?

Generic training plans (from magazines or PDFs) ignore *who you actually are*. A beginner and an elite runner get the same plan. This project's idea:

1. **Use the runner's real data** (from Strava) instead of a generic template.
2. **Apply established coaching science** (Daniels, Pfitzinger, Seiler, Mujika) so the plan is safe and effective.
3. **Personalise everything** — mileage, long-run length, pace targets, and difficulty ramp — to the individual.

Two features deliver this:

| Feature | What it answers |
|---|---|
| **① Goal-Aware Training Plan** | "Give me a week-by-week plan to run *this race* in *this time*, based on where my fitness is *now*." |
| **② Fitness Loss Forecast** | "If I stop training for X days, how much fitness do I lose, and how should I return?" |

---

## 3. Tech Stack (and why each piece)

| Layer | Technology | Why it was chosen |
|---|---|---|
| **Frontend** | React 18 + Vite | Component-based UI; Vite gives instant dev reloads and fast builds |
| **Charts** | Recharts | Simple React charting for the mileage curve + fitness-decay curve |
| **Backend** | Vercel Serverless Functions (Node.js) | No server to manage; each API file is a function that runs on demand |
| **Database** | Supabase (managed PostgreSQL) | Stores Strava tokens; free tier; simple client library |
| **Auth** | Strava OAuth 2.0 | Lets users log in with Strava and grant read access to their runs |
| **Hosting** | Vercel | Hosts the static React app *and* the serverless API together; free tier |

**Cost: ₹0** — everything runs on free tiers (Vercel hobby, Supabase free, Strava API free).

**Key phrase for interviews:** *"It's a serverless full-stack app — the frontend is a static React bundle, and the backend is just two stateless functions."*

---

## 4. Architecture & Data Flow (the most important diagram)

```
 1. User clicks "Connect with Strava"
          │
          ▼
 2. Strava login/consent page  (strava.com)
          │   user approves
          ▼
 3. Strava redirects to →  /api/strava-callback   ← serverless function #1
          │   (Strava sends a temporary "code")
          ▼
 4. Function exchanges the code for:
       access_token + refresh_token + athlete profile
          │
          ▼
 5. Tokens + profile saved to Supabase  (athletes table)
          │
          ▼
 6. Redirect back to the app with  ?athlete_id=12345
          │
          ▼
 7. React app reads athlete_id, calls →  /api/activities?athlete_id=12345  ← serverless function #2
          │
          ▼
 8. Function looks up the athlete, refreshes the token if it's about to expire,
    fetches the last 60 activities from Strava, keeps only the runs
          │
          ▼
 9. React receives the runs → computes stats → auto-classifies level →
    pre-fills the form → builds and displays the training plan + charts
```

**Why OAuth happens on the server, not in the browser:** the `client_secret` must never be exposed to users. Exchanging the code and refreshing tokens are done inside the serverless functions where the secret is a private environment variable.

---

## 5. File-by-File Walkthrough

```
Strava-Integrated-Endurance-intelligence/
├── api/
│   ├── strava-callback.js   ← OAuth: swap code → tokens, save to DB   (serverless)
│   └── activities.js        ← Fetch runs from Strava, auto-refresh tokens (serverless)
├── src/
│   ├── main.jsx             ← React entry point (mounts <App/>)
│   ├── App.jsx              ← THE WHOLE APP: UI + all the algorithms (~1,250 lines)
│   └── supabase.js          ← Creates the Supabase client for the browser
├── index.html               ← HTML shell React mounts into
├── vite.config.js           ← Vite config + local proxy (/api → localhost:3001)
├── vercel.json              ← Routing rules for Vercel (API routes + SPA fallback)
├── package.json             ← Dependencies and scripts
├── .env.example             ← Template of the secrets you need to provide
└── SETUP.md                 ← Step-by-step deployment guide
```

### `api/strava-callback.js` — the login handler
- Receives the temporary `code` from Strava.
- POSTs to Strava's token endpoint with `client_id` + `client_secret` + `code` to get an `access_token`, a `refresh_token`, and the athlete's profile.
- **Upserts** the athlete into Supabase (`onConflict: 'strava_id'`) — so logging in again *updates* the row instead of creating a duplicate.
- Redirects the browser back to the app with `?athlete_id=…`.
- Handles errors by redirecting with an `?error=…` flag.

### `api/activities.js` — the data fetcher
- Takes `athlete_id`, looks up that athlete's stored tokens in Supabase.
- **`refreshIfNeeded()`** — Strava access tokens expire every ~6 hours. If the token expires within the next 5 minutes, it uses the `refresh_token` to get a fresh one and saves it back to the DB. This is why the app keeps working days later without re-login.
- Fetches the last 60 activities, keeps only `type === 'Run'` with distance > 0, and returns a clean array: `{ date, distance, elapsed, elevation, hr, pace }`.
- `pace` is derived server-side as `elapsed_time / (distance_in_km)` → seconds per km.

### `src/App.jsx` — everything else
This one file holds **all the UI and all the algorithms**. It has three logical parts:
1. **Constants** — theme, the four fitness levels, race distances, pace-zone colours, demo profile.
2. **Pure functions** — the "intelligence": stats, level classification, ML prediction, pace zones, the plan builder, the decay model, PDF export.
3. **The `App` component** — React state, the data-fetch effect, and the rendered dashboard (two big sections + a methodology panel + charts).

### `src/supabase.js`
Creates a browser-side Supabase client using the **public anon key** (safe to expose). Note: the *service key* used by the serverless functions is different and secret — it can bypass row-level security, so it lives only on the server.

---

## 6. The Database (Supabase / Postgres)

One table, `athletes`:

```sql
CREATE TABLE athletes (
  id            BIGSERIAL PRIMARY KEY,
  strava_id     BIGINT UNIQUE NOT NULL,   -- Strava's athlete ID (the lookup key)
  firstname     TEXT,
  lastname      TEXT,
  profile_pic   TEXT,
  access_token  TEXT NOT NULL,            -- short-lived, used to call Strava
  refresh_token TEXT NOT NULL,            -- long-lived, used to get new access tokens
  token_expires BIGINT NOT NULL,          -- unix timestamp when access_token dies
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_athletes_strava_id ON athletes(strava_id);
```

**Why store tokens?** So the app can fetch fresh data later without asking the user to log in every time. The index makes lookups by `strava_id` fast.

---

## 7. The "Intelligence" — Every Algorithm Explained Simply

This is the heart of the project and the part interviewers dig into. Take your time here.

### 7.1 Auto Level Classification — `classifyLevel(stats)`
Looks at the runner's **average pace** and **weekly kilometres** and buckets them into one of four levels. Rough logic:

| Level | Roughly means | Trigger |
|---|---|---|
| 🌱 Beginner | 0–1 yr | slower than the thresholds below |
| ⚡ Intermediate | 1–3 yrs | faster than 6:30/km **and** >20 km/week |
| 🔥 Advanced | 3–5 yrs | faster than 5:30/km **and** >40 km/week |
| 🏆 Elite | 5+ yrs | faster than 4:30/km **and** >70 km/week |

The level then controls things like decay rate and peak-mileage caps. Users can still override it with a dropdown.

### 7.2 Computing Stats — `computeStatsFromRuns(runs)`
Turns the raw run list into: average pace (min/km), average heart rate, total km, longest run, and estimated weekly km (`total ÷ (number of runs ÷ 4)` — a rough 4-week window). These pre-fill the form so the user starts from *their* real numbers.

### 7.3 The ML Prediction — `mlPredictTraining(...)`
Given the **goal time**, **race distance**, current **weekly km**, and **longest run**, it outputs three targets:
- **`requiredPace`** — the goal race pace (goal time ÷ race distance).
- **`easyPace`** — how slow your easy runs should be (goal pace × a factor between 1.14 and 1.28, bigger for longer races).
- **`weeklyLoad`** — the peak weekly mileage to build toward (bigger of "race distance × a multiplier" and "current mileage × 1.25").
- **`longRunKm`** — the long-run distance (a fraction of peak weekly volume, capped at the race distance).

> ⚠️ **Honesty note for the interview:** the code *frames* this as a "Random Forest trained on 42,116 runs," and the offline data analysis did inform the numbers. But **at runtime this function is a set of tuned formulas**, not a serialized model loaded into the browser. Be ready to say: *"The heavy data analysis happened offline in Python — it shaped the coefficients and thresholds. What ships in the app is a lightweight formula version of those findings so it runs instantly in the browser with no backend inference."* That's accurate and defensible; claiming a live RF model is running in React is not.

### 7.4 Pace Zones — `getPaceZones(goalPace)`  (Daniels VDOT method)
Every workout pace is derived from your **goal race pace**, following Jack Daniels' running formula:

| Session | Relative to goal pace | Feel |
|---|---|---|
| **Recovery** | +1.5 to +2.0 min/km | very easy |
| **Easy** | +1.0 to +1.5 min/km | conversational |
| **Long** | +0.9 to +1.3 min/km | steady |
| **Tempo** | +0.3 to +0.6 min/km | comfortably hard (lactate threshold) |
| **Intervals** | −0.3 to +0.0 min/km | 5K effort (VO2max) |

### 7.5 The Plan Builder — `buildPlan(...)`  (the biggest algorithm)
This generates the full week-by-week plan. It combines several real coaching principles:

**a) Volume progression (the 10% rule + mesocycles)**
- Runs on a **4-week cycle**: weeks 1–3 *build* (+10% each), week 4 *recovers* (−20%).
- Crucial detail: the +10% is always taken off the **last build week**, not the recovery week (so recovery weeks don't stall the ramp).
- **Back-calculation trick:** the app knows the *peak* mileage the race needs (`RACE_PEAK_KM[race][level]`). It works *backwards* from that peak to figure out where the plan should **start**, so it arrives at peak exactly on time — but never starts below what the runner is already doing.

**b) Taper**
- Last 2 weeks cut volume ~30% each week so the runner arrives at the race fresh.

**c) Long run (Pfitzinger)**
- ~30% of that week's volume, hard-capped at `min(90% of race distance, 38 km)`.

**d) Session distribution (Seiler's 80/20 rule)**
- ~80% of running is easy, ~20% is hard (tempo + intervals). Implemented with per-session *weights* that split the weekly mileage across the days.

**e) Race-week overrides**
- Race day → Rest; day before → short Warmup; final days tapered further.

Output: an array of weeks, each with a date range, total km, long-run km, and 7 days — each day has a session type, distance, and a pace range.

### 7.6 Fitness Loss Forecast — the detraining model
When you stop training, aerobic fitness fades. The model uses **exponential decay**:

```
Fitness(t) = 100 × e^(−k × t)
```

- **`k`** (the decay constant) depends on your **level** — Elite decays slowest because of a bigger aerobic base. Values are described as calibrated from a running dataset blended with **Mujika & Padilla (2000)** detraining research.
- **Training history reduces decay:** more `trainingDaysDone` → slower decay (up to a 40% reduction, plateauing around 6 months).
- It outputs: fitness retained %, estimated loss %, recovery days (~half the days off), a slower **return pace**, and a reduced **return volume** so you don't jump back too hard.
- Shown as an area chart of fitness % over 60 idle days, with a marker at your chosen rest length.

### 7.7 PDF Export — `exportPlanToPDF(...)`
Opens a new browser window with a print-friendly HTML version of the plan (each day on its own line, with paces) so the user can **Save as PDF / print**. No PDF library needed — it uses the browser's print engine.

---

## 8. Frontend State & Screens (how the UI flows)

The `App` component has three **screens** driven by one `screen` state variable:
- `"connect"` → landing page with the Strava button + a **Demo mode** button (loads a fake "Intermediate runner" so you can show the app without a Strava login — great for demos and interviews!).
- `"loading"` → spinner while runs are fetched.
- `"dashboard"` → the main app: Section 01 (training plan), Section 02 (fitness forecast), and an expandable **Methodology panel** (dataset size, models, validation metrics — added for academic review).

Key state: race, current weekly km, longest run, pace, goal time, rest days, start/race dates, plus forecast inputs. Almost everything on screen is **derived** from this state, so changing any input instantly rebuilds the plan (React re-render).

Nice touches worth mentioning: responsive layout (`narrow` state on resize), goal-time validation (rejects impossible paces faster than 2:50/km or slower than 14:00/km), and a live race countdown.

---

## 9. Data Science Background (the "big data" part)

The plan is grounded in a running dataset and published sports science. Numbers cited in the app's Methodology panel:
- **Dataset:** a Kaggle running dataset — 116 athletes, ~42,117 raw rows cleaned to ~40,833, from which **31,656 consecutive run-gap pairs** were extracted to study detraining.
- **Detraining research:** Mujika & Padilla (2000).
- **Coaching frameworks:** Jack Daniels (VDOT pace zones), Pfitzinger (periodization & long runs), Stephen Seiler (80/20 polarised training), Hal Higdon / RRCA guidelines for cross-checking.
- Reported decay model error ≈ 25.85% MAE (high variance is expected — individuals detrain very differently).

**Framing tip:** the "big data" work (cleaning ~40k rows, extracting 31k gap pairs, fitting decay curves, checking feature importance) was the **offline analysis phase**. The web app is the **product** that operationalises those findings into a live tool.

---

## 10. How to Run / Deploy (quick recap)

**Local:**
```bash
cp .env.example .env      # fill in Strava + Supabase keys
npx vercel dev            # runs frontend + serverless functions together
```

**Deploy:**
1. Create a Strava API app → get Client ID + Secret.
2. Create a Supabase project → run the `athletes` table SQL → copy URL + anon key + service key.
3. Push to GitHub, import into Vercel, add all env vars, deploy.
4. Set the Strava **Authorization Callback Domain** to your Vercel URL.

**Environment variables:** `STRAVA_CLIENT_ID`, `STRAVA_CLIENT_SECRET`, `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_KEY` (server only!), `VITE_APP_URL`, `VITE_STRAVA_CLIENT_ID`.

---

## 11. 🎯 Interview Cheat Sheet (practise these out loud)

**Q: Walk me through the project in a minute.**
→ Use the elevator pitch in Section 1.

**Q: What happens when a user clicks "Connect with Strava"?**
→ Recite the 9-step flow in Section 4. Emphasise: the code-for-token exchange happens **server-side** so the secret is never exposed.

**Q: How do you keep fetching data after the access token expires?**
→ Strava access tokens last ~6 hours. Each request checks `token_expires`; if it's within 5 minutes of expiry, `refreshIfNeeded()` uses the stored `refresh_token` to get a new access token and saves it back to Supabase. So the user never has to re-log-in.

**Q: Why serverless instead of a traditional backend?**
→ No server to run or scale; each API route is a stateless function; it's free on Vercel's hobby tier; and it keeps the secret out of the browser. Trade-off: cold starts and no long-running processes.

**Q: How is the training plan actually generated?**
→ Section 7.5. Hit the four science principles by name: **10% weekly rule**, **4-week build/recover mesocycles**, **Seiler 80/20**, **taper**, plus the **back-calculation** trick that picks a starting mileage so the plan peaks right on race week.

**Q: Is there a real machine-learning model running?**
→ Be honest (Section 7.3). The data analysis was done **offline in Python** and shaped the formulas/thresholds; the app ships a **lightweight formula version** so it runs instantly in the browser. Say what actually runs — don't claim a live model.

**Q: Explain the fitness decay model.**
→ Exponential decay `F(t)=100·e^(−k·t)`; `k` varies by fitness level; more prior training slows decay; outputs retained %, recovery days, and a gentler return pace/volume. Calibrated on the dataset + Mujika & Padilla (2000).

**Q: Where's the "big data"?**
→ ~40k cleaned rows, 31k extracted gap pairs, curve fitting, and feature-importance analysis — all in the offline phase. The web app operationalises the result.

**Q: What would you improve / what are the limitations?**
→ Good honest answers: (1) all logic sits in one 1,250-line `App.jsx` — I'd split it into modules/hooks and add tests; (2) it uses inline styles — I'd move to CSS modules or a design system; (3) the "ML" is really tuned formulas — I'd serve a genuine trained model behind an API endpoint; (4) only 60 runs are fetched with no pagination or caching; (5) no user accounts beyond Strava and no row-level security policies shown; (6) the decay model has high variance (25% MAE) — it's directional, not precise.

**Q: How did you handle security?**
→ Secrets (`client_secret`, `service_role` key) live only in serverless env vars, never in the browser. The browser only gets the **anon** key. OAuth means we never see or store the user's Strava password.

**Q: Why Supabase / Postgres?**
→ Managed Postgres with a simple JS client and a generous free tier; one small `athletes` table with a unique `strava_id` and an index for fast lookups; `upsert` avoids duplicate rows on repeat logins.

---

## 12. One-Line Summaries to Memorise

- **Project:** Strava-connected app that builds personalised, science-based running plans + forecasts fitness loss.
- **Frontend:** React 18 + Vite, one big `App.jsx`, Recharts for the two charts.
- **Backend:** two Vercel serverless functions — one for OAuth, one for fetching runs (with auto token refresh).
- **DB:** Supabase Postgres, single `athletes` table storing tokens.
- **The science:** Daniels pace zones, 10% rule, 4-week mesocycles, Seiler 80/20, Pfitzinger long runs, taper, Mujika decay curve.
- **The honest bit:** ML/data work was offline; the app runs tuned formulas derived from it.

---

*Good luck — read Sections 1, 4, 7, and 11 last, right before you walk in.* 🏃💨
