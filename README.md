# HEAT-SHIELD AI

**Hyperlocal Extreme Heat & Thermal Health Impact Early-warning, Simulation, Intervention and
Localized Emergency Decision-support** — SIH 2026, problem statement **26083**
(Ministry of Earth Sciences / NCMRWF, Disaster Management theme).

> **Weather → Thermal Stress → Exposure → Vulnerability → Health Impact Risk → Alert → Action**

The platform answers *"what will the weather do to people?"* rather than *"what will the
temperature be?"*, at **ward level**, for **all of India**, 3–5 days ahead.

---

## 1. Quick start

```bash
pip install -r backend/requirements.txt
cp <checkout>/frontend/.env.example frontend/.env.local 2>/dev/null || true   # API base URL
cd frontend && npm install && cd ..

# API  (http://127.0.0.1:8000, docs at /docs)
python -m uvicorn app.main:app --app-dir backend --host 127.0.0.1 --port 8000

# seed the demo dataset once (~5 min; same job as POST /api/v1/admin/seed)
python scripts/demo_seed.py

# train the ML model (optional, enables the 70/30 blend at runtime)
python ml/training/train.py

# frontend (http://localhost:3000)
cd frontend && npm run dev
```

Demo accounts: `admin/admin123`, `officer/officer123`, `health/health123`, `field/field123`.

Database: Postgres in production (`DATABASE_URL`, see `.env.example` + `docker-compose.yml`);
falls back to SQLite (`backend/heatshield.db`) so the demo runs without credentials.

## 2. Architecture

```
 Weather / population / GIS  ─▶  Ingestion + QC  ─▶  Thermal engine (HI, WBGT, UTCI, HTSI)
                                             ├──▶  Vulnerability engine (PVI)
                                             ├──▶  Forecast engine (120 h)
                                             └──▶  ML engine (GBM, chronological split)
                                                        │
                                    Risk fusion: EHS × PVI ─▶ HIRI + uncertainty + drivers
                                                        │
                        ┌───────────────────────────────┼────────────────────────────┐
                     GIS map                      Action engine                 Alert / API
```

| Layer | Implementation |
|---|---|
| **A — Environmental Heat Stress** | Heat Index (NWS Rothfusz), WBGT (ISO 7243 weighting; BOM/ACSM screening + radiation-aware globe), UTCI (ECMWF thermofeel 6th-order polynomial), HTSI composite |
| **B — Population Vulnerability** | PVI from elderly/children/outdoor-work/density/informal-settlement/healthcare-access/cooling-access |
| **C — Health Impact Risk** | `HIRI = 100 · clamp(EHS · (0.65 + 0.35·PVI/100))` (PRD 7.3), blended 70/30 with the trained GBM when available |
| Forecast | 120 h (5 days) hourly risk + weather per city and ward |
| Alerts | Anti-flap rule (no downgrade without 3 consecutive lower intervals), role-gated approve/acknowledge, multilingual advisory templates (en/hi/ta) |
| Actions | Band-escalating action catalogue with target group and priority |
| Analytics | Forecast-vs-observed, lead-time error, model comparison with chronological validation |

## 3. Risk bands

| HIRI | Band |
|---|---|
| 0–20 | LOW |
| 21–40 | MODERATE |
| 41–60 | HIGH |
| 61–80 | VERY HIGH |
| 81–100 | EXTREME |

Every risk response carries `confidence`, `lower_bound`/`upper_bound` (uncertainty widens with
forecast lead) and human-readable `drivers` for explainability.

## 4. Frontend

Next.js 14 (App Router) + TypeScript + **MUI** (CSS Modules — *no Tailwind*, per PRD) +
MapLibre + Recharts + TanStack Query.

Pages: Dashboard, Live Map (national choropleth → city → ward), Forecast, Alert centre,
Vulnerability explorer, Analytics, Data quality, Model registry, Scenario Lab. Auth is
JWT-based against the demo accounts and role-gated in the UI.

## 5. API (all under `/api/v1`)

`health`, `auth/login`, `admin/seed` · `weather/current|forecast` ·
`thermal/wbgt|utci|heat-index|timeseries` · `risk/current|forecast|ward/{id}` ·
`vulnerability/wards` · `map/india|risk-layer|forecast-peaks|wards` ·
`alerts|alerts/preview|alerts/approve|alerts/acknowledge` · `actions/{ward_id}` ·
`analytics/accuracy|analytics/forecast-vs-observed` · `model/status|model/metrics` ·
`data-quality` · `scenario/run`

Interactive OpenAPI docs: `http://localhost:8000/docs`.

## 6. Scientific & product safeguards (PRD §45)

1. WBGT/UTCI are **heat-stress** metrics, never mortality.
2. The demo dataset is **simulated** and labelled as such in the API, UI and data-quality page;
   health-impact targets used for training are explicitly simulated.
3. Model validation uses a **chronological** split — no random shuffling of time series.
4. Forecasts carry uncertainty and confidence; official IMD/NCMRWF warnings remain authoritative.
5. Data-quality gaps (e.g. health data 0 %) are reported, never silently discarded.

## 7. Tests

```bash
cd backend && python -m pytest tests/ -q    # thermal reference values, risk engine, API + seed flow
cd frontend && npx tsc --noEmit && npm run build
```

## 8. Layout

```
backend/app/    FastAPI: core, models, schemas, thermal, risk, services, ml bridge, main routes
frontend/       Next.js app (app router), components, lib
ml/training/    Model training with chronological validation
data/boundaries/ India GeoJSON outline (Natural Earth, public domain)
docs/           Additional documentation
```

## 9. Limitations (prototype)

Ward geometry is point proxies (centroids); real ward polygons should replace them.
No real health/mortality data is used. Cooling-centre and hospital-capacity data are not
integrated yet. See `docs/` for the production roadmap.