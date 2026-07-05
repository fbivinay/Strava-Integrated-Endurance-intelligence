# 🏃 Endurance Intelligence

A Strava-connected web app that reads your real running history and generates a **personalised, science-backed training plan** for a target race — with the right weekly mileage, long runs, and pace zones for every session. It also includes a **fitness-loss forecast** that estimates how much fitness you lose during rest days and how to return safely.

Built as an MSc Big Data Analytics project.

---

## ✨ Features

- **One-click Strava login** (OAuth 2.0) — no passwords stored.
- **Automatic fitness-level detection** from your last 60 runs (pace + weekly mileage).
- **Goal-aware training plans** for 5K, 10K, Half, Full Marathon, Ultra 50K, or a custom distance.
- **Science-based periodization** — 10% weekly-growth rule, 4-week build/recover cycles, an 80/20 easy-hard split, race-specific peak mileage, and a taper before race day.
- **Daniels VDOT pace zones** for every session (Easy, Long, Tempo, Intervals, Recovery).
- **Fitness-loss forecast** using an exponential detraining model, with a return-pace and return-volume recommendation.
- **PDF export** of the full plan.
- **Demo mode** — explore the app with a sample runner profile, no Strava account needed.

---

## 🧩 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18 + Vite |
| Charts | Recharts |
| Backend | Vercel Serverless Functions (Node.js) |
| Database | Supabase (PostgreSQL) |
| Auth | Strava OAuth 2.0 |
| Hosting | Vercel |

Everything runs comfortably on free tiers.

---

## 🏗️ How It Works

```
1. User clicks "Connect with Strava"
        ↓
2. Strava login & consent  (strava.com)
        ↓
3. Strava redirects to → /api/strava-callback   (serverless)
        ↓
4. Function exchanges the auth code for access + refresh tokens
        ↓
5. Tokens & athlete profile saved to Supabase
        ↓
6. Redirect back to the app with ?athlete_id=...
        ↓
7. App calls → /api/activities?athlete_id=...   (serverless)
        ↓
8. Function refreshes the token if needed and fetches the last 60 runs
        ↓
9. App computes stats, detects the runner's level, and builds the plan
```

The OAuth token exchange and token refresh happen inside the serverless functions, so the Strava `client_secret` is never exposed to the browser.

---

## 📁 Project Structure

```
├── api/
│   ├── strava-callback.js   # OAuth: exchange code for tokens, save to DB
│   └── activities.js        # Fetch runs from Strava, auto-refresh tokens
├── src/
│   ├── main.jsx             # React entry point
│   ├── App.jsx              # Main app: UI + plan/forecast logic
│   └── supabase.js          # Supabase client (browser)
├── index.html
├── vite.config.js
├── vercel.json
├── package.json
└── .env.example
```

---

## 📚 The Science Behind the Plan

The training logic is grounded in established endurance-coaching principles and calibrated against a running dataset:

- **Progressive overload** — mileage builds ~10% per week to reduce injury risk.
- **Mesocycles** — 4-week blocks (three build weeks, one recovery week).
- **80/20 polarised training** (Seiler) — roughly 80% easy volume, 20% quality (tempo + intervals).
- **VDOT pace zones** (Daniels) — all session paces derived from your goal race pace.
- **Long-run rules** (Pfitzinger) — ~30% of weekly volume, capped by race distance.
- **Taper** — volume reduced over the final two weeks so you arrive fresh.
- **Detraining model** — exponential fitness decay `F(t) = 100 · e^(−k·t)`, calibrated from the dataset and informed by Mujika & Padilla (2000).

---

## 🗄️ Database Schema

```sql
CREATE TABLE athletes (
  id            BIGSERIAL PRIMARY KEY,
  strava_id     BIGINT UNIQUE NOT NULL,
  firstname     TEXT,
  lastname      TEXT,
  profile_pic   TEXT,
  access_token  TEXT NOT NULL,
  refresh_token TEXT NOT NULL,
  token_expires BIGINT NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_athletes_strava_id ON athletes(strava_id);
```

---

## 🚀 Getting Started

### Prerequisites
- A [Strava API application](https://www.strava.com/settings/api) (Client ID + Secret)
- A [Supabase](https://supabase.com) project (URL, anon key, service key)
- A [Vercel](https://vercel.com) account

### Local development

```bash
# 1. Install dependencies
npm install

# 2. Configure environment variables
cp .env.example .env      # then fill in your keys

# 3. Run frontend + serverless functions together
npx vercel dev
```

### Environment variables

| Variable | Description |
|---|---|
| `STRAVA_CLIENT_ID` | Strava app Client ID |
| `STRAVA_CLIENT_SECRET` | Strava app Client Secret (server only) |
| `VITE_STRAVA_CLIENT_ID` | Same as Client ID (exposed to frontend) |
| `VITE_SUPABASE_URL` | Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | Supabase anon key (safe for browser) |
| `SUPABASE_SERVICE_KEY` | Supabase service-role key (**server only**) |
| `VITE_APP_URL` | Your deployed app URL |

### Deploy

1. Push the repo to GitHub and import it into Vercel.
2. Add the environment variables above in Vercel's project settings.
3. Deploy, then set your Strava app's **Authorization Callback Domain** to your Vercel URL.

See [`SETUP.md`](./SETUP.md) for a detailed step-by-step guide.

---

## 📝 License

This project was built for educational purposes.
