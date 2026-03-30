# Flowable UI 离线部署文档

## 1. 目标

在 **无法访问外网** 的服务器上部署 `flowable/flowable-ui`，并可通过浏览器访问。

本文档采用“**有网机器提前下载资源 → 无网机器导入运行**”方式，适用于内网、政企、金融、生产隔离区等离线场景。

建议实际落地时不要长期使用 `latest`，而是固定明确版本，例如：`6.8.0`。

---

## 2. 部署对象

- Docker 镜像：`flowable/flowable-ui`
- 默认容器访问端口：`8080`
- 默认访问路径：`/flowable-ui`

访问示例：

```text
http://<服务器IP>:8080/flowable-ui
```

---

## 3. 环境要求

离线服务器需要提前具备以下环境：

- Linux 服务器一台
- Docker 已安装并可正常使用
- 具备足够磁盘空间（建议至少 3GB）
- 防火墙 / 安全组已放行业务端口（默认 `8080`）

检查 Docker：

```bash
docker --version
systemctl status docker
```

如果 Docker 未启动：

```bash
systemctl start docker
systemctl enable docker
```

---

## 4. 离线部署前需提前下载的资源

以下资源需在 **可联网机器** 上提前下载，并带入离线环境。

### 4.1 必需资源

#### 资源 1：Flowable UI Docker 镜像

- Docker Hub 页面：
  - `https://hub.docker.com/r/flowable/flowable-ui`
- Docker Registry 拉取地址：
  - `docker.io/flowable/flowable-ui:latest`
- 如固定版本，示例：
  - `docker.io/flowable/flowable-ui:6.8.0`

在线机器下载命令：

```bash
docker pull flowable/flowable-ui:latest
```

固定版本示例：

```bash
docker pull flowable/flowable-ui:6.8.0
```

导出为离线包：

```bash
docker save -o flowable-ui-latest.tar flowable/flowable-ui:latest
```

或：

```bash
docker save -o flowable-ui-6.8.0.tar flowable/flowable-ui:6.8.0
```

#### 资源 2：镜像校验文件（建议）

生成 SHA256：

```bash
sha256sum flowable-ui-latest.tar > flowable-ui-latest.tar.sha256
```

或：

```bash
sha256sum flowable-ui-6.8.0.tar > flowable-ui-6.8.0.tar.sha256
```

---

### 4.2 建议提前下载的辅助资源

#### 资源 3：镜像说明文档

- Flowable UI Docker Hub：
  - `https://hub.docker.com/r/flowable/flowable-ui`

#### 资源 4：Flowable 官方文档（如需本地留档）

- Flowable 官方网站：
  - `https://www.flowable.com/`
- Flowable 开源代码仓库：
  - `https://github.com/flowable/flowable-engine`

> 如果离线区对资料归档有要求，建议将上述页面内容另存为 PDF 或 Markdown 一并归档。

---

## 5. 在线机器操作

以下操作在 **可以联网的机器** 上执行。

### 5.1 拉取镜像

```bash
docker pull flowable/flowable-ui:latest
```

建议使用固定版本：

```bash
docker pull flowable/flowable-ui:6.8.0
```

### 5.2 导出镜像包

```bash
docker save -o flowable-ui-latest.tar flowable/flowable-ui:latest
```

或：

```bash
docker save -o flowable-ui-6.8.0.tar flowable/flowable-ui:6.8.0
```

### 5.3 生成校验值

```bash
sha256sum flowable-ui-latest.tar > flowable-ui-latest.tar.sha256
```

### 5.4 需要带入离线环境的文件清单

至少包括：

- `flowable-ui-latest.tar`
- `flowable-ui-latest.tar.sha256`

如果使用固定版本，则对应为：

- `flowable-ui-6.8.0.tar`
- `flowable-ui-6.8.0.tar.sha256`

可通过 U 盘、堡垒机、内网传输、文件摆渡等方式带入离线服务器。

---

## 6. 离线服务器操作

### 6.1 校验镜像包完整性

```bash
sha256sum -c flowable-ui-latest.tar.sha256
```

如果提示 `OK`，说明文件未损坏。

### 6.2 导入镜像

```bash
docker load -i flowable-ui-latest.tar
```

或：

```bash
docker load -i flowable-ui-6.8.0.tar
```

### 6.3 查看镜像

```bash
docker images | grep flowable
```

预期可见：

```text
flowable/flowable-ui   latest
```

或：

```text
flowable/flowable-ui   6.8.0
```

---

## 7. 启动 Flowable UI

### 7.1 默认启动方式

```bash
docker run -d \
  --name flowable-ui \
  --restart unless-stopped \
  -p 8080:8080 \
  flowable/flowable-ui:latest
```

固定版本示例：

```bash
docker run -d \
  --name flowable-ui \
  --restart unless-stopped \
  -p 8080:8080 \
  flowable/flowable-ui:6.8.0
```

### 7.2 查看运行状态

```bash
docker ps --filter name=flowable-ui
```

### 7.3 查看启动日志

```bash
docker logs -f flowable-ui
```

若日志中看到类似以下内容，通常说明启动正常：

```text
Tomcat initialized with port(s): 8080 (http)
Initializing Spring embedded WebApplicationContext
```

---

## 8. 访问地址

浏览器访问：

```text
http://<服务器IP>:8080/flowable-ui
```

例如：

```text
http://192.168.1.100:8080/flowable-ui
```

---

## 9. 默认账号密码

默认登录信息如下：

- 用户名：`admin`
- 密码：`test`

> 首次登录后建议立即修改默认密码，避免安全风险。

---

## 10. 端口放行

如果服务器启用了防火墙，需要放行 `8080/tcp`。

### 10.1 firewalld

```bash
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

### 10.2 iptables

```bash
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
service iptables save
```

### 10.3 云主机安全组

若部署在云服务器，还需在安全组中放通：

- 协议：TCP
- 端口：8080
- 来源：按需限制

---

## 11. 常用运维命令

### 启动容器

```bash
docker start flowable-ui
```

### 停止容器

```bash
docker stop flowable-ui
```

### 重启容器

```bash
docker restart flowable-ui
```

### 查看日志

```bash
docker logs --tail 200 flowable-ui
```

### 删除容器

```bash
docker rm -f flowable-ui
```

---

## 12. 修改端口启动

如果服务器上的 `8080` 已被占用，可改为其他端口，例如 `18080`：

```bash
docker rm -f flowable-ui

docker run -d \
  --name flowable-ui \
  --restart unless-stopped \
  -p 18080:8080 \
  flowable/flowable-ui:latest
```

此时访问地址变为：

```text
http://<服务器IP>:18080/flowable-ui
```

---

## 13. 升级方式（离线）

### 13.1 在线机器重新拉取目标版本

```bash
docker pull flowable/flowable-ui:6.8.0
docker save -o flowable-ui-6.8.0.tar flowable/flowable-ui:6.8.0
sha256sum flowable-ui-6.8.0.tar > flowable-ui-6.8.0.tar.sha256
```

### 13.2 离线机器升级

```bash
docker stop flowable-ui
docker rm -f flowable-ui
docker load -i flowable-ui-6.8.0.tar

docker run -d \
  --name flowable-ui \
  --restart unless-stopped \
  -p 8080:8080 \
  flowable/flowable-ui:6.8.0
```

---

## 14. 常见问题排查

### 14.1 页面打不开

检查容器是否运行：

```bash
docker ps | grep flowable-ui
```

检查端口监听：

```bash
ss -lntp | grep 8080
```

检查防火墙配置。

### 14.2 容器启动后退出

查看日志：

```bash
docker logs flowable-ui
```

重点关注：

- 端口冲突
- 内存不足
- 镜像文件损坏
- Java / Spring Boot 启动异常

### 14.3 镜像导入失败

重新校验 tar 包：

```bash
sha256sum -c flowable-ui-latest.tar.sha256
```

如果校验失败，需要重新从在线环境导出并传输。

---

## 15. 推荐的生产建议

1. 固定镜像版本，不建议长期直接使用 `latest`
2. 首次登录后立即修改默认密码
3. 建议通过 Nginx 反向代理统一暴露域名
4. 如对外提供服务，建议配置 HTTPS
5. 保留离线镜像包、校验值和部署文档，方便审计与复现

---

## 16. 一键命令汇总

### 在线机器

```bash
docker pull flowable/flowable-ui:latest
docker save -o flowable-ui-latest.tar flowable/flowable-ui:latest
sha256sum flowable-ui-latest.tar > flowable-ui-latest.tar.sha256
```

### 离线机器

```bash
docker load -i flowable-ui-latest.tar

docker run -d \
  --name flowable-ui \
  --restart unless-stopped \
  -p 8080:8080 \
  flowable/flowable-ui:latest
```

### 访问地址

```text
http://<服务器IP>:8080/flowable-ui
```

### 默认账号密码

```text
admin / test
```
