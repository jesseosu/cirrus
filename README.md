# Cirrus — Automated Fleet Health & Self-Healing Platform

A production-grade automated fleet health monitoring and self-healing platform for AWS EC2 instances. Cirrus detects failures, runs diagnostic playbooks, attempts automated remediation via SSM Run Command, and escalates to on-call engineers when auto-repair fails.

## Architecture

```
                         ┌──────────────────────────────────────────┐
                         │           EventBridge Scheduler          │
                         │            (every 60 seconds)            │
                         └─────────────────┬────────────────────────┘
                                           │
                                           ▼
                         ┌──────────────────────────────────────────┐
                         │         Health Check Lambda              │
                         │  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
                         │  │EC2 Status│ │CW Metrics│ │ Process │  │
                         │  └──────────┘ └──────────┘ │ Health  │  │
                         │  ┌──────────┐               └─────────┘  │
                         │  │Endpoint  │    ThreadPoolExecutor      │
                         │  │ Health   │    (parallel checks)       │
                         │  └──────────┘                            │
                         └─────────────────┬────────────────────────┘
                                           │
                          ┌────────────────┼────────────────┐
                          │                │                │
                          ▼                ▼                ▼
                   ┌─────────────┐  ┌───────────┐  ┌──────────────┐
                   │  CloudWatch  │  │EventBridge│  │  CloudWatch   │
                   │   Metrics    │  │  Events   │  │   Dashboard   │
                   └─────────────┘  └─────┬─────┘  └──────────────┘
                                          │
                         cirrus.health.UNHEALTHY / CRITICAL
                                          │
                                          ▼
               ┌───────────────────────────────────────────────────────┐
               │              Step Functions State Machine              │
               │                                                       │
               │  ┌──────────┐   ┌────────────┐   ┌──────────────┐   │
               │  │  Log      │──▶│ Diagnostics │──▶│ Classify     │   │
               │  │ Incident  │   │  Lambda     │   │ Failure      │   │
               │  └──────────┘   └────────────┘   └──────┬───────┘   │
               │                                          │           │
               │                    ┌─────────────────────┤           │
               │                    │                     │           │
               │                    ▼                     ▼           │
               │  ┌──────────────────────┐    ┌────────────────┐     │
               │  │ Remediation Lambda   │    │   Escalation   │     │
               │  │ ┌─────────────────┐  │    │    Lambda      │     │
               │  │ │ restart_service │  │    │  (SNS Alert)   │     │
               │  │ │ clear_disk      │  │    └────────────────┘     │
               │  │ │ reboot_instance │  │              │            │
               │  │ │ replace_instance│  │              ▼            │
               │  │ └─────────────────┘  │    ┌────────────────┐     │
               │  └──────────┬───────────┘    │   On-Call       │     │
               │             │                │   Engineer      │     │
               │             ▼                └────────────────┘     │
               │  ┌──────────────────┐                               │
               │  │ Verifier Lambda  │──── Healthy? ──▶ RESOLVED     │
               │  └──────────────────┘                               │
               │                                                     │
               └───────────────────────────────────────────────────────┘
                                          │
                                          ▼
                              ┌──────────────────┐
                              │    DynamoDB       │
                              │ cirrus-incidents  │
                              └──────────────────┘
```

## Key Features

| Feature | Implementation |
|---------|---------------|
| Fleet Health Monitoring | EventBridge-triggered Lambda runs 4 parallel health checks (EC2 status, CloudWatch metrics, SSM process liveness, HTTP endpoint) every 60s |
| Automated Diagnostics | Collects CloudWatch logs, metric snapshots, and system info via SSM; classifies failures into 7 types |
| Self-Healing Remediation | Maps failure types to actions (restart service, clear disk, reboot, replace instance) executed via SSM Run Command |
| Incident Tracking | Full lifecycle tracking in DynamoDB with TTL-based retention (DETECTED → DIAGNOSED → REMEDIATED → RESOLVED/ESCALATED) |
| Escalation Pipeline | Detailed SNS alerts with failure context, remediation history, and runbook links |
| Observability | CloudWatch dashboard with fleet health, remediation rates, latency percentiles; 4 configurable alarms |
| Infrastructure as Code | 3 independent CDK stacks with L3 constructs, least-privilege IAM |
| CI/CD | GitHub Actions pipeline: lint → type-check → test (80% coverage) → cdk synth → security audit |

## AWS Services Used

| Service | Purpose |
|---------|---------|
| Lambda | 6 Python 3.12 functions: health checker, diagnostics, remediator, verifier, escalation, incident logger |
| Step Functions | Orchestrates the detect → diagnose → remediate → verify → escalate pipeline with retries and error handling |
| EventBridge | Schedule-based health check triggers + event routing for unhealthy instance events |
| CloudWatch | Custom fleet health metrics, dashboard with 5 widget groups, 4 alarms |
| DynamoDB | Incident records with composite key (INSTANCE#id / INCIDENT#timestamp), PITR, TTL |
| Systems Manager | Run Command for executing Bash remediation scripts on instances |
| SNS | Escalation alerts and CloudWatch alarm notifications |
| IAM | Least-privilege roles per Lambda function |
| X-Ray | Distributed tracing across all Lambda functions |
| CDK | Python-based IaC with 3 stacks and 3 L3 constructs |

## How It Works

### Detect → Diagnose → Remediate → Verify → Escalate

1. **Detect**: Every 60 seconds, the Health Check Lambda discovers EC2 instances tagged `cirrus:monitored=true` and runs 4 checks in parallel using `ThreadPoolExecutor`. Results are aggregated into a health verdict per instance.

2. **Publish**: Healthy/unhealthy metrics are published to CloudWatch. Non-healthy verdicts are published as EventBridge events (`cirrus.health.UNHEALTHY` or `cirrus.health.CRITICAL`).

3. **Diagnose**: The Step Functions state machine catches these events, logs an incident to DynamoDB, and invokes the Diagnostics Lambda. It collects CloudWatch logs, metric snapshots, and system state via SSM, then classifies the failure (DISK_FULL, MEMORY_EXHAUSTED, CPU_SATURATED, PROCESS_CRASHED, INSTANCE_UNREACHABLE, ENDPOINT_DOWN, or UNKNOWN).

4. **Remediate**: Based on the failure classification, the Remediator Lambda selects and executes the appropriate action via SSM Run Command (or EC2 API for reboots/replacements). UNKNOWN failures skip directly to escalation.

5. **Verify**: After a 60-second stabilization wait, the Verifier Lambda re-runs health checks. If healthy → incident resolved. If still unhealthy → escalate.

6. **Escalate**: The Escalation Lambda formats a detailed alert and publishes to SNS. The incident is marked ESCALATED in DynamoDB.

## Project Structure

```
cirrus/
├── .github/workflows/
│   ├── cirrus-ci.yml              # Lint → type-check → test → cdk synth
│   └── cirrus-deploy.yml          # CDK deploy (manual trigger)
├── infra/
│   ├── app.py                     # CDK app entry point
│   ├── stacks/
│   │   ├── monitoring_stack.py    # Health check Lambda + EventBridge
│   │   ├── remediation_stack.py   # Step Functions + Lambdas + DynamoDB
│   │   └── observability_stack.py # Dashboard + alarms + SNS
│   ├── constructs/
│   │   ├── health_checker.py      # L3 construct for health check infra
│   │   ├── remediation_pipeline.py # L3 construct for Step Functions
│   │   └── fleet_dashboard.py     # L3 construct for CloudWatch dashboard
│   └── requirements.txt
├── src/
│   ├── shared/                    # Logger, models, constants, AWS client factory
│   ├── health_checker/            # Health check engine with 4 check types
│   ├── diagnostics/               # Diagnostic collection + failure classification
│   ├── remediator/                # 4 remediation actions + action selector
│   ├── verifier/                  # Post-remediation health verification
│   ├── escalation/                # SNS alert formatting
│   └── incident_logger/           # DynamoDB incident recording
├── scripts/
│   ├── remediation/               # 4 Bash scripts for SSM execution
│   └── seed_data/                 # DynamoDB Local seed data
├── tests/
│   ├── unit/                      # 8 test files covering all modules
│   └── integration/               # Step Functions state machine tests
├── Makefile                       # lint, typecheck, test, synth shortcuts
├── pyproject.toml                 # pytest, mypy, flake8 config
├── requirements.txt               # Runtime dependencies
└── requirements-dev.txt           # Dev dependencies
```

## Quick Start

### Prerequisites
- Python 3.12+
- Node.js 20+ (for CDK CLI)
- AWS CDK CLI (`npm install -g aws-cdk`)
- Docker (for DynamoDB Local)

### Setup
```bash
# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt
pip install -r infra/requirements.txt

# Run tests
make test

# Lint and type check
make lint
make typecheck

# Synthesize CloudFormation templates
make synth

# Deploy (requires AWS credentials)
make deploy
```

## Local Development

```bash
# Start DynamoDB Local
docker-compose up -d dynamodb-local

# Seed sample data
python scripts/seed_data/seed_incidents.py

# Run full quality suite
make lint && make typecheck && make test
```

## Remediation Playbooks

| Failure Type | Diagnostic Evidence | Remediation Action |
|---|---|---|
| DISK_FULL | disk_used_percent > 95%, df shows 99-100% | `clear_disk` — deletes old temp/log files, vacuums journald |
| MEMORY_EXHAUSTED | mem_used_percent > 95%, OOM killer in dmesg | `reboot_instance` — EC2 API reboot with running-state verification |
| CPU_SATURATED | CPUUtilization > 95% sustained | `reboot_instance` — EC2 API reboot |
| PROCESS_CRASHED | Failed systemd services, segfault/core dump in logs | `restart_service` — SSM systemctl restart + verification |
| INSTANCE_UNREACHABLE | EC2 status checks failed, no SSM connectivity | `replace_instance` — terminate for ASG replacement |
| ENDPOINT_DOWN | HTTP health check failures, connection refused in logs | `restart_service` — SSM systemctl restart |
| UNKNOWN | No specific pattern detected | Immediate escalation (no auto-remediation) |

## Observability

### CloudWatch Dashboard
- **Fleet Health Summary**: Healthy vs unhealthy instance count over time
- **Health Check Latency**: p50, p95, p99 check duration
- **Remediation Success/Failure Rate**: Tracking auto-heal effectiveness
- **Active Incidents**: Current open incident count
- **Per-Instance Health**: Individual instance status

### CloudWatch Alarms
| Alarm | Condition | Severity |
|-------|-----------|----------|
| Unhealthy Instances Warning | Any unhealthy instance for 5 minutes | WARNING |
| Unhealthy Instances Critical | >3 unhealthy instances for 5 minutes | CRITICAL |
| Health Check Lambda Errors | >5 errors in 5 minutes | WARNING |
| Step Functions Failures | Any execution failure | CRITICAL |

### X-Ray Tracing
All Lambda functions have active X-Ray tracing enabled. Trace IDs are correlated in structured JSON log entries for end-to-end request tracking across the remediation pipeline.

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Step Functions over direct Lambda chaining** | Provides built-in retry, error handling, wait states, and visual debugging. The remediation pipeline has complex branching (UNKNOWN → escalate, remediation failure → escalate) that's cleaner in a state machine. |
| **SSM Run Command over Lambda-based remediation** | Remediation requires executing commands directly on instances (systemctl, disk cleanup). SSM Run Command is the AWS-native approach and doesn't require SSH key management or security group changes. |
| **EventBridge over SNS for routing** | EventBridge supports content-based filtering on event detail types (UNHEALTHY vs CRITICAL), enabling selective routing without multiple SNS topics. Also provides built-in schema registry and archive/replay. |
| **DynamoDB for incident log** | Serverless, auto-scaling, supports TTL for automatic cleanup of resolved incidents. Composite key (INSTANCE#id / INCIDENT#timestamp) enables efficient queries by instance or time range. |
| **ThreadPoolExecutor for parallel checks** | Lambda's single-process model benefits from I/O-bound parallelism. Running 4 health checks concurrently per instance reduces total check time from ~40s to ~10s. |
| **Pydantic v2 for data models** | Provides runtime validation, serialization, and type safety for event payloads flowing between Lambda functions via Step Functions. |
| **L3 CDK constructs** | Encapsulates related resources (Lambda + IAM + EventBridge) into reusable, testable units. Demonstrates CDK best practices beyond basic L1/L2 usage. |

## CI/CD Pipeline

```
Push to any branch
        │
        ▼
┌───────────────┐
│  flake8 lint  │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  mypy check   │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ pytest + cov  │──── Fail if < 80% coverage
└───────┬───────┘
        │
        ▼
┌───────────────┐
│   cdk synth   │──── Validates infrastructure compiles
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  pip audit    │──── Security vulnerability scan
└───────────────┘

Manual trigger (workflow_dispatch)
        │
        ▼
┌───────────────┐
│  Run tests    │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  cdk deploy   │──── Deploys all 3 stacks
│    --all      │
└───────────────┘
```

## License

MIT
