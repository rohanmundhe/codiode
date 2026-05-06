# Google Cloud Credit Application - Codiode.com
## Technical Scaling Infrastructure Cost Breakdown

**Applicant:** Rohan Mundhe — rohanmundhe766@gmail.com
**Project:** gopim-f7995
**Platform:** [codiode.com](https://codiode.com)
**Credit Requested:** $20,000 USD
**GCP Region:** us-central1

---

## Executive Summary

Codiode is an electronics engineering education platform where students learn digital design by writing real Verilog/SystemVerilog hardware description code, not watching videos. Every problem on the platform requires a student to write actual RTL (Register Transfer Level) code that gets **compiled, simulated, and verified against hardware test vectors in real time**.

This is computationally expensive by nature. Simulating a Verilog design requires running `iverilog` (compiler) and `vvp` (simulator) — the same tools used in professional chip design. Each submission is a 30–120 second CPU-bound workload that cannot be shortened.

We have built a production-grade, multi-tier architecture on Google Cloud to handle this workload safely at scale. Our pipeline is **deeply and exclusively integrated with GCP** — GKE Autopilot for containerized grading, Cloud Run for stateless services, Cloud SQL for persistence, and Artifact Registry for our container supply chain.

The $20,000 credit will allow us to scale from **10,000 to 500,000 users** over 12 months without infrastructure cost becoming a barrier to growth at a critical stage.

---

## What Makes This Infrastructure Expensive

Most web applications serve static content or make simple database queries. Codiode is different. Every student submission triggers this workload:

```
Student submits Verilog code
         │
         ▼
1. Code security scan (pattern matching)
         │
         ▼
2. Job queued in Redis (BullMQ)
         │
         ▼
3. Kubernetes Job spawned — isolated container
   ┌─────────────────────────────────────┐
   │  iverilog compile (5–30 seconds)    │  ← real CPU work
   │  vvp simulate   (10–90 seconds)     │  ← real CPU work
   │  yosys synthesize (5–20 seconds)    │  ← real CPU work
   └─────────────────────────────────────┘
         │
         ▼
4. Results parsed, stored in PostgreSQL
         │
         ▼
5. Real-time results streamed via WebSocket
```

This is equivalent to running a mini chip-design EDA tool for every single student, every single submission. There is no way to cache or short-circuit it. The compute is inherent to the product.

---

## Current Production Architecture

### Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Google Cloud (us-central1)               │
│                                                                 │
│  Cloud Run                     GKE Autopilot                   │
│  ┌─────────────┐               ┌────────────────────────────┐  │
│  │ codiode-web │               │  Cluster: codiode-autopilot│  │
│  │ Next.js SSR │               │  Namespace: codiode        │  │
│  │ min:1 max:5 │               │                            │  │
│  └─────────────┘               │  Deployment: rtl-worker    │  │
│                                │  ├─ Pod (worker + sql proxy)│  │
│  ┌─────────────┐               │  ├─ Pod (worker + sql proxy)│  │
│  │ codiode-api │  BullMQ queue │  └─ HPA: 1–10 pods         │  │
│  │ NestJS API  │ ─────────────▶│                            │  │
│  │ Socket.IO   │◀─ pub/sub ─── │  K8s Jobs: grader-{id}    │  │
│  │ min:1 max:5 │   (Redis)     │  ├─ grader (iverilog+vvp)  │  │
│  └─────────────┘               │  ├─ grader (iverilog+vvp)  │  │
│         │                      │  └─ ... (one per submit)   │  │
│         │ Cloud SQL Auth Proxy │                            │  │
│         ▼                      └────────────────────────────┘  │
│  ┌─────────────────────────┐             │                      │
│  │ Cloud SQL               │◀────────────┘ Cloud SQL Auth Proxy │
│  │ codiode-db (db-f1-micro)│                                    │
│  │ PostgreSQL 15           │                                    │
│  └─────────────────────────┘                                    │
│                                                                 │
│  Artifact Registry: vericircuit (37 GB — 4 container images)   │
│  Secret Manager: 17 secrets (DB, Redis, JWT, OAuth, etc.)      │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
       Upstash Redis (external)
       BullMQ queue + Socket.IO adapter
```

### GCP Services in Use

| Service | Usage | Why GCP-Specific |
|---|---|---|
| **GKE Autopilot** | Containerized grader pods, RTL worker deployment | Node auto-provisioning, Workload Identity for Cloud SQL auth, native K8s Job TTL cleanup |
| **Cloud Run** | API + frontend, auto-scales on HTTP/WebSocket traffic | Session affinity for Socket.IO, native Cloud SQL socket integration |
| **Cloud SQL** | Primary PostgreSQL — users, submissions, problems, leaderboard | Cloud SQL Auth Proxy in-cluster via Workload Identity |
| **Artifact Registry** | All 4 Docker images (api, web, worker, grader) | Same-region pulls are free; GKE pulls from GAR natively |
| **Secret Manager** | 17 production secrets | Native integration with Cloud Run and GKE Workload Identity |
| **Cloud Build** | Was used for CI/CD | Replaced by GitHub Actions to save costs — still on GCP infra |
| **IAM + Workload Identity** | Keyless auth for GKE→Cloud SQL | Core security architecture |

---

## Current Monthly Cost (10,000 Users)

**Usage basis:** 10,000 registered users, ~500 daily active users, ~300 submissions/day

### Fixed Infrastructure Costs

These are paid regardless of how many users are active on any given day.

| Service | Resource | Monthly Cost |
|---|---|---|
| GKE Autopilot | Cluster management fee (1 cluster × $0.10/hr × 720 hrs) | **$72.00** |
| Cloud Run API | Min 1 instance always warm — memory + CPU (WebSocket keepalive) | **$20.00** |
| Cloud Run Web | Min 1 instance always warm — Next.js SSR | **$11.00** |
| Cloud SQL | db-f1-micro instance + 10 GB SSD + backup | **$10.17** |
| Artifact Registry | 37 GB image storage × $0.10/GB | **$3.70** |
| Secret Manager | 17 secrets management | **$1.00** |
| **Fixed Total** | | **$117.87/month** |

### Variable Costs (Scale With Submissions)

| Service | Basis | Monthly Cost |
|---|---|---|
| GKE — Worker pods | 1–3 pods active, CPU + memory billing | **$10.71** |
| GKE — Grader pods | 9,000 submissions × 60s avg × 0.5 vCPU | **$5.87** |
| Upstash Redis | ~1.35M commands/month (BullMQ + Socket.IO) | **$3.00** |
| Cloud Run API scale-out | Occasional burst to 2–3 instances | **$5.00** |
| **Variable Total** | | **$24.58/month** |

### Current Monthly Total

| | |
|---|---|
| Fixed infrastructure | $117.87 |
| Variable (usage-based) | $24.58 |
| Third-party (Cloudinary, Resend, Twilio) | ~$10.00 |
| **Total at 10,000 users** | **~$152/month** |

---

## 12-Month Scaling Projection

### User Growth Assumptions

Codiode is targeting college-level electronics engineering students — a global addressable market of ~15 million students. Based on referral growth from university coursework integration:

| Month | Registered Users | DAU | Submissions/Day | Monthly GCP Cost |
|---|---|---|---|---|
| 1–2 (Now) | 10,000 | 500 | 300 | $152 |
| 3–4 | 25,000 | 1,250 | 750 | $210 |
| 5–6 | 60,000 | 3,000 | 1,800 | $360 |
| 7–8 | 120,000 | 6,000 | 3,600 | $620 |
| 9–10 | 250,000 | 12,500 | 7,500 | $1,150 |
| 11–12 | 500,000 | 25,000 | 15,000 | $2,100 |

### How Each Cost Component Scales

#### Cloud Run API + Web — Scales with HTTP traffic
```
10k users:   2 instances avg  →  $31/month
100k users:  4 instances avg  →  $85/month
500k users:  8 instances avg  →  $190/month  (needs max-instances increase)
```

#### GKE Autopilot — Scales with submission volume
The cluster fee ($72/month) stays fixed. Pod costs grow with submissions.

```
300 submissions/day:   6 grader pod-hours/day  →  $11/month (pods only)
1,800 submissions/day: 36 grader pod-hours/day →  $67/month (pods only)
7,500 submissions/day: 150 pod-hours/day       →  $280/month (pods only)
15,000 submissions/day: 300 pod-hours/day      →  $560/month (pods only)
```

#### Cloud SQL — Needs upgrade at ~50k users
```
Now:       db-f1-micro    (1 shared vCPU, 614 MB, 25 connections)  →  $10/month
50k users: db-g1-small    (1 vCPU, 1.7 GB, 25 connections)         →  $28/month
120k users: db-custom-2   (2 vCPU, 4 GB, 100 connections)          →  $95/month
250k users: db-custom-4   (4 vCPU, 8 GB, 200 connections) + replica →  $280/month
500k users: db-custom-8   (8 vCPU, 16 GB) + read replica + PgBouncer→  $480/month
```

#### Required Infrastructure Upgrades

| Milestone | Trigger | Upgrade | Monthly Cost Added |
|---|---|---|---|
| 50k users | DB connection saturation | Cloud SQL → db-g1-small | +$18 |
| 100k users | API WebSocket capacity | Cloud Run max-instances: 5→20 | +$40 |
| 100k users | DB query throughput | PgBouncer + Cloud SQL upgrade | +$70 |
| 200k users | Grader pod throughput | GKE CPU quota increase request | +$0 (free) |
| 250k users | DB read bottleneck | Cloud SQL Read Replica | +$95 |
| 500k users | CDN + DDoS protection | Cloud Armor + Cloud CDN | +$150 |

### 12-Month Cumulative GCP Cost

| Period | Monthly Cost | Cumulative |
|---|---|---|
| Month 1–2 | $152 | $304 |
| Month 3–4 | $210 | $724 |
| Month 5–6 | $360 | $1,444 |
| Month 7–8 | $620 | $2,684 |
| Month 9–10 | $1,150 | $4,984 |
| Month 11–12 | $2,100 | $9,184 |
| **12-Month Total GCP** | | **~$9,200** |

### Additional GCP Investment (Non-Recurring)

| Item | Cost | Purpose |
|---|---|---|
| Cloud Armor WAF | $500 setup + $200/month | DDoS protection at 200k+ users |
| Cloud CDN | $300/month at scale | Cache static + SSR pages globally |
| Vertex AI API | $1,500 | AI-powered circuit feedback, hint generation |
| Dev + Staging environments | $800 | Separate GKE namespace, Cloud Run revisions for QA |
| Load testing infrastructure | $400 | Validate scaling before each growth phase |
| Cloud Monitoring + Alerting | $300 | Production observability at scale |
| **Additional Investment Total** | **~$4,000** | |

### Total 12-Month GCP Spend Requiring Credit

```
Recurring infrastructure (12 months)   $9,200
Additional GCP investment              $4,000
Buffer for unexpected scale events     $2,800
──────────────────────────────────────────────
Total                                 $16,000

Credit requested                      $20,000
```

The additional $4,000 buffer accounts for GKE node autoprovisioning during traffic spikes (e.g., university exam periods where thousands of students submit simultaneously), emergency Cloud SQL upgrades, and increased Artifact Registry egress during rapid deployment cycles.

---

## Why This Must Run on Google Cloud

Every layer of our architecture is built on GCP-native capabilities that cannot be straightforwardly ported to another cloud:

**1. GKE Autopilot + Workload Identity**
Our security model relies on Workload Identity — GKE pods authenticate to Cloud SQL using GCP service accounts, with no credentials stored anywhere. The `grader-worker` K8s service account is bound to `grader-worker@gopim-f7995.iam.gserviceaccount.com` which holds `roles/cloudsql.client`. This keyless authentication model is GCP-specific.

**2. Cloud SQL Auth Proxy**
The Cloud SQL Auth Proxy sidecar in each worker pod creates an encrypted, IAM-authenticated tunnel to PostgreSQL. This is the only way to securely connect GKE pods to Cloud SQL without exposing the database to the public internet (the instance has no authorized network entries).

**3. Cloud Run Session Affinity**
Our WebSocket architecture (Socket.IO) requires session affinity so that a student's HTTP submission and their WebSocket connection land on the same Cloud Run instance. GCP's Cloud Run session affinity flag makes this a one-line configuration. Implementing this elsewhere requires a dedicated load balancer with sticky session configuration.

**4. Same-Region Artifact Registry Pulls**
Each Kubernetes grader Job pulls the `grader:latest` image (250 MB) when a submission is received. With GKE and Artifact Registry in the same region (us-central1), these pulls are **free and low-latency**. On another cloud, we would pay egress fees on every image pull, which at 15,000 submissions/day × 250 MB = 3.75 TB/day of image transfer would cost thousands per month.

**5. Integrated CI/CD Chain**
Our GitHub Actions pipeline authenticates to GCP using a dedicated service account (`codiode-github@gopim-f7995.iam.gserviceaccount.com`) with scoped roles (`artifactregistry.writer`, `run.admin`, `container.developer`). Every push to `main` builds 4 Docker images, pushes to GAR, and deploys to Cloud Run + GKE in a single workflow. This tight integration is specific to the GCP toolchain.

---

## Technical Differentiation

What makes Codiode technically distinct from other EdTech platforms and why it justifies infrastructure investment:

| Capability | Most EdTech | Codiode |
|---|---|---|
| Code execution | Sandboxed Python/JS | Full EDA toolchain (iverilog, vvp, yosys) |
| Isolation | Process-level | Kubernetes Job per submission (cgroup-enforced limits) |
| Feedback | Test pass/fail | Waveform viewer, synthesis netlist, cell count, latch detection |
| Scalability | Stateless request/response | Async queue → grader pod → WebSocket stream |
| Subject matter | CS algorithms | Digital hardware design (RTL, FSM, sequential circuits) |

The grading pipeline for hardware description languages is significantly more complex than general code execution platforms (HackerRank, LeetCode). There is no off-the-shelf solution. This is custom infrastructure built specifically for hardware education.

---

## Contact & Project Details

| | |
|---|---|
| **Platform** | codiode.com |
| **GCP Project** | gopim-f7995 |
| **Project Created** | April 2024 |
| **GCP Services Active** | Cloud Run, GKE Autopilot, Cloud SQL, Artifact Registry, Secret Manager, IAM |
| **Contact** | rohanmundhe26@gmail.com |
| **Credit Requested** | $20,000 USD — 12-month runway for scaling from 10k to 500k users |

---

*All cost figures use current GCP list prices for us-central1 as of May 2026. Actual costs may vary based on sustained use discounts and committed use contracts, which we would adopt as usage stabilizes at each tier.*
