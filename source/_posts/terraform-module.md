---
title: Terraform module 練習
date: 2025-03-14 14:59:06
description: Terraform 使用自建 Module 建立 GKE 叢集，用以了解 Module 概念。之實作紀錄。
categories:
    - [技術文件, terraform]
tags:
    - terraform
    - gcp
    - IaC
---

## 模組目錄結構
```
project-root/
├── main.tf           # 主要的模組調用文件
├── variables.tf      # （可選）根層級的變數定義
├── outputs.tf        # （可選）根層級的輸出定義
├── provider.tf       # provider 配置
├── terraform.tfvars  # （可選）變數值配置文件
└── modules/
    └── gke/
        ├── main.tf       # GKE 模組的主要資源定義
        ├── variables.tf  # GKE 模組的變數定義
        ├── outputs.tf    # GKE 模組的輸出定義
        └── versions.tf   # （可選）模組的版本要求
```

## modules 編寫
1.首先在根目錄創建資料夾 “modules/gke”
```
## 先建立這個部分
modules/
    └── gke/ 
        ├── main.tf       # GKE 模組的主要資源定義
        ├── variables.tf  # GKE 模組的變數定義
        └── outputs.tf    # GKE 模組的輸出定義
```

---

2.`modules/gke/`底下建立檔案 `main.tf`，寫我們的gke配置
```
# 獲取目前專案配置
data "google_client_config" "default" {}

# 創建 GKE 叢集
resource "google_container_cluster" "cluster" {
  name     = var.cluster_name
  location = var.zone

  # 我們需要先創建一個默認節點池，之後會移除它
  remove_default_node_pool = true
  initial_node_count       = 1

  # 網路配置
  network    = var.network
  subnetwork = var.subnetwork

  # 叢集配置
  release_channel {
    channel = var.release_channel
  }

  # 啟用工作負載身份
  workload_identity_config {
    workload_pool = "${var.project}.svc.id.goog"
  }

  # 啟用其他功能
  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
  }
}

#  創建節點池
resource "google_container_node_pool" "nodes" {
  name     = var.node_pool_name
  location = var.zone
  cluster  = google_container_cluster.cluster.name

  # 節點數量配置
  initial_node_count = var.initial_node_count

  # 自動擴縮配置
  autoscaling {
    min_node_count = var.min_node_count
    max_node_count = var.max_node_count
  }

  # 節點配置
  node_config {
    machine_type = var.machine_type
    labels       = var.node_labels

    # OAuth 範圍
    oauth_scopes = var.oauth_scopes
  }

  # 管理配置
  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```

---

3.`modules/gke/` 底下建立檔案 `variables.tf，寫變數配置
```
variable "project" {
  description = "GCP Project ID"
  type        = string
  default     = "project-id"
}

variable "region" {
  type    = string
  default = "asia-east1"
}

variable "zone" {
  type    = string
  default = "asia-east1-c"
}

variable "cluster_name" {
  type    = string
  default = "lily-tf-cluster"
}

variable "network" {
  description = "VPC 網路名稱"
  type        = string
  default     = "default"
}

variable "subnetwork" {
  description = "子網路名稱"
  type        = string
  default     = "default"
}

variable "release_channel" {
  description = "GKE 版本發布頻道"
  type        = string
  default     = "REGULAR"
}

variable "node_pool_name" {
  description = "節點池名稱"
  type        = string
  default     = "primary-node-pool"
}

variable "initial_node_count" {
  description = "初始節點數量"
  type        = number
  default     = 1
}

variable "min_node_count" {
  description = "最小節點數量"
  type        = number
  default     = 1
}

variable "max_node_count" {
  description = "最大節點數量"
  type        = number
  default     = 1
}

variable "machine_type" {
  description = "節點機器類型"
  type        = string
  default     = "e2-micro"
}

variable "node_labels" {
  description = "節點標籤"
  type        = map(string)
  default     = {
    env = "development"
  }
}

variable "oauth_scopes" {
  description = "節點 OAuth 範圍"
  type        = list(string)
  default     = [
    "https://www.googleapis.com/auth/logging.write",
    "https://www.googleapis.com/auth/monitoring",
    "https://www.googleapis.com/auth/devstorage.read_only",
    "https://www.googleapis.com/auth/service.management.readonly",
    "https://www.googleapis.com/auth/servicecontrol",
    "https://www.googleapis.com/auth/trace.append"
  ]
}
```

---
4.`modules/gke/`底下建立檔案 `output.tf`，寫 output 配置
```
output "cluster_name" {
  value       = google_container_cluster.cluster.name
  description = "GKE 叢集名稱"
}

output "cluster_endpoint" {
  value       = google_container_cluster.cluster.endpoint
  description = "GKE 叢集端點"
}

output "cluster_ca_certificate" {
  value       = base64decode(google_container_cluster.cluster.master_auth[0].cluster_ca_certificate)
  sensitive   = true
  description = "GKE 叢集 CA 證書"
}
```

---

---
### 主配置編寫
1.根目錄創建 `main.tf`，引用 gke 模組
```
module "gke" {
  source = "./modules/gke"

  project   = "project-id"
  cluster_name = "cluster-name"
}
```
---

註: 變數傳遞流程如下
>根目錄 [variables.tf](http://variables.tf/)
↓
根目錄 [main.tf](http://main.tf/)
↓
模組內 [variables.tf](http://variables.tf/)
↓
模組內 [main.tf](http://main.tf/)

---

2.根目錄創建 `provider.tf` ，需要添加 kubernetes provider
```
# CONFIGURATION
# ------------------------------
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "4.83.0"
    }
  }
}


# PROVIDERS
# ------------------------------
provider "google" {
  credentials = file(var.credentials_file)
  project     = var.project
  region      = var.region
  zone        = var.zone
}

provider "kubernetes" {
  host                   = "https://${module.gke.cluster_endpoint}"     # GKE 叢集的 API Server 端點
  cluster_ca_certificate = base64decode(module.gke.cluster_ca_certificate)     # GKE 叢集的 CA 憑證，用於驗證 API Server
  token                  = data.google_client_config.default.access_token     # 用於認證的 access token
}
```