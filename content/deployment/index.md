# 部署运维

部署采用两阶段演进：模拟盘 / 小资金阶段用**单机 EC2** 跑全栈（成本与延迟优先）；真金白银上量后再迁移到 **ECS 托管方案**（可用性优先）。

---

## 阶段一：单机 EC2（当前方案）

### 区域选择：ap-northeast-1（东京）

Binance 的撮合引擎与 API 服务器部署于 AWS 东京区域，**同区域内访问延迟为个位数毫秒**；原新加坡（ap-southeast-1）方案跨区延迟约 70ms。对 WebSocket 行情采集与下单链路而言这是数量级差距，故区域选定东京。

### 架构

一台 **EC2 t4g.medium**（ARM Graviton，2 vCPU / 4 GB 内存）以 docker compose 跑全栈：

| 组件 | 说明 |
|------|------|
| Redis | 开启 **AOF 持久化**，事件总线 + Stream 数据掉电不丢 |
| TimescaleDB | PostgreSQL + 时序扩展，数据卷落 EBS |
| 全部 quant_trading 服务 | data-service / asset-pool-service / feature-service / strategy-service / risk-service / execution-service / account-service / API gateway 等，同一镜像不同 command |
| Caddy | 反向代理 + 自动 HTTPS，唯一对外入口 |

**网络与访问控制：**

- 实例置于**公有子网**，安全组仅放行 443（无对外服务需求时入站全关）
- **不开放 22 端口**：运维统一走 **SSM Session Manager**（免密钥、免暴露端口、自带会话审计）

### 周边 AWS 服务

| 服务 | 用途 |
|------|------|
| S3 | parquet 归档 + 每日 `pg_dump` 备份；生命周期规则自动转 Glacier 降冷存储成本 |
| ECR | CI（GitHub Actions）构建并推送 **arm64** 镜像 |
| SSM Parameter Store | 密钥管理（SecureString，标准参数在免费层内），替代 Secrets Manager |
| CloudWatch | 实例级告警（CPU / 磁盘 / StatusCheck） |
| AWS Budgets | 账单预警 |
| EIP | 固定出口 IP，绑定 Binance API key 的 IP 白名单 |

### 成本估算

| 项目 | 月费 |
|------|------|
| EC2 t4g.medium（按需） | ~$31 |
| EBS 50 GB gp3 | ~$5 |
| 公网 IPv4（EIP） | ~$3.6 |
| S3 / ECR / 杂项 | ~$5 |
| **总计** | **≈ $45/月** |

签 1 年期 Savings Plan 可在此基础上再降。

### 明确不用的服务及原因

| 服务 | 不用的原因 |
|------|-----------|
| NAT Gateway | 实例在公有子网直接出网，无需 NAT；$35/月固定费纯浪费 |
| ALB | 单机一个 Caddy 反代即可，没有多目标负载均衡需求 |
| ElastiCache / RDS | docker 自建 Redis / TimescaleDB + S3 每日备份足够，托管版溢价无收益 |
| Lambda | 行情采集是 WebSocket 长连接常驻进程，不适用事件驱动短任务模型 |

### 告警与日志

应用级告警（心跳丢失、下单失败、风控拒绝等）通过 **Telegram Bot** 直接推送；**不接 CloudWatch Logs** —— tick 级日志的摄入费按 GB 计费，是隐形账单。容器日志留在本机（docker json-file + max-size 轮转），CloudWatch 仅保留实例级指标告警。

---

## 阶段二：ECS 扩展方案（规划）

> **触发条件**（满足任一时启动迁移）：
>
> 1. 真金白银上量，单机单点故障不可接受
> 2. 某个服务需要独立伸缩（如 data-service 吞吐瓶颈）
> 3. 需要滚动发布 / 不停机部署
>
> **迁移顺序**：RDS（先把数据库迁出单机）→ ECS on EC2 → Fargate + 多 AZ。
>
> **区域**：同样建议从 ap-southeast-1 改为 **ap-northeast-1（东京）**，理由同阶段一。

每个服务为独立 ECS Service（独立 Task Definition），共享一个 ECS Cluster；Fargate 自动管理底层节点，无需手动运维 EC2。

**各服务资源配置建议：**

| 服务 | CPU | Memory | 实例数 | 说明 |
|------|-----|--------|--------|------|
| Data Service | 1024 | 2 GB | 1 | 高吞吐, WebSocket 连接池, 写入 Redis Stream |
| Asset Pool Service | 256 | 512 MB | 1 | 低频, 每 24h 执行一次, 写入 Redis SET |
| BarSourceAdapterService | 1024 | 2 GB | 1 | 从 Redis Stream 消费 tick, bar 聚合 + tick 特征计算 |
| Feature Service | 512 | 1 GB | 1 | |
| Strategy Service | 256 | 512 MB | 1 | 轻量计算 |
| Risk Service | 256 | 512 MB | 1 | 规则检查 |
| Order Service | 256 | 512 MB | 1 | API 调用 |
| Account Service | 256 | 512 MB | 1 | 定时轮询 |
| Monitor Service | 256 | 512 MB | 1 | 心跳检查 |
| API Gateway | 512 | 1 GB | 1-2 | 按访问量扩 |
| **总计** | | | **10-11** | Fargate 按使用计费 |

**月费估算：**

```
ECS Fargate:
  10 tasks × 平均 0.5 vCPU × 1 GB × 730h ≈ $30-45/月

ElastiCache Redis (cache.t3.small):
  ≈ $25/月

RDS TimescaleDB (db.t3.small):
  ≈ $30/月 (含 20GB GP3 存储)

NAT Gateway:
  ≈ $35/月 (固定) + 数据传输费

ECR:
  < $5/月

CloudWatch:
  ≈ $10/月

总计: ≈ $140-160/月
```

网络架构（VPC / 公私子网 / 安全组）、密钥管理（Secrets Manager）、CI/CD 多服务部署与 Terraform 模块结构等完整细节见[架构文档 §三 AWS 部署方案](../architecture/index.md#aws-deployment)。
