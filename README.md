# Dental AI ML Pipeline Architecture
Automated YOLOv8 Retraining and Deployment for Dental Disease Detection

## System Overview

Production ML pipeline processing patient images through 6 layers: quality validation, real-time YOLOv8 inference, expert annotation, and automated monthly retraining with blue-green deployment.

**99.9% uptime** -- **less than 5s P99 latency** -- **10,000+ inferences/day**

---

## Architecture Diagram

```mermaid
flowchart TD

    subgraph EXT["LAYER 1 -- External Systems"]
        direction LR
        SUP["Supabase\nPatient image storage"]
        PRIS["Prisma ORM\nApp DB and schema"]
        APIGW["API Gateway\nPOST /upload webhook"]
        SUP --> PRIS --> APIGW
    end

    subgraph ING["LAYER 2 -- Ingestion and Quality Control"]
        direction TB
        LING["Lambda: Image Ingestion\nDownload and upload to S3"]
        S3["S3 Buckets\nraw / verified / augmented"]
        LQUAL["Lambda: Quality Gate\nBlur, brightness, contrast, 512px min"]
        DDB["DynamoDB\nImage metadata and state tracking"]
        LING --> S3 --> LQUAL
        LQUAL -->|"Pass"| DDB
        LQUAL -->|"Fail -- reject"| DDB
    end

    subgraph INF["LAYER 3 -- Inference"]
        direction TB
        SQS["SQS Queue + Dead Letter Queue"]
        LBATCH["Lambda: Batch Processor"]
        SM["SageMaker -- YOLOv8 Endpoint\nml.g4dn.xlarge -- 1 to 10 instances"]
        SNS["SNS Topic\nPrediction complete notification"]
        SQS --> LBATCH --> SM --> SNS
    end

    subgraph ANN["LAYER 4 -- Annotation and Verification"]
        direction TB
        ALB["ALB Load Balancer\nAuto Scaling 1 to 5 instances"]
        EC2["EC2 Annotation App\nFastAPI + React"]
        LVAL["Lambda: Validator\nAnnotation check"]
        SYNC["Supabase Sync\nUpdate verified JSON"]
        ALB --> EC2 --> LVAL --> SYNC
    end

    subgraph RET["LAYER 5 -- Monthly Retraining Pipeline"]
        direction TB
        EB["EventBridge Trigger\ncron 0 2 1 every month -- 1st of month, 2AM UTC"]
        SF["Step Functions\n8-Step State Machine"]

        subgraph STEPS["Retraining Steps"]
            direction TB
            S1["Step 1: Validate\nMin 100 images, quality check"]
            S2["Step 2: Class Analysis\nImbalance check, max 2:1 ratio"]
            S3["Step 3: Augment\nAlbumentations on minority classes"]
            S4["Step 4: Train\nYOLOv8, 50 epochs\n70% current + 30% historical"]
            S5["Step 5: Evaluate\nmAP@0.5, Precision, Recall"]
            S6["Step 6: Compare\n+2% threshold vs production"]
            S7["Step 7: Deploy\nBlue-green 10% to 100% over 2hrs\nAuto-rollback if errors exceed 5%"]
            S8["Step 8: Register\nS3 artifacts + Model Registry\nTag images for Glacier transition"]
            S1 --> S2
            S2 -->|"Imbalanced"| S3 --> S4
            S2 -->|"Balanced"| S4
            S4 --> S5 --> S6
            S6 -->|"Improved"| S7 --> S8
            S6 -->|"No improvement"| S8
        end

        EB --> SF --> S1
    end

    subgraph MON["LAYER 6 -- Monitoring"]
        direction LR
        CW["CloudWatch\nAlarms and dashboards"]
        ALERT["SNS Alerts\nOps team notifications"]
        CW --> ALERT
    end

    %% Inter-layer connections
    APIGW --> LING
    DDB -->|"Verified images accumulate"| SQS
    SNS --> ALB
    SYNC --> DDB
    DDB -->|"Triggers monthly on schedule"| EB
    SM --> CW
    SF --> CW
```

---

## Architecture Layers

| Layer | Key Components | Purpose |
|-------|---------------|---------|
| **1 -- External** | Supabase, Prisma ORM, API Gateway | Receive patient image uploads |
| **2 -- Ingestion** | Lambda, S3, Quality Gate, DynamoDB | Validate images and store metadata |
| **3 -- Inference** | SQS, Lambda, SageMaker, SNS | Run YOLOv8 predictions at scale |
| **4 -- Annotation** | EC2 (FastAPI + React), ALB, Validator | Dentists review and correct predictions |
| **5 -- Retraining** | EventBridge, Step Functions, 8 steps | Monthly automated model improvement |
| **6 -- Monitoring** | CloudWatch, SNS Alerts | Observe, alert, and act |

---

## Monthly Retraining Workflow

**Trigger:** EventBridge cron (`0 2 1 * ? *`) fires on the 1st of each month at 2AM UTC, launching an 8-step Step Functions pipeline.

**Steps 1 to 2 -- Validate and Analyse:** Confirms at least 100 images meet quality standards. Detects class imbalance beyond the 2:1 threshold.

**Step 3 -- Augment:** If imbalanced, Albumentations is applied to minority classes via SageMaker Processing.

**Step 4 -- Train:** YOLOv8 trained for 50 epochs on `ml.g4dn.xlarge` using 70% current data and 30% historical data.

**Steps 5 to 6 -- Evaluate and Compare:** Calculates mAP@0.5, precision, and recall. Deploys only if the new model shows at least +2% improvement over production.

**Step 7 -- Deploy:** Blue-green deployment from 10% to 100% traffic over 2 hours. Auto-rollback triggered if error rate exceeds 5%.

**Step 8 -- Register:** Artifacts stored in S3, lineage tracked in Model Registry, images tagged for Glacier transition.

---

## Technical Specifications

### Quality Gates

| Check | Threshold |
|-------|-----------|
| Blur (Laplacian variance) | Greater than 100 |
| Brightness (mean pixel) | Greater than 30 |
| Contrast (std dev) | Greater than 20 |
| Resolution | Minimum 512x512 px |

### Auto-Scaling

| Service | Scale |
|---------|-------|
| SageMaker | 1 to 10 instances |
| EC2 Annotation App | 1 to 5 instances (70% CPU trigger) |
| DynamoDB | On-demand |
| Lambda | Automatic |

### Data Lifecycle

| Stage | Duration |
|-------|----------|
| S3 Standard | 0 to 90 days |
| S3 Glacier | 90 days to 1 year |
| S3 Deep Archive | 1 to 7 years |

---

## Production SLAs

| Metric | Target |
|--------|--------|
| Uptime | 99.9% (multi-AZ) |
| Latency P99 | Less than 5 seconds |
| Daily throughput | 10,000+ inferences |
| Async reliability | Dead Letter Queues on all async operations |

---

## Security

- HIPAA-compliant architecture
- KMS encryption at rest and in transit
- VPC-only SageMaker endpoints
- CloudTrail audit logging enabled
- GDPR patient data deletion support

---

## Tech Stack

**AWS:** SageMaker, Lambda, S3, DynamoDB, Step Functions, EventBridge, CloudWatch, SQS, SNS, EC2, ALB, API Gateway

**ML:** YOLOv8, Albumentations

**App:** FastAPI, React, Supabase, Prisma ORM

---

*Dental AI Platform -- ML Pipeline Architecture v1.0 -- December 2025*
