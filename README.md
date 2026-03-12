# 🦷 Dental AI ML Pipeline Architecture
> Automated YOLOv8 Retraining & Deployment for Dental Disease Detection

## System Overview

Production ML pipeline processing patient images through **6 layers**: quality validation → real-time YOLOv8 inference → expert annotation → automated monthly retraining & blue-green deployment. Achieves **99.9% uptime**, **<5s P99 latency**, and **10K+ inferences/day**.

---
```mermaid
flowchart TD
    subgraph EXT["① External Systems"]
        SUP["🗄️ Supabase\nPatient Images"]
        PRIS["Prisma ORM\nApp DB / Schema"]
        APIGW["API Gateway\nPOST /upload"]
    end

    subgraph ING["② Ingestion & Quality Control"]
        LING["λ Image Ingestion\nDownload & Upload"]
        S3RAW["S3 Buckets\nraw/ verified/ augmented/"]
        LQUAL["λ Quality Gate\nBlur · Brightness · Contrast · 512px min"]
        DDB["DynamoDB\nImage Metadata & State"]
    end

    subgraph INF["③ Inference"]
        SQS["SQS Queue\n+ Dead Letter Queue"]
        LBATCH["λ Batch Processor"]
        SM["SageMaker\nYOLOv8 Endpoint\nml.g4dn.xlarge\n1–10 instances"]
        SNS["SNS Topic\nPrediction Complete"]
    end

    subgraph ANN["④ Annotation & Verification"]
        EC2["EC2 Annotation App\nFastAPI + React"]
        ALB["ALB Load Balancer\nAuto Scaling 1–5"]
        LVAL["λ Validator\nAnnotation Check"]
        SYNCB["Supabase Sync\nUpdate JSON"]
    end

    subgraph RET["⑤ Monthly Retraining Pipeline"]
        EB["EventBridge\ncron 0 2 1 * ? *\n1st of month, 2 AM UTC"]
        SF["Step Functions\n8-Step State Machine"]

        subgraph STEPS["Retraining Steps"]
            S1["① Validate\nMin 100 imgs · Quality check"]
            S2["② Class Analysis\nImbalance check max 2:1"]
            S3["③ Augmentation\nAlbumentations on minority classes"]
            S4["④ Training\nYOLOv8 · 50 epochs\n70% current + 30% historical"]
            S5["⑤ Evaluate\nmAP@0.5 · Precision · Recall"]
            S6["⑥ Compare\n+2% threshold vs prod"]
            S7["⑦ Deploy\nBlue-Green 10%→100% over 2hrs\nAuto-rollback if errors >5%"]
            S8["⑧ Register\nS3 artifacts · Model Registry\nTag for Glacier transition"]
        end
    end

    subgraph MON["⑥ Monitoring"]
        CW["CloudWatch\nAlarms & Dashboards"]
        ALERT["SNS Alerts\nOps Team"]
    end

    %% External → Ingestion
    SUP --> PRIS --> APIGW --> LING
    LING --> S3RAW --> LQUAL
    LQUAL -->|"Pass"| DDB
    LQUAL -->|"Fail → reject"| DDB

    %% Ingestion → Inference
    DDB --> SQS --> LBATCH --> SM --> SNS

    %% Inference → Annotation
    SNS --> ALB --> EC2
    EC2 --> LVAL --> SYNCB --> DDB

    %% Annotation → Retraining
    DDB -->|"Verified images\naccumulate"| EB
    EB --> SF --> S1 --> S2
    S2 -->|"Imbalanced"| S3 --> S4
    S2 -->|"Balanced"| S4
    S4 --> S5 --> S6
    S6 -->|"Improved"| S7 --> S8
    S6 -->|"No improvement\nskip deploy"| S8

    %% Monitoring
    SM --> CW
    SF --> CW
    CW --> ALERT

    style EXT fill:#1a1a2e,stroke:#4a9eff,color:#fff
    style ING fill:#16213e,stroke:#4a9eff,color:#fff
    style INF fill:#0f3460,stroke:#4a9eff,color:#fff
    style ANN fill:#1a1a2e,stroke:#00d4aa,color:#fff
    style RET fill:#16213e,stroke:#f39c12,color:#fff
    style STEPS fill:#0f0f23,stroke:#f39c12,color:#fff
    style MON fill:#1a1a2e,stroke:#e74c3c,color:#fff
```

---

## Architecture Layers

| Layer | Components | Purpose |
|-------|-----------|---------|
| **① External** | Supabase → Prisma ORM → API Gateway | Receive patient image uploads |
| **② Ingestion** | Lambda → S3 → Quality Gate → DynamoDB | Validate & store images with metadata |
| **③ Inference** | SQS → Lambda → SageMaker → SNS | Run YOLOv8 predictions at scale |
| **④ Annotation** | EC2 (FastAPI+React) → ALB → Validator | Dentists review & correct predictions |
| **⑤ Retraining** | EventBridge → Step Functions → 8 steps | Monthly automated model improvement |
| **⑥ Monitoring** | CloudWatch → Alarms → SNS | Observe, alert, and act |

---

## Monthly Retraining Workflow

**Trigger:** EventBridge cron (`0 2 1 * ? *`) → Step Functions 8-step pipeline

- **Steps 1–2 · Validate & Analyse** — Confirms ≥100 images; checks quality; detects class imbalance (threshold: max 2:1 ratio)
- **Step 3 · Augment** — If imbalanced: Albumentations applied to minority classes via SageMaker Processing
- **Step 4 · Train** — YOLOv8, 50 epochs, `ml.g4dn.xlarge` — 70% current data + 30% historical
- **Steps 5–6 · Evaluate & Compare** — Calculates mAP@0.5, precision, recall; deploys only if **+2% improvement** over prod
- **Step 7 · Deploy** — Blue-green: 10% → 100% traffic over 2 hrs; auto-rollback if error rate >5%
- **Step 8 · Register** — Artifacts to S3, lineage in Model Registry, images tagged for Glacier transition

---

## Technical Specifications

<table>
<tr>
<td>

### Quality Gates
| Check | Threshold |
|-------|-----------|
| Blur (Laplacian) | Variance > 100 |
| Brightness | Mean pixel > 30 |
| Contrast | Std dev > 20 |
| Resolution | Min 512×512 px |

</td>
<td>

### Auto-Scaling
| Service | Scale |
|---------|-------|
| SageMaker | 1–10 instances |
| EC2 | 1–5 (70% CPU) |
| DynamoDB | On-demand |
| Lambda | Automatic |

</td>
<td>

### Data Lifecycle
| Stage | Duration |
|-------|----------|
| S3 Standard | 0–90 days |
| Glacier | 90d – 1yr |
| Deep Archive | 1–7 years |

</td>
</tr>
</table>

---

## Production SLAs

| Metric | Target |
|--------|--------|
| Uptime | 99.9% (multi-AZ) |
| Latency P99 | < 5 seconds |
| Daily throughput | 10,000+ inferences |
| Async reliability | DLQs on all queues |

**Security:** HIPAA-compliant · KMS encryption · VPC-only SageMaker endpoints · CloudTrail audit logging · GDPR patient data deletion

---

**Tech Stack:** AWS (SageMaker, Lambda, S3, DynamoDB, Step Functions, EventBridge, CloudWatch) · YOLOv8 · Albumentations · FastAPI · React · Supabase · Prisma

*Dental AI Platform · ML Pipeline Architecture v1.0 · December 2025*