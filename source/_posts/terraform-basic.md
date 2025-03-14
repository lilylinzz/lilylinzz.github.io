---
title: Terraform 初學
date: 2025-03-14 10:57:53
description: 從安裝terraform，到透過terraform在GCP建立幾個功能，最最新手的練習過程紀錄。
categories: 
    - [技術文件, terraform]
tags: 
    - terraform
    - gcp
    - IaC
---

## 前置作業
### linux 安裝 Terraform
```
wget https://releases.hashicorp.com/terraform/0.14.8/terraform_0.14.8_linux_amd64.zip
sudo apt install unzip
sudo unzip terraform_0.14.8_linux_amd64.zip -d /usr/local/bin
terraform --version
```
- Terraform版本0.14.8 / google provider版本6.9.0
- [Terraform官網](https://developer.hashicorp.com/terraform/tutorials/gcp-get-started/install-cli) 說明很詳細

---
可以加入別名(Alias)，之後操作比較方便
1.打開終端機的 .basrc 檔案 `vim ~/.basrc`
2.在文件最下方加入

```
alias tf="terraform"
```

---

### 終端機登入GCP
[官網](https://cloud.google.com/sdk/docs/install) 照做就好

裝好後登入
```
gcloud auth login
gcloud config set project PROJECT_ID
```

---

## Terraform 實作
### lab 1 - 建立 vpc 網路

1.找個適當的目錄底下，建立 `main.tf`，內容如下
```
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "4.83.0"
    }
  }
}

provider "google" {
  credentials = file("lab-terraform-key.json")

  project = "project-id"
  region  = "asia-east1"
  zone    = "asia-east1-c"
}

resource "google_compute_network" "vpc_network" {
  name = "terraform-network"
}
```
> credentials = GCP服務帳務金鑰(在下一步說明)
> peoject = GCP的專案ID

---

2.建立服務帳戶金鑰
* GCP服務帳戶 > 建立服務帳戶 > 存下金鑰
* IAM 替服務帳戶設定適當的權限
* terraform 根目錄下建立檔案，輸入以下
    ```
    {
  "type": "service_account",
  "project_id": "",             ## 專案id
  "private_key_id": "",         ## 金鑰id
  "private_key": "",            ## 剛剛存下的金鑰內容
  "client_email": "",           ## service accound
  "client_id": "",              ## 服務帳戶的id
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/terraform%40lab-0808.iam.gserviceaccount.com",
  "universe_domain": "googleapis.com"
}
    ```

---

3.初始化 - 當有新的配置，需此步驟下載配置中定義的provider
```
terraform init
```

4.使用fmt格式化代碼，確保代碼風格一致且易讀 / 使用validate驗證代碼有效
```
terraform fmt
terraform validate
```

5.預覽 - 可預覽將要異動的資源，不會執行
```
terraform plan
```

6.建立 - 預覽確認沒有問題，輸入yes則執行
```
terraform apply
```
---
3-6步驟，因前面有加入別名，在輸入時都可將 terraform 簡化為 tf
```
tf init
tf fmt
tf validate
tf plan
tf apply
```

---

註: plan 與 apply 時，終端都會秀出異動的項目，前綴標籤分別有不同意思
> "+" create 建立新項目
  "~" update in-place 不停機更新
 "-/+" destroy and then create replacement 將銷毀並重新建立資源

---

### lab 2 - 建立vm instance
```
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "4.83.0"
    }
  }
}

provider "google" {
  credentials = file("lab-terraform-key.json")

  project = "lab-0808"
  region  = "asia-east1"
  zone    = "asia-east1-c"
}

resource "google_compute_instance" "vm_instance" {
  name         = "terraform-lilytest"
  machine_type = "n1-standard-1"
  zone         = "asia-east1-c"

  boot_disk {
    initialize_params {
      image = "projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20240229"
      size = "10"
      type = "projects/lab-0808/zones/asia-east1-c/diskTypes/pd-standard"
    }
  }

  network_interface {
    network = "default"
  }
}
```

---

### lab 3 - 配置變數
可以從前面的 lab 1或2延伸繼續，根目錄新增檔案 `variables.tf`  設定變數
```
variable "project" { }

variable "credentials_file" { }

variable "region" {
  default = "asia-east1"
}

variable "zone" {
  default = "asia-east1-c"
}
```

編輯 `main.tf` 將 project、credentials_fail、region、zone 改為變數
```
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "4.83.0"
    }
  }
}

provider "google" {
  credentials = file(var.credentials_file)

  project = var.project
  region  = var.region
  zone    = var.zone
}
```


