# Prism DESIGN V2

> 基于 `ROOT.md` 与 `SPEC.md` 的详细设计文档（实现导向，覆盖 MVP 可落地闭环）。

## 1. 目标与范围

### 1.1 目标
- 交付一个可邀请内测用户试用的 Web 产品闭环：账号 + 笔记 + 聚合 + 信息流 + 轻社交 + 管理后台。
- 以稳定、可维护为优先，不追求复杂推荐算法和复杂运营系统。

### 1.2 非目标
- 不实现复杂内容生产能力（例如思维导图、播客生成）。
- 不实现复杂多租户、微服务拆分、在线扩缩容。
- 不实现高复杂度推荐排序（采用规则排序）。

---

## 2. 技术架构

### 2.1 系统组成
- `apps/web`：Next.js（App Router）+ Tailwind + shadcn/ui。
- `apps/api`：FastAPI（REST API）。
- `PostgreSQL`：主业务数据存储。
- `Redis`：会话/验证码/限流/异步任务状态。
- `Worker`（可与 API 同代码基，独立进程）：
  - 内容分析任务（拉取网页、调用 LLM）
  - 聚合刷新任务（RSS/Atom 拉取、去重入库）

### 2.2 部署拓扑（MVP）
- Docker Compose 单机部署（开发 + 小规模生产）。
- 生产推荐同域：
  - Web：`https://<domain>/`
  - API：`https://<domain>/api/v1`
- Caddy/Nginx 作为反向代理与 TLS 入口。

### 2.3 关键设计原则
- **逻辑删除优先**：用户、笔记、聚合源支持逻辑删除。
- **幂等优先**：收藏/点赞/关注/任务重试保证重复请求一致。
- **显式状态机**：分析状态统一 `PENDING -> RUNNING -> SUCCEEDED|FAILED`。
- **可观测性最小闭环**：记录失败诊断、任务状态、最近错误时间。

---

## 3. 领域模型设计（核心实体）

> 命名以 PostgreSQL 表名表达，字段仅列关键字段。

### 3.1 用户与认证

#### `users`
- `id` (uuid, pk)
- `user_id` (varchar, unique，可复用；删除用户后释放)
- `email` (varchar, unique，可复用；删除用户后释放)
- `password_hash` (nullable，SSO-only 可为空)
- `nickname` (varchar)
- `language` (enum: `zh-CN`, `en-US`, default `zh-CN`)
- `role` (enum: `admin`, `user`)
- `status` (enum: `active`, `disabled`, `deleted`)
- `deleted_at` (nullable)
- `created_at`, `updated_at`

#### `user_sso_accounts`
- `id` (uuid, pk)
- `user_id` (fk -> users.id)
- `provider` (enum: `google`)
- `provider_user_id` (varchar)
- `provider_email` (varchar)
- unique(`provider`, `provider_user_id`)

#### `auth_login_attempts`
- `id` (uuid, pk)
- `principal` (varchar: user_id/email/ip 组合摘要)
- `window_start` (timestamp)
- `fail_count` (int)
- `locked_until` (nullable)

#### `email_verification_codes`
- `id` (uuid, pk)
- `scene` (enum: `register`, `reset_password`)
- `email`
- `code_hash`
- `expires_at`
- `used_at` (nullable)
- `attempt_count` (int)

### 3.2 创作者与关注

#### `creators`
- `id` (uuid, pk)
- `type` (enum: `user`, `source`)
- `user_id` (nullable fk -> users.id)
- `source_id` (nullable fk -> aggregation_sources.id)
- `display_name`
- `bio` (nullable)
- unique(`type`, `user_id`) / unique(`type`, `source_id`)

#### `follows`
- `id` (uuid, pk)
- `follower_user_id` (fk -> users.id)
- `creator_id` (fk -> creators.id)
- `created_at`
- unique(`follower_user_id`, `creator_id`)

### 3.3 内容与笔记

#### `contents`
- 内容主表：统一承载“笔记绑定链接”与“聚合条目链接”的归一化信息。
- `id` (uuid, pk)
- `normalized_url` (text, unique)
- `raw_url` (text)
- `domain` (varchar)
- `fetch_strategy` (enum: `direct`, `jina`)
- `language` (nullable)
- `published_at` (nullable)
- `created_at`, `updated_at`

#### `notes`
- `id` (uuid, pk)
- `owner_user_id` (fk -> users.id)
- `content_id` (fk -> contents.id)
- `visibility` (enum: `public`, `private`, default `public`)
- `learning_note` (text, 用户心得)
- `user_tags` (jsonb array, <=5)
- `analysis_status` (enum: `pending`, `running`, `succeeded`, `failed`)
- `analysis_error` (nullable text)
- `analysis_updated_at` (nullable)
- `deleted_at` (nullable)
- `created_at`, `updated_at`
- unique(`owner_user_id`, `content_id`)  // 防止同用户同链接重复创建

#### `aggregates`
- `id` (uuid, pk)
- `source_id` (fk -> aggregation_sources.id)
- `content_id` (fk -> contents.id)
- `analysis_status` (enum: `pending`, `running`, `succeeded`, `failed`)
- `analysis_error` (nullable)
- `analysis_updated_at` (nullable)
- `deleted_at` (nullable)
- `created_at`, `updated_at`
- unique(`source_id`, `content_id`)

### 3.4 聚合源

#### `aggregation_sources`
- `id` (uuid, pk)
- `name`
- `site_url`
- `feed_url`
- `language` (nullable)
- `status` (enum: `enabled`, `disabled`, `deleted`)
- `deleted_at` (nullable)
- `last_refresh_at` (nullable)
- `last_refresh_status` (enum: `idle`, `running`, `succeeded`, `failed`)
- `last_refresh_error` (nullable text)
- `created_at`, `updated_at`

#### `aggregation_refresh_jobs`
- `id` (uuid, pk)
- `source_id` (nullable, fk)
- `triggered_by` (enum: `system`, `admin`)
- `status` (enum: `queued`, `running`, `succeeded`, `failed`)
- `diagnosis` (nullable text)
- `created_at`, `updated_at`, `finished_at` (nullable)

### 3.5 分析结果

#### `content_analysis_results`
- `id` (uuid, pk)
- `content_id` (fk -> contents.id)
- `model_provider` (`openai`/`gemini`/`claude`)
- `model_name`
- `model_version` (nullable)
- `title_original`, `title_zh` (nullable)
- `summary_short_original`, `summary_short_zh` (nullable)
- `summary_long_original`, `summary_long_zh` (nullable)
- `tags_original` (jsonb array <=5)
- `tags_zh` (jsonb array <=5)
- `published_at_inferred` (nullable)
- `analyzed_at`
- `prompt_version`

### 3.6 互动行为

#### `favorites`
- `id` (uuid, pk)
- `user_id`
- `target_type` (enum: `note`, `aggregate`)
- `target_id` (uuid)
- `created_at`
- unique(`user_id`, `target_type`, `target_id`)

#### `likes`
- 与 favorites 类似，unique 口径一致。

> 计数采用“实时聚合 + 缓存”或“异步回写 counter 表”均可。MVP 推荐先实时聚合查询 + 短缓存。

---

## 4. 核心流程设计

### 4.1 注册/登录/退出
1. 注册时检查 `user_id/email` 未被占用（`status != deleted` 的用户中唯一）。
2. 校验邮箱验证码（场景 `register`）。
3. 写入 `users` 与默认 `creators(type=user)`。
4. 登录支持 `user_id/email + password`。
5. Redis 记录失败次数；超阈值触发临时锁定。
6. 登录成功签发会话（HttpOnly Cookie/JWT 二选一；MVP 推荐 HttpOnly Session Cookie）。
7. 退出登录销毁 Redis 会话。

### 4.2 Google SSO
1. `/auth/sso/google/start` 生成 `state` 存 Redis（TTL）。
2. 回调校验 state，换 token，读取用户信息。
3. 若邮箱已绑定本地账号：直接登录并绑定 provider（若未绑定）。
4. 若无匹配账号：生成 `complete_token` 跳转前端补全页。
5. 补全 `user_id` 后创建账号 + creator + sso 绑定。

### 4.3 创建学习笔记
1. 用户提交 URL。
2. URL 归一化：
   - lower-case host
   - 去 fragment
   - 去常见追踪参数（`utm_*`, `spm`, `from` 等）
   - 规范尾斜杠
3. 黑名单域名检查（配置文件）。
4. 查 `contents(normalized_url)`：无则创建。
5. 校验同一用户是否已存在 `notes(owner,content)`：若存在返回冲突并给出跳转 id。
6. 创建 note，状态 `pending`，异步投递分析任务。

### 4.4 内容分析
1. Worker 取任务，置 `running`。
2. 根据开关选择抓取策略：
   - `CONTENT_FETCH_USE_JINA_READER=true` => `JINA_READER_BASE_URL + url`
   - 否则直连 URL。
3. 抓取失败：记录 `analysis_error`，状态 `failed`。
4. 调 LLM（OpenAI/Gemini/Claude 统一 adapter）。
5. 生成结构化结果：标题、双摘要、标签、发布时间推断、语言。
6. 应用摘要兜底逻辑（短/长互补）。
7. 写 `content_analysis_results` 最新记录，状态置 `succeeded`。

### 4.5 聚合刷新
1. 定时或管理员触发 refresh job。
2. 拉取源 RSS/Atom，按 `AGGREGATION_MAX_ITEMS_PER_SOURCE` 截断。
3. 对条目 URL 归一化，写入 `contents` + `aggregates`（去重）。
4. 对新增/待更新条目投递分析任务。
5. 更新 job 状态 + source `last_refresh_*` 字段。

### 4.6 收藏/点赞/关注
- API 采用“set/unset 显式语义”或“toggle + 当前状态返回”。
- 重复提交通过 unique 约束与 `ON CONFLICT DO NOTHING` 保持幂等。
- 公开笔记转私有/删除时：
  - 信息流查询自动过滤；
  - 互动操作返回不可用。

---

## 5. API 设计（补充约束）

### 5.1 通用响应
- 成功：`{ "data": ..., "request_id": "..." }`
- 失败：`{ "error": { "code": "...", "message": "..." }, "request_id": "..." }`

### 5.2 关键错误码建议
- `AUTH_INVALID_CREDENTIALS`
- `AUTH_TOO_MANY_ATTEMPTS`
- `USER_ID_ALREADY_EXISTS`
- `EMAIL_ALREADY_EXISTS`
- `URL_BLACKLISTED`
- `NOTE_ALREADY_EXISTS`
- `ANALYSIS_RETRY_NOT_ALLOWED`
- `FORBIDDEN_PRIVATE_NOTE`
- `NOT_FOUND`

### 5.3 权限矩阵（简）
- 普通用户：仅管理自己的私有资源。
- 公开资源：可读，不可编辑。
- 管理员：访问 `/admin/*`，且不可把最后一个管理员降级。

---

## 6. 信息流与搜索

### 6.1 信息流统一视图
构建逻辑视图 `feed_items`（可用 SQL VIEW 或查询拼接）：
- 来源：`public notes` + `active aggregates`
- 字段统一：`item_type`, `item_id`, `creator_id`, `published_at_for_sort`, `tags`, `summary`, `counters`

### 6.2 排序规则
- 默认 `published_at_for_sort DESC`。
- 笔记使用 `notes.updated_at(learning_note)` 作为发布时间。
- 聚合条目优先 `contents.published_at`，无则 fallback `aggregates.created_at`。

### 6.3 筛选
- 关注/未关注：按 `creator_id` 是否在 `follows` 集合。
- 标签筛选：`tags` overlap。
- 去重：同结果集按 `(item_type,item_id)` 唯一。

---

## 7. 前端交互设计（实现要点）

### 7.1 全局导航
- 左：品牌名 + 全局搜索（作用于笔记/广场）。
- 中：`笔记`、`广场`。
- 右：`写笔记`、通知、账号入口。

### 7.2 页面路由建议
- `/auth` 登录/注册 Tab + Google 按钮。
- `/auth/forgot`、`/auth/reset`。
- `/onboarding/google-complete`。
- `/notes?tab=my|favorites`
- `/notes/new`
- `/notes/:id`
- `/feed?tab=following|discover&tag=...`
- `/profile/:userId`
- `/admin?tab=users|notes|sources|aggregates`

### 7.3 卡片展示规则
- 标题最多两行，摘要分“自动摘要/学习心得”两个块。
- 空块不展示。
- 右下角显示 `聚合/笔记` 标签 + 收藏/点赞。
- Hover 创作者弹层支持关注操作。

---

## 8. 配置文件设计

### 8.1 `config/url_blacklist.yaml`
- 字段：`domains`, `keywords`, `reason`
- 示例：`mp.weixin.qq.com`、需登录且反爬严重站点。

### 8.2 `config/aggregation_sources.yaml`
- 字段：`name`, `site_url`, `feed_url`, `language`, `enabled`
- 首批建议内置 8~15 个高质量 AI 源。

---

## 9. 安全与风控

- 密码：bcrypt/argon2。
- 会话 Cookie：`HttpOnly` + `Secure` + `SameSite=Lax`。
- CSRF：对 Cookie 会话接口加 CSRF Token。
- 登录限流：按 IP + principal 双维度。
- 输入校验：URL 长度、tag 数量、文本长度。
- 管理接口审计：记录管理员操作日志（最少记录 who/what/when）。

---

## 10. 可观测性与运维

- 结构化日志：`request_id`, `user_id`, `path`, `latency_ms`。
- 任务日志：分析任务/聚合任务分别带 `job_id/content_id`。
- 健康检查：`/healthz`（应用）+ `/readyz`（DB/Redis）。
- 指标（MVP 可选）：任务成功率、平均分析耗时、失败分布。

---

## 11. 测试设计

### 11.1 后端测试
- 单元测试：URL 归一化、摘要兜底、权限判断、状态机流转。
- 集成测试：
  - 注册/登录/忘记密码
  - 创建笔记 + 异步分析 + 重试
  - 聚合刷新 + 去重
  - 收藏/点赞/关注幂等
  - 公开转私有后信息流不可见

### 11.2 前端测试
- 关键页面渲染与路由守卫。
- 表单校验（注册、登录、补全、创建笔记）。
- 交互测试（Tab 切换、筛选、收藏点赞按钮状态）。

### 11.3 验收测试映射
- 将 `SPEC.md` 验收要点逐条映射到 e2e Case（Playwright）。

---

## 12. 里程碑建议

### M1（账号闭环）
- 本地注册登录、Google SSO、资料页、管理员初始化。

### M2（笔记闭环）
- URL 归一化、创建笔记、分析任务、公开私有、详情与编辑。

### M3（聚合与信息流）
- 聚合源管理、刷新任务、信息流双 Tab、标签筛选。

### M4（社交与管理完善）
- 关注/收藏/点赞、后台排障视图、失败重试工具。

---

## 13. 风险与缓解

- **外部站点抓取不稳定**：Jina Reader + 失败重试 + 域名黑名单。
- **LLM 成本与延迟**：异步任务、结果缓存、模型超时与重试策略。
- **SSO 配置复杂**：提供明确 env 模板与回调检查清单。
- **数据一致性**：关键关系加 unique 约束，接口幂等化。

---

## 14. 交付产物清单

- `DESIGN.md`（本文件）
- 数据库迁移脚本（按实体建模）
- API OpenAPI 文档
- 前端页面与组件清单
- e2e 验收用例清单
