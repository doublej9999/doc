# 一键离线部署 Rancher + K8s（生产可参考版）

> 目标：在**无外网环境**，通过一套脚本完成  
> 1) K8s（推荐 RKE2）部署  
> 2) 私有镜像仓库准备  
> 3) Rancher 离线安装（Helm Chart + 镜像重定向）  
> 4) 节点导入与基础验收  

---

## 1. 架构说明（离线标准方案）

离线部署推荐分两台角色：

- **跳板/制品机（有网）**
  - 下载离线资源包（RKE2、Rancher Chart、镜像）
  - 打包并传输到内网
- **内网部署机（无网）**
  - 安装私有镜像仓库（Harbor/Registry）
  - 安装 RKE2（或 K3s）
  - 用本地 Chart + 私有仓库安装 Rancher

---

## 2. 环境准备

### 2.1 节点规划（示例）

- `10.0.0.11`：rke2-server-1（控制平面）
- `10.0.0.12`：rke2-agent-1（工作节点）
- `10.0.0.20`：registry.local（私有仓库）
- `10.0.0.30`：lb.local（可选，4层负载）

### 2.2 基础要求

- OS：Ubuntu 20.04+/Rocky 8+/CentOS 7.9+
- CPU/内存（最小）：
  - Rancher 管理集群：4C/8G（建议 8C/16G）
- 必须准备：
  - 内网 DNS（如 `rancher.local`）
  - 自签或企业 CA 证书
  - 时间同步（NTP）

---

## 3. 离线资源下载（有网机器执行）

> 以下版本按需替换  
> - Rancher: `v2.8.x`  
> - RKE2: `v1.28.x+rke2r1`

### 3.1 下载 Rancher Chart

```bash
mkdir -p offline-bundle/charts
cd offline-bundle/charts

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm pull rancher-stable/rancher --version 2.8.5
```

### 3.2 下载 Rancher 镜像列表并拉取镜像

可用 `rancher-images.txt`（按版本匹配），示例：

```bash
mkdir -p ../images
cd ../images

# 假设你已拿到镜像清单 rancher-images.txt
while read -r image; do
  docker pull "$image"
done < rancher-images.txt
```

### 3.3 导出镜像包

```bash
docker save $(cat rancher-images.txt) -o rancher-images-2.8.5.tar
```

### 3.4 下载 RKE2 离线安装包

从 RKE2 release 页面下载（示例）：

- `rke2.linux-amd64.tar.gz`
- `rke2-images.linux-amd64.tar.zst`
- `sha256sum-amd64.txt`

放入：

```bash
mkdir -p ../rke2
mv rke2* ../rke2/
```

### 3.5 打包整体离线包

```bash
cd ..
tar -czf rancher-offline-bundle-2.8.5.tgz charts images rke2
```

传到内网部署机（U 盘、堡垒机中转、scp 等）。

---

## 4. 内网部署（无网机器执行）

### 4.1 解压离线包

```bash
mkdir -p /opt/rancher-offline
tar -xzf rancher-offline-bundle-2.8.5.tgz -C /opt/rancher-offline
```

### 4.2 安装私有镜像仓库（简化示例：registry）

```bash
docker run -d --restart=always --name registry \
  -p 5000:5000 \
  -v /opt/registry/data:/var/lib/registry \
  registry:2
```

### 4.3 导入 Rancher 镜像并推送到私仓

```bash
cd /opt/rancher-offline/images
docker load -i rancher-images-2.8.5.tar

REG=registry.local:5000
while read -r image; do
  new_image="${REG}/${image}"
  docker tag "$image" "$new_image"
  docker push "$new_image"
done < rancher-images.txt
```

> 如果使用 Harbor，建议建项目 `rancher`，并改写目标镜像路径。

---

## 5. 一键安装 RKE2（控制平面）

创建脚本：`install-rke2-server.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

mkdir -p /var/lib/rancher/rke2/agent/images
cp /opt/rancher-offline/rke2/rke2-images.linux-amd64.tar.zst /var/lib/rancher/rke2/agent/images/

tar -xzf /opt/rancher-offline/rke2/rke2.linux-amd64.tar.gz -C /usr/local

mkdir -p /etc/rancher/rke2
cat >/etc/rancher/rke2/config.yaml <<EOF
write-kubeconfig-mode: "0644"
tls-san:
  - rancher.local
cni: canal
