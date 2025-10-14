# 二、链路追踪系统部署文档

## 组件概述

本项目使用Jaeger作为链路追踪系统，结合Elasticsearch作为存储后端，实现对分布式系统中服务调用链路的跟踪、监控和分析。链路追踪系统能够帮助开发人员快速定位和解决微服务架构中的性能瓶颈和故障。

### Jaeger

Jaeger是一个开源的分布式追踪系统，由Uber开发并捐赠给CNCF（云原生计算基金会）。它提供了分布式上下文传播、分布式事务监控、根本原因分析、服务依赖分析和性能/延迟优化等功能。

### Elasticsearch

Elasticsearch作为Jaeger的存储后端，用于存储和检索追踪数据。它提供了强大的搜索和分析能力，使得用户可以快速查询和分析大量的追踪数据。

## 部署前提

在部署链路追踪系统之前，请确保满足以下前提条件：

1. 已完成Kubernetes集群的搭建
2. 已完成Elasticsearch集群的部署（可参考日志系统部署文档）
3. 已准备好足够的磁盘空间用于存储追踪数据

## 2.1、部署jaeger

### 1.1 配置文件准备

创建jaeger.yaml配置文件。附件`jaeger.yaml`文件内容

### 1.2 部署命令

执行以下命令部署Jaeger：

```bash
# 创建tracing命名空间
kubectl create namespace tracing
# 部署Jaeger
kubectl apply -f jaeger.yaml
# 查看部署状态
kubectl get pods -n tracing -l app=jaeger
# 查看服务状态
kubectl get svc -n tracing -l app=jaeger
```

### 1.3 验证部署

部署完成后，可以通过以下方式验证Jaeger是否正常运行：

```bash
# 查看Jaeger日志
kubectl logs -f -n tracing $(kubectl get pods -n tracing -l app=jaeger -o jsonpath='{.items[0].metadata.name}')
# 检查Jaeger UI是否可以访问
curl http://localhost:30202
```

## 2.2、客户端集成

### 2.2.1 Java应用集成

对于Java应用，可以使用OpenTracing API或Spring Cloud Sleuth来集成Jaeger。

#### 使用Spring Cloud Sleuth

1. 在pom.xml中添加依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-otel-autoconfigure</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-otel-exporter</artifactId>
</dependency>
```

2. 在application.yml中添加配置：

```yaml
spring:
  application:
    name: your-service-name
  sleuth:
    otel:
      config:
        trace-id-ratio-based: 1.0
      exporter:
        otlp:
          endpoint: http://<jaeger-host>:14250
```

### 2.2.2 Go应用集成

对于Go应用，可以使用Jaeger客户端库来集成。

1. 安装依赖：

```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/exporters/jaeger
go get go.opentelemetry.io/otel/sdk
```

2. 示例代码：

```go
package main

import (
    "context"
    "log"
    "time"
    
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/jaeger"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
)

func initTracer() (*sdktrace.TracerProvider, error) {
    // 创建Jaeger exporter
    exporter, err := jaeger.New(
        jaeger.WithCollectorEndpoint(jaeger.WithEndpoint("http://<jaeger-host>:14268/api/traces")),
    )
    if err != nil {
        return nil, err
    }

    // 创建TracerProvider
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String("your-service-name"),
        )),
    )

    // 设置全局TracerProvider
    otel.SetTracerProvider(tp)
    // 设置全局TextMapPropagator
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return tp, nil
}

func main() {
    // 初始化tracer
    tp, err := initTracer()
    if err != nil {
        log.Fatal(err)
    }
    defer func() {
        if err := tp.Shutdown(context.Background()); err != nil {
            log.Printf("Error shutting down tracer provider: %v", err)
        }
    }()

    // 开始使用tracer
    ctx := context.Background()
    tracer := otel.Tracer("your-service-name")
    
    // 创建span
    ctx, span := tracer.Start(ctx, "main-operation")
    defer span.End()
    
    // 业务逻辑...
    time.Sleep(time.Second)
    
    // 创建子span
    ctx, childSpan := tracer.Start(ctx, "child-operation")
    defer childSpan.End()
    
    // 更多业务逻辑...
    time.Sleep(time.Millisecond * 500)
}
```

## 2.3、访问指南

### Jaeger UI访问

- 访问地址：http://localhost:30202
- 无需用户名和密码（可根据实际需求配置认证）

### API接口

- 收集器HTTP接口：http://localhost:30204/api/traces
- 收集器gRPC接口：http://localhost:30203
- Zipkin兼容接口：http://localhost:30205/api/v2/spans

## 2.4、使用指南

### Jaeger UI使用

1. **服务列表**：在Jaeger UI首页，可以看到所有集成了Jaeger的服务列表。
2. **追踪查询**：
   - 选择服务名称
   - 选择操作名称（可选）
   - 设置时间范围
   - 设置最大追踪数量
   - 点击"Find Traces"按钮查询追踪数据
3. **追踪详情**：点击具体的追踪记录，可以查看追踪的详细信息，包括：
   - 追踪的总时长
   - 各个span的调用关系和时长
   - 每个span的标签和日志信息
4. **依赖图**：点击"Dependencies"标签页，可以查看服务之间的依赖关系图。

### 高级查询

Jaeger UI支持使用Jaeger Query Language (JQL)进行高级查询：

- 按标签查询：`tags.service=my-service AND tags.http.status_code=200`
- 按操作查询：`operation=GET /api/v1/users`
- 按持续时间查询：`duration>1s`

## 2.5、维护监控

### 日常维护

1. 定期清理过期的追踪数据
2. 监控Jaeger的性能和资源使用情况
3. 根据业务需求调整采样策略
4. 监控Elasticsearch中追踪数据的存储情况

### 性能优化

1. 调整Jaeger的采样率，根据业务需求和系统负载进行配置
2. 增加Jaeger的资源配置（CPU、内存）
3. 优化Elasticsearch的配置，提高查询性能
4. 使用索引生命周期管理，定期删除过期的追踪数据

### 采样策略配置

Jaeger支持多种采样策略，可以在配置文件中进行配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-sampling
  namespace: tracing
data:
  sampling.json: |- {
    "service_strategies": [
      {"service": "my-service", "type": "probabilistic", "param": 0.1},
      {"service": "another-service", "type": "ratelimiting", "param": 10}
    ],
    "default_strategy": {"type": "probabilistic", "param": 0.01}
  }
```

然后在Jaeger部署中挂载这个ConfigMap：

```yaml
volumeMounts:
- name: sampling-config
  mountPath: /etc/jaeger
volumes:
- name: sampling-config
  configMap:
    name: jaeger-sampling
```

## 2.6、常见问题排查

### Jaeger UI无数据显示

1. 检查Jaeger服务是否正常运行
2. 检查客户端是否正确集成了Jaeger
3. 检查采样率设置是否过低
4. 检查Elasticsearch连接是否正常
5. 查看Jaeger日志，查找具体错误信息

### 追踪数据不完整

1. 检查客户端集成是否正确，确保所有服务都集成了Jaeger
2. 检查服务之间的调用是否正确传递了追踪上下文
3. 检查是否有网络问题导致追踪数据丢失

### 查询性能问题

1. 优化查询条件，避免过于宽泛的查询
2. 增加Elasticsearch的资源配置
3. 调整索引策略，合理设置分片和副本数量
4. 定期清理过期的追踪数据

### Elasticsearch存储空间不足

1. 增加Elasticsearch的存储容量
2. 调整索引生命周期策略，缩短数据保留时间
3. 增加采样率，减少追踪数据的生成量

## 2.7、附录

### 2.7.1 环境变量配置参考

Jaeger支持以下环境变量配置：

| 环境变量 | 说明 | 默认值 |
|---------|------|-------|
| SPAN_STORAGE_TYPE | 存储类型（elasticsearch, memory等） | memory |
| ES_SERVER_URLS | Elasticsearch服务器地址 | http://localhost:9200 |
| ES_USERNAME | Elasticsearch用户名 | - |
| ES_PASSWORD | Elasticsearch密码 | - |
| ES_INDEX_PREFIX | Elasticsearch索引前缀 | jaeger-span |
| ES_TAGS_AS_FIELDS_ALL | 是否将所有标签作为字段存储 | false |
| COLLECTOR_ZIPKIN_HOST_PORT | Zipkin兼容接口端口 | - |
| LOG_LEVEL | 日志级别 | info |

### 2.7.2 性能监控指标

Jaeger提供了以下Prometheus指标：

1. **jaeger_collector_spans_received_total**：收集器接收到的span总数
2. **jaeger_collector_spans_dropped_total**：收集器丢弃的span总数
3. **jaeger_query_traces_requests_total**：查询请求总数
4. **jaeger_query_search_latency**：查询延迟
5. **jaeger_ui_traces_requests_total**：UI请求的追踪总数

这些指标可以被Prometheus采集，并在Grafana中展示。
