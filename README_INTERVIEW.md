# AI-Augmented DevSecOps Framework: Interview and Deployment Guide

This guide explains the project from the problem it addresses through its code, deployment assets, current limitations, and a way to present it in an interview. It complements the existing [README.md](README.md), which describes the project goals and research claims.

> **Status and accuracy note:** Treat this repository as a research prototype and implementation scaffold, not as a production-ready security platform. Some modules are implemented at prototype level; others contain placeholders, simulations, missing helper methods, or references to files that are not present. The benchmark numbers in the existing documentation are not independently established by the current benchmark code. This guide calls out those differences so that you can describe the work honestly.

## 1. The real-world problem

Cloud-native applications are made of APIs, containers, Kubernetes workloads, and CI/CD pipelines. Security signals are often split between separate scanners and runtime tools. This creates practical problems:

- A build can pass even when its deployment manifest or runtime behavior is risky.
- Static rules may miss suspicious behavior that was not known when the rules were written.
- Different detection tools can produce conflicting alerts, leaving operators to decide what to do.
- Security findings may not be connected to a measurable response, such as isolating a workload or restricting traffic.
- Teams need visibility into both security events and the health of the detection pipeline.

## 2. The proposed solution

The project explores a single DevSecOps workflow that combines:

1. **Event and data preparation** for security and network observations.
2. **Machine-learning-based classification** using an LSTM model for sequence-shaped input.
3. **Behavioral anomaly analysis** as a route toward detecting unfamiliar attacks.
4. **API and business-logic checks** to look for issues such as access-control mistakes and invalid workflows.
5. **Signal orchestration** to choose a response when components disagree.
6. **Policy and Kubernetes integration** to represent or apply containment actions.
7. **Telemetry and CI/CD assets** for monitoring, validation, and deployment.

The architecture is a useful design story, but the current code does not yet connect every step into a continuously running service. In particular, the main entry point initializes an LSTM model and exits; it does not expose an HTTP API or process a live event stream.

## 3. End-to-end walkthrough

Use this as the high-level explanation in an interview:

1. **A developer changes the application.** The GitHub Actions workflows and Jenkins pipeline describe CI steps such as tests, linting, security scans, image builds, and deployment.
2. **The CI security gate evaluates the change.** `src/pipeline/security_gate.py` is intended to inspect a Kubernetes manifest and fail unsafe deployments. In the current version it imports a missing module, so this path needs repair before it can run.
3. **Security data is prepared.** `data/ingestion.py` loads a CICIDS2017 CSV and scales the feature data. `src/pipeline/preprocessor.py` sketches configurable normalization, categorical encoding, and LSTM sequence creation, but calls helper methods that are not implemented in that file.
4. **The detector scores a sequence.** `src/detection/engine.py` builds a TensorFlow LSTM and emits alerts when its prediction exceeds a threshold. `src/main.py` contains a separate LSTM model definition and training routine. `src/detection/lstm_detector.py` is another model wrapper, but its model is not compiled or trained in that module.
5. **Other components add context.** The zero-day detector, API validator, and business-logic validator describe intended analysis paths. The zero-day detector and business-logic validator currently have unimplemented methods; the behavioral analyzer returns a random score; and API fuzzing references missing helper methods and `src/api/utils.py`.
6. **Signals are combined.** `src/orchestration/orchestrator.py` implements conservative, majority, and confidence-weighted selection and records a proposed policy action. Its Kubernetes response methods currently log the action rather than applying it. A second, simpler orchestrator exists in `src/pipeline/orchestrator.py`.
7. **Policies and runtime telemetry are connected.** `src/pipeline/k8s_integration.py` can create a Kubernetes NetworkPolicy when configured with in-cluster credentials. The policy engine is a stub. `src/telemetry/collector.py` can list pods, but CPU and memory values are placeholders set to zero.
8. **Results are evaluated and monitored.** `evaluation/benchmarking.py` defines metrics and comparison reports, but generates synthetic data and simulates framework and baseline results. `prometheus.yml` describes pod discovery; the collector defines Prometheus metrics, but does not start an HTTP metrics endpoint.

The intended flow is **CI checks -> event/data preparation -> detection -> signal resolution -> policy response -> telemetry/evaluation**. The current codebase contains parts of this flow, not a complete, integrated runtime.

## 4. Important components and what they currently do

| Area | Main files | Current implementation |
|---|---|---|
| Application entry point | `src/main.py` | Loads YAML and builds/trains an LSTM. `main()` initializes the model but does not start a server, monitor, or event loop. |
| Detection | `src/detection/engine.py`, `lstm_detector.py` | TensorFlow LSTM model implementations; model training/inference paths are not fully integrated with the rest of the service. |
| Zero-day / behavior analysis | `src/detection/zero_day_detector.py`, `src/detection/zero_day/behavioral_analyzer.py` | The detector methods are `pass`; the behavioral analyzer requires a baseline but currently returns a random score. |
| API validation | `src/api/security_validator.py`, `src/api/test_runner.py` | The validator outlines OpenAPI fuzzing, but imports missing helpers and calls methods that are not implemented. The test runner is a stub. |
| Business validation | `src/validation/business_logic_validator.py` | Rule container is present; sequence analysis is unimplemented. |
| Orchestration | `src/orchestration/orchestrator.py`, `src/pipeline/orchestrator.py` | The former has signal-weighting and action selection; action execution is logging-only. The latter simulates component signals. |
| Pipeline | `src/pipeline/message_bus.py`, `messagebus.py`, `preprocessor.py`, `policy_engine.py`, `security_gate.py` | Message-bus prototypes and pipeline sketches. There are two differently named message-bus modules; preprocessing and policy generation are incomplete; the security gate has a missing import. |
| Kubernetes | `src/pipeline/k8s_integration.py`, `deploy/kube.yml`, `config/policies/network-policies.yaml` | Kubernetes client integration and example policy/deployment resources. Cluster access, labels, RBAC, model storage, and image availability must be configured. |
| Telemetry | `src/telemetry/collector.py`, `prometheus.yml` | Pod listing and Prometheus metric definitions; actual resource measurements and a scrape endpoint are not implemented. |
| Research evaluation | `evaluation/benchmarking.py`, `models/results.md` | The results document reports research figures, but the benchmark script simulates parts of the comparison; do not present those figures as reproduced by this code without validating them. |

## 5. Repository map: every tracked project file

### Folder map

| Folder | What it contains |
|---|---|
| `.github/workflows/` | GitHub Actions CI and CI/CD workflow definitions. |
| `config/` | Runtime settings and logging configuration; `config/policies/` contains Kubernetes network-policy examples. |
| `data/` | Dataset ingestion code; no dataset is checked in. |
| `deploy/` | Kubernetes deployment example. |
| `docs/` | API reference and architecture assets; `docs/architecture/` contains rendered diagrams and their PlantUML sources. |
| `evaluation/` | Benchmark metric and comparison code. |
| `examples/` | Demonstration code, including simulated event processing. |
| `helm/ai-devsecops/` | Helm chart metadata and values; chart templates are not included. |
| `jenkins/` | Jenkins pipeline example. |
| `models/` | Written model results; no trained model artifact is included. |
| `src/api/` | API validation and test-runner prototypes. |
| `src/detection/` | LSTM, detection-engine, attack-generation, and zero-day detector code; `src/detection/zero_day/` contains behavioral analysis. |
| `src/orchestration/` | Security-signal conflict resolution and action selection. |
| `src/pipeline/` | Message-bus, preprocessing, policy, Kubernetes, gate, and orchestration code. |
| `src/telemetry/` | Kubernetes telemetry collection and Prometheus metric definitions. |
| `src/validation/` | Business-logic validation interface. |
| `terraform/` | AWS VPC and EKS infrastructure-as-code outline. |
| `tests/` | Python tests for the main LSTM/configuration module. |

### Root files

| File | Purpose |
|---|---|
| `.gitattributes` | Git attributes for repository files. |
| `.dockerignore` | Excludes local and generated files from Docker build context. |
| `.gitignore` | Excludes virtual environments, caches, datasets, model artifacts, logs, and build output from Git. |
| `README.md` | Project overview, intended features, architecture, and research claims. |
| `README_INTERVIEW.md` | This project walkthrough, honest implementation status, deployment notes, and interview/resume guidance. |
| `CONTRIBUTING.md` | Contribution workflow and coding/documentation conventions. |
| `LICENSE` | MIT license text and original copyright notice. |
| `requirements.txt` | Python runtime/data/ML dependencies. It does not list every package imported by all modules (for example, Click, Prometheus Client, Requests, and OpenAPI Spec Validator). |
| `Dockerfile` | Multi-stage Python container definition for the detector entry point. The entry point is not a long-running API, and the health check assumes `curl`, which is not installed in the runtime stage. |
| `Dockerfile.dev` | Development image that installs test/formatting tools and runs tests and formatters. |
| `docker-compose.yml` | Compose setup for detector, Redis, and PostgreSQL. The code does not currently wire the detector to those services or expose a complete service endpoint. |
| `docker-compose.dev.yml` | Development container configuration with source and Kubernetes config mounts. |
| `prometheus.yml` | Prometheus Kubernetes pod-discovery scrape configuration. |
| `setup.yml` | Ansible-style host setup outline for Docker, Python packages, and Prometheus. It begins with stray non-YAML text and references template and Compose services that are not present in this repository. |

### Application source: `src/`

| File | Purpose |
|---|---|
| `src/__init__.py` | Marks the source directory as a Python package. |
| `src/main.py` | YAML loader, `LSTMThreatDetector` definition, training method, and initialization entry point. |
| `src/cli.py` | Click CLI with `monitor` and `evaluate` commands; imports `SecurityGate`, whose current module has a missing dependency import. |
| `src/api/security_validator.py` | OpenAPI validation and API fuzzing outline; several helpers and its `.utils` import are absent. |
| `src/api/test_runner.py` | API test-runner interface; methods are placeholders. |
| `src/detection/attack_generator.py` | Begins a synthetic attack scenario generator; only a partial container-escape path is defined. |
| `src/detection/engine.py` | Builds a TensorFlow LSTM and converts high predictions into alert dictionaries. |
| `src/detection/lstm_detector.py` | Smaller LSTM wrapper with a prediction method; model setup/training is incomplete. |
| `src/detection/zero_day_detector.py` | Zero-day detector interface; `fit`, `detect`, and `score` are placeholders. |
| `src/detection/zero_day/behavioral_analyzer.py` | Behavioral feature/result data classes and a placeholder baseline/anomaly analyzer. |
| `src/orchestration/orchestrator.py` | Async security-signal orchestration and confidence/majority/conservative action selection. Kubernetes action methods currently only log. |
| `src/pipeline/k8s_integration.py` | Kubernetes API client helper for creating network policies and a basic pod-isolation policy. |
| `src/pipeline/message_bus.py` | Threaded queue-based message bus prototype with topic subscribers. |
| `src/pipeline/messagebus.py` | Alternate synchronous message-bus prototype; refers to helper methods not defined in the class. |
| `src/pipeline/orchestrator.py` | Lightweight component registry and simulated security-signal conflict resolution. |
| `src/pipeline/policy_engine.py` | Policy generation/enforcement interface; both methods are placeholders. |
| `src/pipeline/preprocessor.py` | Data-cleaning/scaling/encoding/sequence pipeline outline; referenced transformation helpers are absent. |
| `src/pipeline/security_gate.py` | CI deployment-manifest check for privileged containers; references missing `src.pipeline.detection_engine`. |
| `src/telemetry/collector.py` | Kubernetes pod enumeration and Prometheus metric definitions; CPU/memory collection is currently stubbed. |
| `src/validation/business_logic_validator.py` | Business-rule storage interface; transaction analysis is a placeholder. |

### Configuration and policies

| File | Purpose |
|---|---|
| `config/settings.yaml` | Application, database, security, model, and monitoring settings. It contains two `security` keys; YAML parsers generally keep only the later value. |
| `config/logging.yaml` | Console/file logging format and logger levels. The `logs/` directory must exist for the file handler. |
| `config/policies/network-policies.yaml` | Kubernetes default-deny, threat-isolation, and monitoring NetworkPolicy examples. Review carefully before applying: default-deny can interrupt workloads. |

### Data, examples, tests, and evaluation

| File | Purpose |
|---|---|
| `data/ingestion.py` | Loads a CICIDS2017 CSV, drops missing rows, separates labels, and scales features. A compatible data file is not included. |
| `examples/basic_usage.py` | Integrated-looking asynchronous demo; its detector, API, and policy decisions are explicitly simulated and should not be treated as actual ML detection. |
| `tests/__init__.py` | Test package marker. |
| `tests/test_main.py` | Tests LSTM initialization, training, config loading, and expected performance behavior on random synthetic data. |
| `evaluation/benchmarking.py` | Benchmark metric structures, confidence intervals, synthetic events, and simulated baseline comparisons. |
| `models/results.md` | Research-result narrative and reported metrics; it is not a model binary or proof that the checked-in benchmark reproduces those results. |

### Documentation and architecture: `docs/`

| File | Purpose |
|---|---|
| `docs/API_REFERENCE.md` | Component/API documentation. Some examples refer to APIs/classes not implemented in the current source. |
| `docs/architecture/Architecture.png` | Diagram of CI/CD, Kubernetes, monitoring, and ML components. |
| `docs/architecture/DataFlow.png` | Diagram of the proposed developer-to-runtime data flow. |
| `docs/architecture/SecurityModel.png` | Diagram of proposed security controls and monitoring. |
| `docs/architecture/devSecOps.png` | Diagram of the proposed overall DevSecOps system architecture. |
| `docs/architecture/data-flow.puml` | PlantUML source for the data-flow diagram. |
| `docs/architecture/high-level-architecture.puml` | PlantUML source for the high-level architecture. |
| `docs/architecture/latency_comparison.puml` | PlantUML source for a conceptual latency comparison. |
| `docs/architecture/roc_curve.puml` | PlantUML source for a conceptual ROC comparison. |
| `docs/architecture/security-model.puml` | PlantUML source for the security-model diagram. |

### Deployment and automation

| File | Purpose |
|---|---|
| `.github/workflows/ci.yml` | GitHub Actions test/coverage and Docker build/push workflow. The registry setup and image naming need project-specific configuration. |
| `.github/workflows/ci-cd.yml` | GitHub Actions lint, security scan, test, build, and AWS EKS/Helm deployment workflow. Requires secrets, compatible chart templates, image publishing, and deployment configuration. |
| `deploy/kube.yml` | Kubernetes Deployment example for two detector replicas; references an image and model-storage PVC that are not supplied here. |
| `helm/ai-devsecops/Chart.yaml` | Helm chart metadata. |
| `helm/ai-devsecops/values.yaml` | Default image, service, resource, monitoring, and model-storage values. A Helm `templates/` directory is absent, so this chart is not installable as-is. |
| `jenkins/jenkinsfile` | Jenkins pipeline outline for testing, security scanning, image publishing, and deployment. Uses placeholder registry credentials and references a manifest path absent from this tree. |
| `terraform/main.tf` | AWS VPC/EKS Terraform outline. It references variables and module outputs that are not declared in this file; review/complete before use. |

## 6. Local setup and run

The most direct runnable path is the model initialization entry point. It is a one-shot initialization, **not** a web service or continuous event processor.

### Windows PowerShell

Run these commands from the repository root (the directory containing `requirements.txt`):

```powershell
py -3.9 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m src.main
```

If PowerShell prevents activation, use the environment's Python directly:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m src.main
```

`src.main` reads `config/settings.yaml` relative to the current working directory, builds an LSTM, logs initialization, and exits. Run from the repository root. Dependencies for optional modules are incomplete in `requirements.txt`; install those only if you are developing those paths, and track them in the manifest as part of that work.

### macOS/Linux

```bash
python3.9 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m src.main
```

### Tests

`pytest` is not listed in `requirements.txt`, so install it in the development environment before running:

```bash
python -m pip install pytest
python -m pytest tests/ -q
```

The test file uses random labels and asserts accuracy thresholds, so results can vary; test success is not evidence that the documented research metrics have been reproduced.

## 7. Deployment: what is possible now

There are Docker, Compose, Kubernetes, Helm, Terraform, Jenkins, and GitHub Actions deployment artifacts, but the checked-in application is not currently a deployable long-running service:

- `python -m src.main` exits after model initialization, so the container has no server to keep running.
- The Docker health check uses `curl`, which is not installed in the runtime image.
- Compose starts Redis and PostgreSQL, but the application does not currently use them.
- `deploy/kube.yml` requires a published image and a `model-storage` PersistentVolumeClaim that is not included.
- The Helm chart has no `templates/` directory.
- The Terraform file lacks variable declarations, and the CI/CD workflows require credentials and project-specific image settings.

### Build and run the image as a one-shot initialization check

This verifies a container build/run, but is **not** a production deployment:

```powershell
docker build -t ai-devsecops-framework:local .
docker run --rm ai-devsecops-framework:local
```

The image's configured health check should not be relied on until `curl` is installed or the check is changed. A real deployment requires a persistent service entry point, complete pinned dependencies, a functioning health/metrics endpoint, configured secrets/RBAC/storage, and validated Kubernetes/Helm resources.

### Production deployment plan (after completing the missing pieces)

1. Implement and test a long-running service that accepts security events and returns/exports results.
2. Complete the detector, preprocessing, policy, and telemetry paths; add required packages to `requirements.txt`.
3. Build and scan a versioned image; publish it to a registry accessible to the target cluster.
4. Finish Kubernetes manifests or add Helm `templates/`; supply a PVC/model artifact only if the application actually needs it.
5. Create cluster credentials with least privilege, configure Kubernetes RBAC and namespace/network policies, and set secrets outside source control.
6. Validate the chart/manifests in a staging cluster, verify readiness and metrics, then promote the same immutable image tag to production.

Do not apply `config/policies/network-policies.yaml` to a shared/production cluster without reviewing its default-deny effects and testing the required ingress/egress first.

## 8. Interview explanation

### 30-second introduction

> “I built a research prototype for connecting AI-assisted threat detection with DevSecOps workflows in cloud-native environments. The design combines sequence-based LSTM detection, behavioral anomaly analysis, security-signal orchestration, Kubernetes policy concepts, and CI/CD/monitoring assets. I focused on how detection results can become operational decisions rather than isolated alerts. The current repository is a prototype: the LSTM model scaffolding and orchestration logic are present, while the API service, several detectors, and automated response paths still need production hardening.”

### A deeper explanation

- **Why this problem?** Cloud teams use multiple security tools, but detections, build gates, response actions, and operational telemetry are often disconnected.
- **What is the approach?** Normalize event features, score time-series behavior, collect component signals, resolve competing assessments, and represent the response as a policy action.
- **Why use an LSTM?** An LSTM can model ordered observations where sequence context matters. It requires representative labeled sequences, evaluation against leakage and imbalance, and calibration before operational use.
- **Why add orchestration?** A central decision point can apply a consistent policy when different detectors assign different confidence or severity.
- **How would you improve it next?** First close the integration gaps: establish a real event/API contract, implement deterministic detector tests, add missing dependencies and schema validation, make actions auditable and approval-aware, expose health/metrics endpoints, then validate deployment manifests in a staging cluster.

### Questions an interviewer may ask

1. **How do you reduce false positives?** Use a representative held-out dataset, tune thresholds against operational cost, measure precision/recall by attack class, and add review/feedback before automated containment.
2. **How do you avoid unsafe automated isolation?** Validate target identity, scope permissions narrowly, log every decision, make actions idempotent, provide rollback, and use human approval for high-impact actions.
3. **How do you evaluate model drift?** Monitor input and outcome distributions, compare to a labeled validation set, alert on drift, and retrain only through a versioned, tested process.
4. **What is implemented vs. proposed?** Point to the current implementation-status table above; explicitly call out simulations and stubs.
5. **What would you deliver next?** A tested event ingestion API and a reproducible evaluation that uses real, documented data and compares against a baseline without simulated results.

## 9. Resume wording: achievements already listed in README.md

For interview defense of each number, including metric definitions, evidence to prepare, and honest sample answers, see [README_ACHIEVEMENT_DEFENSE.md](README_ACHIEVEMENT_DEFENSE.md).

The existing project README reports the following results. The current benchmark implementation simulates parts of its evaluation, so treat these as **project-reported claims**, not independently verified results. Use a numeric claim on your resume only if you can explain and substantiate how it was measured.

- Prototyped AI-assisted cloud-native threat detection; project documentation reports a **95% attack detection rate** with **sub-2-second latency**.
- Documented **98% accuracy retention over six months** using the project's described adaptive-training approach.
- Designed the framework for a reported throughput of **10,000 security events per second**.
- Reported **0.95 precision** and **0.94 recall** in the project's benchmark comparison.
- Reported **33% lower CPU usage** than the comparison baseline in the project's performance results.

For an interview, be ready to distinguish documented targets/results from measurements reproduced by the checked-in code. If you have not independently validated a metric, describe it as a project-reported result or omit the number rather than presenting it as your own verified achievement.

## 10. Suggested next engineering steps

1. Define one event schema and make ingestion, detection, and CLI/API consumers agree on it.
2. Implement missing pipeline methods and replace random/simulated outputs with deterministic components.
3. Add missing runtime dependencies and use reproducible version constraints.
4. Add unit and integration tests for positive, negative, malformed, and failure-path inputs.
5. Make benchmark inputs and baselines reproducible; separate measured results from illustrative values.
6. Implement a real service entry point, health/readiness/metrics endpoints, and graceful shutdown.
7. Complete and validate Docker, Compose, Helm, Kubernetes, Terraform, and CI workflows before claiming production deployment.
