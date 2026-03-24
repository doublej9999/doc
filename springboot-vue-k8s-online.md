# Spring Boot + Vue 前后端项目上 Kubernetes（在线部署版）

> 适用：K8s 可访问公网或企业在线镜像仓库。  
> 目标：标准化部署 Spring Boot API + Vue 前端，并提供 Ingress 暴露与弹性扩缩容。

---

## 1. 架构

- backend：Spring Boot（Deployment + Service）
- frontend：Vue + Nginx（Deployment + Service）
- ingress：路由 `app.example.com` 与 `api.example.com`
- config/secrets：ConfigMap + Secret

---

## 2. 构建镜像

### 2.1 Spring Boot Dockerfile

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

```bash
cd backend
mvn clean package -DskipTests
docker build -t registry.example.com/demo-backend:1.0.0 .
docker push registry.example.com/demo-backend:1.0.0
```

### 2.2 Vue Dockerfile

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`nginx.conf`：

```nginx
server {
  listen 80;
  location / {
    root /usr/share/nginx/html;
    try_files $uri $uri/ /index.html;
  }
}
```

```bash
cd frontend
docker build -t registry.example.com/demo-frontend:1.0.0 .
docker push registry.example.com/demo-frontend:1.0.0
```

---

## 3. K8s 清单（核心）

## 3.1 Deployment + Service（backend）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels: { app: backend }
  template:
    metadata:
      labels: { app: backend }
    spec:
      containers:
        - name: backend
          image: registry.example.com/demo-backend:1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: demo-app
spec:
  selector: { app: backend }
  ports:
    - port: 8080
      targetPort: 8080
```

## 3.2 Deployment + Service（frontend）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels: { app: frontend }
  template:
    metadata:
      labels: { app: frontend }
    spec:
      containers:
        - name: frontend
          image: registry.example.com/demo-frontend:1.0.0
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: demo-app
spec:
  selector: { app: frontend }
  ports:
    - port: 80
      targetPort: 80
```

## 3.3 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app-ingress
  namespace: demo-app
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

---

## 4. 一键部署

```bash
kubectl create ns demo-app || true
kubectl apply -f k8s/
```

---

## 5. 验收

```bash
kubectl -n demo-app get pods,svc,ingress
kubectl -n demo-app port-forward svc/backend 18080:8080
curl http://127.0.0.1:18080/actuator/health
```

---

## 6. 生产建议

- Ingress 开启 TLS（cert-manager）
- Secret 不落明文（Vault / SOPS / SealedSecrets）
- 配置 HPA、PDB、资源限额
- 接入监控告警（Prometheus + Grafana）
