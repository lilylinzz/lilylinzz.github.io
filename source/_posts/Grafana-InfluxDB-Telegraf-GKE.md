---
title: Grafana/InfluxDB/Telegraf (GKE)
date: 2025-03-10 17:14:40
description: 本篇為使用 GKE 建置 Grafana、InfluxDb、Telegraf 監控工具之操作步驟紀錄
tags: 
    - k8s
    - gke
    - monitor
---

## 建置StorageClass

### 配置
* StorageClass

### 建立 `sc.yaml`
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: grafana
  namespace: lily2
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
reclaimPolicy: Delete #回收策略
allowVolumeExpansion: true #可擴展
volumeBindingMode: WaitForFirstConsumer
#改為等到pod創建才綁定PV，預設是Immediate立即綁頂
```

apply
```
kubectl apply -f sc.yaml
```

---

## 建置Grafana

### 配置 
* Grafana本人 - Deployment、Service 
* 數據儲存 - PVC

### 建立 `grafana.yaml`
```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: lily2
  labels:
    app: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: lily-pool
      containers:
      - name: grafana
        image: grafana/grafana-enterprise:9.0.2
        imagePullPolicy: IfNotPresent
        env: 
        - name: GF_SECURITY_ADMIN_USER
          value: lily
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: lily
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: grafana-pvc
          mountPath: /var/lib/grafana
	  volumes:
      - name: grafana-pvc
        persistentVolumeClaim:
          claimName: grafana-pvc
        # resources:
        #   limits:
        #     memory: 200Mi
        #     cpu: 200m
        #   requests:
        #     cpu: 100m
        #     memory: 100Mi
      #pvc有權限問題，需加這段調整權限
      securityContext:
        runAsUser: 0
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: lily2
  labels:
    app: grafana
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: lily2
spec:
  type: LoadBalancer
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
```

apply
```
> kubectl apply -f grafana.yaml
```


---

## 建置InfluxDB

### 配置
* influxdb-config - ConfigMap 
* 存帳密資訊 - Secret
* InfluxDB本人 - StatefulSet、Service 
* 儲存數據 - PVC

### 建立 `influxdb-config.yaml` 
- 內容參考此篇 [[influxdb-config.yaml]](https://hackmd.io/BNxmcbB3SM2PIL5MEXSpyg) 


### 建立 `influxdb.yaml`
```
apiVersion: v1
kind: Secret
metadata:
  name: influxdb-secret
  namespace: lily2
type: Opaque
stringData: 
  INFLUXDB_CONFIG_PATH: /etc/influxdb/influxdb.conf
  INFLUXDB_DATABASE: test
  INFLUXDB_USERNAME: admin2
  INFLUXDB_PASSWORD: 123456
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: influxdb
  namespace: lily2
  labels:
    app: influxdb
spec:
  serviceName: influxdb
  replicas: 1
  selector:
    matchLabels:
      app: influxdb
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: lily-pool
      containers:
      - name: influxdb
        image: influxdb:1.8.4
        imagePullPolicy: IfNotPresent
        ports:
        - name: influxdb
          containerPort: 8086
          protocol: TCP
        volumeMounts:
        - name: influxdb-data
          mountPath: /var/lib/influxdb
        - name: influxdb-config
          mountPath: /etc/influxdb/influxdb.conf
          subPath: influxdb.conf
          readOnly: true
        envFrom:
        - secretRef:
            name: influxdb-secret
      volumes:
      - name: influxdb-data
        persistentVolumeClaim:
          claimName: influxdb-pvc
      - name: influxdb-config
        configMap:
         name: influxdb-config
---
apiVersion: v1
kind: Service
metadata:
  name: influxdb
  namespace: lily2
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - port: 8086
      targetPort: 8086
      name: influxdb
  selector:
    app: influxdb
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: influxdb-pvc
  namespace: lily2
  labels:
    app: influxdb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 300Mi
```

apply
```
> kubectl apply -f .
```

---


## 建置Telegraf

### 配置:
* telegraf-config - ConfigMap 
* rbac - ClusterRole、ClusterRoleBinding、ServiceAccount
* telegraf - DaemonSet 

### 建立 `telegraf-config.yaml`
- 內容參考此篇 [[telegraf-config.yaml]](https://hackmd.io/BNxmcbB3SM2PIL5MEXSpyg?view#telegrafconf) 

:::info
注意 telegraf.config 要加這段 [[inputs.kube_inventory]]
token 是從 ServiceAccount 創建的Secret取得

```
    [[inputs.kube_inventory]]
      # URL for the kubernetes API
      url = "https://172.16.0.2"
      namespace = ""  
      # namespace空著表示採集所有namespace
      # bearer_token = "/usr/local/telegraf/etc/telegraf/token.csv" # 讀取存放token的檔案，或是用下列方法直接使用token
      bearer_token_string = "eyJhbGciOiJSUzI1NiIsImtp......"
      response_timeout = "5s"
      insecure_skip_verify = true
```
:::


### 建立 `telegraf.yaml`
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: telegraf
  namespace: lily2
  labels:
    app: telegraf
spec:
  selector:
    matchLabels:
      app: telegraf
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: telegraf
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: lily-pool
      serviceAccountName: "telegraf"
      containers:
      - name: telegraf
        image: telegraf:1.25.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 200Mi
            cpu: 200m
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: telegraf-config
          mountPath: /etc/telegraf/telegraf.conf
          readOnly: true
          subPath: telegraf.conf
      terminationGracePeriodSeconds: 30
      volumes:
      - name: telegraf-config
        configMap:
          name: telegraf-config
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: influx:cluster:viewer
  namespace: lily2
  labels:
    rbac.authorization.k8s.io/aggregate-view-telegraf: "true"
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes", "nodes"]
    verbs: ["get", "list"]

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: influx:telegraf
  namespace: lily2
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.authorization.k8s.io/aggregate-view-telegraf: "true"
    - matchLabels:
        rbac.authorization.k8s.io/aggregate-to-view: "true"
rules: []
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: influx:telegraf:viewer
  namespace: lily2
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: influx:telegraf
subjects:
  - kind: ServiceAccount
    name: telegraf
    namespace: lily2
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: telegraf
  namespace: lily2
```

apply
```
kubectl apply -f .
```


---

## 補充

### 參考資料

k8s建立grafana
https://octoperf.com/blog/2019/09/19/kraken-kubernetes-influxdb-grafana-telegraf/#check-the-influxdb-deployment
telegraf資料
https://www.ywdevops.cn/index.php/2022/06/10/telegraf-5/
https://godleon.github.io/blog/Kubernetes/k8s-How-to-access-resource-legally/

---
### influxdb操作

influxdb驗證
https://docs.influxdata.com/influxdb/v1.7/administration/authentication_and_authorization/

INFLUXDB 身分驗證
|> influx
|> auth
創建管理員用戶
|> create user <*admin*> with password '<*password*>' with all privileges
創建非管理員用戶
|> create user <*username*> with password '<*password*>'

---
### InitContainer

Grafana v9 使用pvc會有權限問題
錯誤訊息: 
(1)GF_PATHS_DATA='/var/lib/grafana' is not writable. 
(2)mkdir: can't create directory '/var/lib/grafana/plugins': Permission denied
原因: grafana預設讀取權限 grafana:root, 但掛載的方式會將檔案權限更改為 root:root, 因此須將掛載的權限改為grafana所需
結論: ~~使用initContainer可解決，參考: https://cloud.tencent.com/developer/article/1856854~~
加入這段可解決

```
      #調整權限
      securityContext:
        runAsUser: 0
```
