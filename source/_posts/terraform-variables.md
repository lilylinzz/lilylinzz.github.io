---
title: Terraform 變數概念
date: 2025-03-14 14:41:07
description: 
categories: 
    - [技術文件, terraform]
tags: 
    - terraform
    - gcp
    - IaC
---

### 變數使用優先權

1.命令列參數 (最優先) : 使用 terraform apply -var="" 帶的變數
2.Terraform 環境變數 : 使用 TV_VAR_foo 環境變數定義的變數
3.Terraform 變數文件 : 使用 terraform.tvfars 文件定義變數
4.Terraform 設定文件 : 使用 variables 區塊定義變數
5.變數預設值

<!--more-->
---

## 變數種類
```
# 字串變數（String variables）：用於定義字符串值
variable "region" {
  type    = string
  default = "asia-east1-c"
}

# 數字變數（Number variables）：用於定義數字值
variable "instance_count" {
  type    = number
  default = 123
}

# 布林變數（Boolean variables）：用於定義布林值（True 或 False）
variable "use_ssl" {
  type    = bool
  default = true
}

# 列表變數（List variables）：用於定義多個值
variable "tags" {
  type    = list(string)
  default = ["qc", "rtm", "online"]
}

# 映射變數（Map variables）：用於定義鍵值對（Key-Value Pairs）
variable "region_zone_map" {
  type = map(string)
  default = {
    area1 = "asia-east1-a"
    area2 = "asia-east1-b"
    area3 = "asia-east1-c"
  }
}
```




