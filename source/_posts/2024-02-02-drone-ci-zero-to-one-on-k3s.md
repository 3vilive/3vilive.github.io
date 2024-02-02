---
title: k3s 上从 0 到 1 部署 Drone CI 并应用
tags:
  - Drone
  - k3s
description: Drone 是一个现代化持续集成平台，能让用户基于管道引擎自动化构建、测试、发布应用。
index_img: img/drone-logo-png-dark-512.png
categories:
  - CI
date: 2024-02-02 17:37:10
---


# 在 k3s 上部署 Drone CI

Drone 是一个现代化持续集成平台，能让用户基于管道引擎自动化构建、测试、发布应用。

## 准备

1. 确保你的机器可以在公网被访问，用于接收来自 Github 的 webhooks
2. 一个可以访问的 MySQL 实例
****
### 准备好 Client ID 和 Client secret 和 RPC secret

#### 创建 [Github OAuth application](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app) 

1. Settings -> Developer settings -> OAuth Apps -> Register a new application
2. 填写信息并提交
   - Application name: Drone
   - Homepage URL: https://drone.your-domain.com
   - Authrization callback URL: https://drone.your-domain.com/login
3. 复制 `Client ID`
4. 生成 `Client secret` 并复制保存好（后面不能再复制了）

#### 生成用于 Drone Server 和 Drone Runner 通讯的 `RPC secret`
   
```shell
openssl rand -hex 16
```

### 初始化 MySQL

需要在 MySQL 中创建一个用于存储 drone 数据的数据库，并为 drone 分配一个用户以及设置相应的权限：

```sql
CREATE DATABASE drone;
CREATE USER 'drone'@'%' IDENTIFIED BY '{your-password}';
GRANT SUPER ON *drone*.* TO 'drone'@'%';
FLUSH PRIVILEGES;
SHOW GRANTS FOR 'drone'@'%';
```

注意这里是给 drone 用户设置了超级权限，最好单独启用一个 MySQL 实例。

## Drone Server

### 创建 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: drone-server-cm
  namespace: devops
data:
  TZ: "Asia/Shanghai"
  DRONE_GITHUB_CLIENT_ID: your-github-client-id
  DRONE_GITHUB_CLIENT_SECRET: your-github-client-secret
  DRONE_RPC_SECRET: your-rpc-secret
  DRONE_SERVER_HOST: drone.your-domain.com
  DRONE_SERVER_PROTO: https
  DRONE_DATABASE_DRIVER: mysql
  DRONE_DATABASE_DATASOURCE: drone:your-mysql-password@tcp(your-mysql-host:3306)/drone?parseTime=true
```

### 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drone-server-dep
  namespace: devops
spec:
  selector:
    matchLabels:
      app: drone-server
  replicas: 1
  template:
    metadata:
      labels:
        app: drone-server
    spec:
      containers:
        - name: drone-server
          image: drone/drone:2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: drone-server-cm
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
```

### 创建 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: drone-server-svc
  namespace: devops
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: drone-server
  type: ClusterIP

```

### 创建 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: drone-server-ing
  namespace: devops
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
    - secretName: drone-your-domain-tls
      hosts:
        - drone.your-domain.com
  rules:
    - host: drone.your-domain.com
      http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
              service:
                name: drone-server-svc
                port:
                  number: 80
```

注意这里使用了 `cert-manager` 来配置 `HTTPs` 证书，如果不需要 `HTTPs` 的话，将 `DRONE_SERVER_PROTO` 设置为 `http`，移除这里的 `tls` 和 `annotations` 内容即可。

完成这一步后，你就可以访问页面了：

![drone-welcome-page](drone-welcome-page.png)

### 完成初始化

点击继续使用 Github 登录，登录完成后，操作数据库设置自己为管理员：

```sql 
update drone.users set user_admin = 1 where user_id = 1;
```

## Drone Runner

### 创建 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: drone-runner-cm
  namespace: devops
data:
  TZ: "Asia/Shanghai"
  DRONE_RPC_HOST: drone-server-svc # 直接使用 service 名称
  DRONE_RPC_PROTO: http
  DRONE_RPC_SECRET: your-rpc-secret
  DRONE_RUNNER_CAPACITY: "3"
  DRONE_RUNNER_NAME: "docker-runner"
  DRONE_RESOURCE_REQUEST_CPU: "100" # pipeline 所需的 CPU
  DRONE_RESOURCE_REQUEST_MEMORY: "256Mi" # pipeline 所需的内存
  DRONE_RESOURCE_LIMIT_CPU: "1000" # pipeline 限制 CPU
  DRONE_RESOURCE_LIMIT_MEMORY: "1024Mi" # pipeline 限制内存
  DRONE_NAMESPACE_DEFAULT: devops # 命名空间
```

### 创建 ServiceAccount、Role、RoleBinding

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: drone
  namespace: devops
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: drone
  namespace: devops
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - create
      - delete
  - apiGroups:
      - ""
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - create
      - delete
      - list
      - watch
      - update

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: drone
  namespace: devops
subjects:
  - kind: ServiceAccount
    name: drone
    namespace: devops
roleRef:
  kind: Role
  name: drone
  apiGroup: rbac.authorization.k8s.io
```

### 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drone-runner-dep
  namespace: devops
spec:
  selector:
    matchLabels:
      app: drone-runner
  replicas: 1
  template:
    metadata:
      labels:
        app: drone-runner
    spec:
      serviceAccountName: drone
      containers:
        - name: drone-runner
          image: drone/drone-runner-kube:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: drone-runner-cm
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
```

检查日志：

```
time="2024-02-02T09:11:29Z" level=info msg="starting the server" addr=":3000"
time="2024-02-02T09:11:29Z" level=info msg="successfully pinged the remote server"
time="2024-02-02T09:11:29Z" level=info msg="polling the remote server" capacity=3 endpoint="http://drone-server-svc" kind=pipeline type=kubernetes
```

# 应用

## 编写 .drone.yml

在需要引用 drone 的仓库，创建 .drone.yml 文件，用于描述 pipeline 的步骤，这里我的仓库是一个 Rust 项目，直接使用官方的例子：

```yaml
---
kind: pipeline
type: kubernetes
name: default

steps:
  - name: test
    image: rust:1.71
    commands:
      - cargo build --verbose --all
      - cargo test --verbose --all
```

## 激活仓库

在 drone 的 Web 页面中的仓库列表，找到对应的仓库：

![repo-list](repo-list.png)

点击激活并保存：

![activate-repo](activate-repo.png)

这时候提交即可触发构建：

![repo-builds](repo-builds.png)

# 参考文档

- [Drone CI/CD Documentation](https://docs.drone.io/)
  