# 日志系统部署文档

## EFK组件概述

本项目使用EFK（Elasticsearch + Filebeat + Kibana）技术栈来构建日志收集与展示系统，实现对分布式系统日志的统一收集、存储、查询和可视化。

### Elasticsearch

Elasticsearch是一个分布式的搜索和分析引擎，专为处理大规模数据而设计，提供了实时搜索、分析和存储能力。在日志系统中，Elasticsearch作为存储和检索引擎，负责存储收集到的日志数据并提供强大的查询能力。

### Kibana

Kibana是一个开源的分析和可视化平台，设计用于与Elasticsearch协作。通过Kibana，用户可以搜索、查看存储在Elasticsearch索引中的数据，并创建各种图表、表格和仪表盘来可视化数据。

### Filebeat

Filebeat是一个轻量级的日志收集器，专为转发和集中日志数据而设计。它占用资源少，可以安装在需要收集日志的服务器上，将日志数据发送到Elasticsearch或Logstash进行进一步处理。

## 部署前提

在部署EFK组件之前，请确保满足以下前提条件：

1. 已完成Kubernetes集群的搭建
2. 所有节点的操作系统已配置正确的内存映射限制和文件描述符限制
3. 已准备好足够的磁盘空间用于存储日志数据

## 1. Elasticsearch部署

### 1.1 准备措施

#### 1.1.1 配置最大内存映射区域数量

```shell
vi /etc/sysctl.conf
# 写入文件
vm.max_map_count=262144
# 重新加载配置
sysctl -p
```

#### 1.1.2 配置文件描述符限制

``` shell
vi /etc/security/limits.conf
# 配置内容，*表示所有用户生效
* soft nofile 65536
* hard nofile 65536
# 配置完成后需要重新登录才能生效
# 验证配置是否生效：ulimit -H -n
```

### 1.2 安装步骤

#### 1.2.1 上传软件包并解压

```shell
# 1. 上传压缩包并解压
tar -xvf elasticsearch-7.17.26-linux-x86_64.tar.gz

# 2. 创建启动用户
useradd es; passwd es

# 3. 工作目录移到普通用户下
cd /home/es/; mv ~/install/elasticsearch/elasticsearch-7.17.26 .

# 4. 修改目录归属用户
chown -R es:es elasticsearch-7.17.26
```

#### 1.2.2 修改配置文件

根据集群节点的角色，分别配置各节点的elasticsearch.yml文件：

##### 节点一（主节点）配置

``` shell
cat > config/elasticsearch.yml << EOF
# 集群名称
cluster.name: custom-cluster
# 节点名称
node.name: "node-173"
cluster.initial_master_nodes: ["node-173"]
# 定义为主节点
node.master: true
# 同时作为数据节点
node.data: true
# 访问的IP地址，0.0.0.0表示不限制
network.host: 0.0.0.0
network.publish_host: nodeip001
# HTTP访问端口号
http.port: 9020
# 集群通讯端口号
transport.tcp.port: 9030
# 所有节点的IP地址和通讯端口
discovery.zen.ping.unicast.hosts: ["nodeip001:9030", "nodeip004:9030", "nodeip003:9030"]
EOF
```

##### 节点二（数据节点）配置

``` shell
cat > config/elasticsearch.yml << EOF
# 集群名称
cluster.name: custom-cluster
# 节点名称
node.name: "node-185"
cluster.initial_master_nodes: ["node-173"]
# 不作为主节点
node.master: false
# 作为数据节点
node.data: true
# 访问的IP地址，0.0.0.0表示不限制
network.host: 0.0.0.0
network.publish_host: nodeip004
# HTTP访问端口号
http.port: 9020
# 集群通讯端口号
transport.tcp.port: 9030
# 所有节点的IP地址和通讯端口
discovery.zen.ping.unicast.hosts: ["nodeip001:9030", "nodeip004:9030", "nodeip003:9030"]
EOF
```

##### 节点三（数据节点）配置

``` shell
cat > config/elasticsearch.yml << EOF
# 集群名称
cluster.name: custom-cluster
# 节点名称
node.name: "node-188"
cluster.initial_master_nodes: ["node-173"]
# 不作为主节点
node.master: false
# 作为数据节点
node.data: true
# 访问的IP地址，0.0.0.0表示不限制
network.host: 0.0.0.0
network.publish_host: nodeip003
# HTTP访问端口号
http.port: 9020
# 集群通讯端口号
transport.tcp.port: 9030
# 所有节点的IP地址和通讯端口
discovery.zen.ping.unicast.hosts: ["nodeip001:9030", "nodeip004:9030", "nodeip003:9030"]
EOF
```

#### 1.2.3 添加安全认证

在所有节点的elasticsearch.yml文件中追加以下配置：

``` yaml
# 启用安全认证
xpack.security.enabled: true
# 设置许可证类型
xpack.license.self_generated.type: basic
# 启用传输层SSL
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

在主节点上生成证书文件，并复制到其他节点：

```bash
# 生成证书文件，密码为空
su es
cd /home/es/elasticsearch-7.17.26/bin/
./elasticsearch-certutil cert -out ../config/elastic-certificates.p12 -pass ""
# 将证书文件复制到其他节点
scp ../config/elastic-certificates.p12 es@nodeip004:/home/es/elasticsearch-7.17.26/config/
scp ../config/elastic-certificates.p12 es@nodeip003:/home/es/elasticsearch-7.17.26/config/
```

#### 1.2.4 配置systemctl服务

创建systemd服务文件，以便通过systemctl管理Elasticsearch服务：

``` shell
cat > /usr/lib/systemd/system/elasticsearch.service << EOF
[Unit]
Description=Elasticsearch - distributed search and analytics engine
Documentation=https://www.elastic.co/docs/en/elasticsearch/reference/
Wants=network-online.target
After=network-online.target

[Service]
Type=forking
User=es
Group=es
Environment=ES_HOME=/home/es/elasticsearch-7.17.26
Environment=ES_PATH_CONF=/home/es/elasticsearch-7.17.26/config
Environment=PID_DIR=/var/run/elasticsearch
ExecStart=/home/es/elasticsearch-7.17.26/bin/elasticsearch -d -p /var/run/elasticsearch/elasticsearch.pid
Restart=always
WorkingDirectory=/home/es/elasticsearch-7.17.26
# 设置资源限制
LimitNOFILE=65535
LimitNPROC=65535
LimitAS=infinity
LimitFSIZE=infinity
TimeoutStopSec=0

[Install]
WantedBy=multi-user.target
EOF

# 创建PID目录
mkdir -p /var/run/elasticsearch
chown -R es:es /var/run/elasticsearch

# 重载systemd配置
systemctl daemon-reload
```

### 1.3 启动和验证Elasticsearch

```bash
# 启动Elasticsearch服务
systemctl start elasticsearch

# 设置开机自启
systemctl enable elasticsearch

# 检查服务状态
systemctl status elasticsearch

# 在主节点上设置密码（首次启动后执行）
su es
cd /home/es/elasticsearch-7.17.26/bin/
./elasticsearch-setup-passwords interactive
# 设置密码：password

# 验证集群状态
curl -u elastic:password http://localhost:9020/_cluster/health?pretty

# 查看节点信息
curl -u elastic:password http://localhost:9020/_cat/nodes?v

# 检查节点状态
curl -u elastic:password http://localhost:9020/_cluster/state?pretty
```

## 2. Kibana部署

### 2.1 安装步骤

#### 2.1.1 上传软件包并解压

```bash
# 上传压缩包并解压
tar -zxvf kibana-7.17.26-linux-x86_64.tar.gz
mv kibana-7.17.26 kibana
cd kibana

# 修改目录归属用户
chown -R es:es /home/es/kibana
```

#### 2.1.2 修改配置文件

```yaml
# kibana.yml
server.port: 5601
server.host: "nodeip001"
elasticsearch.hosts: ["http://nodeip001:9020"]
elasticsearch.username: "username"
elasticsearch.password: "password"
logging.dest: /home/es/kibana/logs/kibana.log
i18n.locale: "zh-CN"
```

#### 2.1.3 配置systemctl服务

``` shell
cat > /usr/lib/systemd/system/kibana.service << EOF
[Unit]
Description=Kibana - visualize and analyze your Elasticsearch data
Documentation=https://www.elastic.co/docs/en/kibana/reference/
Wants=elasticsearch.service
After=elasticsearch.service

[Service]
Type=simple
User=es
Group=es
Environment=KIBANA_HOME=/home/es/kibana
ExecStart=/home/es/kibana/bin/kibana
Restart=always
WorkingDirectory=/home/es/kibana

[Install]
WantedBy=multi-user.target
EOF

# 重载systemd配置
systemctl daemon-reload
```

### 2.2 启动和验证Kibana

```bash
# 启动Kibana服务
systemctl start kibana

# 设置开机自启
systemctl enable kibana

# 检查服务状态
systemctl status kibana

# 查看Kibana日志
journalctl -u kibana -f
```

## 3. Filebeat部署

### 3.1 安装步骤

在所有需要收集日志的节点上安装Filebeat：

#### 3.1.1 上传软件包并解压

```bash
# 上传压缩包并解压
tar -zxvf filebeat-7.17.26-linux-x86_64.tar.gz
mv filebeat-7.17.26 filebeat
cd filebeat

# 修改目录归属用户
chown -R es:es /home/es/filebeat
```

#### 3.1.2 修改配置文件

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /path/to/application/logs/*.log
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}'
  multiline.negate: true
  multiline.match: after

output.elasticsearch:
  hosts: ["http://nodeip001:9020", "http://nodeip004:9020", "http://nodeip003:9020"]
  username: "elastic"
  password: "password"
  index: "filebeat-%{+yyyy.MM.dd}"

setup.kibana:
  host: "http://nodeip001:5601"

processors:
- add_host_metadata: ~
- add_cloud_metadata: ~
- add_docker_metadata: ~
- add_kubernetes_metadata: ~
```

#### 3.1.3 配置systemctl服务

``` shell
cat > /usr/lib/systemd/system/filebeat.service << EOF
[Unit]
Description=Filebeat - lightweight shipper for metrics and log files
Documentation=https://www.elastic.co/docs/en/beats/filebeat/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=es
Group=es
Environment=FILEBEAT_HOME=/home/es/filebeat
ExecStart=/home/es/filebeat/filebeat -c /home/es/filebeat/filebeat.yml
Restart=always
WorkingDirectory=/home/es/filebeat

[Install]
WantedBy=multi-user.target
EOF

# 重载systemd配置
systemctl daemon-reload
```

### 3.2 启动和验证Filebeat

```bash
# 启动Filebeat服务
systemctl start filebeat

# 设置开机自启
systemctl enable filebeat

# 检查服务状态
systemctl status filebeat

# 查看Filebeat日志
journalctl -u filebeat -f
```

## 4. 访问指南

### Elasticsearch访问

- 访问地址：http://nodeip001:9020, http://nodeip004:9020, http://nodeip003:9020
- 用户名：elastic
- 密码：password

### Kibana访问

- 访问地址：http://nodeip001:5601
- 用户名：elastic
- 密码：password

## 5. 使用指南

### Kibana日志查询

1. 登录Kibana界面
2. 在左侧菜单中点击"Discover"
3. 选择相应的索引模式（如：filebeat-*）
4. 使用KQL（Kibana Query Language）进行日志查询
5. 可以通过时间范围选择器筛选特定时间段的日志

### 创建日志可视化

1. 登录Kibana界面
2. 在左侧菜单中点击"Visualize Library"
3. 点击"Create visualization"
4. 选择合适的可视化类型（如：柱状图、折线图、饼图等）
5. 选择数据源（索引）
6. 配置可视化的具体参数
7. 保存可视化

### 创建仪表盘

1. 登录Kibana界面
2. 在左侧菜单中点击"Dashboard"
3. 点击"Create dashboard"
4. 点击"Add visualization"
5. 选择之前创建的可视化组件
6. 调整组件的大小和位置
7. 保存仪表盘

## 6. 维护监控

### 日常维护

1. 定期检查Elasticsearch的存储空间使用情况
2. 根据日志量调整索引生命周期策略
3. 监控Filebeat的收集状态和性能
4. 定期备份重要的日志数据

### 性能优化

1. 根据服务器资源和日志量调整Elasticsearch的JVM内存大小
2. 优化Elasticsearch的索引策略，合理设置分片和副本数量
3. 优化Filebeat的配置，减少不必要的日志收集
4. 使用索引模板和索引生命周期管理（ILM）优化存储空间

## 7. 常见问题排查

### Elasticsearch集群健康状态异常

1. 检查集群状态：`curl -u elastic:password http://localhost:9020/_cluster/health?pretty`
2. 查看节点状态：`curl -u elastic:password http://localhost:9020/_cat/nodes?v`
3. 查看分片状态：`curl -u elastic:password http://localhost:9020/_cat/shards?v`
4. 检查是否有未分配的分片，并分析原因
5. 查看Elasticsearch日志，查找具体错误信息

### Kibana无法连接Elasticsearch

1. 检查Elasticsearch是否正常运行
2. 检查Kibana配置文件中的Elasticsearch连接信息是否正确
3. 检查网络连接是否正常
4. 查看Kibana日志，查找具体错误信息

### Filebeat无法收集日志

1. 检查Filebeat配置文件中的路径是否正确
2. 检查Filebeat是否有足够的权限读取日志文件
3. 检查Filebeat与Elasticsearch之间的网络连接是否正常
4. 查看Filebeat日志，查找具体错误信息

### 日志查询性能问题

1. 优化查询语句，避免使用过于复杂的查询
2. 增加Elasticsearch的资源配置（CPU、内存）
3. 调整索引的分片数量和副本数量
4. 使用索引生命周期管理，定期清理过期日志

## 8. 附录

### 8.1 索引生命周期管理配置示例

```bash
# 创建索引生命周期策略
curl -u elastic:password -XPUT "http://localhost:9020/_ilm/policy/logs_policy" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "7d"
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}'

# 创建索引模板，应用生命周期策略
curl -u elastic:password -XPUT "http://localhost:9020/_index_template/logs_template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}'

# 创建初始索引
curl -u elastic:password -XPUT "http://localhost:9020/logs-000001" -H 'Content-Type: application/json' -d'
{
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  }
}'
```

### 8.2 性能监控指标

1. Elasticsearch性能指标：
   - 集群健康状态
   - 节点CPU、内存使用率
   - 索引查询速率、写入速率
   - 分片状态、磁盘使用率

2. Kibana性能指标：
   - 页面加载时间
   - 查询响应时间
   - 仪表盘渲染时间

3. Filebeat性能指标：
   - 日志收集速率
   - 发送成功/失败的事件数
   - 队列大小和延迟
# SIGTERM是停止java进程的信号
KillSignal=SIGTERM
# 信号只发送给给JVM
KillMode=process
# java进程不会被杀掉
SendSIGKILL=no
# 正常退出状态
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
EOF
```

执行命令

1. `systemctl start elasticsearch`
2. `systemctl enable elasticsearch`

## 1.2 配置UI 部署Kibana

1. yaml文件参考[kibana](./kibana.yaml)
2. 按实际环境修改相关配置：
   1. TODO 需列出具体有哪些配置项待修改
3. 在目标集群中执行部署命令: `kubectl apply -f ./`

## 1.3 部署Filebeat

1. yaml文件参考[filebeat](./filebeat.yaml)
2. 按实际环境修改相关配置：
   1. TODO 需列出具体有哪些配置项待修改
3. 在目标集群中执行部署命令: `kubectl apply -f ./`
