# WaterCast - Groundwater Level Forecasting for Pump Control

WaterCast is a groundwater and tank-level forecasting project with a React monitoring dashboard. It combines historical water-level data, weather observations, autoregressive features, and notebook-trained machine-learning artifacts to support early pump planning and overflow monitoring.

The repository contains two related workflows:

1. **Research and training:** Jupyter notebooks prepare data, compare models, and write the serialized artifacts used by the dashboard.
2. **Monitoring dashboard:** An Express API calls a Python forecast adapter, and a React/Vite application presents forecasts, weather conditions, operations, and alerts.

## Repository layout

```text
.
├── water_level.ipynb                    # data cleaning and feature engineering
├── model_water_level.ipynb              # model training and evaluation
├── pool_groundwater_level.csv           # raw water-level records
├── abhishek data.txt                    # raw weather-station export
├── weather_data.csv                     # cleaned weather data
├── preprocessed_water_level_data.csv    # model-ready data
├── best_model_*.pkl, *.pkl              # trained model artifacts
├── lstm_water_level.keras               # optional TensorFlow artifact
└── dashboard/
    ├── backend/                         # Express API and forecast bridge
    │   ├── python/forecast_snapshot.py  # Python forecast and alert engine
    │   └── data/                        # dashboard CSV history
    └── frontend/                        # React/Vite dashboard
```

## How the system works

```text
Notebook data + trained artifacts
              │
              ▼
preprocessed_water_level_data.csv ──► forecast_snapshot.py
                                         │
                 Open-Meteo weather ────┤
                                         ▼
                              JSON dashboard snapshot
                                         │
                                         ▼
                          Express API ──► React dashboard
```

### Forecast and dashboard behavior

- The Python adapter loads `preprocessed_water_level_data.csv` and the checked-in pickle artifacts from the repository root.
- Historical rows are time-shifted so the final historical row represents the current hour for dashboard demonstration purposes.
- A forecast request attempts to fetch hourly and daily weather from Open-Meteo for the configured location near Tirupati, India. If that request fails, the forecast falls back to historical weather and locally estimated rainfall.
- Forecast horizons are clamped by the Python adapter to **6-72 hours**. The frontend currently offers 12, 24, 36, and 48 hours.
- Forecast alerts use thresholds of 75% for warning and 85% for critical operation. Pump recommendations are rule-based and capped at two pumps.
- The backend also starts a local water-level simulation every 15 minutes. It writes `dashboard/backend/data/actual_water_level.csv` and `tank_levels.csv`; this is a simulated feed, not a hardware sensor integration.
- Weather polling uses the configured Davis weather service and writes `actual_weather_data.csv` and `weather_observations.csv` every 15 minutes. The current implementation contains the Davis API URL in source code; credentials should be moved to environment variables and rotated before deploying this project.
- MongoDB persistence is optional. If MongoDB is unavailable, the API continues serving snapshots without persistence.

## Data and modeling workflow

Run the notebooks from the repository root in order:

1. Open `water_level.ipynb` and run all cells to regenerate `preprocessed_water_level_data.csv`.
2. Open `model_water_level.ipynb` and run all cells to train/evaluate models and regenerate the serialized artifacts.

The feature set includes:

- lagged and change-based water-level features such as `wl_lag_1h`, `wl_lag_6h`, and `wl_lag_24h`;
- weather signals including temperature, humidity, pressure, rainfall, solar radiation, and wind;
- rolling rainfall and weather summaries;
- cyclical time features such as hour and month sine/cosine values; and
- event and operational indicators for rising, dropping, draining, and overflow conditions.

The latest metrics recorded in the repository for an 80/20 chronological split are notebook results, not a startup health check:

| Model | MAE | RMSE | R² | MAPE |
| --- | ---: | ---: | ---: | ---: |
| Top-20 XGBoost + naive persistence blend | 0.661 | 1.689 | 0.660 | 1.140% |
| Tuned top-20 XGBoost | 0.672 | 1.707 | 0.653 | 1.156% |

The dashboard primarily loads `best_model_xgb_cv_top20.pkl`, `best_model_huber_cv.pkl`, `hybrid_scaler.pkl`, and `best_hybrid_meta.pkl`. Other repository artifacts support notebook experiments and comparisons.

## Run the dashboard

### Prerequisites

- Node.js and npm
- Python 3.9+ recommended
- Python packages used by the forecast adapter:

```bash
python -m pip install numpy pandas scikit-learn xgboost
```

TensorFlow is only needed when working with the LSTM notebook/artifact; the dashboard forecast adapter does not import TensorFlow.

### 1. Configure the backend

```bash
cd dashboard/backend
cp .env.example .env
```

Available backend variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `5000` | Express listening port |
| `MONGODB_URI` | unset | Optional MongoDB connection for snapshot, alert, and notification persistence |
| `PYTHON_CMD` | platform fallback | Python executable, for example `python`, `python3`, or an absolute path |

`MONGODB_URI` may be left unset. A failed MongoDB connection is logged and does not stop the server.

Install and start the API:

```bash
npm install
npm start
```

For development with automatic restart:

```bash
npm run dev
```

### 2. Start the frontend

In a second terminal:

```bash
cd dashboard/frontend
npm install
npm run dev
```

The Vite development server is normally available at `http://localhost:5173`. The frontend uses `http://localhost:5000/api` by default. To point it elsewhere, create `dashboard/frontend/.env`:

```dotenv
VITE_API_BASE_URL=http://localhost:5000/api
```

The frontend can be built and linted with:

```bash
npm run lint
npm run build
```

## API reference

The backend listens on `http://localhost:5000` by default.

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/` | Service information and endpoint summary |
| `GET` | `/api/health` | Health response with service name and timestamp |
| `GET` | `/api/dashboard/snapshot?horizon=24` | Forecast snapshot for the requested horizon; cached for 60 seconds by default |
| `POST` | `/api/dashboard/refresh` | Clears the snapshot cache, regenerates a snapshot, and persists it when MongoDB is connected |
| `GET` | `/api/weather` | Historical rows from `actual_weather_data.csv` |
| `GET` | `/api/weather/conditions?range=hourly` | Aggregated weather conditions for `hourly`, `daily`, `monthly`, or `yearly` ranges |
| `GET` | `/api/water-level` | Raw CSV contents of `actual_water_level.csv` |

`/api/dashboard/snapshot` returns `overview`, `environmental`, `operations`, `alerts`, `notifications`, and `systemLogs`. A snapshot error is returned as an HTTP 500 response; the Python adapter itself emits a degraded fallback payload when its forecast work fails.

## Frontend sections

- `/` **Overview:** live/simulated level, forecast profile, weekly rainfall, historical level filtering, and operating notes.
- `/weather-conditions` **Weather Conditions:** latest weather metrics and hourly/daily/monthly/yearly aggregates.
- `/environment` **Environment:** rainfall versus groundwater history and regional rainfall outlook.
- `/operations` **Operations:** severity, pump recommendation, pump status, activity log, and maintenance items.
- `/alerts` **Alerts:** active alerts, notifications, and forecast system logs.

## Important limitations

- The dashboard is a demonstration and decision-support interface, not an automatic pump controller. Recommendations must be verified by an operator.
- The water-level stream is simulated in Node.js and does not read a physical tank sensor.
- The Python script uses a fixed location and shifts historical timestamps to the current time, so dashboard timestamps are synthetic.
- Open-Meteo access requires network connectivity. The adapter has a historical/local fallback, but the weather source may be reported as `fallback`.
- The Davis weather integration currently depends on credentials embedded in `dashboard/backend/src/services/weatherService.js`. Do not deploy the repository as-is with those credentials; move them to environment variables and rotate exposed credentials.
- There are no automated backend tests configured yet; `npm test` currently reports that no tests are configured.