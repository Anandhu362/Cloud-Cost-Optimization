# 🧾 Master Case Study: GCP Cloud Cost & Architectural Optimization

**Author**: Anandhu V S (Web Developer & Cloud Engineer)  
**Target Platform**: Google Cloud Platform (GCP)  
**Repository**: [Anandhu362/Cloud-Cost-Optimization](https://github.com/Anandhu362/Cloud-Cost-Optimization)

---

## Abstract

This case study analyzes real-world backend infrastructure cost and operational optimizations on Google Cloud Platform across two distinct production stages. By aligning compute models with application runtime requirements, monthly infrastructure costs were reduced by **>90%** for low-traffic stateless APIs and by **>50%** for high-value persistent enterprise backends, while improving SLA uptime to **100%**.

---

## 1. Background & System Scope

The engineering team operated backends for two distinct business applications:

1. **`alfajar-backend`**: A low-to-moderate traffic Node.js + TypeScript service responsible for handling intermittent REST API queries.
2. **`ferrari-backend`**: A mission-critical financial & operational management system processing 50-60+ daily high-value transactional entries across 2+ enterprise branches. Features include 24/7 WhatsApp real-time customer/management alerts (`@whiskeysockets/baileys`), scheduled daily compliance reporting (`node-cron` at 08:00 AM), Google BigQuery data warehouse syncing, and Firebase Firestore/Realtime DB storage.

---

## 2. Phase 1 Case Study: 90% Cost Cut via Cloud Run (Serverless)

### The Problem
- The initial deployment ran `alfajar-backend` on an unmanaged, always-running GCP Compute Engine VM.
- **Monthly Cost Breakdown (January 2026)**:
  - Compute Engine: **$17.91**
  - Cloud Storage: **$0.01**
  - VM Manager & Networking: **$1.45** (Offset by savings credits)
  - **Total Monthly Billing**: **$17.92 / month**
- **Inefficiency**: The backend received light traffic, yet the VM was billed 24 hours a day, 7 days a week for idle CPU cycles.

### Technical Migration Strategy
- Containerized the Node.js + TypeScript codebase into a production Docker container.
- Deployed to **Google Cloud Run** in `us-central1` with scaling configured from `Min: 0` to `Max: 20`.
- Handled HTTPS, domain mapping, and container execution automatically via GCP managed infrastructure.

### Financial Proof & Results
- **February 2026 Forecasted Total Cost**: **$1.40 / month**
- **Net Savings**: **-90.87% cost reduction** (-$13.94 monthly drop).
- Zero idle server billing; instances scale to zero during off-peak windows.

```
Monthly Cost Comparison (Stateless Microservice):
-------------------------------------------------
Baseline VM Deployment (Jan 2026) :  ████████████████████  $17.92
Serverless Cloud Run  (Feb 2026) :  █                     $1.40  (-90.87%)
```

---

## 3. Phase 2 Case Study: Right-Sized Dedicated VM for Persistent Workloads

### The Serverless Trap for Persistent Sockets & Timers
When scaling `ferrari-backend` on Cloud Run, three operational failures emerged:

1. 🔌 **WebSocket Disconnection (`DisconnectReason 409`)**:
   - The WhatsApp engine (`@whiskeysockets/baileys`) requires a persistent 24/7 WebSocket connection to maintain socket session state.
   - Cloud Run's scale-to-zero algorithm regularly terminated container instances during 15-minute traffic lulls, causing socket disconnects and missed automated alerts.
2. ⏰ **Frozen Background Cron Jobs**:
   - `node-cron` scheduled compliance tasks set to fire at 08:00 AM failed to execute because Cloud Run throttles container CPU to near zero between HTTP requests when CPU is allocated on request processing.
3. 💸 **Unexpected Cost Spikes**:
   - To keep WebSockets alive, enabling `CPU always allocated` on Cloud Run caused billing spikes that surpassed standard fixed VM rates under sustained connection holding.

### The Solution: Dedicated Compute Engine + Caddy + Auto-Rollback CI/CD
- Provisioned a dedicated `e2-medium` VM in `me-central1-a` (Ubuntu 22.04 LTS, 30GB `pd-balanced` SSD).
- Deployed **Caddy Web Server** on Port 443 to manage reverse proxying and automatic Let's Encrypt TLS certificates, resolving browser Mixed Content constraints for frontends hosted on Vercel.
- Mounted sensitive credentials (`.env`, Firebase Admin, BigQuery keys) read-only (`:ro`) into the Docker runtime container.
- Built a zero-downtime automated deployment & rollback pipeline using GitHub Actions, Google Artifact Registry (GAR), native Docker `HEALTHCHECK`, and a 35-second stabilization evaluation period.

### Operational & Financial Results
- **Cost Reduction**: Cut monthly cloud expenditure by **>50%** compared to unoptimized persistent serverless instances.
- **WebSocket Uptime**: **100% continuous uptime** with sub-second WhatsApp alert delivery.
- **Deployment Reliability**: **0 production downtime** with guaranteed auto-rollback protection.

---

## 4. Key Engineering Lessons

1. **Match Infrastructure to Workload Statefulness**:
   - Use **Cloud Run (Serverless)** for stateless, request-driven REST/GraphQL services with variable or low traffic.
   - Use **Dedicated VMs + Containers** for stateful, connection-holding services (WebSockets, continuous cron jobs, background workers).
2. **Automate Zero-Downtime Rollbacks**:
   - Combining native Docker healthchecks with a stabilization buffer (35s) ensures broken builds never reach production users.
3. **Decouple TLS Proxying from App Code**:
   - Utilizing Caddy as an outer reverse proxy simplifies certificate management and guarantees CORS/HTTPS security across multi-cloud setups (Vercel + GCP).
