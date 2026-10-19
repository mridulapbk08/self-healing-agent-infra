# Self-Healing Agent Infrastructure

A workflow orchestration platform written in Go, with Redis task queuing, PostgreSQL persistence and a Next.js monitoring dashboard, built to evaluate automated recovery strategies under controlled fault injection.

This repository is the companion code for the paper:

> Mridula Prabhakar. *Self-Healing Workflow Orchestration Framework Using System-Aware Adaptive Recovery for Fault-Tolerant Distributed Systems.* Independent research preprint, 2026. Zenodo. [https://doi.org/10.5281/zenodo.23428728](https://doi.org/10.5281/zenodo.23428728)

---

## Key Results

The evaluation consists of 20 benchmark runs: four recovery strategies, five runs each, 100 workflows per run (2,000 workflows in total).

* Automated recovery raised workflow completion from 44.8% with no recovery to between 82.4% and 84.4% (p < 0.001 for every recovery strategy against the baseline).
* The three recovery strategies cannot be distinguished on success rate (all pairwise Welch's t-tests p > 0.3; one-way ANOVA p = 0.61). This is the expected outcome: injected faults are independent across attempts, so the delay before a retry cannot change whether it succeeds.
* The measured rates agree with a closed-form prediction. Each attempt fails with probability 0.5, each workflow gets four attempts, and a simulated worker crash ends 10% of workflows before any attempt, so the predicted success rate is 0.90 × (1 − 0.5⁴) = 84.4%, with 39.4 recovered workflows per run. The No Recovery baseline matches its own prediction, 0.90 × 0.5 = 45.0%, against a measured 44.8%.
* The strategies differ in cost. Exponential Backoff has a significantly higher average recovery time and lower throughput than Fixed Retry (p < 0.01). System-Aware Adaptive Recovery cannot be distinguished from Fixed Retry on recovery time (p = 0.43) or throughput (p = 0.36).

| Strategy | Success Rate | Failure Rate | Recovered Tasks | Average Recovery Time | Throughput (workflows/s) |
| --- | ---: | ---: | ---: | ---: | ---: |
| No Recovery | 44.80 ± 2.95% | 55.20 ± 2.95% | 0.00 ± 0.00 | 914.11 ± 8.48 ms | 3.26 ± 0.06 |
| Fixed Retry | 84.40 ± 3.36% | 15.60 ± 3.36% | 40.80 ± 5.50 | 2112.04 ± 141.81 ms | 1.43 ± 0.12 |
| Exponential Backoff | 82.40 ± 2.70% | 17.60 ± 2.70% | 41.40 ± 3.21 | 2509.97 ± 198.15 ms | 1.19 ± 0.08 |
| System-Aware Adaptive Recovery | 82.60 ± 4.04% | 17.40 ± 4.04% | 39.20 ± 6.26 | 2202.96 ± 200.46 ms | 1.35 ± 0.12 |

Values are the mean ± sample standard deviation over five runs. Average Recovery Time is the mean end-to-end duration of every workflow, including workflows that never needed recovery; the No Recovery value reflects the 1,000 ms processing delay, not recovery activity. With five runs per strategy, differences in success rate below about seven percentage points cannot be separated from chance.

The per-run data are in [`experiments/results.csv`](experiments/results.csv), and the summary statistics and significance tests are in [`experiments/analysis.md`](experiments/analysis.md).

---

## Architecture

![Architecture](docs/figures/architecture.png)

| Component | Location | Role |
| --- | --- | --- |
| Workflow API | `backend/cmd/server`, `backend/internal/orchestrator` | Go `net/http` server on port 8080 |
| Workflow Orchestrator | `backend/internal/orchestrator` | Creates one task per request, saves it to PostgreSQL and pushes it to Redis |
| Redis Task Queue | `backend/internal/queue` | Holds tasks waiting for a worker |
| Worker Pool | `backend/internal/orchestrator/worker.go` | Three concurrent workers; each dequeues a task, executes it and retries it in place |
| Recovery Engine | `backend/internal/recovery` | Decides whether to retry a failed task and how long to wait |
| PostgreSQL | `backend/internal/database` | Stores the current state of each task |
| Monitoring and Metrics | `backend/internal/metrics` | In-memory worker, system and benchmark metrics, exposed over REST |
| Dashboard | `frontend/` | Next.js application that displays the metrics |

A failed task is retried by the same worker after the delay chosen by the recovery strategy; it is not returned to the Redis queue. The worker stays occupied for the whole retry sequence, which is why longer retry delays reduce throughput.

---

## Recovery Strategies

The strategy is selected with the `RECOVERY_STRATEGY` environment variable. If it is not set, the server uses `FIXED_RETRY`.

| Value | Behaviour (retry limit of three) |
| --- | --- |
| `NO_RECOVERY` | A failed attempt is final. |
| `FIXED_RETRY` | Waits 500 ms before every retry. |
| `EXPONENTIAL_BACKOFF` | Waits 500, 1,000 and 2,000 ms. |
| `ADAPTIVE_RECOVERY` | Before each retry, selects Exponential Backoff if at least 10 outcomes have been recorded and the cumulative failure rate is at least 30%; otherwise Fixed Retry if at least 50 tasks are queued; otherwise a hybrid schedule of 500, 500 and 1,000 ms. |

The adaptive failure-rate signal is the share of recorded final workflow outcomes that are FAILED, not the share of failed attempts. The delays are configured waiting intervals, not measured recovery durations.

![Recovery workflow](docs/figures/recovery-workflow.png)

---

## Fault Injection

The benchmark settings are defined in [`backend/internal/config/config.go`](backend/internal/config/config.go):

| Setting | Value | Meaning |
| --- | --- | --- |
| `WorkerCount` | 3 | Concurrent workers |
| `MaxRetries` | 3 | Retries allowed after the first attempt |
| `FailureRate` | 0.50 | Each execution attempt fails independently with this probability |
| `CrashRate` | 0.10 | A simulated worker crash marks a workflow as failed before it executes |
| `WorkerDelayMs` | 1000 | Processing time of each attempt, in milliseconds |

---

## Running Locally

Prerequisites: Go 1.24 or later, Docker, a current Node.js LTS release (for the dashboard), and Python 3 with `matplotlib` and `scipy` (for the analysis).

### 1. Clone the repository

```bash
git clone https://github.com/mridulapbk08/self-healing-agent-infra.git
cd self-healing-agent-infra
```

### 2. Start PostgreSQL and Redis

```bash
docker compose up -d
```

This starts PostgreSQL 15 on port 5432 and Redis 7 on port 6379.

### 3. Create the database table

```bash
docker exec -i self_healing_postgres psql -U postgres -d self_healing_db < backend/scripts/schema.sql
```

### 4. Run the backend

```bash
cd backend
RECOVERY_STRATEGY=ADAPTIVE_RECOVERY go run cmd/server/main.go
```

On Windows PowerShell:

```powershell
cd backend
$env:RECOVERY_STRATEGY = "ADAPTIVE_RECOVERY"
go run cmd/server/main.go
```

The API is served at `http://localhost:8080`.

### 5. Run the dashboard

```bash
cd frontend
npm install
npm run dev
```

The dashboard is served at `http://localhost:3000`.

### 6. Submit workflows

From the `backend` directory:

```bash
./scripts/benchmark.sh 100
```

---

## Reproducing the Experiments

The full evaluation is automated. From the `backend` directory, on macOS or Linux (the script needs `bash`, `curl`, `lsof` and `python3`):

```bash
bash scripts/run_experiments.sh
```

The script runs each strategy five times with 100 workflows per run, starts a fresh server for every run, and appends one row per run to `experiments/results.csv`. The file must begin with its header row. To start a new set of runs, move the existing file aside and create a new `experiments/results.csv` containing only:

```text
Strategy,Run,TotalWorkflows,Completed,Recovered,Failed,SuccessRate,FailureRate,AverageRecoveryTimeMs,Throughput
```

Then generate the statistics and figures:

```bash
cd ../experiments
pip install matplotlib scipy
python3 analyze_results.py
```

This writes `analysis.md` (summary table and Welch's t-tests) and Figures 3 to 7 of the paper to `figures/`. Failures are drawn at random, so new runs produce numbers that differ from the reported ones; the reported runs are the ones in `results.csv`.

---

## API Endpoints

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/workflow/start` | Submits a workflow and returns its task ID |
| `GET` | `/workflow/status?id=<task_id>` | Returns the current state of a task |
| `GET` | `/metrics/workers` | Processed, completed, recovered and failed counts for each worker |
| `GET` | `/metrics/system` | Totals, recovery rate and failure rate |
| `GET` | `/metrics/benchmark` | Success rate, failure rate, average recovery time and throughput |

Example:

```bash
curl -X POST http://localhost:8080/workflow/start \
  -H "Content-Type: application/json" \
  -d '{"type":"demo","payload":"hello"}'
```

---

## Repository Layout

```text
backend/
  cmd/server/main.go           HTTP server and worker start-up
  internal/agents/             simulated task execution and per-attempt failures
  internal/config/             worker count, retry limit and fault-injection settings
  internal/database/           PostgreSQL persistence
  internal/logger/             structured JSON logging
  internal/metrics/            worker, system and benchmark metrics
  internal/models/             task model
  internal/orchestrator/       API handlers, worker loop and simulated worker crashes
  internal/queue/              Redis task queue
  internal/recovery/           recovery strategies and the adaptive controller
  scripts/benchmark.sh         submits a number of workflows
  scripts/run_experiments.sh   runs the full evaluation
  scripts/schema.sql           PostgreSQL table definition
experiments/
  results.csv                  per-run results reported in the paper
  analyze_results.py           summary statistics, significance tests and figures
  analysis.md                  output of analyze_results.py
  figures/                     Figures 3 to 7
frontend/                      Next.js monitoring dashboard
docs/                          architecture notes, figures and screenshots
docker-compose.yml             PostgreSQL and Redis services
evaluation_results.md          earlier single-run exploratory results (superseded)
paper/                         earlier project write-up (superseded)
```

`evaluation_results.md` and `paper/self_healing_workflow_system.md` record exploratory single runs made before the reported evaluation, and their numbers differ from the paper. The reported results are the 20 runs in `experiments/results.csv`, described in the [Zenodo preprint](https://doi.org/10.5281/zenodo.23428728).

---

## Screenshots

**Dashboard overview**

![Dashboard overview](docs/screenshots/dashboard-overview.jpg)

**Worker utilization and task distribution**

![Worker utilization and task distribution](docs/screenshots/charts.jpg)

**Worker metrics**

![Worker metrics](docs/screenshots/worker-metrics.jpg)

**Server logs during workflow execution**

![Server logs during workflow execution](docs/screenshots/workflow-execution.png)

---

## Scope and Next Steps

* The experiments ran on a single machine with locally hosted Redis and PostgreSQL, so they do not capture network latency or failures between machines.
* The fault injector draws failures independently on each attempt, so the delay before a retry cannot affect its outcome; this is why the recovery strategies cannot be separated on success rate. The next step is to evaluate them under temporally correlated failures, such as outages that persist across retries, where the timing of a retry does matter.
* The adaptive controller uses deterministic rules. Learned or predictive controllers, and deployment on Kubernetes with distributed worker pools, are further directions.
* CPU, memory, network overhead and energy use were not measured.

---

## Technologies

Go 1.24 (`net/http`), Redis 7, PostgreSQL 15, Docker Compose, Next.js 16, React 19, Recharts, Tailwind CSS 4, and Python with matplotlib and SciPy for the analysis.

---

## Citation

Mridula Prabhakar. *Self-Healing Workflow Orchestration Framework Using System-Aware Adaptive Recovery for Fault-Tolerant Distributed Systems.* Independent research preprint, 2026. Zenodo. https://doi.org/10.5281/zenodo.23428728

```bibtex
@misc{prabhakar2026selfhealing,
  author    = {Prabhakar, Mridula},
  title     = {Self-Healing Workflow Orchestration Framework Using System-Aware Adaptive Recovery for Fault-Tolerant Distributed Systems},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.23428728},
  url       = {https://doi.org/10.5281/zenodo.23428728},
  note      = {Independent research preprint}
}
```

---

## Author

**Mridula Prabhakar** · [GitHub](https://github.com/mridulapbk08)
