---
title: Terraform for_each 練習
date: 2025-03-14 15:19:58
description: Terraform 使用自建 for_each 建立 IP Address，以及建立 gke 節點池，兩個練習用以了解 for_each 概念。之實作紀錄。
categories:
    - [技術文件, terraform]
tags:
    - terraform
    - gcp
    - IaC
---
## 練習1：建立 IP Address

---
`main.tf`
```
resource "google_compute_global_address" "default" {
  for_each = var.global_addresses
  name     = each.value.name
}
```

---
`variables.tf`
```
variable "global_addresses" {
  description = "Global addresses 配置"
  type = map(object({
    name = string
  }))
```

---
`terraform.tfvars`
```
global_addresses = {
  "address-1" = {
    name = "lily-tf-address-1"
  },
  "address-2" = {
    name = "lily-tf-address-2"
  }
}
```

---
## 練習2：一個 gke 叢集下建立兩個 pool
`main.tf`
```
module "gke_node_pool" {
  source   = "./modules/node-pool"
  for_each = var.node_pools

  pool_name    = each.value.pool_name
  location     = "asia-east1-a"
  cluster_name = "lab-gke"

  node_count   = each.value.node_count
  disk_size_gb = each.value.disk_size_gb
  labels       = each.value.labels
}
```

---
`variables.tf`
```
variable "node_pools" {
  description = "Node pools 配置的 Map"
  type = map(object({
    pool_name    = string
    node_count   = number
    disk_size_gb = number
    labels       = map(string)
  }))
```

---
`terraform.tfvars`
```
node_pools = {
  "pool-1" = {
    pool_name    = "lily-tf-pool-1"
    node_count   = 1
    disk_size_gb = 100
    labels = {
      environment = "lilytest-1"
    }
  },
  "pool-2" = {
    pool_name    = "lily-tf-pool-2"
    node_count   = 0
    disk_size_gb = 100
    labels = {
      environment = "lilytest-2"
    }
  }
}
```

---

註1: 使用變數自訂義檔案 `.tfvars` 的好處是：
- 配置更靈活，可以從外部文件(.tfvars)修改
- 可以為不同環境設定不同的 tfvars 文件
- 變數類型有明確的定義，更容易維護


註2: 為不同環境設定不同的 tfvars 文件，用法:

1.建立特定檔案
```
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── dev.tfvars
│   ├── prod.tfvars
│   └── staging.tfvars
```
2.執行時明確指定
```
# 開發環境
terraform plan -var-file="dev.tfvars"
# 生產環境
terraform plan -var-file="prod.tfvars"
# 測試環境
terraform plan -var-file="staging.tfvars"
```
3.未指定檔案時，預設則會是 `terraform.tfvars`