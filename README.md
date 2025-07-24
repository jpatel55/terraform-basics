# Terraform for GCP Data Engineering: A Study Guide

This repository contains a comprehensive study guide for understanding Terraform and its application in a Google Cloud Platform (GCP) data engineering context. It covers fundamental concepts, practical examples, and common interview questions.

## Table of Contents

- [What is Terraform?](#what-is-terraform)
- [Core Concepts & Terminology](#core-concepts--terminology)
- [The Terraform Workflow](#the-terraform-workflow)
- [Terraform with GCP: Data Engineering Examples](#terraform-with-gcp-data-engineering-examples)
  - [Setup: Connecting Terraform to GCP](#setup-connecting-terraform-to-gcp)
  - [Example 1: Creating a GCS Bucket](#example-1-creating-a-gcs-bucket)
  - [Example 2: A Simple Data Pipeline](#example-2-a-simple-data-pipeline)
- [Advanced Topics & Interview Questions](#advanced-topics--interview-questions)
  - [Managing Multiple Environments (dev/prod)](#managing-multiple-environments-devprod)
  - [Handling Infrastructure Drift](#handling-infrastructure-drift)
  - [Managing Secrets](#managing-secrets)
- [Real-World Module Example: GCP Data Landing Zone](#real-world-module-example-gcp-data-landing-zone)
  - [Project Structure](#project-structure-)
  - [Step 1: Creating the Reusable Module](#step-1-creating-the-reusable-module)
  - [Step 2: Using the Module in a Project](#step-2-using-the-module-in-a-project)
  - [Why This is a Good Real-World Example](#why-this-is-a-good-real-world-example-)

---

## What is Terraform?

Terraform is an **Infrastructure as Code (IaC)** tool created by HashiCorp. It lets you define and manage your cloud infrastructure (like servers, databases, storage, and networking) using human-readable configuration files instead of manually clicking through a graphical user interface like the GCP Console.

It's **declarative**, meaning you write code to describe your desired "end state," and Terraform figures out how to create, update, or delete resources to make the real-world infrastructure match your code.

## Core Concepts & Terminology

-   **Provider**: A plugin that allows Terraform to interact with a specific API. For our purposes, the most important one is the `google` provider, which connects Terraform to GCP.
-   **Resource**: A single piece of infrastructure managed by Terraform. Examples in GCP include a `google_storage_bucket`, a `google_bigquery_dataset`, or a `google_compute_instance`.
-   **State File (`terraform.tfstate`)**: A critical JSON file that acts as a map between your configuration code and the real-world resources it manages. It should be stored remotely (e.g., in a GCS bucket) and locked when in use to prevent team conflicts.
-   **Configuration Language (HCL)**: The language used for `.tf` files. It's designed to be easy to read and write.
-   **Module**: A reusable, self-contained package of Terraform configurations. Modules help you organize code and avoid repetition.
-   **Variables**: Used to parameterize your code, making it flexible for different environments (e.g., dev, prod).
-   **Outputs**: Used to extract information from your infrastructure after it's created (e.g., the URL of a new GCS bucket).

## The Terraform Workflow

The workflow is a simple, three-step command-line process.

1.  **`terraform init`**: Initializes your working directory. It downloads the necessary provider plugins and sets up the backend for your state file. Run this once per project.

2.  **`terraform plan`**: Creates an execution plan. This is a **dry run** that shows you what actions (create, update, or destroy) Terraform *will* take to match your configuration. It's a critical safety step.

3.  **`terraform apply`**: Executes the plan. This command applies the changes to your GCP environment.

A fourth command, **`terraform destroy`**, will tear down all the resources managed by the configuration.

---

## Terraform with GCP: Data Engineering Examples

### Setup: Connecting Terraform to GCP

First, you need a `main.tf` file to configure the `google` provider.

```hcl
# main.tf

terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0" # Pin to a major version
    }
  }
}

# Configure the Google Cloud provider
provider "google" {
  project = "your-gcp-project-id" # Your Project ID
  region  = "us-central1"
}
```
**Authentication**: Terraform uses credentials from your local environment. Run `gcloud auth application-default login` before you begin.

### Example 1: Creating a GCS Bucket

A GCS bucket is the cornerstone of most data pipelines.

```hcl
# gcs.tf

resource "google_storage_bucket" "data_lake" {
  name          = "de-interview-data-lake-bucket" # Must be globally unique
  location      = "US-CENTRAL1"
  force_destroy = true # Allows deletion of the bucket even if it has objects

  # Add labels for organization and cost tracking
  labels = {
    env      = "development"
    pipeline = "customer-data"
  }
}
```

### Example 2: A Simple Data Pipeline

Let's provision a pipeline: **GCS Bucket** -> **Cloud Function** -> **BigQuery Table**. Terraform manages the dependencies between these resources.

```hcl
# pipeline.tf

# 1. The BigQuery Dataset
resource "google_bigquery_dataset" "raw_data" {
  dataset_id = "raw_customer_data"
  location   = "US-CENTRAL1"
  description = "Dataset for raw data from GCS"
}

# 2. The BigQuery Table
resource "google_bigquery_table" "customers" {
  dataset_id = google_bigquery_dataset.raw_data.dataset_id
  table_id   = "customers_raw"

  # Define the schema for the table
  schema = <<EOF
[
  {"name": "customer_id", "type": "STRING", "mode": "REQUIRED"},
  {"name": "first_name", "type": "STRING", "mode": "NULLABLE"},
  {"name": "ingestion_timestamp", "type": "TIMESTAMP", "mode": "NULLABLE"}
]
EOF
}

# 3. The GCS Bucket (our trigger source)
resource "google_storage_bucket" "landing_zone" {
  name     = "de-interview-landing-zone"
  location = "US-CENTRAL1"
}

# 4. The Cloud Function to process the data
# (Note: This assumes you have the function's code zipped and in a bucket)
resource "google_cloudfunctions_function" "gcs_to_bq_loader" {
  name        = "gcs-to-bigquery-loader"
  runtime     = "python39"
  entry_point = "process_gcs_event" # The function name within your Python code

  source_archive_bucket = "your-source-code-bucket"
  source_archive_object = "functions/gcs_to_bq.zip"

  # This is the trigger!
  event_trigger {
    event_type = "google.storage.object.finalize" # Triggers on new file creation
    # This is how we link resources!
    # We are referencing the bucket created earlier in this file.
    resource   = google_storage_bucket.landing_zone.name
  }

  # Explicitly wait for the BigQuery table to be created first
  depends_on = [
    google_bigquery_table.customers
  ]
}
```

---

## Advanced Topics & Interview Questions

### Managing Multiple Environments (dev/prod)

> **Question:** "How would you manage infrastructure for dev, staging, and prod environments without duplicating code?"

**Answer:** Use **Terraform Workspaces**. A workspace is an independent instance of a state file. You can create a workspace for each environment (`terraform workspace new dev`) and use different variable files (`.tfvars`) for each one to supply different values (e.g., smaller machine types for dev).

- **`variables.tf`** (defines the variables)
    ```hcl
    variable "machine_type" {
      description = "The machine type for our compute instances."
      type        = string
    }
    ```
- **`dev.tfvars`** (values for the 'dev' workspace)
    ```hcl
    machine_type = "e2-micro"
    ```
- **`prod.tfvars`** (values for the 'prod' workspace)
    ```hcl
    machine_type = "n2-standard-4"
    ```
- Apply a specific configuration with: `terraform apply -var-file="dev.tfvars"`

### Handling Infrastructure Drift

> **Question:** "What happens if a teammate manually changes a resource in the GCP Console that was created by Terraform?"

**Answer:** This is called **infrastructure drift**. `terraform plan` will detect the difference between your configuration file (the desired state) and the actual resource in GCP (the drifted state). `terraform apply` will then correct the drift by changing the resource back to match the code. This enforces your code as the single source of truth.

### Managing Secrets

> **Question:** "How do you handle sensitive data like passwords or API keys in Terraform?"

**Answer:** **Never hardcode secrets** in `.tf` files. The best practice is to use a dedicated secrets management tool like **Google Secret Manager**. You can use a Terraform `data` source to fetch the secret at runtime, so the value never appears in your code or state file.

```hcl
# Fetch the latest version of a secret from Secret Manager
data "google_secret_manager_secret_version" "db_password" {
  secret = "my-database-password"
}

resource "google_sql_database_instance" "main" {
  name     = "my-instance"
  settings {
    # ... other settings
  }
  # Reference the secret data fetched above
  root_password = data.google_secret_manager_secret_version.db_password.secret_data
}
```

---

## Real-World Module Example: GCP Data Landing Zone

A module is a reusable function for your infrastructure. Let's create a **"Data Landing Zone"** module that provisions a GCS bucket, a BigQuery dataset, and a dedicated Service Account.

### Project Structure 📁

```
.
├── main.tf                 # Root configuration - USES the modules
├── terraform.tfvars        # Input values for our environments
└── modules/
    └── gcp_data_landing_zone/
        ├── main.tf         # The actual resource definitions (GCS, BQ, SA)
        ├── variables.tf    # The module's INPUTS (e.g., project name)
        └── outputs.tf      # The module's OUTPUTS (e.g., the bucket name)
```

### Step 1: Creating the Reusable Module

This is the code inside the `modules/gcp_data_landing_zone/` directory.

#### `variables.tf` (The Module's Inputs)
```hcl
variable "project_name" {
  type        = string
  description = "The logical name for the data project (e.g., 'customer_analytics')."
}

variable "environment" {
  type        = string
  description = "The deployment environment (e.g., 'dev', 'prod')."
  default     = "dev"
}

variable "gcp_location" {
  type        = string
  description = "The GCP region for the resources."
}
```

#### `main.tf` (The Module's Logic)
```hcl
# 1. Create a dedicated Service Account
resource "google_service_account" "landing_zone_sa" {
  account_id   = "sa-${var.project_name}-${var.environment}"
  display_name = "Service Account for ${var.project_name}"
}

# 2. Create the GCS Bucket for raw data
resource "google_storage_bucket" "landing_zone_bucket" {
  name     = "${var.project_name}-landing-${var.environment}"
  location = var.gcp_location
  labels   = { project = var.project_name, environment = var.environment }
}

# 3. Create the BigQuery Dataset
resource "google_bigquery_dataset" "data_project_dataset" {
  dataset_id = replace(var.project_name, "-", "_") # BQ doesn't allow hyphens
  location   = var.gcp_location
  labels     = { project = var.project_name, environment = var.environment }
}

# 4. Grant the Service Account necessary permissions
resource "google_storage_bucket_iam_member" "sa_bucket_writer" {
  bucket = google_storage_bucket.landing_zone_bucket.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.landing_zone_sa.email}"
}
resource "google_bigquery_dataset_iam_member" "sa_bq_editor" {
  dataset_id = google_bigquery_dataset.data_project_dataset.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = "serviceAccount:${google_service_account.landing_zone_sa.email}"
}
```

#### `outputs.tf` (The Module's Outputs)
```hcl
output "bucket_name" {
  description = "The name of the GCS landing bucket."
  value       = google_storage_bucket.landing_zone_bucket.name
}

output "dataset_id" {
  description = "The ID of the BigQuery dataset."
  value       = google_bigquery_dataset.data_project_dataset.dataset_id
}

output "service_account_email" {
  description = "The email of the dedicated service account."
  value       = google_service_account.landing_zone_sa.email
}
```

### Step 2: Using the Module in a Project

Now, in the root `main.tf`, you can call this module with very little code.

```hcl
# main.tf in the root directory

# Create a landing zone for the customer data project
module "customer_data_landing_zone" {
  source = "./modules/gcp_data_landing_zone" # Path to the module

  # Provide values for the module's variables
  project_name = "customer-data"
  environment  = "prod"
  gcp_location = "us-central1"
}

# REUSE the module to create another landing zone for the marketing team
module "marketing_data_landing_zone" {
  source = "./modules/gcp_data_landing_zone" # Same module, different inputs

  project_name = "marketing-analytics"
  environment  = "dev"
  gcp_location = "us-central1"
}

# You can access the module's outputs like this:
output "customer_data_bucket" {
  value = module.customer_data_landing_zone.bucket_name
}

output "marketing_service_account" {
  value = module.marketing_data_landing_zone.service_account_email
}
```

### Why This is a Good Real-World Example 🚀

-   **Abstraction**: The root `main.tf` is simple. You don't need to know *how* the landing zone is built, just that you need one.
-   **Reusability**: We provisioned infrastructure for two projects without copying any code, reducing errors and maintenance.
-   **Consistency**: Both landing zones are built exactly the same way, with the same IAM policies and naming conventions, enforcing best practices.
