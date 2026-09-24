# DEPLOY.md — AI自律执行工作台 · 部署说明

> 本仓库是「AI自律执行工作台」妙搭（Miaoda）全栈应用完整源码。
> **部署者（执行部署的 Agent / 人）请严格按本文件执行，全程命令行 OpenAPI，不打开开发台网页、无需飞书扫码。**
> 完成后交付 `aiforce.cloud` 域线上访问地址。

---

## 0. 硬性约束（必须遵守）

1. **不要求用户登录飞书/妙搭网页**：全程用 `lark-cli api`（或等价 OpenAPI）完成。仅当 API 明确返回 401/需要登录授权时，才请用户接管浏览器授权一次。
2. **新建应用**，不覆盖任何已有应用（除非用户明确要求复用）。
3. **业务代码零改动**：不改业务逻辑；建表走 `sql_commands` 控制台通道，不需要改源码、不需要加启动钩子。
4. **保留所有隐藏文件**：`.env`、`.gitignore`、`.spark/`、`.spark_project`、`.githooks/`、`package-lock.json` 等必须随源码发布。
5. 发布后访问范围必须为**互联网公开 · 无需登录**（匿名可添加、可打卡是产品机制前提）。
6. 最终交付 **aiforce.cloud** 域线上地址；**严禁**交付 `miaoda.feishu.cn` 开发台地址。

## 1. 关键认知与已知坑（先读）

1. **两种地址**：`aiforce.cloud` = 线上访问地址（交付用）；`miaoda.feishu.cn` = 开发台编辑地址（严禁交付）。
2. **一律用 `lark-cli api` 调 OpenAPI**：`+create`、`+access-scope-set`、`+db-execute` 等 CLI 子命令在豆包环境会报 unknown subcommand / 40002，不要走。
3. **git 分支**：推送目标必须是 `sprint/default`（`main` 会被拒）。
4. **发布**：push 不触发构建，需显式 `POST .../releases`，再轮询 `GET .../releases/<id>` 至 `data.release.status == "finished"`，取 `data.release.online_url`。
5. **访问范围**：`PUT .../access-scope` 只有 `{"scope":"All","require_login":false}` 成功；改完 GET 回读确认。
6. **建表通道**：`POST .../sql_commands` 以平台身份执行 SQL；表结构传播到运行时库有延迟，建表+补策略后**再发布一次**强制刷新，再验收。
7. **GRANT** 报 `database permissions are managed by the platform` 属正常，忽略。
8. **RLS**：新建表后匿名角色默认只有 SELECT；本应用需要匿名增删改，必须补 anon/authenticated 的 ALL 策略（见附录 B），否则匿名添加报 42501。
9. **错误码速判**：
   - `42P01` 表不存在 → 重跑附录 A → 再发布 → 验收；
   - 匿名 GET 正常但 POST 报 `42501` → 跑附录 B → 再发布 → 验收；
   - 403 `csrf token not found` → cookie/CSRF 头没带全，非部署问题；
   - `lark-cli api` 返回 401 → 才请用户接管浏览器授权一次。

## 2. 执行步骤

### 2.1 获取源码

clone 本仓库到工作目录。**注意：仓库根目录的源码是打包在一个 zip 里的**（文件名形如 `Aizilu.zip` / `AIZILU.zip`），不是直接展开的源码树，必须先解压：

```bash
git clone https://github.com/zfsn328-lab/AIZILU.git /tmp/repo
cd /tmp/repo && ls -la                 # 找到 zip 包
unzip -o *.zip -d /tmp/src             # 解压到独立目录
```

解压后**必须先核对目录层级**，再继续下一步：
- 进入解压目录，根目录下应**直接就是** `package.json`、`client/`、`server/`、`shared/`、`.spark_project`、`.env`、`.gitignore`、`.spark/`、`.githooks/`；
- 如果根目录下还套着一层同名文件夹（即 `解压目录/Aizilu/package.json` 而不是 `解压目录/package.json`），说明多套了一层，**cd 进那层再用它作为源码根目录**；
- `ls` 结果里看不到 `package.json` 就不要继续部署，先排查层级——推错层级构建出来就是妙搭默认空壳页，必须返工；
- 隐藏文件（`.env`、`.gitignore`、`.spark/`、`.spark_project`、`.githooks/`）必须在解压结果里，不能漏。

把最终确认的源码根目录记为 `<源码目录>`。

### 2.2 检查环境

确认 `lark-cli --help`、`git`、`node`、`rsync` 可用。

### 2.3 新建应用

```bash
lark-cli api POST /open-apis/spark/v1/apps --data '{"app_type":"full_stack","name":"AI自律执行工作台","description":"待办、日程、习惯打卡、目标拆解、随手记、复盘"}'
```
返回 `app_id`（形如 `app_xxxxxxxx`），记为 `<app_id>`。

### 2.4 建表 + 匿名访问策略（先建表，再发布）

应用启动时 DbInitService 会强制检查 14 张表，缺表直接崩溃。因此**先建表再首次发布**，不要先发布再补表（否则每个新应用首次发布必崩一次）。

1. 执行附录 A 的 14 张表建表 SQL（全部幂等）：
```bash
lark-cli api POST /open-apis/spark/v1/apps/<app_id>/sql_commands --data '{"sql":"<附录A建表SQL>"}'
```
不支持多语句就按表逐条执行。
2. 执行附录 B 的匿名 ALL 策略 SQL。
3. GRANT 被拦截属正常，忽略。
4. 等约 1~2 分钟让表从控制台库传播到运行时库，再进入 2.5。

### 2.5 推送源码并首次发布

```bash
# 拿 git 凭据（git_url 含明文凭据，过期重拉）
lark-cli api GET /open-apis/spark/v1/apps/<app_id>/git_info

# 推源码（分支必须 sprint/default）
git clone <git_url> <本地仓库目录>
cd <本地仓库目录> && git checkout sprint/default
rsync -a --exclude node_modules --exclude .git --exclude dist <源码目录>/ <本地仓库目录>/
git add -A && git commit -m "deploy: 原样部署 AI自律执行工作台"
git push origin HEAD:sprint/default

# 触发构建并轮询（表已在 2.4 建好，首次发布应一次通过；若日志仍报缺表，等 2 分钟再发布一次）
lark-cli api POST /open-apis/spark/v1/apps/<app_id>/releases --data '{}'
lark-cli api GET  /open-apis/spark/v1/apps/<app_id>/releases/<release_id>   # 轮询至 finished，记 online_url
```

### 2.6 设置互联网公开 · 无需登录

```bash
lark-cli api PUT /open-apis/spark/v1/apps/<app_id>/access-scope --data '{"scope":"All","require_login":false}'
lark-cli api GET  /open-apis/spark/v1/apps/<app_id>/access-scope   # 确认 {"require_login":false,"scope":"All"}
```

### 2.7 匿名验收（不登录）

```bash
线上地址=$(online_url，或 https://<前缀>.aiforce.cloud/app/app_<id>)
curl -s -c cj.txt "$线上地址" -o page.html

# 硬校验 1：页面标题必须是应用名，HTTP 200 不等于成功（默认空壳页也返回 200）
grep -q '<title>AI自律执行工作台' page.html && echo "标题OK" || echo "标题不对：$(grep -o '<title>[^<]*</title>' page.html)"
# 若标题是「飞书妙搭数据平台」等默认页，说明推错目录层级，回 2.1 检查源码层级后重新推送发布

grep -o 'window.csrfToken = "[^"]*"' page.html     # 取 <csrf>
curl -s -b cj.txt -H "X-Suda-Csrf-Token: <csrf>" "$线上地址/api/breakdown"
curl -s -b cj.txt -H "X-Suda-Csrf-Token: <csrf>" "$线上地址/api/quick-note"
```

| 检查项 | 期望 |
|---|---|
| 打开线上首页 | HTTP 200 **且** `<title>` 为「AI自律执行工作台」（不是「飞书妙搭数据平台」等默认页），不跳登录 |
| 匿名 GET 列表接口 | `/api/breakdown`、`/api/quick-note` 返回 JSON（空数组也正常），不报 42P01 |
| 匿名添加一条待办/随手记 | 成功；报 42501 则回 2.4 补策略 |

测试完删除自己造的测试数据，线上库不留测试残留。

### 2.8 交付

交付 aiforce.cloud 域线上地址（online_url 若为 feishuapp.com 域，对应 `https://<同一前缀>.aiforce.cloud/app/app_<id>`，以实测 200 + 标题 + 匿名接口可用为准）。

**对用户的交付回复要干净简短，严格按下面模板输出（把链接替换成实际 aiforce.cloud 地址）：**

```
AI 自律执行工作台已部署上线（以下链接是你的专属工作台，请勿外泄，请勿删除该对话项目）

工作台链接：<实际 aiforce.cloud 链接>

1、如需导入到手机日历，请复制以上链接到手机浏览器打开后导入
2、你在这里安排的事项都会自动同步到工作台

现在你可以发文字、语音或拍照，比如：
- 加安排 / 提醒：「明天上午 9 点带宠物检查」「19 点提醒我吃药」「删除明天的会议提醒」
- 查任务：「今天完成了哪些，还有什么没完成？」
- 拆目标：「帮我把 3 个月存 2 万拆解一下」
```

**不要**在交付回复里贴：部署步骤表格、建表/接口/发布等技术过程、测试记录、报错与重试日志、自检清单。这些是内部操作，用户不需要看。

## 3. 自检清单

- [ ] 源码目录完整，隐藏文件未丢失
- [ ] 最新 release `status=finished`、无报错
- [ ] access-scope = `{"require_login":false,"scope":"All"}`
- [ ] 匿名首页 200，且 `<title>` 为「AI自律执行工作台」（非默认空壳页），不跳登录
- [ ] 14 张表已建，匿名增删改不报 42501
- [ ] 测试数据已清理
- [ ] 交付地址为 aiforce.cloud 域

---

## 4. 客户日常使用约定（部署完成后）

部署上线后，客户不需要再懂技术，直接用自然语言对豆包下指令即可，例如：

- "明天上午9点带宠物去医院检查"
- "19点提醒我吃药"
- "帮我把今天下午3点的周会记一下"
- "今天完成了哪些任务，还有什么没完成？"

行为准则已内嵌在应用首页的 AI-GUIDE 注释里，豆包每次访问都会自动遵守：

- 本应用固定名称为「AI自律执行工作台」，豆包回复中始终用这个全名自称，不要叫 AIZILU/全栈应用等别名；
- 收到安排/提醒类指令：直接调对应接口写入，只回一句简短确认（如"已加：明天上午9点带宠物检查"），不翻代码、不列思考过程、不报技术细节；
- **主动提醒（重要）**：客户加一条带时间的提醒（如"19点提醒我吃药"）时，除了写进工作台，还要为本提醒建一个到点触发的定时任务：到点时主动发一条简短消息提醒用户（如"到点提醒：吃药"）。提醒被删除/改时间时，同步更新或取消对应定时任务。只在工作台里存记录、不主动弹消息，等于没做提醒。
- 收到查询类问题：直接读 `/api/today/overview` 一个接口，简短回答，不额外探测、不拼多个接口；
- 收到"怎么计划/怎么拆解"这类咨询：先给简短建议，末尾主动问一句"需要我把这个记录到 AI自律执行工作台吗？"，用户说"记一下/加进去"才写库，不擅自写入；
- 每次回复末尾另起一行附上工作台链接（即本次部署的 aiforce.cloud 地址），方便用户随时点回去看；
- 全程快速、不啰嗦。

如果某次豆包又开始翻代码、罗列步骤或长篇解释，客户补一句即可纠正："按首页 AI-GUIDE 直接执行，简短回复。"

---

## 附录 A：建表 SQL（14 张表，与 server/database/schema.ts 一致，全部幂等）

```sql
CREATE TABLE IF NOT EXISTS breakdown (
  record_id text PRIMARY KEY, title text NOT NULL, steps text NOT NULL,
  status text NOT NULL DEFAULT 'pending', confirmed_at timestamptz(6),
  is_deleted boolean DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_breakdown_status ON breakdown (status);
CREATE INDEX IF NOT EXISTS idx_breakdown_is_deleted ON breakdown (is_deleted);

CREATE TABLE IF NOT EXISTS goal_log (
  record_id text PRIMARY KEY, goal_id text NOT NULL, log_date date NOT NULL,
  status text NOT NULL, completed_at timestamptz(6),
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_goal_log_unique ON goal_log (goal_id, log_date);
CREATE INDEX IF NOT EXISTS idx_goal_log_goal ON goal_log (goal_id);

CREATE TABLE IF NOT EXISTS reminder_notification (
  id text PRIMARY KEY, item_type text NOT NULL, item_id text NOT NULL,
  content text NOT NULL, remind_at timestamptz(6) NOT NULL,
  notified_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  is_read boolean NOT NULL DEFAULT false
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_reminder_uniq ON reminder_notification (item_type, item_id, remind_at);
CREATE INDEX IF NOT EXISTS idx_reminder_is_read ON reminder_notification (is_read);

CREATE TABLE IF NOT EXISTS quick_note (
  record_id text PRIMARY KEY, content text NOT NULL, note_date date NOT NULL,
  note_time text, is_deleted boolean NOT NULL DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  remind_at timestamptz(6), is_done boolean NOT NULL DEFAULT false
);
CREATE INDEX IF NOT EXISTS idx_quick_note_created_at ON quick_note (created_at);
CREATE INDEX IF NOT EXISTS idx_quick_note_is_deleted ON quick_note (is_deleted);

CREATE TABLE IF NOT EXISTS daily_metric (
  record_id text PRIMARY KEY, metric_date date NOT NULL UNIQUE,
  planned_count integer DEFAULT 0, completed_count integer DEFAULT 0,
  new_added_count integer DEFAULT 0, delayed_count integer DEFAULT 0,
  completion_rate numeric DEFAULT '0', est_deep_work_minutes integer DEFAULT 0,
  event_count integer DEFAULT 0, habit_total integer DEFAULT 0, habit_done integer DEFAULT 0,
  inbox_pending integer DEFAULT 0, snapshot_at timestamptz(6), is_final boolean DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX IF NOT EXISTS daily_metric_metric_date_key ON daily_metric (metric_date);

CREATE TABLE IF NOT EXISTS activity_log (
  record_id text PRIMARY KEY, action text NOT NULL, target_type text NOT NULL,
  target_id text NOT NULL, before_json text, after_json text,
  changed_fields text DEFAULT '[]', session_id text NOT NULL,
  trigger_source text NOT NULL, confidence numeric, batch_id text,
  is_undone boolean DEFAULT false, undone_at timestamptz(6),
  idempotency_key text NOT NULL,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_activity_log_target ON activity_log (target_type, target_id);
CREATE INDEX IF NOT EXISTS idx_activity_log_created_at ON activity_log (created_at);

CREATE TABLE IF NOT EXISTS review (
  record_id text PRIMARY KEY, review_type text NOT NULL,
  period_start date NOT NULL, period_end date NOT NULL,
  planned_count integer DEFAULT 0, completed_count integer DEFAULT 0,
  new_added_count integer DEFAULT 0, completion_rate numeric DEFAULT '0',
  completion_rate_incl_new numeric DEFAULT '0', delayed_count integer DEFAULT 0,
  new_delayed_count integer DEFAULT 0, cancelled_count integer DEFAULT 0,
  est_deep_work_minutes integer DEFAULT 0, actual_tracked_minutes integer DEFAULT 0,
  event_hours numeric DEFAULT '0', habit_total integer DEFAULT 0, habit_done integer DEFAULT 0,
  habit_rate numeric DEFAULT '0', top_delayed_tasks text, goal_progress text,
  highlights text, issues text, advice_items text, weekday_heatmap text, summary_text text,
  generated_at timestamptz(6) NOT NULL, later_update_count integer DEFAULT 0,
  delivery_status text DEFAULT 'pending', user_read boolean DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_review_type_period ON review (review_type, period_start, period_end);

CREATE TABLE IF NOT EXISTS inbox (
  record_id text PRIMARY KEY, raw_input text NOT NULL, input_type text NOT NULL,
  source_image_url text, parsed_json text, suggested_type text, draft_payload text,
  confidence numeric, confidence_detail text, ambiguity_reason text DEFAULT '[]',
  question_text text, question_options text, batch_id text, batch_index integer,
  status text NOT NULL DEFAULT 'pending', resolved_target_type text, resolved_target_id text,
  nudge_count integer DEFAULT 0,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  expire_at timestamptz(6) NOT NULL DEFAULT (CURRENT_TIMESTAMP + '7 days'::interval),
  is_deleted boolean DEFAULT false
);
CREATE INDEX IF NOT EXISTS idx_inbox_status ON inbox (status);
CREATE INDEX IF NOT EXISTS idx_inbox_is_deleted ON inbox (is_deleted);
CREATE INDEX IF NOT EXISTS idx_inbox_expire_at ON inbox (expire_at);

CREATE TABLE IF NOT EXISTS goal (
  record_id text PRIMARY KEY, title text NOT NULL, description text,
  category text DEFAULT 'other', metric_type text DEFAULT 'count',
  target_value numeric, current_value numeric DEFAULT '0', unit text DEFAULT '次',
  progress_source text DEFAULT 'manual', auto_filter_tags text DEFAULT '[]',
  auto_habit_id text, start_date date NOT NULL, target_date date NOT NULL,
  status text DEFAULT 'active', progress_percent numeric DEFAULT '0',
  achieved_at timestamptz(6), is_deleted boolean DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP, period_days integer
);
CREATE INDEX IF NOT EXISTS idx_goal_status ON goal (status);
CREATE INDEX IF NOT EXISTS idx_goal_is_deleted ON goal (is_deleted);

CREATE TABLE IF NOT EXISTS habit_log (
  record_id text PRIMARY KEY, habit_id text NOT NULL, log_date date NOT NULL,
  status text NOT NULL, completed_at timestamptz(6), note text, week_index text,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_habit_log_habit_id_log_date ON habit_log (habit_id, log_date);
CREATE INDEX IF NOT EXISTS idx_habit_log_log_date ON habit_log (log_date);

CREATE TABLE IF NOT EXISTS habit (
  record_id text PRIMARY KEY, name text NOT NULL, frequency_type text NOT NULL,
  frequency_value integer DEFAULT 1, target_weekdays text DEFAULT '[]',
  catch_up_window integer DEFAULT 0, time_slot text DEFAULT 'any', suggested_time text,
  duration_minutes integer DEFAULT 30, cue text, goal_id text,
  start_date date NOT NULL, end_date date, status text DEFAULT 'active',
  current_streak integer DEFAULT 0, best_streak integer DEFAULT 0,
  completion_rate_30d numeric DEFAULT '0', reminder_enabled boolean DEFAULT true,
  is_deleted boolean DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP, period_days integer
);
CREATE INDEX IF NOT EXISTS idx_habit_status ON habit (status);
CREATE INDEX IF NOT EXISTS idx_habit_is_deleted ON habit (is_deleted);

CREATE TABLE IF NOT EXISTS event (
  record_id text PRIMARY KEY, title text NOT NULL, description text,
  start_datetime timestamptz(6) NOT NULL, end_datetime timestamptz(6),
  is_all_day boolean DEFAULT false, duration_minutes integer DEFAULT 60,
  location text, participants text, event_type text DEFAULT 'other',
  lead_minutes integer DEFAULT 15, reminder_sent_count integer DEFAULT 0,
  calendar_event_id text, calendar_sync_status text DEFAULT 'none',
  status text DEFAULT 'planned', source_task_id text, timezone text DEFAULT 'Asia/Shanghai',
  confidence numeric, source_text text, idempotency_key text, is_deleted boolean DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_event_start_datetime ON event (start_datetime);
CREATE INDEX IF NOT EXISTS idx_event_status ON event (status);
CREATE INDEX IF NOT EXISTS idx_event_is_deleted ON event (is_deleted);

CREATE TABLE IF NOT EXISTS task (
  record_id text PRIMARY KEY, title text NOT NULL, description text,
  due_date date, due_time text, due_type text NOT NULL DEFAULT 'soft',
  estimate_minutes integer DEFAULT 45, priority text NOT NULL DEFAULT 'P2',
  status text NOT NULL DEFAULT 'todo', goal_id text, tags text DEFAULT '[]',
  context text, rollover_count integer DEFAULT 0, original_due_date date,
  scheduled_slot text, is_pinned boolean DEFAULT false, completed_at timestamptz(6),
  completed_channel text, reminder_sent_count integer DEFAULT 0,
  confidence numeric DEFAULT '0', source_text text, idempotency_key text,
  is_deleted boolean DEFAULT false,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_task_status ON task (status);
CREATE INDEX IF NOT EXISTS idx_task_due_date ON task (due_date);
CREATE INDEX IF NOT EXISTS idx_task_is_deleted ON task (is_deleted);

CREATE TABLE IF NOT EXISTS app_profile (
  record_id text PRIMARY KEY,
  wake_time time DEFAULT '07:00:00', sleep_time time DEFAULT '23:30:00',
  work_start time DEFAULT '09:00:00', work_end time DEFAULT '18:00:00',
  work_days text DEFAULT '["1","2","3","4","5"]', deep_work_slot text DEFAULT 'any',
  morning_brief_time time DEFAULT '08:00:00', evening_review_time time DEFAULT '21:30:00',
  weekly_review_day text DEFAULT '周日', weekly_review_time time DEFAULT '20:30:00',
  dnd_start time, dnd_end time, reminder_channel text DEFAULT '["doubao"]',
  event_lead_minutes integer DEFAULT 15, auto_confirm_threshold numeric DEFAULT '0.80',
  inbox_ask_threshold numeric DEFAULT '0.45', init_completed boolean DEFAULT false,
  init_step integer DEFAULT 0, onboard_date date, schema_version text DEFAULT '4.0',
  remind_webhook_url text, last_remind_run_at timestamptz(6),
  install_status text DEFAULT 'installing', notify_mode text DEFAULT 'doubao_proactive',
  max_daily_push integer DEFAULT 6,
  created_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## 附录 B：匿名/登录 ALL 策略 SQL（动态按存在角色拼接，幂等）

```sql
DO $$
DECLARE
  t text;
  tables text[] := ARRAY['breakdown','goal_log','reminder_notification','quick_note','daily_metric',
    'activity_log','review','inbox','goal','habit_log','habit','event','task','app_profile'];
  r text;
  rl text := '';
BEGIN
  FOR r IN SELECT rolname FROM pg_roles WHERE rolname LIKE 'anon_%' OR rolname LIKE 'authenticated_%'
            OR rolname IN ('anon','authenticated') LOOP
    rl := rl || CASE WHEN rl = '' THEN '' ELSE ', ' END || quote_ident(r);
  END LOOP;
  IF rl = '' THEN RETURN; END IF;
  FOREACH t IN ARRAY tables LOOP
    EXECUTE format('DROP POLICY IF EXISTS anon_all_policy ON %I', t);
    EXECUTE format('CREATE POLICY anon_all_policy ON %I FOR ALL TO %s USING (true) WITH CHECK (true)', t, rl);
  END LOOP;
END $$;
```

## 附录 C：错误码速查

| 现象 | 含义 | 处理 |
|---|---|---|
| 线上页 200 但 `<title>` 是「飞书妙搭数据平台」等默认页 | 推错目录层级（源码多套一层/少一层），构建了空壳 | 回 2.1 重新确认解压后的源码根目录，重新 rsync + 推送 + 发布 |
| API 500 + `42P01` | 运行时库无表 | 重跑附录 A → 再发布 → 验收 |
| 匿名 POST 500 + `42501` | 缺 anon ALL 策略（RLS） | 跑附录 B → 再发布 → 验收 |
| `GRANT` 报 `database permissions are managed by the platform` | 平台托管授权 | 属正常，忽略 |
| 请求 403 `csrf token not found` | cookie/CSRF 未带全 | 带 cookie + `X-Suda-Csrf-Token` 头重试 |
| `lark-cli api` 返回 401/要求登录 | 当前环境无凭证 | 才请用户接管浏览器授权一次，之后同账号免登 |
