# Self-Healing Agent Infrastructure

## Overview

Self-Healing Agent Infrastructure is a distributed workflow orchestration platform designed to evaluate fault tolerance and automated recovery in modern cloud-native systems. The platform simulates resilient agents capable of detecting failures, recovering from disruptions, and maintaining workflow execution without manual intervention.

Built using Golang, Redis, PostgreSQL, Docker, and Next.js, the system combines distributed task processing, durable queues, configurable recovery strategies, benchmarking, and real-time monitoring to demonstrate self-healing behavior in distributed environments.

Experimental evaluation covers 20 benchmark runs of 100 workflows each. Automated recovery raises workflow completion from 44.8% to 84.4% against a no-recovery baseline. The three recovery strategies are statistically indistinguishable from one another on success rate under the failure model used in these runs.
---

## Technologies Used

* Golang
* PostgreSQL
* Redis
* Next.js
* React
* Recharts
* Docker
* Tailwind CSS

---

## System Architecture

```text
Client Request
      |
      v
API Layer (Go HTTP Server)
      |
      v
Redis Queue
      |
      v
Worker Pool (3 Concurrent Workers)
      |
      +---- Success ----> PostgreSQL
      |
      +---- Failure ----> Recovery Engine
                               |
                               v
                        Self-Healing Logic
                               |
                               v
                          PostgreSQL

Metrics API
      |
      v
Next.js Monitoring Dashboard
```

---

## Architecture Components

### API Layer

* Accepts workflow execution requests
* Exposes monitoring and metrics APIs
* Coordinates workflow orchestration
* Enables frontend-backend communication

### Redis Queue

* Durable task storage
* Asynchronous workflow execution
* Decouples producers and consumers
* Supports distributed processing

### Worker Pool

* Three concurrent workers
* Independent workflow execution
* Distributed task processing
* Load-balanced task consumption

### Recovery Engine

* Detects task failures
* Executes configurable recovery strategies
* Supports Fixed Retry and Exponential Backoff
* Tracks recovery metrics and success rates

### PostgreSQL

* Persists workflow execution history
* Stores task lifecycle states
* Maintains benchmark metrics
* Records worker statistics

### Monitoring Dashboard

* Real-time monitoring
* Worker utilization analytics
* Benchmark visualization
* Mobile-responsive interface

---

## Features

### Distributed Workflow Orchestration

* Queue-based workflow execution
* Concurrent worker processing
* Asynchronous task execution
* Distributed workload management

### Self-Healing Recovery

* Automatic retry mechanisms
* Failure detection and recovery
* Worker crash simulation
* Recovery effectiveness tracking
* Configurable recovery strategies

### Fault Injection

* Random task failures
* Simulated worker crashes
* Reliability testing
* Recovery benchmarking

### Benchmarking

* Success rate measurement
* Throughput analysis
* Recovery latency tracking
* Worker utilization analytics
* Failure pattern evaluation

### Monitoring Dashboard

* System metrics
* Worker metrics
* Benchmark analytics
* Task distribution visualization
* Workflow execution controls
* Responsive UI

---

## Screenshots

### Dashboard Overview

Displays workflow execution metrics, worker utilization, benchmark results, and system health indicators.

### Worker Analytics

Visualizes completed, recovered, and failed tasks across distributed workers.

### Benchmark Dashboard

Tracks workflow throughput, recovery performance, and success rates.

### Workflow Execution Logs

Shows task execution, failure injection, recovery events, and workflow completion.

---

## Running Locally

### Clone Repository

```bash
git clone https://github.com/mridulapbk/self-healing-agent-infra.git
cd self-healing-agent-infra
```

### Start Infrastructure

```bash
docker compose up -d
```

### Run Backend

```bash
cd backend
go run cmd/server/main.go
```

Backend:

```text
http://localhost:8080
```

### Run Frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend:

```text
http://localhost:3000
```

---

## Recovery Strategy Configuration

The platform supports configurable recovery strategies through environment variables.

### Fixed Retry

```bash
cd backend
RECOVERY_STRATEGY=FIXED_RETRY go run cmd/server/main.go
```

### Exponential Backoff

```bash
cd backend
RECOVERY_STRATEGY=EXPONENTIAL_BACKOFF go run cmd/server/main.go
```

### No Recovery Baseline

```bash
cd backend
RECOVERY_STRATEGY=NO_RECOVERY go run cmd/server/main.go
```

### Supported Strategies

| Strategy            | Description                      |
| ------------------- | -------------------------------- |
| FIXED_RETRY         | Constant retry interval          |
| EXPONENTIAL_BACKOFF | Increasing delay between retries |
| NO_RECOVERY         | No retry attempts (baseline)     |

Default Strategy:

```text
FIXED_RETRY
```

---

## API Endpoints

### Start Workflow

```http
POST /workflow/start
```

### Worker Metrics

```http
GET /metrics/workers
```

### System Metrics

```http
GET /metrics/system
```

### Benchmark Metrics

```http
GET /metrics/benchmark
```

---

## Experimental Evaluation

The platform was evaluated under identical fault-injection conditions using multiple recovery strategies to measure workflow reliability and recovery effectiveness.

### Recovery Strategy Comparison

| Strategy            | Success Rate | Failure Rate | Recovered Tasks | Avg Recovery Time | Throughput         |
| ------------------- | ------------ | ------------ | --------------- | ----------------- | ------------------ |
| No Recovery         | 41.0%        | 59.0%        | 0               | 872.63 ms         | 3.41 workflows/sec |
| Exponential Backoff | 84.0%        | 16.0%        | 34              | 3156.92 ms        | 0.93 workflows/sec |
| Fixed Retry         | 89.0%        | 11.0%        | 44              | 2409 ms           | 1.23 workflows/sec |

### Large Scale Benchmark Results

| Metric                | Value   |
| --------------------- | ------- |
| Total Workflows       | 620     |
| Completed Tasks       | 285     |
| Recovered Tasks       | 239     |
| Failed Tasks          | 96      |
| Success Rate          | 84.52%  |
| Failure Rate          | 15.48%  |
| Average Recovery Time | 1709 ms |
| Concurrent Workers    | 3       |

### Worker Distribution

| Worker   | Processed | Completed | Recovered | Failed |
| -------- | --------- | --------- | --------- | ------ |
| Worker 1 | 196       | 84        | 78        | 34     |
| Worker 2 | 215       | 105       | 82        | 28     |
| Worker 3 | 209       | 96        | 79        | 34     |

### Key Findings

* Self-healing mechanisms improved workflow completion from 44.8% to 84.4% over a no-recovery baseline.
* Fixed Retry automatically recovered 44 failed workflows.
* The platform automatically recovered 239 workflow failures across 620 executions.
* Results demonstrate the effectiveness of automated recovery strategies in distributed workflow orchestration systems.

---

## Key Capabilities Demonstrated

* Distributed workflow orchestration
* Fault-tolerant execution
* Self-healing recovery mechanisms
* Worker crash simulation
* Queue-based architecture
* Benchmarking and performance evaluation
* Real-time monitoring dashboards
* Persistent workflow storage
* Concurrent worker processing
* Recovery strategy experimentation

---

## Future Enhancements

* Adaptive recovery strategy selection
* Kubernetes deployment
* Dynamic worker auto-scaling
* AI-based failure prediction
* Distributed tracing
* Prometheus integration
* Grafana dashboards
* GitHub Actions CI/CD pipeline

---

## Author

**Mridula Prabhakar**

Master of Science in Software Engineering Systems
Northeastern University

* Software Engineer
* Distributed Systems Enthusiast
* Backend & Cloud Development
* AI Infrastructure and Automation
