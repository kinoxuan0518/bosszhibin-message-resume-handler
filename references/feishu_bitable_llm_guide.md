# 飞书多维表格（Bitable）接入说明（给 LLM 工具）

本说明用于让同学在自己的环境中完成：

1. 创建飞书多维表格（作为招聘数据层）
2. 配置应用权限与凭据
3. 建立与 LLM 工具的稳定数据通信（可重试、可幂等）

## 1. 目标与数据流

推荐数据流：

```text
BOSS会话处理 -> 规则评估 -> 飞书招聘批量提交 -> Bitable结构化同步
```

同步建议拆成 3 张表：

- `candidates`：候选人主档（去重主表）
- `interactions`：触达/回复/淘汰等行为流水
- `jobs_dashboard`：职位漏斗统计（聚合视图）

## 2. 创建你自己的 Bitable

1. 在飞书中创建一个新的多维表格（不要复用生产表）。
2. 新建 3 张表：
- `candidates`
- `interactions`
- `jobs_dashboard`
3. 为每张表创建字段（建议字段如下）。

### 2.1 `candidates` 字段建议

| 字段名 | 类型 | 说明 |
|---|---|---|
| `platform_id` | 单行文本 | 平台候选人唯一标识（主去重键） |
| `platform` | 单选/文本 | 平台名（如 BOSS） |
| `name` | 单行文本 | 候选人姓名 |
| `status` | 单选 | 当前状态（已触达/已获简历/已上传飞书/已淘汰等） |
| `current_company` | 单行文本 | 当前公司 |
| `current_title` | 单行文本 | 当前岗位 |
| `experience_years` | 数字 | 工作年限 |
| `education_school` | 单行文本 | 学校 |
| `education_degree` | 单选 | 本科/硕士/博士 |
| `school_tier` | 单选 | C9/985/211/QS100/其他 |
| `city` | 单行文本 | 城市 |
| `first_seen_date` | 日期时间 | 首次出现时间 |
| `last_seen_date` | 日期时间 | 最近出现时间 |
| `times_seen` | 数字 | 出现次数 |

### 2.2 `interactions` 字段建议

| 字段名 | 类型 | 说明 |
|---|---|---|
| `interaction_id` | 单行文本 | 交互唯一键（去重键） |
| `platform_id` | 单行文本 | 关联候选人 |
| `job_name` | 单行文本 | 职位名 |
| `action_type` | 单选 | 打招呼/请求简历/同意收简历/淘汰/上传飞书 |
| `action_date` | 日期时间 | 动作时间 |
| `match_score` | 数字 | 匹配分 |
| `reply_content` | 多行文本 | 候选人回复 |
| `skip_reason` | 单选/文本 | 淘汰原因 |
| `outcome` | 单选 | 无回复/已回复/拒绝/通过 |

### 2.3 `jobs_dashboard` 字段建议

| 字段名 | 类型 | 说明 |
|---|---|---|
| `job_name` | 单行文本 | 职位名（聚合键） |
| `total_seen` | 数字 | 总浏览/总处理人数 |
| `total_contacted` | 数字 | 已触达人数 |
| `total_replied` | 数字 | 已回复人数 |
| `total_resume_received` | 数字 | 收到简历人数 |
| `total_uploaded_feishu` | 数字 | 上传飞书人数 |
| `total_interviewed` | 数字 | 面试人数 |
| `reply_rate` | 数字 | 回复率 |
| `resume_rate` | 数字 | 简历率 |
| `interview_pass_rate` | 数字 | 面试通过率 |
| `avg_match_score` | 数字 | 平均匹配分 |
| `last_run_date` | 日期时间 | 最近同步时间 |

## 3. 创建飞书应用并授权

1. 在飞书开发者后台创建企业自建应用。
2. 打开“权限管理”，添加 Bitable 相关读写权限（至少包含“应用/表/记录”的读取与写入能力）。
3. 安装应用到目标租户（企业）。
4. 获取以下配置项：
- `<APP_ID>`
- `<APP_SECRET>`
- `<APP_TOKEN>`（多维表格 App Token）
- `<TABLE_ID_CANDIDATES>`
- `<TABLE_ID_INTERACTIONS>`
- `<TABLE_ID_DASHBOARD>`

注意：不同租户权限命名可能有微差异，按“bitable + read/write + record/table/app”关键字筛选授权。

## 4. 本地配置（禁止入库）

建议使用 `.env`：

```bash
FEISHU_APP_ID=<APP_ID>
FEISHU_APP_SECRET=<APP_SECRET>
FEISHU_APP_TOKEN=<APP_TOKEN>
FEISHU_TABLE_CANDIDATES=<TABLE_ID_CANDIDATES>
FEISHU_TABLE_INTERACTIONS=<TABLE_ID_INTERACTIONS>
FEISHU_TABLE_DASHBOARD=<TABLE_ID_DASHBOARD>
```

安全要求：

- `.env` 加入 `.gitignore`
- 不在日志中打印 `APP_SECRET` 与 Access Token
- 只在本地或密钥系统注入，不写入 `SKILL.md`、`README.md`

## 5. LLM 与 Bitable 的数据通信协议

建议让 LLM 产出统一结构，再由同步器写入 Bitable：

```json
{
  "session_date": "2026-03-04",
  "job_name": "<JOB_NAME>",
  "platform": "BOSS直聘",
  "candidates": [
    {
      "platform_id": "boss_123456",
      "name": "<NAME>",
      "status": "已获简历",
      "experience_years": 4,
      "education_degree": "本科",
      "education_school": "<SCHOOL>",
      "school_tier": "985",
      "match_score": 84,
      "action": "请求简历",
      "outcome": "已回复"
    }
  ]
}
```

协议约束：

- `platform_id` 不能为空（候选人主键）。
- `interaction_id` 推荐规则：`<platform_id>__<job_name>__<date>__<action>`。
- 所有时间统一输出 ISO 格式或 `YYYY-MM-DD`，同步器负责转毫秒时间戳。

## 6. API 调用顺序（推荐）

1. 获取租户 token：
- `POST /open-apis/auth/v3/tenant_access_token/internal`
2. 读取已有记录（用于 upsert 去重）：
- `GET /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records`
3. 新增记录：
- `POST /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/batch_create`
4. 更新记录：
- `PUT /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/{record_id}`

推荐策略：

- 候选人主档：按 `platform_id` upsert
- 交互流水：按 `interaction_id` 仅新增
- 职位看板：按 `job_name` upsert

## 7. 最小可用同步器（伪代码）

```python
# 1) token
# 2) 拉 candidates 现有索引 {platform_id: record_id}
# 3) 遍历候选人，存在则 update，不存在则 batch_create
# 4) 拉 interactions 已有 interaction_id 集合
# 5) 仅写入未出现的 interaction
# 6) 聚合 session 统计，按 job_name upsert 到 jobs_dashboard
```

## 8. 重试、限流、幂等

- 批量写入分片：每批不超过 500 条。
- 失败重试：指数退避（1s/2s/4s），最多 3 次。
- 限流防护：控制每分钟请求量，优先批量接口。
- 幂等保障：
  - `candidates`: `platform_id`
  - `interactions`: `interaction_id`
  - `jobs_dashboard`: `job_name`

## 9. 诊断清单

当“同步失败”时按顺序检查：

1. 应用是否已安装到当前租户。
2. 应用权限是否包含 Bitable 读写。
3. `APP_TOKEN`、`TABLE_ID` 是否对应同一个多维表格。
4. 字段名是否与同步器中的字段映射完全一致。
5. Access Token 是否过期（建议做 token 缓存与刷新）。

## 10. 与技能联动建议

- 在技能主流程中，把 Bitable 同步放在“飞书招聘提交成功”之后。
- 同步失败时仅标记 `ERROR_BITABLE_SYNC`，不回滚已完成的飞书招聘提交。
- 输出里显式给出：成功写入条数、失败条数、可重试数据包。
