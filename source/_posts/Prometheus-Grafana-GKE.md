---
title: Prometheus/Grafana (GKE)
date: 2025-03-11 10:50:29
description: 本篇為使用 GKE 建置 Prometheus、Grafana 監控工具，之操作步驟紀錄。
categories: 
  - [技術文件, k8s]
tags:
    - k8s
    - gke
    - monitor
---

## Prometheus 架構圖
![](https://i.imgur.com/ubC5Dxe.png)

來源: [yasongxu|container-monitor](https://github.com/yasongxu/container-monitor/blob/master/opensource/agent/prometheus/Prometheus%E5%9F%BA%E6%9C%AC%E6%9E%B6%E6%9E%84.md

---
## 建置Prometheus
### 1.創建 ClusterRole + ServiceAccount
`prometheus-rbac.yaml`
```yaml=
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    name: prometheus
  name: prometheus
  namespace: monitor
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: monitor
rules:
- apiGroups: 
  - ""
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: 
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs: 
  - get
- nonResourceURLs: 
  - /metrics
  verbs: 
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    name: prometheus
  name: prometheus
  namespace: monitor
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
```
```linux=
kubectl apply -f prometheus-rbac.yaml
```

### 2.創建Prometheus ConfigMap 
`prometheus-conf.yaml`

```yaml=
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitor
data:
  prometheus.yml: |-
    global:
      scrape_interval: 1m
      scrape_timeout: 1m

    rule_files:
    - /etc/prometheus/alert-rules-configmap.yaml

    alerting:
      alertmanagers:
        - static_configs:
          - targets: ["localhost:9093"]

    scrape_configs:

      - job_name: 'prometheus'
        scrape_interval: 5s
        static_configs:
          - targets: ['prometheus:9090']

      - job_name: 'node-exporter'
        static_configs:
        - targets: ['node-exporter:9100']
          labels:
            instance: node
      
      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics:8080']

      - job_name: kubernetes-pods
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: "(.+)"
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+) 
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
        - source_labels: [__meta_kubernetes_pod_node_name]
          action: replace
          target_label: kubernetes_node_name

      - job_name: kubernetes-cadvisor
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        metric_relabel_configs:
        - source_labels: [instance]
          separator: ;
          regex: (.+)
          target_label: node
          replacement: $1
          action: replace
```
```linux=
kubectl apply -f prometheus-conf.yaml
```

### 3.創建Prometheus 的 Deployment、Service
`prometheus.yaml`
```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitor
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: lily-pool
      serviceAccountName: prometheus
      containers:
        - name: prometheus
          image: prom/prometheus:v2.24.0
          imagePullPolicy: IfNotPresent
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf 
        - name: prometheus-storage-volume
          emptyDir: {}         
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitor
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector: 
    app: prometheus
  type: LoadBalancer
  ports:
    - port: 9090
      targetPort: 9090
```
```linux=
kubectl apply -f prometheus.yaml
```

### 4.檢查透過 EXTERNAL-IP:9090 可查看Prometheus Dashboard
![](https://i.imgur.com/26RhYf7.png)



---
## 加入 Kube State metrics
Kube狀態指標-用以獲取有關如Deployment、pod、daemonsets、Statefulsets 等的資源指標。

### 1.建立ClustorRole + ServiceAccount `metrics-rbac.yaml`
```yaml=
apiVersion: v1
automountServiceAccountToken: false
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.7.0
  name: kube-state-metrics
  namespace: lily
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.7.0
  name: kube-state-metrics
  namespace: lily
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: lily
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.7.0
  name: kube-state-metrics
  namespace: lily
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - serviceaccounts
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs:
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - list
  - watch
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - list
  - watch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - list
  - watch
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - mutatingwebhookconfigurations
  - validatingwebhookconfigurations
  verbs:
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  - ingressclasses
  - ingresses
  verbs:
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - list
  - watch
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - clusterroles
  - rolebindings
  - roles
  verbs:
  - list
  - watch
```
```linux=
kubectl apply -f metrics-rbac.yaml
```
### 2.創建Metrics State metrics 的 Deployment、Service 
`metrics.yaml`
```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.7.0
  name: kube-state-metrics
  namespace: monitor
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/version: 2.7.0
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: lily-pool
      serviceAccountName: kube-state-metrics
      automountServiceAccountToken: true
      containers:
      - image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.7.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        name: kube-state-metrics
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsUser: 65534
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.7.0
  name: kube-state-metrics
  namespace: monitor
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
```
```linux=
kubectl apply -f metrics.yaml
```
### 3.檢查 prometheus-conf.yaml 中有加入這段
```
- job_name: 'kube-state-metrics'
  static_configs:
    - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']
```
### 4.從Prometheus UI確認狀態為up
![](https://i.imgur.com/LfEXhVG.png)


---
## 加入 Node Exporter
節點導出器-收集node機器本身相關資訊，包含cpu、記憶體、硬碟用量等，並透過metrics來獲取這些資訊

### 1.建立 Daemonset、Service 在每個節點中運行 Node Exporter
`node-exporter.yaml`
```yaml=
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
  name: node-exporter
  namespace: monitor
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: node-exporter
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: node-exporter
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: lily-pool
      containers:
      - args:
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --no-collector.wifi
        - --no-collector.hwmon
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/)
        - --collector.netclass.ignored-devices=^(veth.*)$
        name: node-exporter
        image: prom/node-exporter
        ports:
          - containerPort: 9100
            protocol: TCP
        resources:
          limits:
            cpu: 250m
            memory: 180Mi
          requests:
            cpu: 102m
            memory: 180Mi
        volumeMounts:
        - mountPath: /host/sys
          mountPropagation: HostToContainer
          name: sys
          readOnly: true
        - mountPath: /host/root
          mountPropagation: HostToContainer
          name: root
          readOnly: true
      volumes:
      - hostPath:
          path: /sys
        name: sys
      - hostPath:
          path: /
        name: root
---
kind: Service
apiVersion: v1
metadata:
  name: node-exporter
  namespace: monitor
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9100'
spec:
  selector:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: node-exporter
  ports:
  - name: node-exporter
    protocol: TCP
    port: 9100
    targetPort: 9100
```
```linux=
kubectl appy -f node-exporter.yaml
```

### 2.檢查node-exporder服務的endpoint是否指向所有Node
```linux=
kubectl get endpoints -n monitor
```
![](https://i.imgur.com/tqJOgei.png)

### 3.檢查 prometheus-conf.yaml 中有加入這段
```
- job_name: 'node-exporter'
  static_configs:
  - targets: ['node-exporter:9100']
    labels:
      instance: node
```
### 4.從Prometheus UI確認狀態為up
![](https://i.imgur.com/4P2GHkX.png)

---
## 設置 Grafana

### 1.建立 Deployment、PVC、Service、datasource-configmap
`grafana.yaml`
```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      name: grafana
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
        # 更改UI路徑，預設為http://localhost:3000
        - name: GF_SERVER_ROOT_URL
          value: http://34.80.42.154:3000
        ports:
        - name: grafana
          containerPort: 3000
        volumeMounts:
          - name: grafana-pvc
            mountPath: /var/lib/grafana
          - name: grafana-datasources
            mountPath: /etc/grafana/provisioning/datasources
            readOnly: false
      securityContext:
        runAsUser: 0
      volumes:
        - name: grafana-pvc
          persistentVolumeClaim:
            claimName: grafana-pvc
        - name: grafana-datasources
          configMap:
              defaultMode: 420
              name: grafana-datasources
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: lily
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
  namespace: monitor
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port: '3000'
spec:
  selector: 
    app: grafana
  type: LoadBalancer 
  ports:
    - port: 3000
      targetPort: 3000
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitor
data:
  prometheus.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
                "access":"proxy",
                "editable": true,
                "name": "prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://prometheus:9090",
                "version": 1
            }
        ]
    }
```
```linux=
kubectl appy -f grafana.yaml
```


---
## 設置 Alertmanager


---
## 加入k8s集群外部監控

### LinuxVM

#### 1.VM安裝prometheus-node-exporter

```linux=
---安裝---
apt update
apt install prometheus-node-exporter
---檢查---
service prometheus-node-exporter status
curl http://localhost:9100/metrics | grep "node_"
```

#### 2.prometheus-conf增加job
基於文件的動態服務發現 `file_sd_config`
```yaml=
- job_name: 'linuxvm'
        scrape_interval: 10s
        file_sd_configs:
        - files:
          - /etc/prometheus/externalvm.yml
```

#### 3.新增configmap
targets可同時列出需要監控的多部主機
`prometheus-externalvm.yaml`
```yaml=
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-externalvm
  labels:
    name: prometheus-externalvm
  namespace: monitor
data:
  externalvm.yml: |-
    - targets:
      - "ip:9100"
      labels:
        instance: node
```

#### 4.重新apply prometheus-externalvm.yaml 、prometheus.yaml
從Prometheus UI確認狀態為up
![image](https://hackmd.io/_uploads/HysgnKtYyg.png)



---
參考:
https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/
http://www.mydlq.club/article/113/