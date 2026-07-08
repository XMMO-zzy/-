# 06 数据库设计（Web联调版）

> 目标：硬件侧由外部后端写库；本项目只负责 Web 前后端，直接读数据库并做业务处理。

## 1. 设计原则

- 外部数据原样落库，不在入库时做强耦合字段转换。
- Web 业务层使用标准化表，避免前端依赖原始 JSON 结构。
- 保留可追溯能力：任一告警可回溯到原始 `raw_json/raw_hex`。
- 兼容增量联调：先跑通查询与展示，再逐步完善规则计算。

## 2. 分层模型

### 2.1 原始接入层（外部后端写）

`annotated_data` / `annotated_data_raw`

- 保存外部后端原始写入内容
- 字段与外部口径保持一致，不做业务含义扩展
- 说明：当前代码默认读取 `annotated_data`（兼容历史库）；新部署建议统一到 `annotated_data_raw`

### 2.2 标准业务层（本项目维护）

- `telemetry_events`：标准化遥测事件
- `alerts`：告警主表
- `alert_events`：告警事件流（创建/确认/恢复/关闭）
- `prediction_records`：预测查询结果（RUL/风险趋势）
- `llm_analysis`：大模型分析记录
- `maintenance_records`：维护记录

### 2.3 知识库层（MySQL FULLTEXT）

- `kb_documents`：知识文档主表
- `kb_chunks`：文档切片表（`FULLTEXT(content)`）

### 2.4 主数据层

- `devices`：设备主档
- `edge_nodes`：节点主档（可选）

## 3. 核心表结构

### 3.1 原始接入表：`annotated_data` / `annotated_data_raw`

- `id` bigint PK
- `device_id` varchar(64)
- `timestamp` bigint
- `raw_json` longtext
- `raw_hex` longtext
- `diagnosis_result` longtext
- `created_at` datetime

索引：
- `(device_id, timestamp)`
- `(created_at)`

### 3.2 标准事件表：`telemetry_events`

- `id` bigint PK
- `raw_id` bigint FK -> `annotated_data_raw.id`
- `device_id` varchar(64)
- `event_ts` bigint
- `fault_class` varchar(32)  // normal/bearing_fault/winding_fault/rotor_fault/overload/unknown
- `confidence` int
- `health_score` int
- `vib_rms` double
- `temperature` double
- `current` double
- `need_deep_confirm` tinyint
- `created_at` datetime

索引：
- `(device_id, event_ts)`
- `(need_deep_confirm, event_ts)`
- `(fault_class, event_ts)`

### 3.3 告警表：`alerts`

- `id` bigint PK
- `device_id` varchar(64)
- `node_id` int
- `raw_id` bigint nullable
- `event_id` bigint nullable
- `alert_level` tinyint  // 1~4
- `alert_source` varchar(16)  // threshold/edge/deep/llm/predictive
- `fault_class` varchar(32)
- `confidence` int
- `health_score` int
- `temperature` double
- `current` double
- `vib_rms` double
- `message` text
- `need_deep_confirm` tinyint
- `status` varchar(32)  // pending/handled/ignored
- `handled_at` datetime nullable
- `handler` varchar(128) nullable
- `notes` text nullable
- `created_at` datetime

索引：
- `(device_id, created_at)`
- `(status, created_at)`
- `(alert_level, created_at)`

### 3.4 告警事件表：`alert_events`

- `id` bigint PK
- `alert_id` bigint nullable
- `device_id` varchar(64)
- `event_type` varchar(32)  // created/acknowledged/resolved/escalated/closed
- `event_source` varchar(32)  // rule/llm/operator
- `detail` text
- `created_at` datetime

### 3.5 预测表：`prediction_records`

- `id` bigint PK
- `device_id` varchar(64)
- `timestamp` bigint
- `risk_level` varchar(16)
- `risk_score` double
- `remaining_life_days` double
- `trend` varchar(16)
- `recommendation` text
- `source_event_id` bigint nullable
- `created_at` datetime

### 3.6 知识库表：`kb_documents` / `kb_chunks`

- `kb_documents`：标题、来源、标签、正文
- `kb_chunks`：切片内容、切片序号、`FULLTEXT(content)`

## 4. 建议读写流程

1. 外部后端写 `annotated_data`（当前兼容）或 `annotated_data_raw`（目标）
2. 本项目后端轮询增量 `id`，执行预处理（清洗/特征提取），写 `telemetry_events`
3. 预测引擎基于事件生成 `prediction_records`
4. 告警路由生成 `alerts`，同时写 `alert_events` 并通过 WebSocket 推送
5. 知识库检索从 `kb_chunks` 召回上下文，辅助大模型分析
6. 前端统一读 `telemetry_events + alerts + prediction_records`，订阅事件流，减少轮询

## 5. 联调阶段简化策略

- 第一阶段：先只用原始接入表（`annotated_data`/`annotated_data_raw`）+ 解析后 API 返回（不强制落 `telemetry_events`）
- 第二阶段：补充标准化落表，前端切换到标准接口
- 第三阶段：加物化视图/定时聚合，提升报表性能

## 6. 开发日志（2026-05-05）

- 知识库新增 10 篇电机运维文档，当前 `kb_documents` 共 11 篇，`kb_chunks` 共 11 条。
- 修复 MySQL 中文知识检索问题：`kb_chunks.content` 的 FULLTEXT 索引改为 `WITH PARSER ngram`。
- 新增 `kb_documents.content` 的 ngram FULLTEXT 索引，便于后续按主文档内容扩展检索口径。
- 实测查询（`轴承 振动 温度` / `转子 断条 电流` / `告警 WebSocket 事件`）均可命中，接口 `POST /api/v1/kb/search` 返回 200。

