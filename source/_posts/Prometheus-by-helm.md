---
title: Prometheus-by-helm
date: 2025-08-27 16:33:40
description: Kube-Prometheus-Stack 已經將整組 prometheus 打包好，非常方便，本篇簡單紀錄安裝過程。
categories:
  - [技術文件, k8s]
tags:
  - k8s
  - helm
  - monitor
---

## Helm 安裝
安裝方式可參考 [Helm官網](https://helm.sh/zh/docs/intro/install/)

---
## 安裝 Kube-Prometheus-Stack
來源： [Kube-Prometheus-Stack](https://github.com/prometheus-community/helm-charts)

### 1.在本地添加 Helm Chart 倉庫
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. 將 values.yaml 複製到本機，以便後續調整

### 3. helm install
--values 改為自己的 values.yaml 檔案路徑
```
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack --values=values.yaml -n prometheus --create-namespace
```

> 到這邊基本上就算是安裝好了

## 確認 Dashboard
### 1.導出 Prometheus Dashboard
```
kubectl port-forward service/prometheus-stack-kube-prom-prometheus 9090:9090 -n prometheus
```

### 2.導出 Grafana Dashboard
```
kubectl port-forward service/prometheus-stack-grafana 3000:80 -n prometheus
```
> Grafana 預設帳密：admin/prom-operator