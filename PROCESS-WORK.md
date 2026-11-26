# Process Work: Service Monitoring and Alerting

A record of working through this repository end to end: running the stack, completing lessons 01–05, and the improvements made afterwards (two dashboard panels, finer latency buckets and error-rate alerts). It includes the commands used, the results observed and the lessons learned.

## Contents

1. [What the system is](#1-what-the-system-is)
2. [Setup](#2-setup)
3. [Lesson 01: Run the stack](#3-lesson-01-run-the-stack)
4. [Lesson 02: Explore the API](#4-lesson-02-explore-the-api)
5. [Lesson 03: Prometheus](#5-lesson-03-prometheus)
6. [Lesson 04: Grafana](#6-lesson-04-grafana)
7. [Lesson 05: Observability experiments](#7-lesson-05-observability-experiments)
8. [Improvement 1: Error and p95 latency panels](#8-improvement-1-error-and-p95-latency-panels)
9. [Improvement 2: Error-rate alerts](#9-improvement-2-error-rate-alerts)
10. [Gotchas found along the way](#10-gotchas-found-along-the-way)
11. [Git history](#11-git-history)
12. [Command cheat sheet](#12-command-cheat-sheet)

---

## 1. What the system is

Three containers run on one Docker Compose network:

```
 Browser / curl
   │  localhost:8080        localhost:9090          localhost:3000
   ▼                         ▼                        ▼
┌──────────────┐  scrape  ┌────────────┐  query   ┌─────────┐
│ model-service│◄─────────│ Prometheus │◄─────────│ Grafana │
│  FastAPI +   │ /metrics │ stores and │  PromQL  │ draws   │
│  model.bin   │ every 30s│ evaluates  │          │ panels  │
└──────────────┘          │ alert rules│          └─────────┘
                          └────────────┘
```

| Component | Role | Config |
|---|---|---|
| **model-service** | Serves spam predictions; counts and times every request; exposes them at `/metrics` | [service/main.py](service/main.py), [service/Dockerfile](service/Dockerfile) |
| **Prometheus** | Pulls `/metrics` every 30s, stores the history, evaluates alert rules every minute | [prometheus/prometheus.yaml](prometheus/prometheus.yaml), [prometheus/alert.yaml](prometheus/alert.yaml) |
| **Grafana** | Runs PromQL queries against Prometheus and draws dashboards. It stores no metrics of its own | [grafana/provisioning/](grafana/provisioning/), [grafana/dashboards/](grafana/dashboards/) |

**One request, end to end:**

```
curl …/spam_detection_path/free%20prize%20now
 → API answers {"label":"spam","spam_probability":0.81}; counter goes from 41 to 42
 → within 30s Prometheus scrapes 42 and stores it with a timestamp
 → Grafana asks "increase over the last minute?" and Prometheus answers about 1
 → the "Requests Per Minute" panel goes up
```

**Why it's built this way**
- **Pull model:** if the API dies, Prometheus's scrape fails and `up` becomes 0. A push model would just go silent.
- **Separate parts:** the API doesn't know monitoring exists, Prometheus stores, Grafana displays. Each can be swapped or restarted on its own.
- **Everything as code:** the scrape config, alert rules, data source and dashboard are all files in Git.

**What it does *not* do:** it measures whether the service is up, busy, fast and error-free, not whether the predictions are *correct*. That would need model-quality metrics.

---

## 2. Setup

| Step | Status |
|---|---|
| Create the repo from the GitHub template | Done (`origin` → `github.com/Jassalam/mle-service-monitoring-and-alerting`) |
| Clone it | Done |
| Install `uv` and Docker | Both present |
| `uv sync` (local venv for the editor) | Run as part of setup |

```bash
cd mle-service-monitoring-and-alerting
uv sync
```

---

## 3. Lesson 01: Run the stack

Docker Desktop wasn't running at first, so it was started with `open -a Docker`. Then:

```bash
docker compose up --build -d
docker compose ps
```

| Service | Status | Check |
|---|---|---|
| model-service | Up (healthy) | `{"message":"Welcome to the spam detection API"}` |
| prometheus | Up | `Prometheus Server is Ready.` |
| grafana | Up | `/api/health` → `database: ok`, version 13.2.3 |

**Learned:**
- `--build` builds the API image, and `-d` runs everything in the background.
- "healthy" comes from the Dockerfile `HEALTHCHECK`, which calls `/health` every 30s. Only model-service has one.
- Containers reach each other by service name (`model-service:8080`), and the browser reaches them through `localhost`.

---

## 4. Lesson 02: Explore the API

**How the code works ([service/main.py](service/main.py)):**
- `model.bin` is loaded once at startup.
- `GET /spam_detection_path/{message}` cleans the text, runs the model and returns `label` and `spam_probability`.
- `GET /health` returns `{"status":"ok"}` without touching the model.
- `Instrumentator().instrument(app).expose(app)` times and counts every request and adds `/metrics`.

**Test requests:**

| Request | Result |
|---|---|
| `/health` | `{"status":"ok"}`, 200 |
| `free prize now` | **spam**, 0.81 |
| `meeting at ten` | **ham**, 0.02 |
| `claim your reward` | **spam**, 0.59 (low confidence) |
| `can we reschedule` | **ham**, 0.02 |
| `/does-not-exist` | **404** Not Found |

**Reading `/metrics`:**

```
http_requests_total{handler="/spam_detection_path/{message}",status="2xx"} 3
http_requests_total{handler="none",status="4xx"}                         2
http_requests_total{handler="/health",status="2xx"}                      34
http_requests_total{handler="/metrics",status="2xx"}                     32
```

**Learned:**
- **Counters only go up** and reset when the process restarts.
- All messages share one `handler` (the route template), which keeps the number of series small.
- Unknown routes are grouped as `handler="none"`. There were 2 errors, not 1, because the browser also requested `/favicon.ico`.
- `/health` and `/metrics` are **background traffic** from Docker healthchecks and Prometheus scrapes.
- A **histogram** is published as `_count`, `_sum` and `_bucket{le=…}`. Average latency = `_sum / _count`, about 4 ms here.
- The interactive docs are at <http://localhost:8080/docs>.

---

## 5. Lesson 03: Prometheus

**Config:** scrape every 30s; targets `prometheus:9090` and `model-service:8080`; load rules from `alert.yaml`.

**The four signals as queries:**

| Signal | Query | Result |
|---|---|---|
| Uptime | `up{job="services"}` | `1` for both targets |
| Traffic | `increase(http_requests_total[1m])` | spam route +6, 404 +2 |
| Latency | `rate(…_sum[5m]) / rate(…_count[5m])` | about 2.3 ms per prediction |
| Errors | `http_requests_total{status=~"4xx\|5xx"}` | 3, all `handler="none"` |

**Alert test:** the model service was stopped and `InstanceDown` was watched:

```
t+0s    up=1  alert=inactive
t+30s   up=0  alert=inactive    ← next scrape fails
t+60s   up=0  alert=pending     ← rule evaluated, condition true, waiting out for: 5m
t+364s  up=0  alert=firing      ← down for 5 full minutes
```

After `docker compose start model-service`, `/health` answered at once and the alert returned to **inactive** about 52s later, because rules are evaluated once a minute.

**Learned:**
- `increase()`/`rate()` **extrapolate**. With a 30s scrape interval, a `[1m]` window often holds only two samples, so 3 real requests can be reported as 6. Use windows of several scrape intervals (`[2m]`, `[5m]`).
- `for: 5m` prevents paging on short blips, at the cost of 5 minutes of detection delay.
- A restart resets counters, and `rate`/`increase` handle that automatically.
- `handler!~"/metrics|/health"` filters out background traffic.

---

## 6. Lesson 04: Grafana

**Provisioning (no setup clicks needed):**

| File | Effect | Check |
|---|---|---|
| [prometheus_ds.yaml](grafana/provisioning/datasources/prometheus_ds.yaml) | Default `Prometheus` data source at `http://prometheus:9090` | "Successfully queried the Prometheus API" |
| [all.yaml](grafana/provisioning/dashboards/all.yaml) | Loads dashboard JSON into a **Service Monitoring** folder | Found |
| [service-dashboard.json](grafana/dashboards/service-dashboard.json) | **Model Service Monitoring** dashboard | `/d/model-service-monitoring` |

Login: `admin` / `admin`.

**After 1 warm-up request, a 35s wait and a burst of 20 predictions:**

| Panel | Value |
|---|---|
| Requests Per Minute | spam route about 33 (extrapolated), `/health` and `/metrics` about 2 |
| Average Response Time | prediction about 2 ms, `/metrics` about 2.6 ms, `/health` about 0.4 ms |
| Scrape Target Health | both Up |

**Learned:**
- `/health` is about 5× faster than a prediction because it skips the model.
- The dashboard has no auto-refresh by default, so it needs the refresh button or an interval.
- UI edits are saved only in Grafana's Docker volume. To keep them in Git, export as Classic JSON and replace the dashboard file.

---

## 7. Lesson 05: Observability experiments

| # | Action | Expected | Observed | Signal |
|---|---|---|---|---|
| 1 | Check `/health` and `up` | both `1` | both `1`, health `ok` | Uptime |
| 2 | 4 predictions + 1 bad request | prediction route separate from `/health`; error visible | predictions separate; **404 showed `increase = 0`** | Traffic / Errors |
| 3 | 30 predictions + 30 health checks | volume spike, latency low | about 60/min each; prediction about 1.9 ms, health about 0.8 ms | Traffic / Latency |
| 4 | Stop the API, restart **while pending** | pending, then healthy, no alert | `up=0` at 30s → pending at 70s → restarted → inactive at 131s, **never fired** | Alerting |
| 5 | Request and latency queries | answer "how much / how fast" | matched experiments 2–3 | Traffic / Latency |

**Why the first 404 went missing:** the restart in lesson 03 wiped the counters. The first 404 afterwards *created* the `status="4xx"` series at value `1`. `increase()` only measures change between samples it has seen, so it reported 0. After a second 404, it reported about 1.4.

**Two alert paths now seen:**
- Lesson 03: down for more than 5 min → pending → **firing** (a real outage).
- Experiment 4: down for about 90s → pending → **healthy**, no alert (a restart or deploy).

**Reflection answers:**
1. **API unavailable?** Check `up{job="services"}` first. If it's 0, the problem is the process or network, not the code.
2. **API slow?** Check per-route latency, preferably **p95/p99** from histogram buckets, because averages hide outliers.
3. **Why wait 5 minutes?** To avoid paging people for blips, as experiment 4 showed. The trade-off is slower detection of real outages.
4. **Before production:** error-rate and latency alerts, saturation (CPU/memory), Alertmanager to route alerts to Slack, email or a pager, and model-quality metrics (label mix, probability spread, drift).

---

## 8. Improvement 1: Error and p95 latency panels

Commit `c5dc642`, merged into `main` and pushed.

### New panels in [service-dashboard.json](grafana/dashboards/service-dashboard.json)

| Panel | Query |
|---|---|
| **Are Requests Failing?** (bars; 4xx orange, 5xx red) | `sum by (handler, status) (increase(http_requests_total{status=~"4xx\|5xx"}[2m]))` |
| **p95 Response Time** | `histogram_quantile(0.95, sum by (le, handler) (rate(http_request_duration_seconds_bucket{handler!="none"}[5m])))` |

### Required fix in [service/main.py](service/main.py)

The instrumentator's default latency buckets are `(0.1, 0.5, 1)`. Every request took about 2 ms, so all of them landed in the first bucket, and `histogram_quantile` interpolated across 0–100 ms. It reported a **p95 of 95 ms** when the real value was about 5 ms.

```python
LATENCY_BUCKETS = (0.001, 0.0025, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1)
Instrumentator().instrument(app, latency_lowr_buckets=LATENCY_BUCKETS).expose(app)
```

### Checked with live traffic (after rebuilding the API)

| | Before the fix | After |
|---|---|---|
| p95, prediction route | 95 ms (wrong) | **4.8 ms** |
| average, prediction route | 2.6 ms | 2.6 ms |
| errors in 2 min (6 bad requests) | — | 6.4 |

**Notes:**
- Percentiles look wrong for about 5 minutes after a restart, while the window mixes data from before and after.
- No traffic means no p95: you get gaps or "No data".

---

## 9. Improvement 2: Error-rate alerts

Two rules added to [prometheus/alert.yaml](prometheus/alert.yaml), next to `InstanceDown`:

| Alert | Fires when | `for` | Severity |
|---|---|---|---|
| **HighServerErrorRate** | more than 5% of requests return 5xx over 5 min | 5m | critical |
| **HighClientErrorRate** | more than 20% of requests return 4xx over 5 min | 5m | warning |

```yaml
expr: |
  sum(rate(http_requests_total{job="services", status="5xx", handler!~"/metrics|/health"}[5m]))
  /
  sum(rate(http_requests_total{job="services", handler!~"/metrics|/health"}[5m]))
  > 0.05
```

**Design choices:**
- It's a **ratio**, not a raw count, so the threshold works at any traffic level.
- `/metrics` and `/health` are excluded so background traffic doesn't dilute the ratio.
- With no user traffic the ratio is 0/0 = NaN, which never fires.
- 5xx is the "we are broken" signal, so it's critical. 4xx usually means a misbehaving client, so it's a warning with a higher threshold.

**Validation and reload (no restart needed):**

```bash
docker compose exec -T prometheus promtool check rules /etc/prometheus/alert.yaml
# SUCCESS: 3 rules found
curl -X POST http://localhost:9090/-/reload
```

**Live test:** steady traffic of about 50% predictions and 50% 404s.

```
t+10s   HighClientErrorRate inactive
t+80s   HighClientErrorRate pending   value=0.5
t+380s  HighClientErrorRate firing    value=0.5
        "50% of requests failed with 4xx over the last 5 minutes (threshold 20%)."
HighServerErrorRate stayed inactive (no 5xx), as expected.
```

**Limitations:**
- The 5xx rule couldn't be triggered live, because the API has no route that returns a server error.
- At very low traffic, a single error can cross the threshold (1 of 2 requests = 50%). A minimum-traffic guard could be added.
- Like the error panel, `rate()` misses the very first error after a restart.
- Prometheus has no Alertmanager configured, so firing alerts are visible in the UI but not sent anywhere.

---

## 10. Gotchas found along the way

| # | Gotcha | Why | Fix / habit |
|---|---|---|---|
| 1 | `increase()` reports about 2× the real count | Extrapolation across a window with few samples | Use windows of several scrape intervals; treat values as estimates |
| 2 | First error after a restart is invisible to `increase`/`rate` | The new series starts at 1 with no earlier sample | Make counters start at 0 at app startup, or also alert on the raw counter |
| 3 | p95 read 95 ms instead of 5 ms | Default histogram buckets too coarse | Finer low-end buckets (Improvement 1) |
| 4 | Percentiles odd for about 5 min after a restart | Window mixes old and new bucket sets | Wait out the window |
| 5 | Extra 404s you didn't send | The browser requests `/favicon.ico` | Expect background traffic; filter it out |
| 6 | `/health` and `/metrics` dominate the request counts | Healthchecks and scrapes | Filter with `handler!~"/metrics\|/health"` |
| 7 | Alerts stay firing for up to 1 min after recovery | Rules evaluated once a minute | Expected behaviour |
| 8 | Grafana UI edits not in Git | Saved in the Docker volume only | Export Classic JSON into the repo |

---

## 11. Git history

```
c5dc642  Add error and p95 latency panels to Grafana dashboard   (pushed to origin/main)
a97bb22  Initial commit
```

- The work was done on the branch `add-error-and-p95-panels`, fast-forward merged into `main`, pushed, and the branch was deleted.
- **Not yet committed:** the error-rate alerts in `prometheus/alert.yaml` and this `PROCESS-WORK.md`.
- Untracked and unrelated: `ruvector.db`, a local file from a Claude Code plugin.

---

## 12. Command cheat sheet

```bash
# Stack lifecycle
docker compose up --build -d          # start (rebuild API image)
docker compose ps                     # status
docker compose logs -f model-service  # follow logs
docker compose stop | start           # pause / resume, keeps data
docker compose down                   # remove containers, keep volumes
docker compose down -v                # remove containers and data

# Generate traffic
curl "http://localhost:8080/spam_detection_path/free%20prize%20now"
curl "http://localhost:8080/does-not-exist"          # 404
curl -s http://localhost:8080/metrics | grep '^http_requests_total'

# Prometheus
curl -X POST http://localhost:9090/-/reload           # reload config and rules
docker compose exec -T prometheus promtool check rules /etc/prometheus/alert.yaml
```

**Useful PromQL:**

```promql
up{job="services"}
sum by (handler) (increase(http_requests_total{handler!~"/metrics|/health"}[2m]))
sum by (handler) (rate(http_request_duration_seconds_sum[5m])) / sum by (handler) (rate(http_request_duration_seconds_count[5m]))
histogram_quantile(0.95, sum by (le, handler) (rate(http_request_duration_seconds_bucket[5m])))
sum by (handler, status) (increase(http_requests_total{status=~"4xx|5xx"}[2m]))
```

| URL | What |
|---|---|
| <http://localhost:8080/docs> | Interactive API docs |
| <http://localhost:9090/alerts> | Prometheus alert states |
| <http://localhost:3000/d/model-service-monitoring> | Grafana dashboard (`admin`/`admin`) |
