# 001 - 多维动态加权路由 (Dynamic Weighted Routing)

## 功能概述 (Feature Overview)

### 功能名称
**PocketRouter - 多维动态加权评分路由系统**

### 功能描述

#### 背景
当前 NewAPI 的路由策略是**静态**的：基于手动配置的优先级和权重进行渠道选择。在多供应商、多渠道的 LLM 网关场景下，各渠道的**价格、可用性、延迟**都是实时变化的。静态配置无法适应这种动态环境，导致：
- 无法自动规避高延迟或故障渠道
- 无法根据实时成本优化支出
- 缺乏反馈闭环，路由决策与实际表现脱节

#### 目标用户
- 个人开发者：需要低成本、高可用的 LLM API 聚合服务
- 中小企业：需要管理多个 AI 供应商，优化成本与体验的平衡
- AI 应用开发团队：需要稳定、可观测的 LLM 网关

#### 核心价值
构建一个**基于反馈控制环 (Feedback Control Loop) 的动态路由系统**，在收益与体验之间找到最佳平衡点。

### 业务目标
1. **降低成本**：自动选择性价比最优的渠道
2. **提升体验**：优先选择低延迟、高可用的渠道
3. **增强稳定性**：自动熔断故障渠道，支持智能降级
4. **可观测性**：提供实时的渠道健康度与路由决策可视化

---

## 用户故事 (User Stories)

### US-001: 基于延迟的智能路由
**As a** API 调用者  
**I want to** 系统自动选择响应最快的渠道  
**So that** 我的用户能获得最佳的首字响应体验 (TTFT)

**Acceptance Criteria:**
- [ ] 系统记录每个渠道的 TTFT (Time To First Token) 和 TPS (Tokens Per Second)
- [ ] 使用指数移动平均 (EMA) 平滑延迟数据，避免抖动
- [ ] 路由决策时，延迟因子参与评分计算
- [ ] 支持"极速模式"，调高延迟权重

### US-002: 基于可用性的自动降级
**As a** 系统管理员  
**I want to** 故障渠道自动被熔断和降级  
**So that** 用户请求不会因单点故障而失败

**Acceptance Criteria:**
- [ ] 系统记录每个渠道的成功率 (基于滑动窗口)
- [ ] 连续失败 N 次后触发熔断，暂停该渠道
- [ ] 熔断期间发送探测流量，检测渠道恢复
- [ ] 渠道恢复后自动解除熔断
- [ ] 支持手动强制启用/禁用渠道

### US-003: 基于成本的经济路由
**As a** 成本敏感的用户  
**I want to** 系统优先选择最便宜的渠道  
**So that** 我能在保证可用性的前提下最小化支出

**Acceptance Criteria:**
- [ ] 系统维护每个渠道的 Token 价格配置
- [ ] 路由决策时，成本因子参与评分计算
- [ ] 支持"经济模式"，调高成本权重
- [ ] 提供成本分析报表

### US-004: 多维度综合评分
**As a** 系统管理员  
**I want to** 自定义各维度的权重配比  
**So that** 我能根据业务场景调整路由策略

**Acceptance Criteria:**
- [ ] 支持配置 Cost、Latency、Availability 三个维度的权重
- [ ] 提供预设模式：经济模式、极速模式、高可用模式
- [ ] 支持按 Group 或 Model 粒度配置不同策略
- [ ] 权重变更实时生效，无需重启

### US-005: 概率路由与探索
**As a** 系统管理员  
**I want to** 避免流量全部打到最优渠道  
**So that** 防止惊群效应，并持续探测其他渠道状态

**Acceptance Criteria:**
- [ ] 使用 Softmax 温度采样替代 Greedy 选择
- [ ] 高分渠道获得更高的选中概率，但不是 100%
- [ ] 低分渠道偶尔获得探测流量
- [ ] 支持配置温度参数，控制探索程度

### US-006: 实时监控与可视化
**As a** 系统管理员  
**I want to** 在管理后台看到渠道健康度和路由决策  
**So that** 我能及时发现问题并优化配置

**Acceptance Criteria:**
- [ ] Dashboard 展示各渠道的实时评分
- [ ] 展示各渠道的延迟 P50/P95/P99
- [ ] 展示各渠道的成功率和错误分布
- [ ] 展示路由决策的分布统计
- [ ] 支持时间范围筛选

---

## 功能需求 (Functional Requirements)

### FR-001: 指标采集模块 (Metric Collector)

#### 描述
在每次请求结束后，异步采集并存储渠道的性能指标。

#### 输入
- Channel ID
- 请求开始时间
- 首字响应时间 (TTFT)
- 请求结束时间
- 输出 Token 数量
- HTTP 状态码
- 错误信息 (如有)

#### 处理逻辑
1. 计算 TTFT = 首字响应时间 - 请求开始时间
2. 计算 TPS = 输出 Token 数 / (结束时间 - 首字响应时间)
3. 更新该渠道的 EMA 延迟值：`EMA_new = α * current + (1-α) * EMA_old`
4. 更新滑动窗口内的成功/失败计数
5. 将数据写入 DuckDB (异步批量)

#### 输出
- 更新内存中的 `GlobalRoutingTable`
- 持久化到 DuckDB 日志表

#### 代码位置
- 埋点位置：`relay/compatible_handler.go` 的 `TextHelper` 函数结束处
- 新增文件：`service/metric_collector.go`

### FR-002: 评分引擎 (Scoring Engine)

#### 描述
定期计算每个渠道的综合评分，并更新到内存路由表。

#### 评分公式
```
Score = w_cost * (1/Cost) + w_latency * (1/Latency) + w_availability * Availability
```

其中：
- `w_cost`, `w_latency`, `w_availability` 为可配置权重
- `Cost` 为 Token 单价 (归一化)
- `Latency` 为 EMA 延迟 (归一化)
- `Availability` 为滑动窗口成功率

#### 处理逻辑
1. 每 N 秒 (可配置，默认 10s) 触发一次计算
2. 从 DuckDB 查询最近 M 分钟的聚合数据
3. 结合内存中的实时数据，计算综合评分
4. 按 Model 维度，将渠道评分写入 Redis ZSET 或内存 Map

#### 代码位置
- 新增文件：`service/scoring_engine.go`

### FR-003: 智能路由选择器 (Smart Router)

#### 描述
替换现有的 `GetRandomSatisfiedChannel` 逻辑，实现基于评分的概率选择。

#### 输入
- Model ID
- Group ID
- 路由策略配置

#### 处理逻辑
1. 获取该 Model 下所有可用渠道的评分列表
2. 过滤掉被熔断的渠道
3. 应用 Softmax 函数计算选择概率：`P(i) = exp(Score_i / T) / Σ exp(Score_j / T)`
4. 按概率随机选择一个渠道
5. 记录本次路由决策 (用于分析)

#### 输出
- 选中的 Channel ID

#### 代码位置
- 修改文件：`model/channel_cache.go` 的 `GetRandomSatisfiedChannel` 函数
- 新增文件：`service/smart_router.go`

### FR-004: 熔断器 (Circuit Breaker)

#### 描述
实现渠道级别的熔断机制，防止故障扩散。

#### 状态机
```
CLOSED (正常) → OPEN (熔断) → HALF_OPEN (探测)
     ↑                              ↓
     ←──────────── 成功 ─────────────
```

#### 触发条件
- **CLOSED → OPEN**：滑动窗口内错误率 > 阈值 (默认 50%)，且请求数 > 最小样本数
- **OPEN → HALF_OPEN**：熔断时间到期 (默认 30s)
- **HALF_OPEN → CLOSED**：探测请求成功
- **HALF_OPEN → OPEN**：探测请求失败

#### 代码位置
- 新增文件：`service/circuit_breaker.go`

### FR-005: DuckDB 日志存储

#### 描述
使用 DuckDB 作为高性能日志分析引擎，替代 SQLite 存储大量请求日志。

#### 表结构
```sql
CREATE TABLE request_logs (
    id              BIGINT PRIMARY KEY,
    timestamp       TIMESTAMP,
    channel_id      INTEGER,
    model           VARCHAR,
    ttft_ms         INTEGER,
    tps             FLOAT,
    total_tokens    INTEGER,
    status_code     INTEGER,
    error_type      VARCHAR,
    user_id         INTEGER,
    token_id        INTEGER
);

CREATE INDEX idx_logs_channel_time ON request_logs(channel_id, timestamp);
CREATE INDEX idx_logs_model_time ON request_logs(model, timestamp);
```

#### 写入策略
- 使用 Go Channel 缓冲日志
- 每 1000 条或每 1 秒批量写入
- 使用 DuckDB Appender API 提升性能

#### 代码位置
- 新增文件：`store/duckdb.go`

### FR-006: 配置管理

#### 描述
支持动态配置路由策略参数。

#### 配置项
| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `routing.weight.cost` | float | 0.3 | 成本权重 |
| `routing.weight.latency` | float | 0.4 | 延迟权重 |
| `routing.weight.availability` | float | 0.3 | 可用性权重 |
| `routing.softmax.temperature` | float | 1.0 | Softmax 温度 |
| `routing.ema.alpha` | float | 0.3 | EMA 平滑系数 |
| `routing.window.size` | int | 60 | 滑动窗口大小 (秒) |
| `routing.scoring.interval` | int | 10 | 评分计算间隔 (秒) |
| `circuit_breaker.threshold` | float | 0.5 | 熔断错误率阈值 |
| `circuit_breaker.min_samples` | int | 10 | 熔断最小样本数 |
| `circuit_breaker.timeout` | int | 30 | 熔断超时时间 (秒) |

#### 代码位置
- 修改文件：`setting/routing.go` (新增)

---

## 非功能需求 (Non-Functional Requirements)

### NFR-001: 性能要求
- **QPS 目标**：单机支持 1000 QPS
- **并发连接**：支持 10,000 并发长连接 (SSE 流式)
- **路由决策延迟**：< 1ms (纯内存操作)
- **日志写入**：异步批量，不阻塞主请求

### NFR-002: 资源占用
- **内存**：< 2GB (不含 DuckDB 缓存)
- **CPU**：8 核利用率 < 50%
- **磁盘**：DuckDB 日志按天滚动，保留 7 天

### NFR-003: 可用性
- **单点故障**：路由状态丢失后自动重建 (从 DuckDB 恢复)
- **优雅降级**：评分引擎故障时回退到静态权重

### NFR-004: 可观测性
- **指标暴露**：支持 Prometheus 格式
- **日志格式**：结构化 JSON 日志
- **追踪**：支持 Request ID 贯穿全链路

### NFR-005: 兼容性
- **API 兼容**：完全兼容现有 NewAPI 接口
- **数据兼容**：不破坏现有数据库结构
- **配置兼容**：新功能默认关闭，可渐进式启用

---

## 约束与假设 (Constraints & Assumptions)

### 技术约束
1. **语言**：Go 1.21+
2. **数据库**：保留 SQLite/MySQL 用于业务数据，新增 DuckDB 用于日志分析
3. **部署**：单机部署优先，暂不考虑分布式状态同步
4. **依赖**：最小化外部依赖，避免引入 Redis

### 业务假设
1. 渠道配置 (API Key、BaseURL) 变更频率低，可缓存
2. 用户 Token 验证可缓存，减少数据库压力
3. 日志分析可接受秒级延迟

### 风险与缓解
| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| DuckDB 写入阻塞 | 请求延迟上升 | 异步写入 + 丢弃策略 |
| 内存泄漏 | OOM | 定期清理过期数据 + 监控告警 |
| 评分计算错误 | 路由失效 | 回退到静态权重 + 单元测试 |

---

## 术语表 (Glossary)

| 术语 | 定义 |
|------|------|
| **TTFT** | Time To First Token，首字响应时间 |
| **TPS** | Tokens Per Second，生成速度 |
| **EMA** | Exponential Moving Average，指数移动平均 |
| **Softmax** | 将评分转换为概率分布的函数 |
| **熔断 (Circuit Breaker)** | 当错误率过高时暂停使用该渠道 |
| **滑动窗口** | 只统计最近 N 秒内的数据 |
| **探测流量** | 发送少量请求测试故障渠道是否恢复 |

---

## 实施路线图 (Implementation Roadmap)

### Phase 1: 基础指标采集 (Week 1-2)
- [ ] 引入 go-duckdb 依赖
- [ ] 实现日志表结构和批量写入
- [ ] 在 `relay/compatible_handler.go` 埋点采集 TTFT/TPS
- [ ] 验证日志写入性能

### Phase 2: 评分引擎 (Week 3)
- [ ] 实现内存路由表 (`GlobalRoutingTable`)
- [ ] 实现 EMA 延迟计算
- [ ] 实现评分公式
- [ ] 定时任务更新评分

### Phase 3: 智能路由 (Week 4)
- [ ] 修改 `GetRandomSatisfiedChannel` 接入评分
- [ ] 实现 Softmax 概率选择
- [ ] 添加路由策略配置
- [ ] 单元测试

### Phase 4: 熔断器 (Week 5)
- [ ] 实现熔断状态机
- [ ] 集成到路由选择流程
- [ ] 添加熔断配置

### Phase 5: 可视化 (Week 6)
- [ ] 后台 API 返回渠道评分
- [ ] 前端 Dashboard 展示
- [ ] Prometheus 指标暴露

---

## 验收清单 (Review & Acceptance Checklist)

### 功能验收
- [ ] 延迟低的渠道获得更高选中率
- [ ] 故障渠道被自动熔断
- [ ] 熔断渠道恢复后自动解除
- [ ] 权重配置变更实时生效
- [ ] 日志正确写入 DuckDB

### 性能验收
- [ ] 1000 QPS 下 P99 延迟 < 100ms (不含上游耗时)
- [ ] 内存占用 < 2GB
- [ ] CPU 利用率 < 50%

### 兼容性验收
- [ ] 现有 API 调用无需修改
- [ ] 现有渠道配置无需迁移
- [ ] 可通过配置关闭新功能

---

## 附录

### A. 数据结构定义

```go
// ChannelScore 渠道评分数据
type ChannelScore struct {
    ChannelID   int       `json:"channel_id"`
    LatencyEMA  float64   `json:"latency_ema"`   // 延迟 EMA (ms)
    TPSEMA      float64   `json:"tps_ema"`       // TPS EMA
    SuccessRate float64   `json:"success_rate"`  // 成功率
    CostFactor  float64   `json:"cost_factor"`   // 成本因子
    Score       float64   `json:"score"`         // 综合评分
    UpdatedAt   time.Time `json:"updated_at"`
}

// CircuitState 熔断状态
type CircuitState int

const (
    CircuitClosed   CircuitState = iota // 正常
    CircuitOpen                          // 熔断
    CircuitHalfOpen                      // 探测
)

// RoutingConfig 路由配置
type RoutingConfig struct {
    WeightCost         float64 `json:"weight_cost"`
    WeightLatency      float64 `json:"weight_latency"`
    WeightAvailability float64 `json:"weight_availability"`
    SoftmaxTemperature float64 `json:"softmax_temperature"`
    EMAAlpha           float64 `json:"ema_alpha"`
    WindowSize         int     `json:"window_size"`
    ScoringInterval    int     `json:"scoring_interval"`
}
```

### B. 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Request                          │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Auth Middleware (Cache)                       │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Smart Router                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Score Table │  │Circuit Break│  │  Softmax    │              │
│  │  (Memory)   │  │   (Memory)  │  │  Sampler    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Protocol Adaptor                              │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │
│  │ OpenAI │ │ Claude │ │ Gemini │ │  Ali   │ │  ...   │        │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘        │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Upstream Providers                            │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Metric Collector                              │
│  ┌─────────────┐           ┌─────────────┐                      │
│  │  Log Buffer │ ────────► │   DuckDB    │                      │
│  │  (Channel)  │  Batch    │  (Parquet)  │                      │
│  └─────────────┘           └─────────────┘                      │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Scoring Engine (Ticker)                       │
│  ┌─────────────┐           ┌─────────────┐                      │
│  │ DuckDB SQL  │ ────────► │ Score Table │                      │
│  │  Aggregate  │           │  (Memory)   │                      │
│  └─────────────┘           └─────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

### C. 关键代码修改点

| 文件 | 修改类型 | 说明 |
|------|----------|------|
| `relay/compatible_handler.go` | 修改 | 添加 TTFT/TPS 采集埋点 |
| `model/channel_cache.go` | 修改 | 替换 `GetRandomSatisfiedChannel` 逻辑 |
| `service/metric_collector.go` | 新增 | 指标采集服务 |
| `service/scoring_engine.go` | 新增 | 评分计算服务 |
| `service/smart_router.go` | 新增 | 智能路由选择器 |
| `service/circuit_breaker.go` | 新增 | 熔断器实现 |
| `store/duckdb.go` | 新增 | DuckDB 封装 |
| `setting/routing.go` | 新增 | 路由配置管理 |
| `controller/routing.go` | 新增 | 路由分析 API |
| `web/src/pages/Routing/` | 新增 | 前端 Dashboard |
