# Retail Shelf Analytics on Azure

A reference solution for **on‑shelf availability (OSA), out‑of‑stock (OOS) detection, planogram compliance, and refill‑job orchestration** using computer vision, machine learning, and Microsoft Fabric for analytics.

The companion architecture diagram is in `retail-shelf-analytics-azure.drawio` (open with [draw.io](https://app.diagrams.net)).

---

## 1. Business problem

Retailers lose a meaningful share of revenue to **empty or under‑stocked shelves**. Today the gap is detected manually (associate walk‑bys, customer complaints) and acted on slowly. This solution:

- Detects missing or low SKUs from shelf images **and** live in‑store camera feeds.
- Quantifies the **lost‑sales impact** of each OOS event by tracking how long a slot stays empty.
- Generates **refill jobs**, assigns them to associates, and tracks closure with **before/after photo evidence**.
- Flags **planogram non‑compliance** (wrong product, wrong facings, wrong location) for corrective action.
- Surfaces everything through **Microsoft Fabric** dashboards for store managers, HQ, and category managers.

---

## 2. Use cases (MVP scope)

| # | Use case | Primary outcome |
|---|----------|-----------------|
| 1 | **On‑Shelf Availability detection** | Identify zero‑stock (reactive) and near‑OOS (proactive) SKUs in near real time. |
| 2 | **Lost Sales & Refill Tracking** | Measure OOS duration per SKU/store; quantify lost sales and savings from timely refills. |
| 3 | **Job Assignment & Closure** | Auto‑create refill jobs, assign to associates, enforce SLA, capture before/after photos. |
| 4 | **Product Location** | Identify where missing stock physically is (backroom, top shelf) to speed refill. |
| 5 | **Planogram Compliance** | Compare detected layout vs. planogram; flag wrong facings/placement. |
| 6 | **Computer Vision pipeline** | Detect empty spaces, identify SKUs, diff against planogram from mobile uploads + live camera frames. |
| 7 | **Analytics & Reporting** | Microsoft Fabric–based KPIs: OSA %, OOS duration, lost sales, compliance score, job SLA. |
| 8 | **ML lifecycle** | Train, register, deploy, and monitor detection / compliance models. |

> **MVP note:** No Azure IoT Hub or IoT Edge is required. Live camera feeds are sampled by a small on‑prem capture agent that posts frames over HTTPS to the same ingestion endpoint as the mobile app.

---

## 3. High‑level architecture

The solution is organised into seven layers:

1. **Stores / Edge** — Associate mobile app, in‑store cameras, planogram source.
2. **Ingestion & API** — Front Door + WAF, API Management, App Service / AKS, Event Hubs, Microsoft Entra ID.
3. **Storage & Operational Data** — ADLS Gen2 / Blob, Cosmos DB, Azure SQL DB, Key Vault, Service Bus.
4. **Computer Vision & ML** — Azure AI Vision / Custom Vision, Azure Machine Learning, Azure Functions, Stream Analytics, Logic Apps.
5. **Microsoft Fabric — Analytics & Reporting** — OneLake, Data Pipelines / Dataflows Gen2, Lakehouse / Warehouse, Notebooks (Spark), Power BI.
6. **Consumers** — Store Associate, Store Manager, HQ / Category Management, Data Scientists.
7. **Security & Observability** — Azure Monitor + App Insights, Log Analytics, Defender for Cloud, Microsoft Purview, Azure DevOps / GitHub.

---

## 4. End‑to‑end data flows

### Flow A — On‑Shelf Availability (image upload)

1. Associate snaps a shelf photo in the **Mobile App**.
2. Image hits **Front Door (+ WAF)** → **API Management** (Entra ID auth) → **App Service / AKS**.
3. App Service writes the image to **ADLS Gen2 / Blob** and metadata to **Cosmos DB**.
4. Blob event triggers **Azure Functions** (image pipeline).
5. Functions calls **Azure AI Vision / Custom Vision** for empty‑space + SKU detection, and **Azure Machine Learning** for planogram‑compliance scoring.
6. Detection results are written back to **Cosmos DB**.

### Flow B — Live camera feed (no IoT Edge)

1. On‑prem capture agent samples RTSP frames from cameras and HTTPS‑posts them to the same **Front Door → APIM → App Service** path.
2. App Service publishes a lightweight event to **Event Hubs**; **Stream Analytics** does windowed aggregation and forwards to **Functions**, which run the same vision pipeline as Flow A.

### Flow C — Job Assignment & Closure

1. When OOS / compliance issues are detected, **Functions** publish a job request to **Service Bus**.
2. **Logic Apps** pick up the message, create a job in **Azure SQL DB**, assign it to an associate, and send a push notification.
3. The associate completes the refill in the mobile app and uploads a **before/after photo**, closing the job (audit trail in SQL).

### Flow D — Analytics & Reporting (Microsoft Fabric)

1. **Fabric Data Pipelines / Dataflows Gen2** ingest from **Cosmos DB**, **Azure SQL**, and **Blob** into **OneLake**.
2. Data is curated through a **Bronze → Silver → Gold** lakehouse (OOS events, lost‑sales calculations, compliance, job SLA).
3. **Fabric Notebooks (Spark)** compute KPIs (OOS duration, lost‑sales estimates, refill effectiveness).
4. **Power BI** delivers dashboards to Store Managers, HQ, and Category Management.

### Flow E — ML lifecycle

- Data Scientists train and register models in **Azure Machine Learning**, with experiments and features sourced from **OneLake** / Fabric notebooks.
- Models are deployed as managed online endpoints consumed by Functions in Flows A and B.
- **Azure DevOps / GitHub** handles CI/CD and MLOps.

---

## 5. Component responsibilities

| Layer | Service | Responsibility |
|-------|---------|----------------|
| Edge | Mobile App | Capture shelf images, view assigned jobs, upload before/after photos |
| Edge | Cameras + capture agent | Sample frames, push to ingestion API |
| Edge | Planogram source | Authoritative store layout per category / fixture |
| Ingestion | Front Door + WAF | Global entry, TLS, OWASP protection |
| Ingestion | API Management | Routing, throttling, Entra ID token validation |
| Ingestion | App Service / AKS | Image, Jobs, Planogram microservices |
| Ingestion | Event Hubs | Live frame & event ingestion |
| Ingestion | Microsoft Entra ID | AuthN / AuthZ for users, services, MSI |
| Storage | ADLS Gen2 / Blob | Raw images & video frames |
| Storage | Cosmos DB | Planogram, SKU catalog, detection results |
| Storage | Azure SQL DB | Jobs, assignments, associates, audit |
| Storage | Key Vault | Secrets, keys, certificates |
| Storage | Service Bus | Decoupled job & alert messaging |
| AI/ML | Azure AI Vision / Custom Vision | Empty‑space + SKU detection |
| AI/ML | Azure Machine Learning | Custom OOS / planogram‑compliance models, MLOps |
| AI/ML | Azure Functions | Image pipeline, planogram diff, OOS scoring |
| AI/ML | Stream Analytics | Windowed aggregation of live frames |
| AI/ML | Logic Apps | Job assignment workflow, notifications |
| Analytics | OneLake | Unified data lake (Fabric) |
| Analytics | Data Pipelines / Dataflows Gen2 | Ingest from Cosmos / SQL / Blob |
| Analytics | Lakehouse / Warehouse | Bronze → Silver → Gold curation |
| Analytics | Notebooks (Spark) | KPI computation (lost sales, refill effectiveness) |
| Analytics | Power BI | Operational + executive dashboards |
| Sec & Obs | Monitor + App Insights | Metrics, traces, alerts |
| Sec & Obs | Log Analytics | Centralised logs |
| Sec & Obs | Defender for Cloud | Posture & threat protection |
| Sec & Obs | Purview | Data governance & lineage |
| Sec & Obs | DevOps / GitHub | CI/CD and MLOps |

---

## 6. Key KPIs

- **On‑Shelf Availability %** (per store / category / SKU)
- **OOS duration** (mean and p95) per slot
- **Estimated lost sales** (₹ / $) per OOS event
- **Refill SLA** (job created → job closed) and **first‑time‑fix rate**
- **Planogram compliance score** per fixture / store
- **Detection model precision / recall** per SKU class

---

## 7. Security & governance

- All user and service auth via **Microsoft Entra ID**; managed identities for service‑to‑service calls.
- Secrets and signing keys in **Key Vault**; rotation enforced.
- Private endpoints for Cosmos DB, SQL, Blob, Key Vault; public ingress only via Front Door + WAF + APIM.
- **Defender for Cloud** for posture and threat protection across the subscription.
- **Microsoft Purview** for catalog, classification, and lineage across OneLake and operational stores.
- Image data PII review (face blurring) handled before persistence where applicable.

---

## 8. MVP boundaries & future enhancements

**In scope (MVP)**
- Image upload from mobile + live camera feed via on‑prem capture agent
- AI Vision / Custom Vision + Azure ML for detection and compliance
- Job orchestration with before/after evidence
- Microsoft Fabric reporting

**Out of scope for MVP, candidates for later phases**
- **Azure IoT Hub / IoT Edge** for on‑device inference, offline operation, and lower bandwidth.
- Edge GPU acceleration (Azure Stack Edge) for sub‑second in‑aisle response.
- Shopper‑behavior analytics (dwell time, heatmaps) from the same camera feeds.
- Automated price/label compliance via OCR.
- Closed‑loop replenishment to the supply chain / DC system.

---

## 9. Production hardening checklist

This section captures the architecture-review feedback from Cloud Solution Architects, prioritised for incremental rollout. Items are grouped by phase; each is a concrete, testable change rather than a principle.

### P0 — Before any pilot expansion

- [ ] **Direct-to-Blob uploads via SAS.** Mobile app and capture agent upload images straight to ADLS Gen2 with short‑lived, per‑object SAS tokens. APIM/App Service handle only metadata + token issuance. Removes APIM payload limits and cost as a bottleneck.
- [ ] **Cosmos DB partition key locked in** (`storeId` or `storeId + shelfId`); autoscale RU enabled; RU consumption dashboarded; budget alert on RU spend.
- [ ] **Functions on Premium / Flex Consumption plan** for the image pipeline, with at least one always‑ready instance to eliminate cold‑start latency.
- [ ] **Idempotency key** on the image-processing Function (`hash(image) + storeId + capturedAt`) to safely handle Event Grid at‑least‑once delivery and retries.
- [ ] **Defender for Cloud** plans enabled for Storage, SQL, Cosmos DB, App Service, Key Vault, and Containers.
- [ ] **FinOps baseline:** subscription tag taxonomy (`env`, `costCenter`, `useCase`, `store`), Azure Cost Management budgets per resource group, anomaly alerts.
- [ ] **APIM tier:** Standard v2 or Premium for VNet integration; do **not** rely on Front Door + WAF alone in place of APIM.

### P1 — Before broad rollout

- [ ] **Evaluate Azure AI Product Recognition + Planogram Compliance** (pretrained) against the custom Custom Vision + AML pipeline. If precision/recall is within ~5%, retire the custom AML endpoint to collapse the ML layer to a single managed call.
- [ ] **Mobile offline outbox** with retry/back‑off so associates don't lose images when in‑store Wi‑Fi degrades.
- [ ] **Face blurring in the ingest Function** (AI Vision face detection → blur → persist). No raw faces in ADLS.
- [ ] **Image lifecycle policy:** Hot → Cool at 30 days → Archive or delete at 90 days; align with regional retention rules.
- [ ] **Right-size Microsoft Fabric capacity.** Start small (F2/F4 for non‑prod with pause schedule, F8/F16 for prod) and scale on measured usage. Fabric capacity is typically the largest single line item.
- [ ] **Front Door → APIM → App Service** access restrictions: App Service accepts traffic only from Front Door + APIM; private endpoints on every PaaS data store; NSGs on integration subnets.
- [ ] **Data residency review.** Confirm region selection per market (GDPR, India DPDP, etc.) before onboarding cross‑border stores.

### P2 — Production hardening

- [ ] **DR plan:** Cosmos DB multi‑region (write or active‑passive), Azure SQL geo‑replica, Front Door failover origin, documented runbook with RTO/RPO targets.
- [ ] **Job SLA escalation** via Durable Functions or timer‑triggered Function — auto‑escalate any job open past SLA to a manager.
- [ ] **Model drift monitoring** in Azure ML (data drift, prediction drift, performance metrics); scheduled retraining; shadow deployments before promotion.
- [ ] **Ground‑truth capture loop:** mis‑detections flagged by associates / managers feed back into the labeling dataset (closes the data flywheel).
- [ ] **Backups:** Cosmos DB point‑in‑time restore enabled; Azure SQL geo‑redundant automated backups; Key Vault soft‑delete + purge protection.
- [ ] **Microsoft Sentinel** on top of Log Analytics for security analytics and incident response.
- [ ] **Penetration test** + WAF rule tuning before go‑live.
- [ ] **App Insights sampling** tuned for production to control telemetry cost.

### P3 — Future evolution

- [ ] **Azure IoT Edge / Stack Edge** for on‑device inference, offline operation, and lower WAN bandwidth — only when latency or connectivity actually hurts.
- [ ] **APIM Self‑hosted Gateway** in flagship stores as a lighter alternative to IoT Edge for buffered, authenticated egress.
- [ ] **Azure Data Explorer (ADX)** for high‑volume detection telemetry analytics if Cosmos becomes uneconomical at scale.
- [ ] **Shopper analytics** (dwell time, heatmaps), **OCR price/label compliance**, **closed‑loop replenishment** to the supply chain.

### Notes on review trade-offs we accepted as-is

- **Stream Analytics for camera frames:** keep for windowed aggregation, late‑arrival handling, and reference‑data joins; revisit only if camera volume stays low long‑term.
- **App Service over AKS** for the API layer until there is a clear need for multi‑container orchestration or hybrid deployment.
- **Cosmos DB + Azure SQL split** retained — different access patterns (event vs. relational/transactional) justify two stores.

---


