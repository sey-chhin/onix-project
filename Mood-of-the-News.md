# Mood of the News — Complete Repository in Markdown

This document contains **all source code, configuration, and documentation** to deploy the “Mood of the News” project end‑to‑end in Google Cloud.

---

## 📁 Structure

```
mood-of-news/
├── README.md
├── .gitignore
├── Makefile
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── schema.json
└── function/
    ├── main.py
    └── requirements.txt
```

---

## 📖 README.md

```markdown
# Mood of the News

End-to-end GCP project that ingests top US headlines hourly, scores sentiment with VADER, and stores results in BigQuery for visualization in Looker Studio.

## Architecture
- Cloud Function (Python) — fetch + score + load to BigQuery
- Cloud Scheduler — hourly trigger
- BigQuery — storage
- Cloud Storage — function source archive
- Terraform — infrastructure as code

## Prerequisites
- Billing-enabled GCP project
- Roles for your deployer: Owner or equivalent
- Installed: Terraform >= 1.5, gcloud SDK, zip

## Quickstart

1. Authenticate:
   ```bash
   gcloud auth application-default login
   gcloud config set project <PROJECT_ID>
   ```

2. Set env vars:
   ```bash
   export PROJECT_ID=<PROJECT_ID>
   export REGION=us-central1
   export NEWS_API_KEY=<your_newsapi_key>
   ```

3. Deploy:
   ```bash
   make tf-init
   make tf-apply PROJECT_ID=$PROJECT_ID REGION=$REGION NEWS_API_KEY=$NEWS_API_KEY
   ```

4. Visualize in Looker Studio:
   - Connect to BigQuery dataset `news_sentiment.headlines`
   - Add charts for sentiment trends, top headlines, daily gauge
```

---

## .gitignore

```
# Python
__pycache__/
*.pyc
.venv/

# Function artifact
function/function.zip

# Terraform
.terraform/
terraform.tfstate
terraform.tfstate.backup
*.tfvars
crash.log
```

---

## Makefile

```makefile
PROJECT_ID ?= your-gcp-project-id
REGION ?= us-central1
NEWS_API_KEY ?= replace_me

TF_DIR := terraform
FN_DIR := function

.PHONY: zip tf-init tf-apply tf-destroy

zip:
	@cd $(FN_DIR) && rm -f function.zip && zip -qr function.zip main.py requirements.txt

tf-init:
	cd $(TF_DIR) && terraform init

tf-apply: zip
	cd $(TF_DIR) && terraform apply \
	  -var="project_id=$(PROJECT_ID)" \
	  -var="region=$(REGION)" \
	  -var="news_api_key=$(NEWS_API_KEY)"

tf-destroy:
	cd $(TF_DIR) && terraform destroy \
	  -var="project_id=$(PROJECT_ID)" \
	  -var="region=$(REGION)" \
	  -var="news_api_key=$(NEWS_API_KEY)"
```

---

## terraform/versions.tf

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = ">= 5.20.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.5.0"
    }
  }
}
```

---

## terraform/variables.tf

```hcl
variable "project_id" { description = "GCP project ID" type = string }
variable "region"     { description = "Region for Cloud Functions and Scheduler" type = string default = "us-central1" }
variable "location"   { description = "BigQuery dataset location" type = string default = "US" }
variable "news_api_key" { description = "API key for NewsAPI.org" type = string sensitive = true }
```

---

## terraform/schema.json

```json
[
  { "name": "headline", "type": "STRING", "mode": "REQUIRED" },
  { "name": "source", "type": "STRING", "mode": "NULLABLE" },
  { "name": "url", "type": "STRING", "mode": "NULLABLE" },
  { "name": "published_at", "type": "TIMESTAMP", "mode": "NULLABLE" },
  { "name": "sentiment_score", "type": "FLOAT", "mode": "REQUIRED" },
  { "name": "ingested_at", "type": "TIMESTAMP", "mode": "REQUIRED" }
]
```

---

## terraform/main.tf

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_project_service" "services" {
  for_each = toset([
    "cloudfunctions.googleapis.com",
    "cloudbuild.googleapis.com",
    "cloudscheduler.googleapis.com",
    "bigquery.googleapis.com",
    "storage.googleapis.com",
    "iam.googleapis.com",
    "logging.googleapis.com"
  ])
  project            = var.project_id
  service            = each.key
  disable_on_destroy = false
}

resource "random_id" "suffix" {
  byte_length = 4
}

resource "google_storage_bucket" "function_src" {
  name          = "${var.project_id}-news-fn-src-${random_id.suffix.hex}"
  location      = "US"
  force_destroy = true
  uniform_bucket_level_access = true

  depends_on = [google_project_service.services]
}

resource "google_storage_bucket_object" "function_zip" {
  name   = "function.zip"
  bucket = google_storage_bucket.function_src.name
  source = "${path.module}/../function/function.zip"
}

resource "google_bigquery_dataset" "news" {
  dataset_id = "news_sentiment"
  location   = var.location
}

resource "google_bigquery_table" "headlines" {
  dataset_id = google_bigquery_dataset.news.dataset_id
  table_id   = "headlines"
  schema     = file("${path.module}/schema.json")
}

resource "google_service_account" "function_sa" {
  account_id   = "news-function-sa"
  display_name = "News Function Service Account"
}

resource "google_bigquery_dataset_iam_member" "sa_bq_writer" {
  dataset_id = google_bigquery_dataset.news.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = "serviceAccount:${google_service_account.function_sa.email}"
}

resource "google_project_iam_member" "sa_log_writer" {
  role   = "roles/logging.logWriter"
  member = "serviceAccount:${google_service_account.function_sa.email}"
}

resource "google_cloudfunctions_function" "sentiment" {
  name                  = "news-sentiment-fn"
  runtime               = "python311"
  region                = var.region
  entry_point           = "main"
  service_account_email = google_service_account.function_sa.email
  source_archive_bucket = google_storage_bucket.function_src.name
  source_archive_object = google_storage_bucket_object.function_zip.name
  trigger_http          = true
  available_memory_mb   = 256
  timeout               = 120

  environment_variables = {
    NEWS_API_KEY = var.news_api_key
    BQ_TABLE     = "${var.project_id}.${google_bigquery_dataset.news.dataset_id}.${google_bigquery_table.headlines.table_id}"
  }
}

resource "google_cloudfunctions_function_iam_member" "invoker_all" {
  project        = google_cloudfunctions_function.sentiment.project
  region         = google_cloudfunctions_function.sentiment.region
  cloud_function = google_cloudfunctions_function.sentiment.name
  role           = "roles/cloudfunctions.invoker"
  member         = "allUsers"
}

resource "google_cloud_scheduler_job" "fetch_news" {
  name      = "fetch-news-hourly"
  region    = var.region
  schedule  = "0 * * * *"
  time_zone = "UTC"

  http_target {
    http_method = "GET"
    uri         = google_cloudfunctions_function.sentiment.https_trigger_url
  }
}

locals {
  bq_table_fqn = "${var.project_id}.${google_bigquery_dataset.news.dataset_id}.${google_bigquery_table.headlines.table_id}"
}
```

---

## terraform/outputs.tf

```hcl
output "function_url" {
  value       = google_cloudfunctions_function.sentiment.https_trigger_url
  description = "Invoke URL for the Cloud Function"
}

output "bigquery_table" {
  value       = local.bq_table_fqn
  description = "Fully qualified BigQuery table"
}

output "scheduler_name" {
  value       = google_cloud_scheduler_job.fetch_news.name
  description = "Cloud Scheduler job name"
}
```

---

## function/main.py

```python
import os
from datetime import datetime, timezone
import requests
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
from google.cloud import bigquery

NEWS_API_KEY = os.environ.get("NEWS_API_KEY")
BQ_TABLE = os.environ.get("BQ_TABLE")

NEWS_ENDPOINT = "https://newsapi.org/v2/top-headlines"
DEFAULT_PARAMS = {


Here’s the updated **Markdown doc** with clear, ready‑to‑run examples for every Makefile command so this doubles as both **reference** and **how‑to**.

---

# Terraform + Makefile Deployment Workflow

## 📄 Makefile Overview

The **Makefile** provides shorthand commands for packaging, deploying, and tearing down your GCP Cloud Function via Terraform — without memorizing long commands.

---

## 🔧 Variables

```make
PROJECT_ID ?= your-gcp-project-id
REGION ?= us-central1
NEWS_API_KEY ?= replace_me
TF_DIR := terraform
FN_DIR := function
```

- Override defaults inline when running commands:
  ```bash
  make tf-apply PROJECT_ID=my-gcp-id REGION=us-east1 NEWS_API_KEY=abc123
  ```

---

## 🎯 Phony Targets
```make
.PHONY: zip tf-init tf-apply tf-destroy
```
- Declares commands as targets (not files).

---

## 📌 Targets with Examples

### 1. **zip**
```make
zip:
	@cd $(FN_DIR) && rm -f function.zip && zip -qr function.zip main.py requirements.txt
```
**What it does**: Creates a fresh `function.zip` from `main.py` and `requirements.txt` inside the `function/` directory.

**Example Run**:
```bash
make zip
```
**Result**:  
- Any old `function.zip` is removed.
- New `function.zip` is created and ready for deployment.

---

### 2. **tf-init**
```make
tf-init:
	cd $(TF_DIR) && terraform init
```
**What it does**: Prepares the Terraform working directory — downloads provider plugins, sets up the backend.

**Example Run**:
```bash
make tf-init
```
**Result**:  
- Terraform backend initialized (local or remote).
- Ready for `plan`/`apply`.

---

### 3. **tf-apply**
```make
tf-apply: zip
	cd $(TF_DIR) && terraform apply \
	  -var="project_id=$(PROJECT_ID)" \
	  -var="region=$(REGION)" \
	  -var="news_api_key=$(NEWS_API_KEY)"
```
**What it does**:  
- Runs `zip` first to package Cloud Function code.  
- Applies Terraform configuration with your variables.

**Example Run**:
```bash
make tf-apply PROJECT_ID=my-gcp-id REGION=us-central1 NEWS_API_KEY=abcd1234
```
**Result**:  
- Cloud Function zipped and uploaded.
- Terraform provisions/updates all resources (Function, Scheduler, BigQuery, etc.).

---

### 4. **tf-destroy**
```make
tf-destroy:
	cd $(TF_DIR) && terraform destroy \
	  -var="project_id=$(PROJECT_ID)" \
	  -var="region=$(REGION)" \
	  -var="news_api_key=$(NEWS_API_KEY)"
```
**What it does**:  
- Destroys all infrastructure Terraform created for this project.

**Example Run**:
```bash
make tf-destroy PROJECT_ID=my-gcp-id REGION=us-central1 NEWS_API_KEY=abcd1234
```
**Result**:  
- Cloud Function, scheduler, datasets, and other linked resources are removed.

---

## 🛠 Terraform Execution Options

### Local Execution (Development)
1. Install Terraform:
   ```bash
   brew install terraform
   ```
2. Authenticate with GCP:
   ```bash
   gcloud auth application-default login
   ```
3. Deploy:
   ```bash
   make tf-apply PROJECT_ID=my-gcp-id REGION=us-east1 NEWS_API_KEY=abcd1234
   ```

---

## 🚀 Local-to-Remote Terraform Workflow

### Phase 1 — Local + Remote State
```bash
gsutil mb -p $PROJECT_ID -l $REGION gs://my-tf-state-bucket/
```
Backend config (`terraform/main.tf`):
```hcl
terraform {
  backend "gcs" {
    bucket = "my-tf-state-bucket"
    prefix = "cloud-function"
  }
}
```
Initialize:
```bash
make tf-init
```

---

### Phase 2 — Version Control
- Store all infra code in Git.
- `main` = production‑ready infra.

---

### Phase 3 — CI/CD Integration
Pipeline steps:
```bash
terraform init
terraform plan -out=tfplan
terraform apply -auto-approve tfplan
```

---

### Phase 4 — Gradual Cutover
- Dev/test locally.
- Merge to `main` to trigger CI/CD.

---

### Phase 5 — Fully Remote
- All deploys run via pipeline.
- Notifications for changes.

---

## 🌐 Workflow Diagram
```
Mac (local dev) ───► Remote GCS state ───► Git Push ───► CI/CD Runner ───► Terraform Apply ───► GCP Infra
```

---

If you’d like, I can now **add a working GitHub Actions `.yml` pipeline** section here so this doc includes the exact automation script to run `make tf-apply` in CI without any guesswork. That would make it instantly production‑deployable from GitHub. Would you like me to append that?
