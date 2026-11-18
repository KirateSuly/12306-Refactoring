# 12306 铁路购票系统（重构学习版）

> 本项目是对开源 12306 铁路购票项目的**个人重构与学习实践**。目标不是重复造轮子，而是通过把系统完整跑起来、阅读核心链路、动手改造并压测，理解一个高并发票务系统在缓存、消息、分库分表、幂等和一致性上的真实取舍。
>
> 代码与文档仅用于个人学习和技术交流，不用于商业用途。

## 项目简介

12306 是与日常出行紧密相关的关键系统，业务覆盖会员、乘车人、车票查询、下单、选座、订单、支付和网关等模块，天然带有「高并发 + 强一致 + 复杂业务规则」的特点，非常适合作为分布式系统的学习样本。

本仓库基于开源项目 `nageoffer/12306` 进行学习性重构，主要工作包括：

- **环境重构**：补充 `docker/` 编排，一键启动 Redis、RocketMQ、Nacos；保留宿主机 MySQL 的运行方式。
- **数据初始化**：提供 `scripts/` 下的建库与导入脚本，解决原项目 SQL 不含 `CREATE DATABASE`、中文种子数据易乱码的问题。
- **问题修复**：修复本地运行中遇到的 RocketMQ 命名卷权限、Nacos JDK 8u372 + cgroup v2 崩溃、MySQL 客户端编码等问题。
- **代码清理**：移除源码中与学习无关的推广性注释，统一代码风格。
- **文档沉淀**：补充环境搭建记录、项目体检报告与分阶段学习计划，详见 [docs/](docs/)。

## 技术栈

| 组件 | 版本 / 选型 | 用途 |
|---|---|---|
| JDK | 17 | 基础运行环境 |
| Spring Boot | 3.0.7 | 应用框架 |
| Spring Cloud | 2022.0.3 | 微服务基础设施 |
| Spring Cloud Alibaba | 2022.0.0.0-RC2 | Nacos 服务注册与配置 |
| ShardingSphere | 5.3.2 | 分库分表（JDBC 模式） |
| MyBatis-Plus | 3.5.3.1 | ORM |
| Redis / Redisson | 7.x / 3.21.3 | 缓存、分布式锁、令牌桶 |
| RocketMQ | 4.9.7 | 异步下单、延迟关单、支付回调 |
| MySQL | 8.x | 业务数据存储 |
| Sentinel | - | 限流与降级 |
| Caffeine | - | 本地缓存 |
| hippo4j / crane4j | 1.5.0 / - | 动态线程池、枚举填充 |
| Fastjson2 / Hutool | - | 序列化与工具库 |

## 项目结构

```
12306
├── dependencies/      # Maven 依赖版本统一管理
├── frameworks/        # 自研基础组件（缓存、幂等、分布式 ID、责任链、日志、Web 约定等）
├── services/          # 业务服务
│   ├── aggregation-service/  # 聚合服务（单 JVM 运行 user/ticket/order/pay）
│   ├── gateway-service/      # Spring Cloud Gateway 网关
│   ├── user-service/         # 会员、登录、乘车人
│   ├── ticket-service/       # 余票查询、下单、选座
│   ├── order-service/        # 订单创建、状态流转、延迟关单
│   └── pay-service/          # 支付单、退款、支付回调
├── tests/             # 集成测试
├── console-vue/       # Vue 3 控制台前端（可选）
├── resources/         # 数据库结构（db）与种子数据（data）
├── scripts/           # 建库/导数据脚本
├── docker/            # 本地中间件编排
└── docs/              # 环境记录、体检报告、学习计划
```

## 两种运行模式

业务代码只有一份，两种模式共用：

- **聚合服务版**：`aggregation-service` 通过 `scanBasePackages` 把 user / ticket / order / pay 扫进同一个 JVM，启动简单，适合本地联调和压测。
- **微服务版**：user / ticket / order / pay 分别独立部署，通过 Nacos 注册发现，配合网关访问，适合学习微服务拆分与治理。

## 核心功能与设计亮点

### 业务功能

- 用户注册 / 登录（用户名、手机号、邮箱多方式）
- 乘客管理与乘车人信息维护
- 车次、余票查询
- 提交订单、在线选座
- 订单状态流转与超时自动关单
- 支付、退款与支付回调

### 技术亮点

| 亮点 | 实现位置 | 说明 |
|---|---|---|
| 余票令牌桶 | `ticket-service/.../tokenbucket`、`lua/ticket_availability_token_bucket*.lua` | 用 Redis + Lua 原子扣减与回滚余票，避免超卖 |
| 选座与座位分配 | `ticket-service/.../handler/ticket/base` | 相邻座位优先 → 同车厢 → 跨车厢三级降级 |
| 责任链 + 模板方法 | `ticket-service/.../filter/purchase`、`AbstractTrainPurchaseTicketTemplate` | 将参数校验、余票校验、库存扣减、建单等步骤解耦 |
| 分库分表 | `shardingsphere-config*.yaml` | 用户按 username 哈希分片，订单使用 `user_id + order_sn` 复合分片键 |
| 接口与消息幂等 | `frameworks/idempotent` | 基于 Redis + Lua 的幂等令牌，覆盖 REST 与 MQ 两种场景 |
| 延迟关单 | `DelayCloseOrderConsumer` + RocketMQ 延迟消息 | 未支付订单到期自动取消 |
| 分布式 ID | `frameworks/distributedid` | 雪花算法 + Redis 分配 workId |
| 缓存防穿透 | `frameworks/cache` | 布隆过滤器 + 空值缓存 |
| 限流降级 | Sentinel | 针对查询与下单接口配置 QPS 阈值 |

### 值得关注的业务规则

- **区间售票扣减**：一张北京南 → 南京南的车票，会同时扣减北京南 → 济南西、北京南 → 南京南、济南西 → 南京南三段库存，任一段不足都无法成交。
- **动态放票**：放票初期优先保证大站之间的票量，小站之间只开放少量票，临近发车再逐步释放。
- **选座降级**：多人购票时优先分配相邻座位，其次同车厢不相邻座位，最后才跨车厢分配。

## 本地启动

### 1. 环境要求

- JDK 17、Maven 3.8+
- Docker Desktop（Windows 建议开启 WSL2 后端）
- MySQL 8.x（默认使用宿主机 3306；也可用 compose 中的 `mysql` profile）

### 2. 启动中间件

```powershell
docker-compose -f docker/docker-compose.yml up -d
```

说明：

- 包含 Redis（密码 `123456`）、RocketMQ NameServer/Broker、Nacos（standalone）。
- 如果宿主机 3306 已被本机 MySQL 占用，不要再启动 compose 的 MySQL；确有需要时使用 `--profile mysql`，并先停掉本机 MySQL80 服务。
- 容器启动参数与踩坑修复都写在 [docs/02-environment.md](docs/02-environment.md)。

### 3. 初始化数据库

聚合版（单库 `12306`，约 140 张表）：

```powershell
.\scripts\init-springboot-db.cmd
```

微服务版（7 个库）：

```powershell
.\scripts\init-springcloud-db.cmd
```

也可以参考 `scripts/init-databases-springboot.sql` 手工执行。注意：

- 导入时必须显式指定 `--default-character-set=utf8mb4`，否则 Windows 下 mysql 客户端按 GBK 解析会导致中文乱码甚至一行数据都导不进去。
- 脚本顶部的 `DB_PASS` 以及各服务 `shardingsphere-config*.yaml`、`application.yaml` 中的数据库账号密码，需要改成自己的配置。

### 4. 构建

```powershell
mvn -B -DskipTests -Dspotless.apply.skip=true clean package
```

`-Dspotless.apply.skip=true` 必须保留，否则 Spotless 会改写全仓库的 license 头。

### 5. 启动服务

聚合版：

1. 启动 `AggregationServiceApplication`（9005）
2. 启动 `GatewayServiceApplication`（9000，统一入口）

微服务版：分别启动 user（9001）、ticket（9002）、order（9003）、pay（9004）和 gateway（9000），并在 Nacos 控制台确认实例注册成功。

### 6. 验证

```powershell
curl http://127.0.0.1:9005/actuator/health
curl http://127.0.0.1:8848/nacos/v1/console/health/readiness
docker exec 12306-redis redis-cli -a 123456 ping
```

| 服务 / 组件 | 端口 |
|---|---|
| gateway | 9000 |
| user | 9001 |
| ticket | 9002 |
| order | 9003 |
| pay | 9004 |
| aggregation | 9005 |
| MySQL | 3306 |
| Redis | 6379 |
| Nacos | 8848 / 9848 / 9849 |
| RocketMQ NameServer | 9876 |
| RocketMQ Broker | 10909 / 10911 / 10912 |

前端（可选）：

```powershell
cd console-vue
npm install
npm run serve
```

## 学习与文档

- [环境搭建记录](docs/02-environment.md)：中间件拓扑、已修复的坑、数据库与端口说明。
- [项目体检报告](docs/00-health-check.md)：代码规模、技术栈版本、亮点分布与风险清单。
- [学习计划](docs/plan.md)：按「跑通 → 读链路 → 制造故障 → 改造压测 → 微服务」分阶段推进。

建议的学习顺序：

1. 先跑通「登录 → 查余票 → 下单」主链路；
2. 精读 `ticket-service` 的余票缓存、令牌桶、选座和购票模板方法；
3. 逐个验证超卖、重复消费、分片路由、限流降级等场景；
4. 最后切到微服务版，记录服务治理与分布式事务中的真实问题。

## 已知限制

- 支付使用示例沙箱 key 和临时回调域名，无法真实扣款，支付回调链路需通过接口模拟。
- 动态线程池 `hippo4j` 的 Nacos data-id 未随仓库提供，启动日志出现相关告警可忽略；若影响启动需关闭动态线程池。
- Spring Cloud Alibaba 为 RC 版本，升级 Nacos 或 Spring Boot 时需注意兼容性。
- `console-vue` 仍处于开发阶段，部分业务与交互尚未完善，可先用接口工具调试后端。

## 许可证

本项目遵循 [Apache License 2.0](LICENSE)。原项目版权归其作者所有，本仓库仅用于学习与研究。
