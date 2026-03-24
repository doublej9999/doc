# Spring Boot + Vue 前后端项目上 Kubernetes 实战指南

> 目标：把一个典型的 **Spring Boot API + Vue 前端** 项目，标准化部署到 K8s（支持开发/测试/生产场景）。

---

## 1. 总体架构

推荐架构：

- **backend**：Spring Boot（Deployment + Service）
- **frontend**：Vue 打包为静态资源，由 Nginx 提供（Deployment + Service）
- **database**：MySQL/PostgreSQL（建议托管或独立中间件，不建议直接耦合在同集群）
- **ingress**：统一入口（`api.example.com`、`app.example.com`）
- **config/secrets**：ConfigMap + Secret 管理环境配置

---

## 2. 目录建议

```text
project/
├── backend/                      # Spring Boot
│   ├── Dockerfile
│   └── src/...
├── frontend/                     # Vue
│   ├── Dockerfile
│   └── src/...
└── k8s/
    ├── namespace.yaml
    ├── backend-deploy.yaml
    ├── backend-svc.yaml
    ├── frontend-deploy.yaml
    ├── frontend-svc.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── ingress.yaml
    └── hpa.yaml
```

---

## 3. 镜像构建

## 3.1 Spring Boot Dockerfile

`backend/Dockerfile`

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

构建：

```bash
cd backend
mvn clean package -DskipTests
docker build -t registry.local:5000/demo-backend:1.0.0 .
docker push registry.local:5000/demo-backend:1.0.0
```

## 3.2 Vue Dockerfile

`frontend/Dockerfile`

```dockerfile
# build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# runtime stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`frontend/nginx.conf`（支持前端路由）

```nginx
server {
  listen 80;
  server_name _;

  location / {
    root /usr/share/nginx/html;
    try_files $uri $uri/ /index.html;
  }
}
```

构建：

```bash
cd frontend
docker build -t registry.local:5000/demo-frontend:1.0.0 .
docker push registry.local:5000/demo-frontend:1.0.0
```

---

## 4. K8s 资源清单

## 4.1 namespace

`k8s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
```

## 4.2 ConfigMap（后端非敏感配置）

`k8s/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: demo-app
data:
  SPRING_PROFILES_ACTIVE: "prod"
  APP_LOG_LEVEL: "INFO"
  DB_HOST: "mysql.default.svc.cluster.local"
  DB_PORT: "3306"
  DB_NAME: "demo"
```

## 4.3 Secret（敏感配置）

`k8s/secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: demo-app
type: Opaque
stringData:
  DB_USER: "demo_user"
  DB_PASSWORD: "ReplaceMe_StrongPassword"
  JWT_SECRET: "ReplaceMe_JWT_Secret"
```

## 4.4 Spring Boot Deployment

`k8s/backend-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: registry.local:5000/demo-backend:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: backend-config
            - secretRef:
                name: backend-secret
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:mysql://$(DB_HOST):$(DB_PORT)/$(DB_NAME)?useSSL=false&serverTimezone=Asia/Shanghai
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: backend-secret
                  key: DB_USER
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: backend-secret
                  key: DB_PASSWORD
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
          resources:
            requests:
              cpu: "200m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1024Mi"
```

> 建议启用 Spring Boot Actuator：
> - `management.endpoint.health.probes.enabled=true`
> - `management.health.livenessstate.enabled=true`
> - `management.health.readinessstate.enabled=true`

## 4.5 Spring Boot Service

`k8s/backend-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: demo-app
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

## 4.6 Vue Deployment

`k8s/frontend-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: registry.local:5000/demo-frontend:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

## 4.7 Vue Service

`k8s/frontend-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: demo-app
spec:
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
```

## 4.8 Ingress

`k8s/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app-ingress
  namespace: demo-app
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 8080
```

## 4.9 HPA（可选）

`k8s/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: demo-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 5. 一键部署命令

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/backend-deploy.yaml
kubectl apply -f k8s/backend-svc.yaml
kubectl apply -f k8s/frontend-deploy.yaml
kubectl apply -f k8s/frontend-svc.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml
```

或者直接：

```bash
kubectl apply -f k8s/
```

---

## 6. 验收步骤

## 6.1 基本状态

```bash
kubectl get ns
kubectl -n demo-app get pods -o wide
kubectl -n demo-app get svc
kubectl -n demo-app get ingress
```

## 6.2 健康检查

```bash
kubectl -n demo-app port-forward svc/backend 18080:8080
curl http://127.0.0.1:18080/actuator/health
```

## 6.3 前端访问

- 配置本地 DNS 或 hosts：
  - `app.example.com` -> Ingress LB IP
  - `api.example.com` -> Ingress LB IP
- 浏览器访问 `http://app.example.com`

---

## 7. CI/CD 建议（简版）

推荐流水线步骤：

1. 后端单测 + 打包
2. 前端构建 + 产物校验
3. 构建并推送镜像（带版本号/commit sha）
4. 执行 `helm upgrade` 或 `kubectl apply`
5. 部署后健康检查，不通过自动回滚

镜像 tag 推荐：

- `demo-backend:1.0.0-<gitsha>`
- `demo-frontend:1.0.0-<gitsha>`

---

## 8. 常见问题

## 8.1 前端 404（刷新页面丢路由）

Nginx 需配置：

```nginx
try_files $uri $uri/ /index.html;
```

## 8.2 后端连不上数据库

- 检查 `DB_HOST/DB_PORT/DB_NAME`
- 检查 Secret 账号密码
- 检查网络策略（NetworkPolicy）是否阻断

## 8.3 Pod 一直重启

```bash
kubectl -n demo-app logs <pod-name> --previous
kubectl -n demo-app describe pod <pod-name>
```

重点看：

- 探针路径是否正确
- JVM 内存限制是否过低
- 配置项是否缺失

---

## 9. 生产落地建议

- 开启 HTTPS（Ingress + cert-manager）
- 开启日志采集（ELK/Loki）与指标监控（Prometheus + Grafana）
- 使用外部数据库高可用方案
- 使用 Helm/Kustomize 管理多环境配置
- 对 Secret 使用 Vault/密文方案（SealedSecrets/SOPS）

---

## 10. 快速 Checklist

- [ ] 前后端镜像均已推送私仓
- [ ] `demo-app` 命名空间下 Pod 全部 Running
- [ ] Ingress 路由可用（前后端域名正常）
- [ ] 后端健康检查通过
- [ ] HPA 正常工作（可选）
- [ ] 日志/监控/备份策略已配置（生产）

---

> 如果你需要，我可以再补一版：
> - **Helm Chart 版本**（前后端同一 Chart，支持 values 切 dev/test/prod）
> - **离线版**（含私仓镜像重定向与完整脚本）
