# Spring Boot + Vue 前后端项目上 Kubernetes（离线部署版）

> 适用：内网隔离、无公网访问环境。  
> 目标：通过离线资源包 + 私有镜像仓库完成完整部署。

---

## 1. 离线部署核心思路

1. 在有网机器下载/构建镜像与资源
2. 导出 tar 包并传输到内网
3. 导入私有仓库（Harbor/Registry）
4. 在 K8s 使用私仓地址部署

---

## 2. 目录结构建议

```text
offline-sbv/
├── images/
│   ├── backend_1.0.0.tar
│   ├── frontend_1.0.0.tar
│   └── images.txt
├── manifests/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── backend-deploy.yaml
│   ├── backend-svc.yaml
│   ├── frontend-deploy.yaml
│   ├── frontend-svc.yaml
│   └── ingress.yaml
├── 00_import_images.sh
└── 01_deploy_k8s.sh
```

---

## 3. 有网机器准备镜像

```bash
# 构建
cd backend && mvn clean package -DskipTests
cd .. && docker build -t demo-backend:1.0.0 ./backend

docker build -t demo-frontend:1.0.0 ./frontend

# 导出
docker save demo-backend:1.0.0 -o backend_1.0.0.tar
docker save demo-frontend:1.0.0 -o frontend_1.0.0.tar
```

`images.txt`：

```text
demo-backend:1.0.0
demo-frontend:1.0.0
```

---

## 4. 内网导入私仓脚本

`00_import_images.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

REGISTRY="registry.local:5000"

docker load -i images/backend_1.0.0.tar
docker load -i images/frontend_1.0.0.tar

for img in demo-backend:1.0.0 demo-frontend:1.0.0; do
  new_img="${REGISTRY}/${img}"
  docker tag "$img" "$new_img"
  docker push "$new_img"
  echo "Pushed: $new_img"
done
```

---

## 5. K8s 部署清单要点

把镜像改为私仓地址：

- `registry.local:5000/demo-backend:1.0.0`
- `registry.local:5000/demo-frontend:1.0.0`

如果私仓开启认证：

```bash
kubectl -n demo-app create secret docker-registry regcred \
  --docker-server=registry.local:5000 \
  --docker-username=<user> \
  --docker-password=<password>
```

Deployment 里增加：

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: regcred
```

---

## 6. 一键部署脚本

`01_deploy_k8s.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

kubectl create ns demo-app || true
kubectl apply -f manifests/
kubectl -n demo-app rollout status deploy/backend --timeout=300s
kubectl -n demo-app rollout status deploy/frontend --timeout=300s
kubectl -n demo-app get pods,svc,ingress
```

执行：

```bash
chmod +x 00_import_images.sh 01_deploy_k8s.sh
./00_import_images.sh
./01_deploy_k8s.sh
```

---

## 7. 常见问题

## 7.1 ImagePullBackOff

- 私仓地址是否正确
- 节点是否可达私仓
- `imagePullSecrets` 是否配置
- 私仓证书是否信任

## 7.2 前端访问白屏/404

- Nginx 是否配置 `try_files ... /index.html`
- 前端 API 地址是否指向内网可达域名

## 7.3 后端启动失败

```bash
kubectl -n demo-app logs deploy/backend
kubectl -n demo-app describe pod -l app=backend
```

重点检查 DB 连接参数与 Secret。

---

## 8. 生产加固建议

- 所有敏感配置改 Secret/密文管理
- Ingress 开启 TLS 与访问控制
- 增加日志、监控、告警
- 使用 Helm/Kustomize 进行多环境参数化
- 配置备份与回滚策略
