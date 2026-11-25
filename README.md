# RAGATE: Reliability-Aware Gating and Adaptive Trading Exposure

A full-stack research platform for cryptocurrency forecasting, reliability-aware expert weighting, risk-managed decisions, **paper trading** and backtesting. Runs locally on Windows 11.

> Research software. It does not place real orders, does not promise profit, and makes no claim of proven novelty. Whether RAGATE beats the baselines is something **you** measure with the backtester on your own data.

## Architecture

| Service | Tech | Port | Role |
|---|---|---|---|
| Frontend | React, TypeScript, Tailwind, Lightweight Charts, Recharts | 5173 | Dashboard |
| Backend | Java 21, Spring Boot 3.3, Spring Security (JWT), JPA, Flyway | 8080 | Auth, datasets, orchestration, paper trading, persistence |
| ML service | Python, FastAPI, PyTorch, XGBoost, hmmlearn, scikit-learn | 8000 | Validation, features, HMM, experts, RAGATE, risk, decisions, backtests |
| Database | PostgreSQL 16 (Docker) | 5432 | All state |

Browser → Spring Boot → (ML service, PostgreSQL). The ML service is never called by the browser; it needs the `X-Internal-Key` header. Spring Boot and the ML service share the `storage/` folder (uploads, canonical datasets, model artifacts). Model files live on disk, not in the database.

## Prerequisites (Windows 11)

Python 3.11 or 3.12 · Node.js 20+ · JDK 21 or later · Apache Maven 3.9+ · Docker Desktop (running).
Open the folder in VS Code and use the integrated terminal (Command Prompt or PowerShell).

## Quick start

From PowerShell, in the extracted project root:

```powershell
.\setup.bat      # first run: installs Python/Node dependencies, builds Java backend, starts PostgreSQL
.\start.bat      # starts PostgreSQL, ML service, backend and frontend
.\stop.bat       # stops the app windows and PostgreSQL (keeps database data)
```

Wait until the backend terminal prints `Started RagateApplication` before registering. Do not keep an older backend copy running on port 8080 while starting this copy.

`setup.bat` creates `.env` from `.env.example` (change the secrets). PyTorch is large; the first install takes several minutes.

Then: open http://localhost:5173 → **Register** (first account becomes ADMIN and can train models) → **Datasets** → **Models** → **Predictions**.

### Manual start (3 terminals)

```bat
docker compose up -d db
:: terminal 1: ML service
cd ml-service && python -m venv .venv && .venv\Scripts\activate && pip install -r requirements.txt
uvicorn app.main:app --port 8000
:: terminal 2: backend
cd backend && mvn spring-boot:run
:: terminal 3: frontend
cd frontend && npm install && npm run dev
```
Set the variables from `.env` in the shell first (the scripts do this for you via `scripts\load_env.bat`).

## Typical workflow

1. **Get data.** Datasets page: upload CSV/XLSX (columns are auto-detected; fix them in the mapping table) *or* fetch BTC/ETH/SOL (or any valid pair) from Binance. Review the quality report, choose how to handle duplicates, invalid rows, missing values and gaps, then *Validate, clean and save*. Every change is logged.
2. **Train.** Models page (admin): trains HMM, XGBoost, Mean Reversion and R-CTE on the chronological training split, tunes on validation, leaves the last 15% untouched as the test period. You need ~1,500+ candles (about 250 days of 4h data); shorter data is rejected with an explanation. CPU training takes minutes. Alternative without the UI: `python scripts\train_cli.py --csv file.csv --symbol BTCUSDT`.
3. **Predict.** Predictions page: runs the experts, RAGATE weighting, risk engine and decision engine. Forecasts are stored; once their target candle exists they are scored (once) and update EWMA/CUSUM. Reliability builds up as you run predictions on successive candles (for Binance data: refetch the dataset as new candles close).
4. **Paper trade.** Execute a BUY/REDUCE/EXIT decision into the virtual portfolio, or place manual simulated orders.
5. **Backtest.** Backtesting page: compares the ten strategies on the untouched test period.

## How RAGATE works (what the code actually does)

* **Experts** (`ml-service/app/experts`): R-CTE (PyTorch attention encoder + regime fusion; FiLM by default, with concat/add/none variants for comparison), XGBoost (regressor + classifier), Mean Reversion (z-score regression whose sign is *estimated*; trend and volatility filters). Targets are forward log returns for 1/3/5 days (6/18/30 candles at 4h).
* **Regimes** (`regime/hmm.py`): 3-state Gaussian HMM fit on training rows only; states labelled by volatility/return; probabilities are *filtered* with the forward algorithm (causal, tested).
* **RAGATE** (`ragate/engine.py`): matured-only scoring (each forecast scored once); loss normalised by volatility known at forecast time; EWMA reliability shrunk toward neutral when few forecasts exist; one-sided CUSUM deterioration flag (flags only, never swaps models); regime-expert compatibility from **validation** performance; weights = softmax over `β₁·compat − β₂·EWMA − β₃·CUSUM`, non-negative, sum to 1, over available experts only; disagreement, within-model variance and regime entropy combine into an uncertainty estimate scaled by a validation-calibrated factor.
* **Risk** (`risk/engine.py`): fees, slippage, spread, net return, signal-to-noise, stress block, adaptive exposure cap, mean-variance sizing, portfolio-volatility limit (Ledoit-Wolf covariance when several assets' returns are supplied). It can veto a positive forecast.
* **Decision** (`decision/engine.py`): BUY / HOLD / REDUCE / EXIT / ABSTAIN with reasons. ABSTAIN on stale data, missing experts, all experts flagged, or a risk veto of a positive signal.
* **Backtest** (`backtest/engine.py`): signal at candle close, fill at next open, costs on turnover. RAGATE state is warmed up on validation data (no trading). Strategies: Buy & Hold, Equal-Weight, MA Crossover, RSI, standalone XGBoost / R-CTE / Mean Reversion, static equal-weight ensemble, regime-aware static ensemble, Full RAGATE. Metrics: MAE, RMSE, directional accuracy, precision, recall, F1 (forecast quality) and cumulative/annualised return, Sharpe, Sortino, max drawdown, CVaR, win rate, profit factor, turnover (portfolio). Start dates inside training/validation are clamped to the test period.

## Honest status

| Area | Status |
|---|---|
| ML service (data validation, features, HMM, 3 experts, RAGATE, risk, decisions, backtest, API) | Implemented. **32 automated tests pass** (see `docs/TESTING.md`). |
| Frontend | Implemented. Type-checks and builds for production. Not exercised by browser tests. |
| Spring Boot backend | Implemented but **not compiled or run in the environment where this project was generated** (no access to Maven Central). Expect to fix small compile or wiring issues on first run; see Troubleshooting. No Java tests are included. |
| Binance REST/WebSocket | Implemented with retry/rate-limit/gap recovery. **Not tested against the live API** (no network from the build environment); the logic is unit-tested only for freshness. |
| `market_features` table | Created but not populated (features are computed on demand). |
| Training | Synchronous inside the ML service, run as a background job by Spring. No progress bar; status is TRAINING until done. |
| Backtests | Per-asset simulation with equal capital across assets. No cross-asset portfolio optimisation or shared cash constraint in the backtester (the risk engine does support covariance for live decisions). Walk-forward validation is provided for XGBoost only (R-CTE is trained once for cost reasons). |
| Documents | `docs/` contains architecture, algorithms, report skeleton. **Results and screenshots sections are intentionally empty**: fill them from your own runs. Nothing is invented. |
| Required 20 pages | Consolidated into 9 screens (below). |

Page mapping: Dashboard (Main Dashboard, Market Overview widgets) · Datasets (Upload, Management, Data Quality) · Features & regimes (Feature Explorer, Regime Monitor) · Model registry (Registry, expert metrics) · Predictions & RAGATE (Prediction Monitor, Expert Analysis, Reliability Monitor, Weight Comparison, Risk Analytics, Decision Monitor) · Paper portfolio (Portfolio, Orders & Trades) · Backtesting (Workspace, Strategy Comparison) · Research results · Settings · Login/Register.

## Ports, URLs and API

Frontend http://localhost:5173 · Backend http://localhost:8080 · ML docs http://localhost:8000/docs · PostgreSQL localhost:5432.
Backend endpoints: `POST /api/auth/register|login`, `GET /api/v1/market/overview`, `GET /api/v1/market/{symbol}/candles`, `POST /api/v1/datasets/upload`, `POST /api/v1/datasets/{id}/commit`, `POST /api/v1/datasets/binance`, `GET /api/v1/datasets[/{id}]`, `POST /api/v1/predictions`, `GET /api/v1/predictions/{id}`, `GET /api/v1/portfolio`, `POST /api/v1/paper/orders`, `GET /api/v1/trades`, `GET /api/v1/orders`, `POST /api/v1/backtests`, `GET /api/v1/backtests/{id}`, `GET /api/v1/models`, `POST /api/v1/models/train`, `GET /api/v1/research/metrics`.
ML endpoints: `GET /health`, `POST /internal/v1/{data/validate, data/features, data/binance, regime/predict, train, predict, ragate/evaluate, risk/evaluate, backtest}`, `GET /internal/v1/models`. (`risk/evaluate` returns both the risk result and the decision.)

## Security notes

JWT auth, BCrypt hashes, role check for training, request validation, upload extension/size limits with random stored names, path-traversal guard on ML file paths, per-IP rate limiting (stricter on auth), CORS limited to the frontend origin, secrets only in `.env` (never in frontend code), internal ML API key. Database backups: `scripts\backup_db.bat`. Change every default secret before exposing anything beyond localhost.

## Troubleshooting

* **Backend fails to start with "password authentication failed"**: `.env` DB values must match the running container. Reset: `docker compose down -v` then `start.bat`.
* **Port already in use**: stop the other process or change the port (frontend `vite.config.ts`, backend `server.port`, ML `--port` and `ML_BASE_URL`).
* **Flyway/Hibernate "schema validation" error**: the database holds an old schema; `docker compose down -v` to recreate it.
* **Maven compile error**: run `cd backend` then `mvn -U clean package`. This project pins Lombok 1.18.42 and configures its annotation processor in Maven; the build must succeed before starting the backend. The compiler targets Java 21. A JDK 21 installation is the recommended runtime.
* **Binance errors (451/403/timeouts)**: Binance blocks some regions/networks. Set `BINANCE_BASES=https://data-api.binance.vision` in `.env`, or use a VPN, or upload CSV data instead.
* **"ML service unavailable"**: start the ML service; check http://localhost:8000/health.
* **Training says dataset too short**: fetch more history (earlier start date) or use a finer interval.
* **Everything is ABSTAIN/HOLD**: expected when forecasts do not clear costs or experts show no skill. Check the reliability table and the risk checks on the Predictions page; the model may simply have no edge on that data.
* **Frontend shows "Cannot reach the backend"**: wait for the backend to finish starting (~1 min first time). Check that the backend terminal is running from this extracted project and that port 8080 is not occupied by an older RAGATE copy.
* **CORS / `Invalid CORS request`**: use `http://localhost:5173` for the frontend (not a different host/port), restart the backend after changing `.env`, and verify the `CORS_ORIGIN` value. The backend explicitly permits localhost and 127.0.0.1 Vite origins and does not rate-limit OPTIONS preflight requests.
* **PyTorch install is slow/huge**: CPU-only wheels: `pip install torch --index-url https://download.pytorch.org/whl/cpu` before `pip install -r requirements.txt`.
* **Tests**: `scripts\run_ml_tests.bat`.
