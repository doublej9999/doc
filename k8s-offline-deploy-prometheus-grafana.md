# 一键离线部署 Prometheus + Grafana（K8s 实战版）

> 目标：在**无外网 Kubernetes 环境**，一键完成 Prometheus + Grafana 部署、持久化、NodePort 暴露与基础验收。  
> 适用：k8s 1.24+，containerd/docker 均可（镜像已提前导入私仓）。

---

## 1. 方案说明

采用 `kube-prometheus-stack`（Helm Chart）离线安装：

- 包含：Prometheus、Alertmanager、Grafana、Node Exporter、Kube State Metrics
- 离线关键点：
  1) Chart 本地化（`.tgz`）
  2) 镜像推送到私有仓库
  3) `values.yaml` 里统一改为 `global.imageRegistry`

---

## 2. 目录结构（建议）

```text
offline-monitoring/
├── 00_import_images.sh
├── 01_install_monitoring.sh
├── values-prom-stack.yaml
├── kube-prometheus-stack-*.tgz
└── images.txt
```

---

## 3. 有网机器准备离线包

### 3.1 拉取 Chart

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm pull prometheus-community/kube-prometheus-stack --version 61.9.0
```

### 3.2 收集镜像（示例思路）

```bash
helm show values kube-prometheus-stack-61.9.0.tgz > values-default.yaml
# 根据 values 与官方文档整理 images.txt（建议固定版本）
```

`images.txt` 示例（按你版本调整）：

```text
quay.io/prometheus/prometheus:v2.53.1
quay.io/prometheus-operator/prometheus-operator:v0.75.2
quay.io/prometheus/alertmanager:v0.27.0
registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.13.0
grafana/grafana:11.1.0
quay.io/prometheus/node-exporter:v1.8.2
```

### 3.3 打包镜像

```bash
while read -r img; do docker pull "$img"; done < images.txt
docker save $(cat images.txt) -o prom-grafana-images.tar
```

把以下文件拷贝进内网：

- `kube-prometheus-stack-61.9.0.tgz`
- `images.txt`
- `prom-grafana-images.tar`
- `values-prom-stack.yaml`（下一节提供模板）

---

## 4. 无网环境一键部署

### 4.1 values 文件（核心）

`values-prom-stack.yaml`：

```yaml
global:
  imageRegistry: "registry.local:5000"

grafana:
  enabled: true
  adminPassword: "Admin@123456"
  service:
    type: NodePort
    nodePort: 30300
  persistence:
    enabled: true
    size: 10Gi

prometheus:
  prometheusSpec:
    retention: 15d
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```

---

### 4.2 一键脚本 1：导入并推送镜像

`00_import_images.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

REGISTRY="registry.local:5000"
TAR_FILE="prom-grafana-images.tar"
IMAGES_FILE="images.txt"

echo "[1/3] Loading images from tar..."
docker load -i "${TAR_FILE}"

echo "[2/3] Tag & push to private registry: ${REGISTRY}"
while read -r IMG; do
  [ -z "$IMG" ] && continue
  NEW_IMG="${REGISTRY}/${IMG}"
  docker tag "${IMG}" "${NEW_IMG}"
  docker push "${NEW_IMG}"
  echo "Pushed: ${NEW_IMG}"
done < "${IMAGES_FILE}"

echo "[3/3] Done."
```

---

### 4.3 一键脚本 2：安装 Prometheus + Grafana

`01_install_monitoring.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="monitoring"
CHART="kube-prometheus-stack-61.9.0.tgz"
VALUES="values-prom-stack.yaml"

kubectl get ns "${NAMESPACE}" >/dev/null 2>&1 || kubectl create ns "${NAMESPACE}"

helm upgrade --install prom-stack "${CHART}" \
  -n "${NAMESPACE}" \
  -f "${VALUES}" \
  --wait --timeout 15m

echo "===== Deploy done ====="
kubectl -n "${NAMESPACE}" get pods -o wide
kubectl -n "${NAMESPACE}" get svc
```

执行顺序：

```bash
chmod +x 00_import_images.sh 01_install_monitoring.sh
./00_import_images.sh
./01_install_monitoring.sh
```

---

## 5. 访问方式

### 5.1 Grafana

- URL: `http://<任一K8s节点IP>:30300`
- 用户名: `admin`
- 密码: `values-prom-stack.yaml` 中 `grafana.adminPassword`

### 5.2 Prometheus

```bash
kubectl -n monitoring port-forward svc/prom-stack-kube-prometheus-prometheus 9090:9090
# 本机访问 http://127.0.0.1:9090
```

---

## 6. 常见故障排查

### 6.1 ImagePullBackOff

```bash
kubectl -n monitoring describe pod <pod-name>
```

检查：

- 镜像是否已推送至 `registry.local:5000`
- `global.imageRegistry` 是否生效
- 节点是否信任私仓 TLS 证书

### 6.2 PVC Pending

```bash
kubectl -n monitoring get pvc
kubectl get sc
```

检查：

- 默认 StorageClass 是否存在
- 存储后端（NFS/Longhorn/Ceph）是否正常

### 6.3 Grafana 登录失败

```bash
kubectl -n monitoring get secret | grep grafana
```

可重置管理员密码（升级 values 后 `helm upgrade`）。

---

## 7. 生产建议

- 把 `adminPassword` 改为 K8s Secret 管理
- 启用 Ingress + TLS（企业 CA）
- 配置告警通知（企业微信/钉钉/Slack/Webhook）
- 保留策略按磁盘容量调整（如 15d/30d）
- 对关键命名空间配置额外 Rule（API、DB、MQ）

---

## 8. 验收清单

- [ ] `kubectl -n monitoring get pods` 全部 Running  
- [ ] Grafana 可登录，内置 dashboard 有数据  
- [ ] Prometheus Targets 大部分为 UP  
- [ ] 节点/Pod 指标正常采集  
- [ ] PVC 已 Bound，重启后数据保留  

---

> 需要的话，我可以下一步给你一份 **“适配你当前 doc 仓库的版本”**，并直接生成：
> - `offline-monitoring/00_import_images.sh`
> - `offline-monitoring/01_install_monitoring.sh`
> - `offline-monitoring/values-prom-stack.yaml`
> - `offline-monitoring/README.md`
> 然后帮你一键提交到 GitHub。
