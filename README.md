# Valyria Backend

Valyria 后端服务：统一提供认证鉴权、多集群 Kubernetes 管理、发布流水线（Dragon）、仪表盘与审计等能力。

## 技术栈

- **语言**: Go 1.24+
- **Web 框架**: Gin
- **ORM**: GORM
- **数据库**: MySQL
- **缓存**: Redis
- **认证**: JWT（Access Token + Refresh Token）
- **Kubernetes**: client-go，支持多集群
- **流水线**: Tekton Pipeline / Task / TaskRun

## 项目结构概览

```
cmd/server/                 # 程序入口
config/                     # 配置文件（如 config.yaml）
internal/
  apps/
    authn/                  # 认证与权限（用户、角色、菜单、权限）
    audit/                  # 审计（登录审计、操作审计）
    dashboard/              # 仪表盘（集群/流水线汇总、趋势、最近发布等）
    dragon/                 # 发布平台（项目、环境、流水线、发布、审批）
    kubernetes/             # 多集群 kubernetes 管理（集群、节点、命名空间、工作负载等）
  core/                     # 配置、数据库、日志、分页等基础设施
  pkg/
    di/                     # 依赖注入与各模块 Provider
    encryption/             # 敏感信息加密
  routes/                   # 路由注册
```

## 实现功能

### 1. 认证与权限（`/api/v1/authn`）

- 登录、刷新 Token、修改密码
- 用户、角色、菜单、权限的 CRUD
- 用户关联角色/菜单、角色关联权限
- 按用户返回前端路由（`/me/routes`）

### 2. 审计（`/api/v1/audit`）

- 登录审计日志（`/audit/auth-logs`）
- 操作审计日志（`/audit/logs`）

### 3. 仪表盘（`/api/v1/dashboard`）

- **总览**: 集群数、节点数、命名空间数、Pod 数；今日发布总数/成功/失败/回滚（resources 预留）
- **流水线趋势**: `/dashboard/pipelines/trend?range=today|7d|30d`，按小时或按天统计成功/失败/回滚
- **最近发布**: `/dashboard/pipelines/recent?env=prod&limit=10`，按环境取每个 pipeline 最近一次发布
- **项目统计**: `/dashboard/pipelines/projects`，按项目统计发布次数、pipeline 数、成功/失败/回滚次数

### 4. Kubernetes 多集群管理（`/api/v1/kubernetes`）

```bash
$ kubectl create serviceaccount admin-user -n kube-system
$ kubectl create clusterrolebinding admin-user-binding --clusterrole=cluster-admin --serviceaccount=kube-system:admin-user
$ kubectl -n kube-system create token admin-user --duration=87600h
```

- 集群管理：增删改查、Token 更新
- 权限：K8s 资源权限的同步与 CRUD（含批量）
- 按集群管理：节点、命名空间、Deployment、StatefulSet、DaemonSet、ReplicaSet、Pod、Service、ConfigMap、Secret、HPA、Ingress、RBAC（Role/RoleBinding/ClusterRole/ClusterRoleBinding）、ServiceAccount、StorageClass、PV/PVC、Event 等
- Tekton：Task、TaskRun、Pipeline、PipelineRun
- 后台 Worker：Kubernetes RBAC 权限同步、HPA 扩缩容历史同步

### 5. 发布平台 Dragon（`/api/v1/dragon`）

- 项目、环境、流水线、流水线 ACL 的 CRUD
- 发布（Release）：创建发布、回滚、查看日志、删除
- 审批：发布评审（Review）、评审阶段配置（Review Stage）
- 凭证与 GitLab 集成：凭证管理、GitLab 配置
- 报告：`/reports/summary`

### 6. Prometheus 配置管理（`/api/v1/prometheus`）

- 分组（Group）、目标组（Target Group）、目标（Target）
- 记录（Record）、规则（Rule）的 CRUD  
（与 ConfigMap 等联动，用于 Prometheus 监控配置）

## 配置与运行

### 配置

- 默认配置文件：`config/config.yaml`
- 启动参数：`-c config/config.yaml` 指定配置文件
- 主要配置项：服务端口、JWT、数据库、Redis、日志、Kubernetes 超时/QPS、Dragon Kustomize、权限 Worker 等

### 运行

```bash
# 依赖：MySQL、Redis 已就绪，并已初始化数据库
go run ./cmd/server -c config/config.yaml
```

默认监听 `http://localhost:8080`，API 前缀为 `/api/v1`。登录、刷新 Token、健康检查、指标等路径可在配置中加入白名单免鉴权。