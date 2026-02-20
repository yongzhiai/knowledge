# KnowForge P0 技术实施方案

## 1. 文档元信息
- 文档名称：KnowForge P0 技术实施方案
- 文档版本：v1.0
- 对应 PRD：`document/prd/KnowForge-PRD-v1-ES全文检索版.md`
- 文档路径：`document/wiki/KnowForge-P0-技术实施方案.md`
- 编写日期：2026-02-20
- 适用阶段：P0（1-2 天）
- 目标：输出可直接指导开发执行的全链路技术方案，不在本轮实现代码。

## 2. P0 范围与非范围
### 2.1 P0 范围（必须交付）
1. Markdown 笔记 CRUD。
2. 全文检索（Elasticsearch）。
3. CLI 重写后回写（rewrite apply）。
4. 操作审计 + Git 历史与回滚。
5. 最小 Web 页面能力约束（列表/编辑/搜索交互要求）。

### 2.2 P0 非范围（明确不做）
1. 向量检索、混合检索（仅预留接口与结构）。
2. 原子卡片提取、关系图谱、模板系统（P1+）。
3. 重型任务管理。
4. 多机分布式部署与高可用集群设计。

## 3. 当前代码基线与差距清单
### 3.1 当前仓库基线（已存在）
1. Spring Boot 启动骨架：`src/main/java/com/yongzhiai/knowledge/KnowledgeApplication.java`
2. 基础配置：`src/main/resources/application.properties`
3. 最小测试：`src/test/java/com/yongzhiai/knowledge/KnowledgeApplicationTests.java`
4. 构建依赖：Spring WebMVC、Devtools、Docker Compose、Lombok、WebMVC Test。

### 3.2 与 P0 目标的差距
1. 缺少业务分层（controller/service/repository/domain）。
2. 缺少全部 P0 API 实现。
3. 缺少 MySQL 实体与持久化层。
4. 缺少文件系统存储适配层。
5. 缺少 Elasticsearch 索引与检索实现。
6. 缺少 Git 提交/历史/回滚模块。
7. 缺少审计日志模型与写入链路。
8. 缺少 API 契约测试和全链路集成测试。

## 4. 目标架构与模块边界
### 4.1 架构风格
- 模块化单体（Modular Monolith），按领域能力划分模块。
- 通过 SPI 预留 P1/P2 扩展（向量检索、混合检索、图谱）。

### 4.2 模块拆分
1. `note-core`：笔记领域模型、CRUD 编排。
2. `note-storage-fs`：Markdown 文件读写。
3. `note-meta-mysql`：元数据与审计持久化。
4. `search-es`：全文检索与索引同步。
5. `history-git`：提交、历史查询、版本回滚。
6. `api-mcp`：REST 接口、参数校验、统一响应。

### 4.3 职责边界
1. 后端负责存取、检索、审计、回滚。
2. 内容生成由外部 CLI + 大模型完成，后端仅执行“应用写入”。
3. 检索统一走 ES，P0 仅全文字段，P1 扩展 dense_vector。

## 5. 公共 API 契约
### 5.1 统一响应结构
```json
{
  "code": "OK",
  "message": "success",
  "data": {},
  "requestId": "c5f1f4f7-8b83-4d6d-b7ce-24836d4eb3db",
  "timestamp": "2026-02-20T10:30:00Z"
}
```

### 5.2 统一错误码最小集
- `NOTE_NOT_FOUND`
- `INVALID_MARKDOWN`
- `ES_INDEX_FAILED`
- `GIT_COMMIT_FAILED`
- `VALIDATION_FAILED`

### 5.3 P0 接口定义（逐项）
1. `POST /api/mcp/notes`
- 用途：创建笔记并落盘、落库、入索引、提交 Git、写审计。
- 幂等语义：非幂等（每次调用创建新 `noteId`）。
- 主要错误：`VALIDATION_FAILED`、`ES_INDEX_FAILED`、`GIT_COMMIT_FAILED`。

2. `GET /api/mcp/notes/{noteId}`
- 用途：读取笔记内容与元数据。
- 幂等语义：幂等。
- 主要错误：`NOTE_NOT_FOUND`。

3. `PUT /api/mcp/notes/{noteId}`
- 用途：更新笔记正文/标题/标签并同步检索与审计。
- 请求体 DTO（`UpdateNoteRequest`）：

| 字段名 | 数据类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `title` | `string` | 否 | 笔记标题；为空时保持原值。 |
| `content` | `string` | 是 | Markdown 正文内容。 |
| `tags` | `array<string>` | 否 | 标签列表；用于元数据和检索过滤。 |
| `type` | `string` | 否 | 笔记类型（如 `inbox`/`project`/`wiki`）。 |
| `domain` | `string` | 否 | 子领域标识（如 `redis`、`kafka`）。 |
| `status` | `string` | 否 | 生命周期状态（如 `draft`、`published`）。 |
| `operator` | `string` | 否 | 操作人标识，用于审计日志。 |
| `source` | `string` | 否 | 变更来源（`web`/`cli`/`api`）。 |

- 幂等语义：对同一版本内容重复更新可视为业务幂等。
- 主要错误：`NOTE_NOT_FOUND`、`INVALID_MARKDOWN`、`ES_INDEX_FAILED`、`GIT_COMMIT_FAILED`。

4. `POST /api/mcp/notes/{noteId}/rewrite-apply`
- 用途：应用 CLI 产出的新正文。
- 请求体 DTO（`ApplyRewriteRequest`）：

| 字段名 | 数据类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `content` | `string` | 是 | CLI/AI 重写后的完整 Markdown 正文。 |
| `requestId` | `string(UUID)` | 是 | 幂等键；防止同一重写结果重复应用。 |
| `operator` | `string` | 否 | 触发回写的操作者。 |
| `source` | `string` | 否 | 建议固定值 `cli`，用于审计来源归因。 |

- 幂等语义：同一 rewrite payload 可按幂等键控制重复应用（建议 requestId）。
- 主要错误：`NOTE_NOT_FOUND`、`INVALID_MARKDOWN`、`ES_INDEX_FAILED`。

5. `GET /api/mcp/search/fulltext`
- 用途：全文检索。
- 固定参数：
  - `q`：必填关键词。
  - `topK`：可选，默认 20，最大 50。
  - `type`：可选，按笔记类型过滤。
  - `domain`：可选，按子领域过滤。
- 幂等语义：幂等。
- 主要错误：`VALIDATION_FAILED`。

6. `GET /api/mcp/audit/logs`
- 用途：查询审计日志。
- 幂等语义：幂等。

7. `GET /api/mcp/git/history/{noteId}`
- 用途：查询笔记 Git 提交历史。
- 幂等语义：幂等。
- 主要错误：`NOTE_NOT_FOUND`。

8. `POST /api/mcp/git/revert/{noteId}`
- 用途：将笔记回滚到指定提交版本。
- 固定请求体：
```json
{
  "commitId": "a1b2c3d4"
}
```
- 幂等语义：对同一目标 commit 的重复回滚可视为幂等。
- 主要错误：`NOTE_NOT_FOUND`、`GIT_COMMIT_FAILED`、`VALIDATION_FAILED`。

### 5.4 P1 预留接口声明
- `POST /api/mcp/index/vectorize/{noteId}`
- `GET /api/mcp/search/vector`
- `GET /api/mcp/search/hybrid`

说明：以上仅预留，不计入 P0 完成定义。

## 6. 数据模型与存储设计
### 6.1 存储分工
1. 文件系统：Markdown 正文源数据。
2. MySQL：元数据、审计日志、查询索引辅助字段。
3. Elasticsearch：全文检索索引。

### 6.2 核心表设计
1. `notes`（P0）

| 字段名 | 数据类型（MySQL 8） | 约束 | 含义 |
| --- | --- | --- | --- |
| `id` | `char(36)` | PK, NOT NULL | 笔记主键 UUID。 |
| `type` | `varchar(32)` | NOT NULL | 笔记类型（`inbox`/`project`/`wiki`）。 |
| `path` | `varchar(512)` | UNIQUE, NOT NULL | Markdown 文件相对路径。 |
| `title` | `varchar(255)` | NOT NULL | 笔记标题。 |
| `tags` | `json` | NULL | 标签数组。 |
| `status` | `varchar(32)` | NOT NULL | 笔记状态（`draft`/`published`/`archived`）。 |
| `domain` | `varchar(64)` | NULL | 子领域标识。 |
| `created_at` | `datetime(3)` | NOT NULL | 创建时间。 |
| `updated_at` | `datetime(3)` | NOT NULL | 最近更新时间。 |

2. `audit_logs`（P0）

| 字段名 | 数据类型（MySQL 8） | 约束 | 含义 |
| --- | --- | --- | --- |
| `id` | `char(36)` | PK, NOT NULL | 审计主键 UUID。 |
| `note_id` | `char(36)` | INDEX, NOT NULL | 关联 `notes.id`。 |
| `operator` | `varchar(128)` | NOT NULL | 操作人或系统主体。 |
| `source` | `varchar(32)` | NOT NULL | 操作来源（`web`/`cli`/`api`/`system`）。 |
| `before_hash` | `char(64)` | NULL | 变更前内容哈希（SHA-256）。 |
| `after_hash` | `char(64)` | NULL | 变更后内容哈希（SHA-256）。 |
| `diff` | `longtext` | NULL | 结构化或文本 diff 内容。 |
| `created_at` | `datetime(3)` | NOT NULL | 审计记录创建时间。 |

3. `links`（P1 预留）

| 字段名 | 数据类型（MySQL 8） | 约束 | 含义 |
| --- | --- | --- | --- |
| `from_id` | `char(36)` | NOT NULL | 起始笔记 ID。 |
| `to_id` | `char(36)` | NOT NULL | 目标笔记 ID。 |
| `relation_type` | `varchar(32)` | NOT NULL | 关系类型（如 `reference`、`derive`）。 |

## 7. ES 全文检索方案
### 7.1 索引定义
- 索引名：`knowforge_notes_v1`
- 分词：`ik + pinyin`（与 PRD 约束一致）
- 字段设计：

| 字段名 | ES 数据类型 | 分词/索引策略 | 含义 |
| --- | --- | --- | --- |
| `id` | `keyword` | 精确匹配 | 对应 `notes.id`，用于唯一定位。 |
| `title` | `text` | `ik_max_word` + `pinyin` 子字段 | 标题全文检索与召回加权。 |
| `content` | `text` | `ik_max_word` + `pinyin` 子字段 | 正文全文检索主字段。 |
| `tags` | `keyword` | 精确匹配 | 标签过滤与聚合。 |
| `type` | `keyword` | 精确匹配 | 笔记类型过滤。 |
| `domain` | `keyword` | 精确匹配 | 子领域过滤。 |
| `path` | `keyword` | 精确匹配 | 文件路径回显与跳转。 |
| `status` | `keyword` | 精确匹配 | 生命周期状态过滤。 |
| `updated_at` | `date` | 时间排序 | 最近更新时间排序。 |

### 7.2 查询策略
1. 默认 `multi_match` 覆盖 `title/content/tags`。
2. `topK` 默认 20，最大 50。
3. 可选过滤：`type`, `domain`。
4. 返回高亮片段，用于最小 Web 检索结果展示。

### 7.3 一致性与失败处理
1. 写入文件 + MySQL 成功后同步刷新 ES，保证“写后可搜”。
2. ES 更新失败：返回 `ES_INDEX_FAILED`，并写入审计日志标记失败原因。
3. 失败补偿：记录待重试任务（可先基于本地重试队列，P1 可升级消息队列）。

## 8. 写入链路与检索链路时序
### 8.1 写入链路（创建/更新/rewrite-apply）
1. API 参数校验。
2. Markdown 语法基础校验。
3. 写文件系统正文。
4. 更新 MySQL `notes`。
5. 写 `audit_logs`（before/after hash + diff）。
6. 提交 Git。
7. 同步更新 ES 索引。
8. 返回统一响应。

### 8.2 写入失败补偿原则
1. 文件写失败：直接失败返回，不进入后续步骤。
2. MySQL 失败：文件变更需回滚或标记脏数据待修复。
3. Git 失败：返回 `GIT_COMMIT_FAILED` 并写审计。
4. ES 失败：返回 `ES_INDEX_FAILED`，保留重试记录。

### 8.3 检索链路
1. 参数校验（`q`, `topK` 范围）。
2. 组装 ES 查询 DSL。
3. 执行检索。
4. 结果映射（高亮、摘要、路径、更新时间）。
5. 统一响应返回。

## 9. Git 历史与审计日志机制
### 9.1 Git 规则
1. 每次写入自动提交。
2. 提交信息建议格式：`note(<noteId>): <action>`。
3. 历史查询基于 note path 过滤。

### 9.2 回滚规则
1. 回滚请求必须提供 `commitId`。
2. 执行回滚后需再次写审计日志并重建 ES 文档。
3. 回滚结果需可被全文检索即时反映。

### 9.3 审计要求
1. 必须记录操作来源（Web/CLI/API）。
2. 必须记录 before/after hash 与 diff。
3. 审计查询按 noteId、时间区间可过滤。

## 10. 最小 Web 交互约束（纳入 P0）
1. 列表区必须支持多级目录导航，不可仅平铺列表。
2. 编辑器必须支持编辑/预览双模式。
3. 必须支持全屏编辑与 `Esc` 退出。
4. 目录节点支持 `Enter/Space` 键盘操作。
5. 搜索区必须支持关键词结果、高亮、无结果态。

说明：此处是交互约束，不绑定具体前端框架实现细节。

## 11. 非功能指标与容量假设
### 11.1 性能目标（P0）
1. 笔记读写 < 1s。
2. 全文检索 < 1.5s（topK=20）。

### 11.2 稳定性目标
1. Git 自动提交成功率 > 99%。
2. 写入链路失败必须可追溯（审计可查）。

### 11.3 容量假设（单机）
1. 单用户本地部署。
2. 初期笔记规模 1 万条以内。
3. 以可用性优先，容量扩展放入 P1/P2。

## 12. 测试策略与验收场景
### 12.1 测试策略
1. API 契约测试：覆盖 7 个 P0 业务接口的参数校验、成功响应、错误码。
2. 核心流程集成测试：文件系统 + MySQL + ES + Git 全链路。
3. 关键失败路径测试：ES 失败、Git 失败、无效 Markdown、noteId 不存在。

### 12.2 验收场景
1. Web 新建笔记后，全文检索可命中。
2. CLI 回写后，检索结果实时更新。
3. 误改后可通过 Git 回滚恢复。
4. 审计日志可完整追溯改动。

### 12.3 验收命令（示例）
1. `./mvnw clean compile`（Windows：`.\mvnw.cmd clean compile`）
2. `./mvnw test`（Windows：`.\mvnw.cmd test`）
3. `./mvnw clean verify`（Windows：`.\mvnw.cmd clean verify`）

## 13. P0 排期（D1/D2）与里程碑 DoD
### 13.1 D1 上午
- 工作项：项目配置、数据模型、笔记 CRUD。
- DoD：能完成笔记创建/读取/更新，MySQL 与文件系统可用。
- 验证：CRUD 契约测试通过。

### 13.2 D1 下午
- 工作项：ES 索引初始化、全文检索接口、写后可搜链路。
- DoD：新建/更新后检索可命中，支持 `topK/type/domain` 参数。
- 验证：检索相关测试通过，性能接近目标。

### 13.3 D2 上午
- 工作项：CLI 回写应用、Git 历史与回滚、审计日志。
- DoD：回写、历史、回滚、审计接口可用。
- 验证：回滚后检索与内容一致。

### 13.4 D2 下午
- 工作项：最小 Web 联调、全链路集成测试、风险收口。
- DoD：四个验收场景全部通过。
- 验证：`clean verify` 通过，风险清单已闭环或已备案。

## 14. 风险、回滚与 P1 演进边界
### 14.1 主要风险
1. 本地 Git 与文件系统权限差异导致提交失败。
2. ES 分词与高亮配置不当导致召回不稳定。
3. 写入链路跨存储一致性在异常时复杂。

### 14.2 风险缓解
1. 启动时做依赖可用性检查（MySQL/ES/Git）。
2. 为 ES 失败提供重试与审计可观测。
3. 对关键链路增加集成测试与失败注入。

### 14.3 回滚策略
1. 代码回滚：按模块化提交粒度回退。
2. 数据回滚：优先使用 Git note 级回滚。
3. 索引回滚：按 noteId 重建 ES 文档。

### 14.4 P1 演进边界
1. 仅在现有 ES 索引中扩展 `dense_vector`，不更换检索存储。
2. 在现有 API 风格下新增向量与混合检索接口。
3. 原子卡片/关系图谱作为独立模块扩展，不破坏 P0 CRUD 与审计链路。

## 15. 需求映射校对清单（PRD -> 技术方案）
1. P0 功能闭环：已覆盖（CRUD、全文检索、CLI 回写、审计、Git、最小 Web 约束）。
2. 接口集合：已覆盖（P0 全量 + P1 预留声明）。
3. 存储分工：已覆盖（文件系统 + MySQL + ES）。
4. 统一检索策略：已覆盖（ES 全文、ik+pinyin、写后可搜）。
5. 性能指标：已覆盖（读写 < 1s，检索 < 1.5s）。
6. 验收场景：已覆盖（4 个核心场景）。

