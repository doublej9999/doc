# 一键离线部署 ELK（K8s 实战版）

> 目标：在**无外网 Kubernetes 环境**，通过离线包 + 脚本一键部署 ELK（Elasticsearch / Logstash / Kibana），并完成基础日志接入验证。  
> 适用：K8s 1.24+，建议至少 3 节点（生产）。

---

## 1. 部署方案说明

离线场景采用 Helm + 私有镜像仓库：

- Elasticsearch：有状态集群（StatefulSet + PVC）
- Kibana：可视化入口（NodePort/Ingress）
- Logstash：日志处理管道（可接 Beats / TCP / HTTP）

推荐命名空间：`logging`

---

## 2. 目录结构（建议）

```text
offline-elk/
├── 00_import_images.sh
├── 01_install_elasticsearch.sh
├── 02_install_kibana.sh
├── 03_install_logstash.sh
├── 04_postcheck.sh
├── values-elasticsearch.yaml
├── values-kibana.yaml
├── values-logstash.yaml
├── elasticsearch-*.tgz
├── kibana-*.tgz
├── logstash-*.tgz
└── images.txt
```

---

## 3. 有网机器准备离线包

### 3.1 拉取 Elastic Helm Charts

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm pull elastic/elasticsearch --version 8.14.3
helm pull elastic/kibana --version 8.14.3
helm pull elastic/logstash --version 8.14.3
```

### 3.2 准备镜像清单

`images.txt` 示例（版本保持一致）：

```text
docker.elastic.co/elasticsearch/elasticsearch:8.14.3
docker.elastic.co/kibana/kibana:8.14.3
docker.elastic.co/logstash/logstash:8.14.3
```

### 3.3 拉取并打包镜像

```bash
while read -r img; do docker pull "$img"; done < images.txt
docker save $(cat images.txt) -o elk-images-8.14.3.tar
```

把以下内容传入内网：

- `elasticsearch-8.14.3.tgz`
- `kibana-8.14.3.tgz`
- `logstash-8.14.3.tgz`
- `images.txt`
- `elk-images-8.14.3.tar`
- 三个 values 文件 + 四个脚本

---

## 4. 无网环境部署（核心配置）

### 4.1 Elasticsearch values

`values-elasticsearch.yaml`

```yaml
image: "registry.local:5000/docker.elastic.co/elasticsearch/elasticsearch"
imageTag: "8.14.3"

replicas: 3
minimumMasterNodes: 2

esConfig:
  elasticsearch.yml: |
    xpack.security.enabled: false
    discovery.seed_hosts: []
    cluster.name: elk-offline
    network.host: 0.0.0.0

resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"

volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Gi
```

> 说明：`xpack.security.enabled: false` 仅用于快速离线验证；生产建议开启安全认证与 TLS。

---

### 4.2 Kibana values

`values-kibana.yaml`

```yaml
image: "registry.local:5000/docker.elastic.co/kibana/kibana"
imageTag: "8.14.3"

service:
  type: NodePort
  nodePort: 30601

elasticsearchHosts: "http://elasticsearch-master.logging.svc.cluster.local:9200"

resources:
  requests:
    cpu: "200m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

---

### 4.3 Logstash values

`values-logstash.yaml`

```yaml
image: "registry.local:5000/docker.elastic.co/logstash/logstash"
imageTag: "8.14.3"

logstashConfig:
  logstash.yml: |
    http.host: 0.0.0.0
    xpack.monitoring.enabled: false

logstashPipeline:
  logstash.conf: |
    input {
      tcp {
        port => 5044
        codec => json
      }
    }
    filter { }
    output {
      elasticsearch {
        hosts => ["http://elasticsearch-master.logging.svc.cluster.local:9200"]
        index => "logstash-%{+YYYY.MM.dd}"
      }
      stdout { codec => rubydebug }
    }

service:
  type: NodePort
  ports:
    - name: beats
      port: 5044
      targetPort: 5044
      protocol: TCP
      nodePort: 30544

resources:
  requests:
    cpu: "200m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "2Gi"
```

---

## 5. 一键脚本

### 5.1 导入镜像到私仓

`00_import_images.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

REGISTRY="registry.local:5000"
TAR_FILE="elk-images-8.14.3.tar"
IMAGES_FILE="images.txt"

echo "[1/3] docker load..."
docker load -i "${TAR_FILE}"

echo "[2/3] tag & push..."
while read -r IMG; do
  [ -z "$IMG" ] && continue
  NEW_IMG="${REGISTRY}/${IMG}"
  docker tag "${IMG}" "${NEW_IMG}"
  docker push "${NEW_IMG}"
  echo "Pushed => ${NEW_IMG}"
done < "${IMAGES_FILE}"

echo "[3/3] done."
```

---

### 5.2 安装 Elasticsearch

`01_install_elasticsearch.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

NS="logging"
kubectl get ns "${NS}" >/dev/null 2>&1 || kubectl create ns "${NS}"

helm upgrade --install elasticsearch elasticsearch-8.14.3.tgz \
  -n "${NS}" \
  -f values-elasticsearch.yaml \
  --wait --timeout 20m

kubectl -n "${NS}" get pods -l app=elasticsearch-master -o wide
```

---

### 5.3 安装 Kibana

`02_install_kibana.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

NS="logging"
helm upgrade --install kibana kibana-8.14.3.tgz \
  -n "${NS}" \
  -f values-kibana.yaml \
  --wait --timeout 10m

kubectl -n "${NS}" get svc kibana-kibana
```

---

### 5.4 安装 Logstash

`03_install_logstash.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

NS="logging"
helm upgrade --install logstash logstash-8.14.3.tgz \
  -n "${NS}" \
  -f values-logstash.yaml \
  --wait --timeout 10m

kubectl -n "${NS}" get svc logstash-logstash
```

---

### 5.5 验收脚本

`04_postcheck.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

NS="logging"

echo "==== Pods ===="
kubectl -n "${NS}" get pods -o wide

echo "==== Services ===="
kubectl -n "${NS}" get svc

echo "==== Elasticsearch health ===="
kubectl -n "${NS}" run es-curl --rm -it --restart=Never --image=curlimages/curl:8.7.1 -- \
  curl -s http://elasticsearch-master.logging.svc.cluster.local:9200/_cluster/health || true

echo "==== Done ===="
```

---

## 6. 一键执行顺序

```bash
chmod +x 00_import_images.sh 01_install_elasticsearch.sh 02_install_kibana.sh 03_install_logstash.sh 04_postcheck.sh

./00_import_images.sh
./01_install_elasticsearch.sh
./02_install_kibana.sh
./03_install_logstash.sh
./04_postcheck.sh
```

---

## 7. 访问入口

- Kibana：`http://<任一K8s节点IP>:30601`
- Logstash TCP 输入：`<任一K8s节点IP>:30544`

---

## 8. 快速日志验证（模拟写入）

从任意可访问 Logstash 的机器发送一条 JSON：

```bash
echo '{"app":"demo","level":"info","msg":"hello elk offline","ts":"2026-03-24T12:00:00+08:00"}' | nc <NODE_IP> 30544
```

然后进 Kibana 创建索引模式：`logstash-*`，即可搜索日志。

---

## 9. 常见问题排查

### 9.1 Elasticsearch 启动失败（内存/虚拟内存）

检查：

```bash
kubectl -n logging describe pod <elasticsearch-pod>
```

常见点：

- 节点内存不足（建议单 ES Pod 至少 2Gi+）
- 存储类异常，PVC 未绑定
- 宿主机 `vm.max_map_count` 未满足（需在节点设置）

> 节点执行：
```bash
sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
```

### 9.2 Pod ImagePullBackOff

- 镜像路径是否改为 `registry.local:5000/...`
- 私仓证书是否被节点信任
- 镜像是否已成功 push

### 9.3 Kibana 无法连接 ES

检查 `elasticsearchHosts` 是否正确；  
确认 `elasticsearch-master.logging.svc.cluster.local:9200` 可达。

---

## 10. 生产建议

- 开启 xpack 安全（用户名/密码、TLS）
- ES 主数据节点角色拆分，避免单点瓶颈
- 配置 ILM（冷热分层 + 自动删除）
- Logstash 前加 Kafka 做缓冲（高吞吐场景）
- 定期快照备份（S3/MinIO/NFS）

---

## 11. 验收清单

- [ ] `logging` 命名空间 Pod 全部 Running  
- [ ] ES `_cluster/health` 为 green/yellow（可接受）  
- [ ] Kibana 可访问并能查到 `logstash-*` 日志  
- [ ] PVC 全部 Bound  
- [ ] 节点重启后服务可恢复  

---

> 如果你愿意，我下一步可以给你“**Filebeat/Fluent Bit -> Logstash -> Elasticsearch**”完整链路离线方案（含 DaemonSet 配置），直接用于采集 K8s 容器日志。
