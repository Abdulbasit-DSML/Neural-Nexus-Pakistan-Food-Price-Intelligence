# Pakistan Food Price Intelligence (Django + Gemini, Vercel-ready)

A Django web app that analyses the WFP Pakistan food price dataset (2004–2023) and includes an **AI analyst powered by the Gemini API**. The AI doesn't guess: it calls tools that query the real data, then explains the results or writes a full report.

## 1. What the dataset contains
- 9,996 retail price records, Jan 2004 – Dec 2023, nominal PKR plus USD equivalent. No missing values or duplicates.
- 5 city markets (Karachi, Lahore, Multan, Peshawar, Quetta) and 17 items (cereals, pulses, oil, dairy, meat/eggs, fuel, daily wage).
- Uneven coverage: wheat flour and basmati rice start in 2004, most others 2013–2015, salt only 2014–2018.
- The CSV has a `#hashtag` row under the header (HDX format); `analytics/data.py` skips it.

## 2. Project structure
```
manage.py
foodai/                 Django project (settings, urls, wsgi)
analytics/
  data.py               CSV loading + aggregations (dashboard AND AI tools use these)
  forecast.py           Price forecasting engine (pure Python) + backtesting
  ai.py                 Gemini REST client with function calling (tool loop)
  views.py              index page, POST /api/analyze/, GET /api/forecast/, GET /api/outlook/
  templates/analytics/index.html   Dashboard (Chart.js) + AI chat panel
  tests.py              Unit tests (Gemini is mocked)
data/wfp_food_prices_pak.csv
vercel.json             60s function timeout for AI requests
requirements.txt        Django + requests (no pandas, keeps the bundle small)
.python-version         3.12
```
There is no database: the CSV is read once per server instance and cached in memory.

## 3. Run locally
```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
export GEMINI_API_KEY=your-key DEBUG=1                # Windows (PowerShell): $env:GEMINI_API_KEY="your-key"; $env:DEBUG="1"
python manage.py runserver                            # http://127.0.0.1:8000
python manage.py test                                 # optional: run tests
```

## 4. Deploy to Vercel
Vercel detects Django automatically from `manage.py`; no extra build setup is needed.
1. Push the project to GitHub.
2. In Vercel: **Add New → Project**, import the repo (Framework should show Django/Python).
3. Add **Environment Variables**: `GEMINI_API_KEY` (required), `SECRET_KEY` (random string), and optionally `ACCESS_CODE` and `GEMINI_MODEL`.
4. Click **Deploy**. CLI alternative: `npm i -g vercel && vercel --prod`.

## 5. How the AI works
The browser posts the chat to `/api/analyze/` (CSRF-protected). `ai.py` sends it to Gemini (`generateContent`) with two functions:
- `get_overview`: coverage notes and per-item stats (CAGR, PKR vs USD multiple, YoY, volatility, cheapest/priciest city).
- `query_prices`: filter by item, city, dates; group by year/month/city/province/item; metric PKR or USD.

Gemini calls them as needed (up to 6 rounds) and answers from the returned numbers. The UI shows every query it ran. Notes for Gemini 3.x: the model's reply (including its thought signature) is sent back unchanged, and each function result carries the call's `id`; both are handled in `ai.py`.

## 5b. Price forecasting
**What it does:** predicts monthly prices 1–24 months ahead for any item (national average or one city), in PKR or USD, with an 80% range. The dashboard shows the chart, expected change, and the model's backtest error. "Compare all items" ranks every commodity by expected change. The AI can call the same engine (`forecast_prices`, `forecast_outlook`), so you can ask "forecast wheat flour for 12 months" or "which items will rise most?".

**How it works (`analytics/forecast.py`):** monthly average price → log scale → three candidate models (damped-trend Holt smoothing, the same with monthly seasonality, and naive "last value"). The best one is chosen per item by a rolling backtest of 6-month-ahead errors over the last ~2 years. Intervals widen with the horizon.

**Read this before trusting the numbers**
- The dataset ends in **Dec 2023**, so "upcoming" means the months after that. For real future predictions, download the latest Pakistan food prices CSV from WFP's page on HDX (data.humdata.org), replace `data/wfp_food_prices_pak.csv` (same columns) and redeploy.
- Backtest errors are roughly 3–13% over 6 months, and for many items the model only slightly beats the naive baseline. Treat forecasts as trend guides, not exact prices.
- It cannot predict shocks (floods, subsidy or policy changes, currency crises). Salt stopped being recorded in 2018, so its forecast is flagged stale.
- Not financial advice.

**Next upgrades:** add fuel price and USD/PKR exchange rate as inputs, or try SARIMA/Prophet (heavier dependencies; run on a separate service if the Vercel bundle gets too big).

## 6. Model choice
Default is `gemini-3.5-flash`. If your key can't use it or you hit quota limits, set `GEMINI_MODEL` to another id from [Google AI Studio](https://aistudio.google.com), e.g. `gemini-3.5-flash-lite` or `gemini-3.1-flash-lite`. Older `gemini-2.5-*` models are scheduled for retirement and may stop working.

## 7. Security and cost
Anyone with your URL can use your Gemini quota. Set `ACCESS_CODE` (visitors then enter it in the chat panel) and set a quota/budget alert in Google AI Studio. For public launch, add rate limiting (Vercel Firewall rules).

## 8. Customise
- **New data:** replace the CSV in `data/` (same columns) and redeploy.
- **AI behaviour:** edit `SYSTEM` in `analytics/ai.py`.
- **More tools:** add a function declaration in `TOOLS` and handle it in `_run_tool` (e.g. forecasting, fuel-vs-food correlation).
- **Ideas:** CPI-adjusted prices, map using latitude/longitude, CSV export, streaming answers.

## 9. Troubleshooting
- *"GEMINI_API_KEY is not set"*: add it under Vercel → Settings → Environment Variables and redeploy.
- *Gemini error 400/404 about the model*: set `GEMINI_MODEL` to a model your key supports.
- *Gemini error 429*: free-tier quota reached; wait, use a lighter model, or enable billing.
- *Timeouts*: full reports can take 20–40s; `vercel.json` allows 60s.
- *CSRF 403 on the AI button*: reload the page so the CSRF cookie is set; on a custom domain add it to `ALLOWED_HOSTS`.
