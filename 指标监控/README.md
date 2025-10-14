# 监控

# 三、指标监控系统部署文档

## 组件概述

本项目使用Prometheus作为监控系统的核心，结合Grafana进行可视化展示，并通过各种Exporter（如Node Exporter、Elasticsearch Exporter等）采集不同类型的数据。指标监控系统能够实时监控Kubernetes集群和各种服务的运行状态，及时发现并告警潜在的问题。

### Prometheus

Prometheus是一个开源的监控和告警系统，由SoundCloud开发并捐赠给CNCF。它具有多维数据模型、强大的查询语言（PromQL）、灵活的告警规则和多种可视化选项等特点。

### Grafana

Grafana是一个开源的可视化平台，支持多种数据源（包括Prometheus），可以创建丰富的仪表盘来展示监控数据。它提供了直观的用户界面和丰富的可视化选项。

### Node Exporter

Node Exporter是一个用于收集主机系统指标的Exporter，可以采集CPU、内存、磁盘、网络等系统级指标。

### Elasticsearch Exporter

Elasticsearch Exporter用于采集Elasticsearch集群的运行指标，如节点状态、索引统计、搜索性能等。

## 部署前提

在部署指标监控系统之前，请确保满足以下前提条件：

1. 已完成Kubernetes集群的搭建
2. 已完成Elasticsearch集群的部署（如需要监控Elasticsearch）
3. 已完成Redis部署（如需要监控Redis）

## 3.1、部署Prometheus

### 3.1.1 配置文件准备

创建prometheus.yaml配置文件，包含以下内容：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
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
    - job_name: 'kubernetes-services'
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
    - job_name: 'node-exporter'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__meta_kubernetes_node_name]
        target_label: instance
      - target_label: __address__
        replacement: $1:9100
      - regex: (__meta_kubernetes_node_label_(.+))
        replacement: $1
        target_label: $2
      metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'go_.*|process_.*'
        action: drop
    - job_name: 'elasticsearch-exporter'
      static_configs:
      - targets: ['elasticsearch-exporter.monitoring.svc:9114']
    - job_name: 'redis-exporter'
      static_configs:
      - targets: ['redis-exporter.monitoring.svc:9121']
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
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
      containers:
      - name: prometheus
        image: prom/prometheus:v2.47.0
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: storage-volume
          mountPath: /prometheus
        resources:
          limits:
            memory: "2Gi"
            cpu: "1000m"
          requests:
            memory: "1Gi"
            cpu: "500m"
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: storage-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30206
  type: NodePort
```

### 3.1.2 创建prometheus账号

```shell
kubectl create serviceaccount prometheus -n monitoring
kubectl create clusterrolebinding prometheus-clusterrolebinding -n monitoring --clusterrole=cluster-admin --serviceaccount=monitoring:prometheus
```

### 3.1.3 部署命令

执行以下命令部署Prometheus：

```bash
# 创建monitoring命名空间
kubectl create namespace monitoring
# 部署Prometheus
kubectl apply -f prometheus.yaml
# 查看部署状态
kubectl get pods -n monitoring -l app=prometheus
# 查看服务状态
kubectl get svc -n monitoring -l app=prometheus
```

### 3.1.4 验证部署

部署完成后，可以通过以下方式验证Prometheus是否正常运行：

```bash
# 查看Prometheus日志
kubectl logs -f -n monitoring $(kubectl get pods -n monitoring -l app=prometheus -o jsonpath='{.items[0].metadata.name}')
# 检查Prometheus UI是否可以访问
curl http://localhost:30206
```

## 3.2、部署Grafana

### 3.2.1 配置文件准备

创建grafana.yaml配置文件，包含以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:9.5.10
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin@2025
        - name: GF_INSTALL_PLUGINS
          value: grafana-clock-panel,grafana-simple-json-datasource
        resources:
          limits:
            memory: "1Gi"
            cpu: "500m"
          requests:
            memory: "512Mi"
            cpu: "250m"
      volumes:
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30207
  type: NodePort
```

### 3.2.2 部署命令

执行以下命令部署Grafana：

```bash
# 部署Grafana
kubectl apply -f grafana.yaml
# 查看部署状态
kubectl get pods -n monitoring -l app=grafana
# 查看服务状态
kubectl get svc -n monitoring -l app=grafana
```

### 3.2.3 验证部署

部署完成后，可以通过以下方式验证Grafana是否正常运行：

```bash
# 查看Grafana日志
kubectl logs -f -n monitoring $(kubectl get pods -n monitoring -l app=grafana -o jsonpath='{.items[0].metadata.name}')
# 检查Grafana UI是否可以访问
curl http://localhost:30207
```

## 3.3、部署Node Exporter

### 3.3.1 配置文件准备

创建exporter/node-exporter.yaml配置文件，包含以下内容：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.0
        ports:
        - containerPort: 9100
          hostPort: 9100
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m"
          requests:
            memory: "64Mi"
            cpu: "50m"
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/rootfs
        - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+)($|/)
        - --collector.filesystem.fs-types-exclude=^(tmpfs|devtmpfs|overlay|aufs|squashfs)$
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: rootfs
        hostPath:
          path: /
```

### 3.3.2 部署命令

执行以下命令部署Node Exporter：

```bash
# 部署Node Exporter
kubectl apply -f exporter/node-exporter.yaml
# 查看部署状态
kubectl get pods -n monitoring -l app=node-exporter
# 查看DaemonSet状态
kubectl get daemonset -n monitoring -l app=node-exporter
```

### 3.3.3 验证部署

部署完成后，可以通过以下方式验证Node Exporter是否正常运行：

```bash
# 在任一节点上检查Node Exporter端口
curl http://localhost:9100/metrics
```

## 3.4、部署Elasticsearch Exporter

### 3.4.1 配置文件准备

创建exporter/elasticsearch-exporter.yaml配置文件，包含以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch-exporter
  template:
    metadata:
      labels:
        app: elasticsearch-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9114"
    spec:
      containers:
      - name: elasticsearch-exporter
        image: justwatch/elasticsearch_exporter:1.5.0
        ports:
        - containerPort: 9114
        env:
        - name: ES_URI
          value: http://elasticsearch-service:9200
        - name: ES_USERNAME
          value: username
        - name: ES_PASSWORD
          value: password
        resources:
          limits:
            memory: "256Mi"
            cpu: "100m"
          requests:
            memory: "128Mi"
            cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-exporter
  namespace: monitoring
spec:
  selector:
    app: elasticsearch-exporter
  ports:
  - port: 9114
    targetPort: 9114
```

### 3.4.2 部署命令

执行以下命令部署Elasticsearch Exporter：

```bash
# 部署Elasticsearch Exporter
kubectl apply -f exporter/elasticsearch-exporter.yaml
# 查看部署状态
kubectl get pods -n monitoring -l app=elasticsearch-exporter
# 查看服务状态
kubectl get svc -n monitoring -l app=elasticsearch-exporter
```

### 3.4.3 验证部署

部署完成后，可以通过以下方式验证Elasticsearch Exporter是否正常运行：

```bash
# 查看Elasticsearch Exporter日志
kubectl logs -f -n monitoring $(kubectl get pods -n monitoring -l app=elasticsearch-exporter -o jsonpath='{.items[0].metadata.name}')
# 检查Elasticsearch Exporter端口
kubectl port-forward svc/elasticsearch-exporter -n monitoring 9114:9114
curl http://localhost:9114/metrics
```

## 3.5、部署Redis Exporter

### 3.5.1 配置文件准备

创建exporter/redis-exporter.yaml配置文件，包含以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
    spec:
      containers:
      - name: redis-exporter
        image: oliver006/redis_exporter:v1.51.0
        ports:
        - containerPort: 9121
        env:
        - name: REDIS_ADDR
          value: redis:6379
        - name: REDIS_PASSWORD
          value: redis@2025
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m"
          requests:
            memory: "64Mi"
            cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: redis-exporter
  namespace: monitoring
spec:
  selector:
    app: redis-exporter
  ports:
  - port: 9121
    targetPort: 9121
```

### 3.5.2 部署命令

执行以下命令部署Redis Exporter：

```bash
# 部署Redis Exporter
kubectl apply -f exporter/redis-exporter.yaml
# 查看部署状态
kubectl get pods -n monitoring -l app=redis-exporter
# 查看服务状态
kubectl get svc -n monitoring -l app=redis-exporter
```

### 3.5.3 验证部署

部署完成后，可以通过以下方式验证Redis Exporter是否正常运行：

```bash
# 查看Redis Exporter日志
kubectl logs -f -n monitoring $(kubectl get pods -n monitoring -l app=redis-exporter -o jsonpath='{.items[0].metadata.name}')
# 检查Redis Exporter端口
kubectl port-forward svc/redis-exporter -n monitoring 9121:9121
curl http://localhost:9121/metrics
```

## 3.6、Grafana配置

### 3.6.1 添加Prometheus数据源

1. 访问Grafana UI：http://localhost:30207
2. 登录Grafana，默认用户名和密码：admin/admin@2025
3. 点击左侧菜单栏的"Configuration" -> "Data Sources"
4. 点击"Add data source"按钮
5. 选择"Prometheus"
6. 在"HTTP"部分，设置URL为：http://prometheus.monitoring.svc:9090
7. 点击"Save & Test"按钮，确认数据源连接成功

### 3.6.2 导入常用仪表盘

Grafana提供了许多社区贡献的仪表盘模板，可以直接导入使用。

#### 导入Kubernetes集群仪表盘

1. 点击左侧菜单栏的"Create" -> "Import"
2. 在"Import via grafana.com"输入框中输入：3119（Kubernetes集群监控仪表盘ID）
3. 点击"Load"按钮
4. 选择刚刚添加的Prometheus数据源
5. 点击"Import"按钮

#### 导入Node Exporter仪表盘

1. 点击左侧菜单栏的"Create" -> "Import"
2. 在"Import via grafana.com"输入框中输入：1860（Node Exporter Full仪表盘ID）
3. 点击"Load"按钮
4. 选择刚刚添加的Prometheus数据源
5. 点击"Import"按钮

#### 导入Elasticsearch仪表盘

1. 点击左侧菜单栏的"Create" -> "Import"
2. 在"Import via grafana.com"输入框中输入：2322（Elasticsearch Metrics仪表盘ID）
3. 点击"Load"按钮
4. 选择刚刚添加的Prometheus数据源
5. 点击"Import"按钮

#### 导入Redis仪表盘

1. 点击左侧菜单栏的"Create" -> "Import"
2. 在"Import via grafana.com"输入框中输入：763（Redis Dashboard仪表盘ID）
3. 点击"Load"按钮
4. 选择刚刚添加的Prometheus数据源
5. 点击"Import"按钮

## 3.7、访问指南

### Prometheus UI访问

- 访问地址：http://localhost:30206
- 无需用户名和密码（可根据实际需求配置认证）

### Grafana UI访问

- 访问地址：http://localhost:30207
- 默认用户名：admin
- 默认密码：admin@2025

### API接口

- Prometheus API：http://localhost:30206/api/v1/
- Grafana API：http://localhost:30207/api/

## 3.8、使用指南

### Prometheus使用

#### 查询语言（PromQL）

Prometheus提供了强大的查询语言PromQL，可以用来查询和聚合时间序列数据。

**常用查询示例：**

1. 查询CPU使用率：
   ```
   100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
   ```

2. 查询内存使用率：
   ```
   (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
   ```

3. 查询磁盘使用率：
   ```
   100 - (node_filesystem_avail_bytes{fstype!="tmpfs",fstype!="rootfs"} / node_filesystem_size_bytes{fstype!="tmpfs",fstype!="rootfs"}) * 100
   ```

4. 查询Kubernetes Pod数量：
   ```
   kube_pod_info{namespace="default"}
   ```

#### 告警规则

Prometheus支持配置告警规则，当满足特定条件时触发告警。

可以在Prometheus配置文件中添加告警规则：

```yaml
rule_files:
  - "alerts.yml"
```

然后创建alerts.yml文件，包含告警规则：

```yaml
groups:
- name: node.rules
  rules:
  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      description: "CPU usage is above 80% (current value: {{ $value }}%)"
  - alert: HighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      description: "Memory usage is above 85% (current value: {{ $value }}%)"
```

### Grafana使用

#### 创建自定义仪表盘

1. 点击左侧菜单栏的"Create" -> "Dashboard"
2. 点击"Add new panel"
3. 在"Query"选项卡中，选择Prometheus数据源，然后输入PromQL查询语句
4. 在"Visualization"选项卡中，选择合适的图表类型
5. 在"General"选项卡中，设置面板标题和描述
6. 点击"Apply"按钮，保存面板
7. 点击左上角的"Save Dashboard"按钮，保存仪表盘

#### 仪表盘共享

1. 打开要共享的仪表盘
2. 点击右上角的"Share"按钮
3. 在"Link"选项卡中，可以获取直接链接或嵌入代码
4. 在"Snapshot"选项卡中，可以创建一个静态的快照，便于分享给其他人

## 3.9、维护监控

### 日常维护

1. 定期备份Grafana仪表盘和Prometheus配置
2. 监控Prometheus和Grafana的性能和资源使用情况
3. 根据业务需求调整告警规则和阈值
4. 定期更新监控组件到最新版本

### 性能优化

1. 调整Prometheus的抓取间隔和保留策略
   ```yaml
global:
  scrape_interval: 30s  # 增加抓取间隔
  evaluation_interval: 30s
  scrape_timeout: 10s
  external_labels:
    monitor: 'kubernetes-monitor'

# 在Prometheus启动参数中设置保留策略
# --storage.tsdb.retention.time=7d  # 保留7天数据
```

2. 增加Prometheus和Grafana的资源配置
3. 使用Prometheus联邦集群，分散监控负载
4. 定期清理过期的数据

## 3.10、常见问题排查

### Prometheus无法抓取目标数据

1. 检查目标服务是否正常运行
2. 检查Prometheus配置是否正确
3. 检查网络连接是否通畅
4. 查看Prometheus日志，查找具体错误信息

### Grafana无法连接Prometheus数据源

1. 检查Prometheus服务是否正常运行
2. 检查Grafana中Prometheus数据源的配置是否正确
3. 检查网络连接是否通畅
4. 查看Grafana日志，查找具体错误信息

### 监控数据不完整或有延迟

1. 检查抓取间隔是否合适
2. 检查目标服务是否有性能问题
3. 检查网络带宽是否足够
4. 增加Prometheus的资源配置

### 告警没有触发

1. 检查告警规则配置是否正确
2. 检查告警条件是否满足
3. 检查Prometheus是否正常运行
4. 查看Prometheus日志，查找具体错误信息

## 3.11、附录

### 3.11.1 Prometheus配置参考

| 配置项 | 说明 | 默认值 |
|-------|------|-------|
| scrape_interval | 抓取间隔 | 15s |
| evaluation_interval | 规则评估间隔 | 15s |
| scrape_timeout | 抓取超时时间 | 10s |
| storage.tsdb.retention.time | 数据保留时间 | 15d |
| storage.tsdb.retention.size | 数据保留大小 | - |

### 3.11.2 Grafana配置参考

| 环境变量 | 说明 | 默认值 |
|---------|------|-------|
| GF_SECURITY_ADMIN_USER | 管理员用户名 | admin |
| GF_SECURITY_ADMIN_PASSWORD | 管理员密码 | admin |
| GF_INSTALL_PLUGINS | 安装的插件 | - |
| GF_SERVER_HTTP_PORT | HTTP端口 | 3000 |
| GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH | 默认首页仪表盘路径 | - |

### 3.11.3 常用Prometheus指标

1. **node_cpu_seconds_total**：CPU时间统计
2. **node_memory_MemTotal_bytes**：总内存
3. **node_memory_MemAvailable_bytes**：可用内存
4. **node_filesystem_size_bytes**：文件系统大小
5. **node_filesystem_avail_bytes**：文件系统可用空间
6. **node_network_receive_bytes_total**：网络接收字节数
7. **node_network_transmit_bytes_total**：网络发送字节数
8. **container_cpu_usage_seconds_total**：容器CPU使用时间
9. **container_memory_usage_bytes**：容器内存使用量
10. **kube_pod_info**：Pod信息
11. **elasticsearch_cluster_health_status**：Elasticsearch集群健康状态
12. **redis_connected_clients**：Redis连接客户端数
