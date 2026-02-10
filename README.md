# Petabyte-Scale Web & Multimodal Data Acquisition Pipeline (Kafka + Docker)

A Kafka-backed, distributed pipeline that crawls, renders (when needed), extracts, filters, deduplicates, and materializes **versioned ML-ready datasets** from multimodal web data: HTML, images, code, and video metadata.

**Core design rule:** Kafka carries **events + pointers**; object storage holds **bytes**. Everything runs in **Docker** locally, and the same containers deploy to **AWS / GCP / Azure** using managed services.

---

## Architecture at a glance

### Components
- **frontier (Rust):** URL frontier + robots + politeness scheduling → emits fetch tasks
- **fetcher (Rust):** fetches bytes → stores to object storage → emits fetch events
- **renderer (Docker headless):** renders JS pages → stores output → emits rendered-page events
- **extractor (Rust):** parses DOM + extracts text/assets/metadata → emits extracted events
- **quality (Python):** rule filters + ML scoring → emits quality events with reason codes
- **dedup (Rust):** exact + near-dup clustering → emits dedup decisions/clusters
- **dataset_builder (Spark):** builds Parquet datasets + manifests
- **registry (API + DB):** metadata + dataset version registry

### Illustrated flow

```
          +-----------+
          | frontier  |
          +-----+-----+
                |
                v
     Kafka: crawl.fetch.tasks.v1
                |
                v
+--------+   +--------+      +-------------------+
| robots |-->| fetcher |----->| object storage    |
+--------+   +--------+      | raw/*             |
                |            +-------------------+
                v
      Kafka: crawl.raw_fetch.v1
                |
        needs JS? (router/consumer)
          | yes                 | no
          v                     v
Kafka: crawl.render.tasks.v1   extractor
          |                     |
          v                     v
       renderer          Kafka: extract.page.v1
          |                     |
          v                     v
 object storage           quality + dedup
 rendered/*                    |
          \                    /
           v                  v
            Spark dataset_builder
                    |
                    v
        object storage datasets/* + manifests/*
                    |
                    v
              registry (Postgres)
```

---

## Kafka topic map (baseline)

### Tasks (short retention)
- `crawl.fetch.tasks.v1` (key: `host_id`)
- `crawl.render.tasks.v1` (key: `host_id`)

### Events (replayable)
- `crawl.raw_fetch.v1` (key: `url_hash`)
- `crawl.rendered_page.v1` (key: `url_hash`)
- `extract.page.v1` (key: `url_hash`)
- `quality.page_scored.v1` (key: `url_hash`)
- `dedup.text_decision.v1` (key: `content_hash`)
- `dedup.image_decision.v1` (key: `image_hash`)

### Compacted indexes (latest state)
- `index.url_latest.v1` (compacted)
- `index.content_cluster.v1` (compacted)
- `index.dataset_membership.v1` (compacted)

### DLQs
- `dlq.fetch.v1`, `dlq.render.v1`, `dlq.extract.v1`, `dlq.quality.v1`, `dlq.dedup.v1`

---

## Repo layout

```
repo/
  rust/
    common/
    frontier/
    fetcher/
    renderer/          # optional wrapper; or use standalone headless container
    extractor/
    dedup/
  python/
    quality/
    validation/
  spark_jobs/
    dataset_builder/
  schema/
    proto/             # or avro/
  infra/
    local/
      docker-compose.yml
      env.example
    aws/
      terraform/       # recommended
    gcp/
      terraform/
    azure/
      terraform/
  docs/
    runbook.md
    architecture.md
```

---

# 1) Local setup (Docker)

## Prerequisites
- Docker + Docker Compose
- (Optional) `make`
- If you want to build locally:
  - Rust toolchain
  - Python 3.10+
  - Spark (or run Spark in Docker)

## Clone and run
```bash
git clone <YOUR_REPO_URL>
cd repo
cp infra/local/env.example .env
docker compose -f infra/local/docker-compose.yml up -d
```

### What the local stack includes
- Kafka (KRaft)
- Schema Registry (if using Avro/Protobuf registry)
- Postgres
- MinIO (S3-compatible storage)
- Prometheus + Grafana (optional but recommended)
- Your services (frontier/fetcher/extractor/quality/dedup)
- Spark container (optional; or submit Spark locally)

## Sanity checks
```bash
docker ps
# check Kafka broker, minio, postgres, and services are healthy
```

### Seed a crawl (example)
1) Add seed URLs (file or API):
- `seeds.txt` (example)
2) Start frontier configured to read seeds and emit tasks:
- frontier reads `seeds.txt` and publishes to `crawl.fetch.tasks.v1`

### Expected local outputs
- MinIO buckets:
  - `raw/`, `rendered/`, `extracted/`, `datasets/`, `manifests/`
- Postgres tables:
  - url status, fetch history, content hashes, dataset versions

---

# 2) Container build & publish (all clouds)

Everything deploys as Docker containers. You typically:
1) Build images
2) Push to a container registry
3) Deploy to Kubernetes or container services + managed Kafka + managed DB + object storage

### Build (example)
```bash
docker build -t pipeline/frontier:latest rust/frontier
docker build -t pipeline/fetcher:latest rust/fetcher
docker build -t pipeline/extractor:latest rust/extractor
docker build -t pipeline/dedup:latest rust/dedup
docker build -t pipeline/quality:latest python/quality
```

---

# 3) Deployment on AWS

Below is a “managed-services first” deployment that maps cleanly to this pipeline.

## AWS service mapping
- **Kafka:** Amazon **MSK**
- **Object storage:** **S3**
- **Metadata DB:** **RDS Postgres** (or Aurora Postgres)
- **Compute (containers):** **EKS** (Kubernetes) or ECS Fargate
- **Spark:** EMR on EKS (or EMR Serverless)
- **Secrets:** AWS Secrets Manager
- **Observability:** CloudWatch + Prometheus/Grafana (managed or self-hosted)

### Recommended: EKS + MSK + S3 + RDS
**High-level steps**
1) Create networking
   - VPC with private subnets for MSK/RDS, public/private for EKS
2) Provision core services
   - MSK cluster (TLS enabled)
   - S3 buckets (`raw`, `rendered`, `extracted`, `datasets`, `manifests`)
   - RDS Postgres
3) Push images to ECR
4) Deploy services to EKS
5) Configure Spark jobs to read/write S3 and write manifests + registry updates

## AWS deployment outline (practical)
1) **Provision infrastructure** (Terraform recommended)
- `infra/aws/terraform` creates:
  - VPC + subnets
  - MSK
  - RDS Postgres
  - S3 buckets
  - EKS cluster + node groups
  - IAM roles for service accounts (IRSA)

2) **Build & push to ECR**
```bash
aws ecr create-repository --repository-name pipeline/frontier
# ...
docker tag pipeline/frontier:latest <acct>.dkr.ecr.<region>.amazonaws.com/pipeline/frontier:latest
docker push <acct>.dkr.ecr.<region>.amazonaws.com/pipeline/frontier:latest
```

3) **Deploy containers**
- Use Helm charts or Kubernetes manifests:
  - Set env:
    - `KAFKA_BOOTSTRAP_SERVERS` (MSK endpoints)
    - `S3_BUCKET_*`
    - `DATABASE_URL` (RDS)
    - `SCHEMA_REGISTRY_URL` (if used)
  - Provide credentials via IRSA + Secrets Manager

4) **Run Spark**
- Option A: **EMR on EKS** Spark Operator
- Option B: **EMR Serverless**
- Jobs read from `extracted/` and write to `datasets/` + `manifests/`

---

# 4) Deployment on GCP

## GCP service mapping
- **Kafka:** Managed Kafka option varies by org:
  - Best practice: **Confluent Cloud on GCP** (managed Kafka)
  - Alternative: run Kafka on **GKE** (Strimzi) if you must self-manage
- **Object storage:** **GCS**
- **Metadata DB:** **Cloud SQL for Postgres**
- **Compute:** **GKE**
- **Spark:** Dataproc on GKE, or run Spark on GKE (Spark Operator)
- **Secrets:** Secret Manager
- **Observability:** Cloud Monitoring + Prometheus/Grafana

### Recommended: GKE + Confluent Cloud + GCS + Cloud SQL
**High-level steps**
1) Provision GKE
2) Provision Confluent Cloud Kafka cluster + Schema Registry (optional)
3) Create GCS buckets
4) Create Cloud SQL Postgres
5) Push images to Artifact Registry
6) Deploy services to GKE
7) Run Spark jobs (Dataproc on GKE or Spark Operator)

## GCP deployment outline
1) **Provision infrastructure** (Terraform recommended)
- `infra/gcp/terraform` creates:
  - VPC
  - GKE cluster
  - Cloud SQL Postgres
  - GCS buckets
  - IAM bindings for Workload Identity

2) **Build & push to Artifact Registry**
```bash
gcloud artifacts repositories create pipeline --repository-format=docker --location=<region>
docker tag pipeline/frontier:latest <region>-docker.pkg.dev/<project>/pipeline/frontier:latest
docker push <region>-docker.pkg.dev/<project>/pipeline/frontier:latest
```

3) **Deploy to GKE**
- Set env:
  - `KAFKA_BOOTSTRAP_SERVERS` (Confluent endpoints)
  - `GCS_BUCKET_*`
  - `DATABASE_URL` (Cloud SQL)
- Prefer Workload Identity + Secret Manager

4) **Spark**
- Use Spark Operator on GKE, or Dataproc (if your org prefers managed Spark)
- Output Parquet to GCS + manifests to GCS; registry updates to Cloud SQL

---

# 5) Deployment on Azure

## Azure service mapping
- **Kafka:** Common managed option is **Confluent Cloud on Azure**
  - Alternative: self-managed Kafka on AKS (Strimzi)
- **Object storage:** **Azure Blob Storage** (ADLS Gen2 recommended)
- **Metadata DB:** **Azure Database for PostgreSQL**
- **Compute:** **AKS**
- **Spark:** Azure Databricks or Spark on AKS (Spark Operator)
- **Secrets:** Key Vault
- **Observability:** Azure Monitor + Grafana/Prometheus

### Recommended: AKS + Confluent Cloud + ADLS + Azure Postgres
**High-level steps**
1) Provision AKS
2) Provision Confluent Cloud Kafka cluster
3) Create Storage Account + containers for `raw/ rendered/ extracted/ datasets/ manifests/`
4) Create Azure Database for PostgreSQL
5) Push images to Azure Container Registry (ACR)
6) Deploy services to AKS
7) Run Spark on Databricks or AKS

## Azure deployment outline
1) **Provision infrastructure** (Terraform recommended)
- `infra/azure/terraform` creates:
  - VNet
  - AKS
  - Storage Account + containers
  - Azure Postgres
  - ACR
  - Managed identities + Key Vault

2) **Build & push to ACR**
```bash
az acr create -n <acrName> -g <rg> --sku Standard
az acr login -n <acrName>
docker tag pipeline/frontier:latest <acrName>.azurecr.io/pipeline/frontier:latest
docker push <acrName>.azurecr.io/pipeline/frontier:latest
```

3) **Deploy to AKS**
- Set env:
  - `KAFKA_BOOTSTRAP_SERVERS`
  - `AZURE_STORAGE_ACCOUNT`, `CONTAINER_*`
  - `DATABASE_URL`
- Use Managed Identity + Key Vault

4) **Spark**
- Azure Databricks (recommended for managed Spark)
- Or Spark Operator on AKS if you want everything in Kubernetes

---

# 6) Configuration: environment variables (common)

Typical shared env (service-specific variants apply):
- Kafka:
  - `KAFKA_BOOTSTRAP_SERVERS`
  - `KAFKA_SECURITY_PROTOCOL` (PLAINTEXT for local, SASL_SSL/TLS for cloud)
  - `KAFKA_SASL_MECHANISM` / `KAFKA_USERNAME` / `KAFKA_PASSWORD` (if needed)
- Object storage (pick one based on env):
  - Local MinIO (S3-compatible): `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET_*`
  - AWS: `S3_BUCKET_*` + IAM
  - GCP: `GCS_BUCKET_*` + Workload Identity
  - Azure: `AZURE_STORAGE_ACCOUNT` + `AZURE_CONTAINER_*`
- DB:
  - `DATABASE_URL` (Postgres)
- Tracing/metrics:
  - `OTEL_EXPORTER_OTLP_ENDPOINT`
  - `LOG_LEVEL`

---

# 7) Kafka configuration profiles (quick reference)

### Profile A: “Millions/day”
- RF=3, minISR=2
- Tasks: 24h retention
- Events: 7–14d retention
- Index topics: compacted
- Typical partitions:
  - `crawl.fetch.tasks.v1`: 48
  - `crawl.raw_fetch.v1`: 96
  - `extract.page.v1`: 96
  - indexes: 24

### Profile B: “Hundreds of millions/day”
- RF=3, minISR=2
- Tasks: 12–24h retention
- Events: 3–14d retention
- indexes: compacted, more partitions, tighter compaction
- Typical partitions:
  - tasks: 192–384
  - raw/extract: 384–768
  - indexes: ~192

---

# 8) Running Spark dataset builds

Spark job responsibilities:
- load extracted + quality + dedup outputs
- apply final filters and dedup membership
- output partitioned Parquet:
  - partition keys: date, language, domain category (or host)
- write deterministic manifest:
  - includes configs, code versions, input windows, counts, distributions

Example (pseudo):
```bash
# local mode
docker run --rm pipeline/spark:latest \
  spark-submit /app/dataset_builder.py --input s3://extracted/... --output s3://datasets/...
```

---

# 9) Operational notes

### Key things to monitor
- Consumer lag per service group
- 429/503 rates (politeness/throttling)
- renderer crash rate
- extraction failures by content-type
- reject rate by quality reason codes
- dedup cluster growth and duplicate % over time
- dataset build success rate and output sizes

### Replay and reprocessing
Kafka enables replay, but you must keep side effects idempotent:
- content-addressed object paths (safe overwrites)
- DB upserts keyed by `url_hash` / `content_hash`
- DLQs for poison messages

---

## License / Responsible crawling
- Strict robots.txt compliance
- Opt-out/deny list support
- PII redaction hooks (MVP acceptable)
- Content policy hooks (unsafe/adult scoring and filtering)
