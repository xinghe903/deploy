# 监控系统部署文档

## 项目简介

本项目提供了一套完整的监控系统部署方案，包括Kubernetes集群搭建、日志收集与展示、链路追踪以及指标监控等功能组件的部署指南。通过本项目，您可以快速构建一个全面的监控体系，实现对分布式系统的有效监控和管理。

## 主要功能

### 部署功能
- Kubernetes集群部署（使用kubeadm）
- 日志组件部署：Elasticsearch、Kibana、Filebeat
- 链路追踪组件部署：Jaeger、Elasticsearch
- 指标监控组件部署：Prometheus、Grafana、各类Exporter

### 展示功能
- 日志展示：通过Kibana展示日志
- 追踪展示：通过Jaeger展示链路追踪
- 指标展示：通过Grafana展示各类监控指标

## 项目结构

```
├── README.md                 # 项目主文档
├── kubeadm搭建k8s集群.md     # Kubernetes集群部署文档
├── 日志/
│   ├── README.md             # 日志组件部署文档
│   ├── filebeat.yaml         # Filebeat配置文件
│   └── kibana.yaml           # Kibana配置文件
├── 链路追踪/
│   ├── README.md             # 链路追踪组件部署文档
│   └── jaeger.yaml           # Jaeger配置文件
├── 指标监控/
│   ├── README.md             # 指标监控组件部署文档
│   ├── exporter/             # 各类Exporter配置文件
│   │   ├── elasticsearch-exporter.yaml
│   │   ├── node-exporter.yaml
│   │   └── redis-exporter.yaml
│   ├── grafana.yaml          # Grafana配置文件
│   └── prometheus.yaml       # Prometheus配置文件
├── kubeadm软件包/             # Kubernetes相关软件包
├── pic/                      # 文档中使用的图片
└── 证书/                      # 证书文件
```

## 部署指南

### 1. 部署顺序

建议按照以下顺序进行部署，以确保各组件之间的依赖关系正确：

1. Kubernetes集群搭建
2. 日志组件部署
3. 链路追踪组件部署
4. 指标监控组件部署

### 2. Kubernetes集群搭建

请参考<mcfile name="k8s集群搭建文档" path="d:\data\code\go\project\deploy\kubeadm搭建k8s集群.md"></mcfile>，完成Kubernetes集群的搭建。

### 3. 日志组件部署

请参考<mcfile name="日志组件部署文档" path="d:\data\code\go\project\deploy\日志\README.md"></mcfile>，完成Elasticsearch、Kibana和Filebeat的部署。

### 4. 链路追踪组件部署

请参考<mcfile name="链路追踪组件部署文档" path="d:\data\code\go\project\deploy\链路追踪\README.md"></mcfile>，完成Jaeger的部署。

### 5. 指标监控组件部署

请参考<mcfile name="指标监控组件部署文档" path="d:\data\code\go\project\deploy\指标监控\README.md"></mcfile>，完成Prometheus、Grafana和各类Exporter的部署。

## 访问指南

部署完成后，可以通过以下方式访问各组件：

### 集群信息
- 登录token生成：`kubectl -n kubernetes-dashboard create token admin-user`


## 维护指南

### 1. 日志系统维护
- 定期检查Elasticsearch的存储空间使用情况
- 根据日志量调整索引生命周期策略
- 监控Filebeat的收集状态和性能

### 2. 链路追踪系统维护
- 定期清理过期的追踪数据
- 监控Jaeger的性能和资源使用情况
- 根据业务需求调整采样策略

### 3. 指标监控系统维护
- 定期更新Grafana的仪表盘
- 优化Prometheus的查询性能
- 监控Exporter的状态和指标采集情况
