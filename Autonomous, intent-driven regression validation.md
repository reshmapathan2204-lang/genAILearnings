# Intent-Driven QA Agent — Technical Architecture & System Design (Production-Grade)
*Author:* Chief Product Architect & AI Agentic Architect (QA Domain)  
*Primary Goal:* Autonomous, intent-driven regression validation: *Feature(Tag) + Version → Fresh K8 SUT → Minimal Regression Scope → Execute → Human-readable Confidence Report*  
*SLO:* End-to-end validation in *< 1 hour* (request → instance ready → execution → report) (validated requirement from user input)  
*Primary Feature Identifier:* *Cucumber Tag, pattern *@TC-FeatureName* *(validated)  
*Version Formats Supported:* 24.1, 24.2, 25.1.2, 25.1.4 (validated)  
*Build Source of Truth:* Grafana dashboard *CDH CI Build Status: https://cockpit.pega.com/d/9ZcmVijWz/cdh-ci-build-status?... *(validated)  
*Execution Policy:*  
- Default *parallel* execution  
- If request includes *@NBADregression* or *@NBADregression2, execute **sequentially* in *top-to-bottom feature order* (validated; NBAD sequential job pattern exists) [1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)[2](https://pegasystems.sharepoint.com/sites/SP-GlobalBusinessUnits/_layouts/15/Doc.aspx?sourcedoc=%7B2C1A3DEC-1E22-4270-B8A9-062CE56546E5%7D&file=Pega%20GenAI%20-%20Architecture,%20Data%20Flow%20%26%20Security.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)  
*Execution Backends:*  
- *Primary:* Selenium (Java Cucumber)  
- *Add-on:* Playwright (TypeScript Cucumber) (validated add-on requirement; both backends exist in automation ecosystem) [3](https://knowledgehub.pega.com/COPLINSV:Onboarding_control-plane_services_via_pegacloud_cli)[4](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAATWmYgbAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  

---

## 0. Non-Hallucination Contract (Strict Grounding)
This design uses only *validated inputs* from:
- User-provided decisions (tags, versions, SLO, policies, confidence factors)
- Enterprise sources found via search:
  - Tag-driven execution & suite tags documentation [5](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAAUN3ALlAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)[1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)  
  - NBAD sequential runtime context & splitting practices [2](https://pegasystems.sharepoint.com/sites/SP-GlobalBusinessUnits/_layouts/15/Doc.aspx?sourcedoc=%7B2C1A3DEC-1E22-4270-B8A9-062CE56546E5%7D&file=Pega%20GenAI%20-%20Architecture,%20Data%20Flow%20%26%20Security.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)  
  - Grafana token/service-account usage guidance [6](https://pegasystems.sharepoint.com/sites/SP-InfrastructureProjectManagement/_layouts/15/Doc.aspx?sourcedoc=%7BEC2F5880-F84E-492F-A85A-4966366E1AB5%7D&file=IT_Infra_Nov_2025.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)[7](https://knowledgehub.pega.com/CLDRLSEN:Cloud3_Staging_Promotion_Pipeline)[8](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAASaOsR1AAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  
  - Helm/self-service K8 deployment references [9](https://pegasystems.sharepoint.com/sites/InternalDevelopmentExperience/_layouts/15/Doc.aspx?sourcedoc=%7B52FF029B-F83F-432B-AA33-99ACA4A10CB1%7D&file=Link%20between%20git%20repository%20and%20Agile%20Studio.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  
  - Agile Studio API integration references [10](https://pegasystems.sharepoint.com/sites/SP-PRD-1-1CustomerEngagementAlliance/_layouts/15/Doc.aspx?sourcedoc=%7B19873B87-3751-4DA5-848F-96F9ECE7A2BA%7D&file=Dtrans%20QA%20syncup.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  

Where the organization-specific mechanism is *unknown* (e.g., exact provisioning API endpoint), this document explicitly marks it as an *ASSUMPTION / INTERFACE TO CONFIRM* and provides a modular interface that can be plugged into your actual systems.

---

## 1. Problem Statement & Product Requirements

### 1.1 Problem
Current UI automation runs are long and environment-heavy. Desired system must let a user say:
> “Validate <feature> in <version>”  
and the system should do everything end-to-end autonomously.

### 1.2 Requirements (Validated)
1. Accept input: *Cucumber tag + version*
2. Provision *fresh* Kubernetes SUT per request (validated)
3. Resolve *latest green build* for requested version from *Grafana* dashboard (validated)
4. Identify minimal regression scope using existing *tag assets/taxonomy* (validated; tag-driven suites exist) [5](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAAUN3ALlAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)[1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)  
5. Execute tests:
   - Parallel by default
   - Sequential for @NBADregression, @NBADregression2 [1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)[2](https://pegasystems.sharepoint.com/sites/SP-GlobalBusinessUnits/_layouts/15/Doc.aspx?sourcedoc=%7B2C1A3DEC-1E22-4270-B8A9-062CE56546E5%7D&file=Pega%20GenAI%20-%20Architecture,%20Data%20Flow%20%26%20Security.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)  
6. Generate report including:
   - Instance details, build details, scope, duration, pass/fail, confidence with reasoning
7. Optimize for speed: *instance creation → validation < 1 hour* (validated)

---

## 2. High-Level Architecture (Fresh Design)

### 2.1 System Overview
The solution is a *multi-agent orchestrated platform* composed of:
- *Intent Router Agent* (NL → structured request)
- *Build Resolver Agent* (version → latest green build)
- *Provisioning Agent* (fresh K8 SUT, deploy build, configure)
- *Scope Minimizer Agent* (tag → tests → minimal scope)
- *Execution Agent* (parallel/sequential execution, Selenium primary; Playwright optional)
- *Triage & Confidence Agent* (flakiness + defect correlations + env health + pass rate)
- *Report Agent* (human readable report, artifacts, links)

The agents run under a deterministic orchestration layer to ensure:
- Idempotency
- Retries (only for infra failures)
- Auditability
- Policy control & security

### 2.2 Core Components
1. *QA Orchestrator API*
   - Receives requests from Copilot/CLI/UI
   - Persists run intent and state
2. *Workflow Engine*
   - Executes steps as a DAG with retries and timeouts
   - Supports step-level parallelism (e.g., indexing + provisioning)
3. *Knowledge & Retrieval Layer (RAG)*
   - Uses vector embeddings (V) and partial vector embeddings (PV)
4. *Provisioning & Deployment Layer*
   - Fresh K8 environment provisioning + app deployment
   - Helm-based PRPC/CDH deployment is a validated pattern in internal docs [9](https://pegasystems.sharepoint.com/sites/InternalDevelopmentExperience/_layouts/15/Doc.aspx?sourcedoc=%7B52FF029B-F83F-432B-AA33-99ACA4A10CB1%7D&file=Link%20between%20git%20repository%20and%20Agile%20Studio.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  
5. *Execution Runners*
   - Selenium Java Cucumber runner exists in pipeline execution logs [4](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAATWmYgbAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  
   - Playwright cucumber project exists alongside Selenium [3](https://knowledgehub.pega.com/COPLINSV:Onboarding_control-plane_services_via_pegacloud_cli)  
6. *Observability & Evidence Store*
   - Logs, metrics, traces, and evidence artifacts per run
7. *Results & Confidence Service*
   - Calculates confidence and produces a structured report

---

## 3. Data & Knowledge Inputs (As Required)

### 3.1 Knowledge Sources & Embeddings Strategy
The solution implements the following stores:

#### (V) Requirements Store (Vector)
- Instance preparation details
- Env configs
- DB configs
- Resources needed during setup
- System dependencies

> *Note:* Current enterprise search revealed a detailed “Instance_Setup_Guide” describing setup/config steps and artifacts import patterns. [11](https://pegasystems-my.sharepoint.com/personal/reshma_pathan_in_pega_com/_layouts/15/Doc.aspx?sourcedoc=%7B3A849055-9011-4F5B-ACF2-BFF149C97488%7D&file=CDH-Applications-Merge-RAP-Process.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  
This document (and similar) should be embedded as part of Requirements Store.

#### (V) Existing Test Assets Store (Vector + Symbolic Index)
- Feature files stored in Git (GitHub/Bitbucket)
- Tag extraction and normalization
- Exact feature identified by unique tags (@TC-FeatureName)
- Suite tags such as @cismoke, @ciregression, and NBAD tags appear in internal docs [1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)[5](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAAUN3ALlAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  

*Implementation detail:* Build a *dual-index*:
- *Symbolic Index:* tag → file path → scenarios → step definitions pointer
- *Vector Index:* semantic search over feature metadata and descriptions

> Internal docs show tag-based selection criteria is already used (example returns @TC-Segment-09). [5](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAAUN3ALlAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)

#### (PV) Code Metadata Store (Partial Vector)
- Only metadata: branches, PRs, modules, commit messages, owners
- No raw source code embedding

#### (V) Technical Documentation Store (Vector)
- Pega Knowledge Hub, Agile Studio docs, Swagger APIs, Architecture docs  
Example: Agile Studio integration doc includes API endpoint patterns and navigation to API docs [10](https://pegasystems.sharepoint.com/sites/SP-PRD-1-1CustomerEngagementAlliance/_layouts/15/Doc.aspx?sourcedoc=%7B19873B87-3751-4DA5-848F-96F9ECE7A2BA%7D&file=Dtrans%20QA%20syncup.docx&action=default&mobileredirect=true&DefaultItemOpen=1)

#### (V) Defect Database Store (Vector)
- Agile Studio defect records (title, tags, component, affected version, root-cause label, linked tests)

#### (V) Release Notes / Test Strategy Store (Vector)
- Change summaries
- Known limitations
- Test strategy notes

#### (V) Latest Stable Build URLs Store (Vector + Structured Cache)
- Primary resolution via Grafana dashboard (validated)
- Store: version → build id → artifact URL → timestamp → “green” status

---

## 4. Security, Identity, and Access (Production-Grade)

### 4.1 Authentication for Grafana
*Recommendation:* Use *Grafana service account / access policy token* (read-only) via Bearer auth.
This approach is directly supported by internal Grafana API guidance and token patterns. [6](https://pegasystems.sharepoint.com/sites/SP-InfrastructureProjectManagement/_layouts/15/Doc.aspx?sourcedoc=%7BEC2F5880-F84E-492F-A85A-4966366E1AB5%7D&file=IT_Infra_Nov_2025.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)[7](https://knowledgehub.pega.com/CLDRLSEN:Cloud3_Staging_Promotion_Pipeline)[8](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAASaOsR1AAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  

### 4.2 Secret Management
- Store tokens in enterprise secret vault
- Short-lived tokens where possible
- Rotate regularly
- Per-environment least privilege

### 4.3 Audit & Compliance
- Every run persists:
  - requester identity
  - requested tag/version
  - build resolution evidence
  - provisioning logs
  - execution logs
  - report output
- Immutable audit trail

---

## 5. Detailed Workflow (End-to-End)

### Phase A — Intake & Intent Normalization
*Input:*
- feature_tag: @TC-FeatureName (required)
- version: 24.2 / 25.1.4 (required)
- optional: runner_preference: selenium | playwright | both (default selenium)

*Output:*
- normalized request object:
  - canonical tag
  - normalized version
  - execution mode (parallel/sequential policy)

---

### Phase B — Build Resolution (Version → Latest Green Build)
1. Build Resolver Agent queries Grafana for selected version’s latest “green build”
2. Extract:
   - build identifier
   - artifact reference URL(s)
   - supporting metadata for report

*Validated dependency:* user-provided Grafana dashboard URL is the source-of-truth.

*ASSUMPTION / INTERFACE TO CONFIRM:*  
- Whether Grafana is queried via:
  - dashboard API + panel query
  - underlying datasource query API
The design supports either via a “BuildResolverAdapter”.

---

### Phase C — Fresh SUT Provisioning on Kubernetes
*Target:* Kubernetes-based SUT cluster (EKS/Cloud) (validated)

*Validated reference pattern:* internal documentation describes fully automated self-service provisioned Kubernetes instances and PRPC deployment using Helm charts from Git. [9](https://pegasystems.sharepoint.com/sites/InternalDevelopmentExperience/_layouts/15/Doc.aspx?sourcedoc=%7B52FF029B-F83F-432B-AA33-99ACA4A10CB1%7D&file=Link%20between%20git%20repository%20and%20Agile%20Studio.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  

*Provisioning steps (abstract interface):*
1. Provision namespace / ephemeral environment
2. Deploy PRPC/CDH + dependencies using Helm
3. Apply configuration:
   - environment configs
   - DB configs
   - operator/persona bootstrap
4. Import required resources (as defined by Requirements Store)
5. Health checks:
   - PRPC login endpoint
   - DB connectivity checks (if required)
   - “ready to test” gate

*ASSUMPTION / INTERFACE TO CONFIRM:*
- Exact provisioning API/portal used internally (EIS Labs / internal platform / custom).  
This architecture isolates it behind ProvisioningAdapter.

---

### Phase D — Minimal Regression Scope Selection
*Validated policy:* run tag-targeted tests directly (no smoke gate).  
*Scope Minimizer Agent:*
1. Locate all scenarios bound to feature_tag in the Test Assets Store
2. Expand scope based on taxonomy rules:
   - For @NBADregression / @NBADregression2, enforce sequential execution [1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)[2](https://pegasystems.sharepoint.com/sites/SP-GlobalBusinessUnits/_layouts/15/Doc.aspx?sourcedoc=%7B2C1A3DEC-1E22-4270-B8A9-062CE56546E5%7D&file=Pega%20GenAI%20-%20Architecture,%20Data%20Flow%20%26%20Security.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)  
3. Optionally include “supporting setup” feature(s) if the tag references shared system setup
   - Tag-driven selection patterns exist and should be stored in the symbolic index [5](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAAUN3ALlAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  

*Output:*
- Ordered list of feature/scenario executions
- Execution mode:
  - parallel: shard by feature/scenario
  - sequential: maintain file order (top-to-bottom)

---

### Phase E — Test Execution (Selenium Primary, Playwright Add-on)
*Selenium (Primary):*
- Execute Cucumber tests via your existing Java/Gradle runner (observed in execution logs). [4](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAATWmYgbAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  

*Playwright (Add-on):*
- Execute TypeScript Cucumber tests via Playwright project configuration (exists alongside Selenium). [3](https://knowledgehub.pega.com/COPLINSV:Onboarding_control-plane_services_via_pegacloud_cli)  

*Execution strategy:*
- Parallel mode:
  - compute optimal shard count based on SLO and cluster capacity
  - run shards concurrently
- Sequential mode (@NBADregression, @NBADregression2):
  - execute strictly in order
  - stop-on-critical-failure policy configurable (default: continue to gather evidence)

---

### Phase F — Triage, Correlation, and Confidence Scoring
*User-validated confidence factors:*
- % tests passed
- historical flakiness
- defect correlations
- environment health signals (validated)

*Confidence Service responsibilities:*
1. Parse test results (JUnit/JSON/Cucumber reports)
2. Classify failures into:
   - probable test flake (based on flakiness DB)
   - probable environment issue (based on env health signals)
   - probable product defect (based on defect correlations)
3. Compute confidence score

*Design (suggested, not sourced):*  
Use a 0–100 score with weighted components:
- Pass rate: 50%
- Flakiness penalty: 20%
- Defect correlation penalty: 20%
- Environment health penalty: 10%

> Weights are configurable and must be validated with QA leadership.

---

### Phase G — Human-Readable Report Generation (Required Output)
The Report Agent produces:
- Instance details:
  - environment name/namespace
  - base URLs
  - configuration summary
- Build details:
  - version requested
  - green build resolved (id + artifact)
  - resolution evidence (Grafana query reference)
- Functionality executed:
  - tag requested
  - resolved list of scenarios/features
- Minimal regression scope selected:
  - why included/excluded
  - parallel/sequential reasoning
- Execution time:
  - provisioning time
  - execution time
  - reporting time
- Pass/Fail summary:
  - total scenarios
  - passed/failed/skipped
- Confidence score + reasoning (explicitly referencing the 4 factors)

---

## 6. Optimization Strategy for < 1 Hour SLO

### 6.1 Time Budget Model (Design Target)
To meet <1 hour:
- Provisioning + deploy must be optimized (ephemeral namespace, fast Helm deploy)
- Tests must be minimized and parallelized whenever allowed

### 6.2 Key Techniques (Design Suggestions)
1. *Fast-path provisioning*
   - Use a pre-created cluster with dynamic namespaces (fresh per request still satisfied)
   - Use immutable base images; deploy deltas only
2. *Parallel execution by default*
   - Sharding across runners
3. *Tag-scope minimization*
   - Execute only tag-targeted tests
4. *Early environment readiness checks*
   - Fail fast if infra not healthy
5. *Evidence-driven retry*
   - Retry infra failures once; do not retry product failures by default

---

## 7. Observability & Operations

### 7.1 Logging
- Correlation ID per request
- Structured logs per step:
  - build resolution
  - provisioning
  - execution
  - report generation

### 7.2 Metrics
- Provision time p50/p90
- Execution time p50/p90
- Failure category breakdown
- Confidence score distribution

### 7.3 Run Artifacts
- Store:
  - deployment logs
  - test reports
  - screenshots/videos (where applicable)
  - environment snapshot metadata

---

## 8. Failure Modes & Recovery

### 8.1 Common Failure Classes
- *Build not found / no green build*
- *Provisioning failure*
- *Environment readiness failure*
- *Execution runner failure*
- *Test failures (product/test/env)*

### 8.2 Recovery Policies
- Infra/provision errors:
  - 1 retry with backoff
- Build resolution errors:
  - retry Grafana query
- Execution errors:
  - rerun only failed shard if infra stable

---

## 9. Phased Delivery Plan (Clearly Divided)

### Phase 1 — MVP (Tag → Build → Fresh SUT → Run Selenium → Report)
*Scope*
- Inputs: tag + version
- Resolve build via Grafana
- Provision fresh K8 namespace + deploy build (via adapter)
- Execute Selenium tag-targeted suite
- Report with pass rate and basic confidence

*Data*
- Index feature files + tags
- Store run results and durations

*Exit Criteria*
- Works for initial functional areas:
  - Contact Policies
  - NBAD taxonomy properties
  - Context Dictionary
  - Personas (validated initial scope)

---

### Phase 2 — Confidence Intelligence (Flakiness + Defects + Env Health)
*Add*
- Flakiness store and trend computation
- Agile Studio defect ingestion and correlation
- Environment health signals integration
- Enhanced confidence reasoning

*Validated source relevance*
- Agile Studio API integration patterns exist [10](https://pegasystems.sharepoint.com/sites/SP-PRD-1-1CustomerEngagementAlliance/_layouts/15/Doc.aspx?sourcedoc=%7B19873B87-3751-4DA5-848F-96F9ECE7A2BA%7D&file=Dtrans%20QA%20syncup.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  

---

### Phase 3 — Dual Runner Support (Playwright Add-on)
*Add*
- Optional Playwright execution backend
- Comparative reporting (Selenium vs Playwright)
- Migration support: keep parity checks

*Evidence*
- Playwright configuration exists in current automation ecosystem [3](https://knowledgehub.pega.com/COPLINSV:Onboarding_control-plane_services_via_pegacloud_cli)  

---

### Phase 4 — Scale & Governance
*Add*
- Multi-tenant execution
- Policy engine
- Role-based access control
- Cost controls
- Organization-wide dashboards

---

## 10. Interfaces (Adapters) — Implementable Without Guessing

### 10.1 BuildResolverAdapter
- resolveGreenBuild(version) -> {buildId, artifactUrls, metadata, evidence}  
- Backend: Grafana API using service-account token [6](https://pegasystems.sharepoint.com/sites/SP-InfrastructureProjectManagement/_layouts/15/Doc.aspx?sourcedoc=%7BEC2F5880-F84E-492F-A85A-4966366E1AB5%7D&file=IT_Infra_Nov_2025.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)[7](https://knowledgehub.pega.com/CLDRLSEN:Cloud3_Staging_Promotion_Pipeline)[8](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAASaOsR1AAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  

### 10.2 ProvisioningAdapter
- provisionSut(buildArtifact, configProfile) -> {sutUrl, namespace, credentialsRef, readiness}  
- Backend: Helm/self-service K8 provisioning pattern [9](https://pegasystems.sharepoint.com/sites/InternalDevelopmentExperience/_layouts/15/Doc.aspx?sourcedoc=%7B52FF029B-F83F-432B-AA33-99ACA4A10CB1%7D&file=Link%20between%20git%20repository%20and%20Agile%20Studio.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  

### 10.3 TestExecutionAdapter
- run(tagExpression, mode, sutUrl, runnerType) -> {reports, artifacts, duration}
- RunnerType: selenium (primary), playwright (add-on) [4](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAATWmYgbAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)[3](https://knowledgehub.pega.com/COPLINSV:Onboarding_control-plane_services_via_pegacloud_cli)  

### 10.4 DefectAdapter (Phase 2)
- searchDefects(tag/version) -> defectSet
- Backend: Agile Studio APIs (pattern reference) [10](https://pegasystems.sharepoint.com/sites/SP-PRD-1-1CustomerEngagementAlliance/_layouts/15/Doc.aspx?sourcedoc=%7B19873B87-3751-4DA5-848F-96F9ECE7A2BA%7D&file=Dtrans%20QA%20syncup.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  

---

## 11. Agent Prompt Blueprint (Ready for Copilot)

### 11.1 System Prompt (Drop-in)
You are an *Intent-Driven QA Agent*.  
You must accept a request containing:
- feature_tag (Cucumber tag, e.g., @TC-FeatureName)
- version (e.g., 24.2, 25.1.4)

Your job is to:
1. Resolve the *latest green build* for the given version from the Grafana dashboard:
   - https://cockpit.pega.com/d/9ZcmVijWz/cdh-ci-build-status?...
2. Provision a *fresh Kubernetes SUT instance* and deploy that build.
3. Determine *minimal regression scope* from existing test assets:
   - Run tag-targeted tests directly
   - If tag includes @NBADregression or @NBADregression2, execute *sequentially; otherwise execute **in parallel*.
4. Execute tests using:
   - Selenium Java Cucumber (primary)
   - Playwright (optional add-on if requested)
5. Generate a *human-readable report* including:
   - Instance details
   - Build details
   - Scope selected
   - Total time
   - Pass/Fail summary
   - Confidence score with reasoning based on:
     - % tests passed
     - historical flakiness
     - defect correlations
     - environment health signals

*Constraints*
- Do not invent data.
- If a required integration detail is unavailable (e.g., provisioning endpoint), call the corresponding adapter and surface a deterministic error with required missing fields.

### 11.2 User Prompt Template
### 11.3 Output Template (Report)
- Request: <feature_tag>, <version>
- Build resolved: <buildId>, <artifactUrls>
- Instance: <sutUrl>, <namespace>, <configProfileSummary>
- Scope:
  - Mode: parallel|sequential (reason)
  - Scenarios executed: list
- Results:
  - Total: X, Passed: Y, Failed: Z, Skipped: N
  - Top failures (classified): product/test/env
- Time:
  - Provision: A
  - Execution: B
  - Total: A+B+...
- Confidence:
  - Score: S/100
  - Reasoning:
    - pass rate contribution
    - flakiness adjustment
    - defect correlation adjustment
    - env health adjustment
- Evidence links:
  - Grafana build status evidence
  - Deployment logs
  - Test reports
  - Artifacts

---

## 12. Explicit Assumptions / Items to Confirm (Non-Hallucinatory)
1. *Grafana API query method* for retrieving “latest green build” (dashboard panel API vs datasource query). The token-based access pattern is validated; the exact endpoint must be finalized in implementation. [6](https://pegasystems.sharepoint.com/sites/SP-InfrastructureProjectManagement/_layouts/15/Doc.aspx?sourcedoc=%7BEC2F5880-F84E-492F-A85A-4966366E1AB5%7D&file=IT_Infra_Nov_2025.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)[7](https://knowledgehub.pega.com/CLDRLSEN:Cloud3_Staging_Promotion_Pipeline)[8](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAASaOsR1AAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  
2. *Provisioning API/portal* for fresh K8 SUT creation. Helm-based deployment + self-service provisioned K8 instances is a validated internal pattern; the exact control plane endpoint must be mapped. [9](https://pegasystems.sharepoint.com/sites/InternalDevelopmentExperience/_layouts/15/Doc.aspx?sourcedoc=%7B52FF029B-F83F-432B-AA33-99ACA4A10CB1%7D&file=Link%20between%20git%20repository%20and%20Agile%20Studio.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  
3. *Defect correlation integration* with Agile Studio: API usage patterns exist; final endpoints/permissions must be confirmed. [10](https://pegasystems.sharepoint.com/sites/SP-PRD-1-1CustomerEngagementAlliance/_layouts/15/Doc.aspx?sourcedoc=%7B19873B87-3751-4DA5-848F-96F9ECE7A2BA%7D&file=Dtrans%20QA%20syncup.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  

---

## 13. Appendix — Grounded Internal Signals (Why these design choices are realistic)
- Tag-based test selection is already part of internal practice (“selection criteria returns tag string”, custom tags for grouping). [5](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAAUN3ALlAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)[1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)  
- NBAD regression is known to be sequential and runtime-heavy; splitting has been discussed internally. [2](https://pegasystems.sharepoint.com/sites/SP-GlobalBusinessUnits/_layouts/15/Doc.aspx?sourcedoc=%7B2C1A3DEC-1E22-4270-B8A9-062CE56546E5%7D&file=Pega%20GenAI%20-%20Architecture,%20Data%20Flow%20%26%20Security.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)[1](https://knowledgehub.pega.com/OBSERVE:How_to_access_Grafana_enterprise_%26_Pagerduty)  
- Grafana automation patterns rely on service account tokens / API keys. [6](https://pegasystems.sharepoint.com/sites/SP-InfrastructureProjectManagement/_layouts/15/Doc.aspx?sourcedoc=%7BEC2F5880-F84E-492F-A85A-4966366E1AB5%7D&file=IT_Infra_Nov_2025.pptx&action=edit&mobileredirect=true&DefaultItemOpen=1)[7](https://knowledgehub.pega.com/CLDRLSEN:Cloud3_Staging_Promotion_Pipeline)[8](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAASaOsR1AAA%3d&exvsurl=1&viewmodel=ReadMessageItem)  
- Kubernetes-based, self-service provisioned PRPC stacks and Helm deployments are documented internally. [9](https://pegasystems.sharepoint.com/sites/InternalDevelopmentExperience/_layouts/15/Doc.aspx?sourcedoc=%7B52FF029B-F83F-432B-AA33-99ACA4A10CB1%7D&file=Link%20between%20git%20repository%20and%20Agile%20Studio.docx&action=default&mobileredirect=true&DefaultItemOpen=1)  
- Both Selenium and Playwright execution tracks appear in your automation ecosystem logs/config. [4](https://outlook.office365.com/owa/?ItemID=AAMkADYzN2JkOTViLTZkOTktNGNhOS1iOWU0LWVlMDZkY2ZjNWI4ZgBGAAAAAAAxPtLckBYCSK0Sp9g%2frJH9BwD5YP9GrqNIRJLiTZkOwkDOAAAAAAEMAAD5YP9GrqNIRJLiTZkOwkDOAATWmYgbAAA%3d&exvsurl=1&viewmodel=ReadMessageItem)[3](https://knowledgehub.pega.com/COPLINSV:Onboarding_control-plane_services_via_pegacloud_cli)  

---
*End of Technical Design Document*
