# Kubernetes Course Final Project 操作手冊

# 實驗環境

- 2 Node Cluster ([Setup with Kubespray](https://github.com/kubernetes-sigs/kubespray))

### 安裝流程

- 準備兩台 VM
- Disable Swap on both machines
    - `sudo swapoff -a`
    - 到 /etc/fstab 註解或刪掉 swap 的那行
- Enable root login
- Enable password free login
    
    ```
    ssh-copy-id root@<node_ip>
    
    ```
    
- Download Repo
    
    ```
    sudo apt -y install git
    git clone <https://github.com/kubernetes-sigs/kubespray.git>
    cd kubespray
    
    ```
    
- Download Requirements
    
    ```
    sudo apt -y install python3-pip
    python3 -m pip install -r requirements.txt
    
    ```
    
- 準備 hosts.yaml file
    - `cp -rfp inventory/sample inventory/mycluster`
    - `declare -a IPS=(<node1_ip> <node2_ip>)
    CONFIG_FILE=inventory/mycluster/hosts.yaml
    python3 contrib/inventory_builder/inventory.py ${IPS[@]}`
- 調整要安裝的參數(可略)
    
    ```
    cat inventory/mycluster/group_vars/all/all.yml
    cat inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
    
    ```
    
- 開始 Build Cluster
    
    ```
    ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
    
    ```
    

# Create Honeypot

- 創建資料夾

```bash
mkdir honeypot
```

- 創建 honeypot 所需的 yaml 檔案
    - vim mysql-honey-pot.yaml
    
    ```bash
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mysql-honey-pod
    spec:
      selector:
        matchLabels:
          app: mysql-honey-pod
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: mysql-honey-pod
        spec:
          nodeName: node1    # Select Node Here
          containers:
          - image: mysql:5.6
            name: mysql-honey-pod
            env:
            - name: MYSQL_ROOT_PASSWORD   # using secret to set password
              valueFrom: 
                secretKeyRef:
                  name: mysql-secret
                  key: password
                  optional: false
            ports:
            - containerPort: 3306
              name: mysql-honey-pod
            volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
          volumes:
          - name: mysql-persistent-storage
            persistentVolumeClaim:
              claimName: mysql-pv-claim
    ```
    
    - vim mysql-pv.yaml
    
    ```bash
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: mysql-pv-volume
      labels:
        type: local
    spec:
      storageClassName: manual
      capacity:
        storage: 5Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: "/mnt/data"
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mysql-pv-claim
    spec:
      storageClassName: manual
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
    ```
    

- vim mysql-secret.yaml

```bash
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: kubernetes.io/basic-auth
stringData:
  password: test1234
```

### 創建 real mysql

- vim mysql-real.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-real
spec:
  selector:
    matchLabels:
      app: mysql-real
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql-real
    spec:
      nodeName: node2                 # Select Your Node Here
      containers:
      - image: mysql:5.6
        name: mysql-real
        env:
        - name: MYSQL_ROOT_PASSWORD   # using secret to set password
          valueFrom: 
            secretKeyRef:
              name: mysql-secret
              key: password
              optional: false
        ports:
        - containerPort: 3306
          name: mysql-real
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

### 創建 Web

- vim next-web-demo.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: next-web
spec:
  selector:
    matchLabels:
      app: next-web
  replicas: 1
  template:
    metadata:
      labels:
        app: next-web
    spec:
      nodeName: node2
      containers:
      - name: next-web
        image: youmin1017/term-project-web:latest
        # imagePullPolicy: IfNotPresent
        env:
        - name: MYSQL_HOST
          value: "real-db-service"
        - name: MYSQL_PORT
          value: "3306"
        - name: MYSQL_DATABASE
          value: "realDB"
        - name: MYSQL_USER
          value: "root"
        - name: MYSQL_PASSWORD
          value: "test1234"
        ports:
        - containerPort: 3000
```

### 創建 Service

- vim services.yaml

```bash
apiVersion: v1
kind: Service
metadata:
  name: next-web-service
spec:
  type: NodePort 
  ports:
  - port: 3000 
    targetPort: 3000 
    nodePort: 30120 
  selector:
    app: next-web
---
apiVersion: v1
kind: Service
metadata:
  name: honey-pot-db-service
spec:
  type: NodePort 
  ports:
  - port: 3306 
    targetPort: 3306 
    nodePort: 30126 
  selector:
    app: mysql-honey-pod
  # clusterIP: None               # headless service
---
apiVersion: v1
kind: Service
metadata:
  name: real-db-service
spec:
  ports:
  - port: 3306
  selector:
    app: mysql-real
  clusterIP: None
```

- 開始 Create honeypot 的檔案

```bash
sudo kubectl create -f honeypot/.
```

### ****MySQL HoneyPotDB & RealDB****

- 設定`mysql`權限，允許任何IP登入Optional

```bash
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'test1234' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

- By HoneyService Connect to HoneyPot SQL

```bash
mysql -h 10.22.22.81 -P 30126 -u root -ptest1234
```

- Creating new pod to test realdb is accessible

```bash
sudo kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h 10.233.19.119:3306 -u root -ptest1234
```

# Install Prometheus

## Install P****rometheus Image****

```bash
docker pull prom/prometheus
```

## Create Namespace

```bash
k create namespace monitoring
```

## Create ****ClusterRole****

- 創建 ClusterRole，Prometheus 使用 k8s API 從 Node、Pod、Deployment 去 read 指標。所以下面是創建一個 RBAC policy，對 API 所需的類別進行 access
- vim clusterRole.yaml

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```

- kubectl create -f clusterRole.yaml

## ****Create a Config Map To Externalize Prometheus Configurations****

### 建立 Prometheus rules

- vim config-map.yaml

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: HoneypotDetect
      rules:
      - alert: Detect somebody ingress honeypot
        expr: mysql_global_status_threads_connected > 2
        for: 0m
        labels:
          severity: slack
        annotations:
          summary: Somebody enter mysql. (instance {{ $labels.instance }})
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'kubernetes-nodes'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name

      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']

      - job_name: kubernetes-services
        scrape_interval: 15s
        scrape_timeout: 10s
        kubernetes_sd_configs:
        - role: service
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: (.+)(?::\d+);(\d+)
          replacement: $1:$2

      - job_name: mysqld-exporter
        scrape_interval: 5s
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
            - default
        relabel_configs:
        - action: keep
          source_labels:
          - __meta_kubernetes_service_label_app_kubernetes_io_name
          regex: mysqld-exporter
        - action: keep
          source_labels:
          - __meta_kubernetes_endpoint_port_name
          regex: http
```

- kubectl create -f config-map.yaml

## ****Create a Prometheus Deployment****

### 部屬 Prometheus

- vim prometheus-deployment.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
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
```

- kubectl create -f prometheus-deployment.yaml

- kubectl get deployments --namespace=monitoring

## ****Exposing Prometheus as a Service****

- 為 Prometheus expose 為 service
- vim prometheus-service.yaml

```bash
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector: 
    app: prometheus-server
  type: NodePort  
  ports:
    - port: 8080
      targetPort: 9090 
      nodePort: 30000
```

- kubectl create -f prometheus-service.yaml --namespace=monitoring

- Connection http://10.22.22.82:30000

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled.png)

# Create MySQL-Exporter

### Create yaml file

- vim mysql-exporter

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysqld-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysqld-exporter
  template:
    metadata:
      labels:
        app: mysqld-exporter
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "9104"
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: mysqld-exporter
        image: prom/mysqld-exporter:v0.12.1
        args:
        - --collect.info_schema.tables
        - --collect.info_schema.innodb_tablespaces
        - --collect.info_schema.innodb_metrics
        - --collect.global_status
        - --collect.global_variables
        - --collect.slave_status
        - --collect.info_schema.processlist
        - --collect.perf_schema.tablelocks
        - --collect.perf_schema.eventsstatements
        - --collect.perf_schema.eventsstatementssum
        - --collect.perf_schema.eventswaits
        - --collect.auto_increment.columns
        - --collect.binlog_size
        - --collect.perf_schema.tableiowaits
        - --collect.perf_schema.indexiowaits
        - --collect.info_schema.userstats
        - --collect.info_schema.clientstats
        - --collect.info_schema.tablestats
        - --collect.info_schema.schemastats
        - --collect.perf_schema.file_events
        - --collect.perf_schema.file_instances
        - --collect.perf_schema.replication_group_member_stats
        - --collect.perf_schema.replication_applier_status_by_worker
        - --collect.slave_hosts
        - --collect.info_schema.innodb_cmp
        - --collect.info_schema.innodb_cmpmem
        - --collect.info_schema.query_response_time
        - --collect.engine_tokudb_status
        - --collect.engine_innodb_status
        ports:
        - containerPort: 9104
          protocol: TCP
        env:
        - name: DATA_SOURCE_NAME
          value: "exporter:YOUR-PASSWORD@(10.233.90.16:3306)/"
---
apiVersion: v1
kind: Service
metadata:
  name: mysqld-exporter
  labels:
    app: mysqld-exporter
spec:
  type: ClusterIP
  ports:
  - port: 9104
    protocol: TCP
    name: http
  selector:
    app: mysqld-exporter
```

### 檢查是否有檢測到 mysql-exporter

- 至 prometheus 首頁上方的 “Status” 點擊 “Targets”

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%201.png)

- Check 是否有看到 mysql-exporter

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%202.png)

# 如何創建 slack

 

- 至 slack 官網去創建帳號
    - [https://slack.com/get-started#/createnew](https://slack.com/get-started#/createnew)
    - 輸入 學校信箱

- 辦理完成後 slack 會將你導入另一個頁面，然後創建一個屬於自己的 Channel，Channel 名稱自行取名，創建完成後就會如下圖所示 (這邊我是創建 monitor 的 channel) :

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%203.png)

- 如何取得 API
    - 至上方的 Monitoring 下拉，選擇 "Seetings & administration”，旁邊再點擊 "Manage apps”
    
    ![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%204.png)
    

- 輸入 hoo 就可以看到選項，選擇 "Incoming WebHooks”

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%205.png)

- 將 Incoming WebHooks Add to Slack

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%206.png)

- 之後往下拉就可以看到 API，後續的 AlertManager 會需要使用到

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%207.png)

# Create AlertManager

### 設定 AlterManager configmap

- vim AlertManagerConfigmap.yaml

```bash
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  config.yml: |-
    global:
    route:
      receiver: slack_demo
      group_by: ['alertname', 'priority']

    receivers:
    - name: slack_demo
      slack_configs:
      - api_url: https://hooks.slack.com/services/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
        channel: '#monitor'
```

- kubectl create -f AlertManagerConfigmap.yaml

### 佈署 AlertManager

- vim Depolyment.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      name: alertmanager
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:latest
        args:
          - "--config.file=/etc/alertmanager/config.yml"
          - "--storage.path=/alertmanager"
        ports:
        - name: alertmanager
          containerPort: 9093
        resources:
            requests:
              cpu: 500m
              memory: 500M
            limits:
              cpu: 1
              memory: 1Gi
        volumeMounts:
        - name: config-volume
          mountPath: /etc/alertmanager
        - name: alertmanager
          mountPath: /alertmanager
      volumes:
      - name: config-volume
        configMap:
          name: alertmanager-config
      - name: alertmanager
        emptyDir: {}
```

- kubectk create -f Depolyment.yaml

### ****Exposing AlertManager as a Service****

- vim service.yaml

```bash
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9093'
spec:
  selector:
    app: alertmanager
  type: NodePort
  ports:
    - port: 9093
      targetPort: 9093
      nodePort: 31000
```

- kubectl create -f service.yaml

### 登入 AlertManager Web Page

![Untitled](Kubernetes%20Course%20Final%20Project%20%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8A%208fa4a651849f48d0b150cee432b75807/Untitled%208.png)