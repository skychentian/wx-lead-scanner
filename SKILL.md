---
name: wx-lead-scanner
description: 微信销售线索自动扫描器。每天从微信私聊和小群消息中提取新销售线索，去重后自动录入飞书多维表格（客户商机表/渠道表）。当用户说"扫线索"、"今天线索"、"微信线索"、"录线索"、"扫一下线索"、"scan leads"、"有什么新线索"、"看看今天有没有客户"、"帮我录一下线索"时触发。也适用于用户说"扫一下最近几天"、"这周线索"等指定时间范围的场景。
---

# 微信销售线索扫描器

从微信本地消息中自动识别销售线索，去重后写入飞书 CRM 多维表格。

## 前置条件

- [wx-cli](https://github.com/jackwener/wx-cli) 已安装且 `wx init` 已完成（微信保持登录）
- lark-cli 已认证（`lark-cli auth login`）
- 所有 `wx` 命令必须使用 `dangerouslyDisableSandbox: true`
- 用户需在 `EXTEND.md` 中配置自己的参数（见下方配置说明）

## 配置（EXTEND.md）

首次使用前，在以下路径之一创建 `EXTEND.md`（优先级从高到低）：

1. `.wx-lead-scanner/EXTEND.md`（项目目录）
2. `~/.wx-lead-scanner/EXTEND.md`（用户目录）

```markdown
# wx-lead-scanner 配置

## 飞书 Base
base_token: <你的飞书多维表格 token>
opportunity_table_id: <客户商机表 ID>
channel_table_id: <渠道表 ID>

## 微信身份
self_wxid_prefix: <你的微信 wxid 前缀，如 wxid_abc123>
self_display: <你的显示名>

## 业务关键词（可选，根据你的业务自定义）
# 默认关键词已覆盖常见场景，如需追加：
extra_keywords: 你的产品名, 你的服务名, 行业术语

## 扫描范围（可选）
max_group_size: 20
```

## 工作流

按以下步骤执行，不要跳步。

### Step 1: 确定扫描范围

解析用户输入，确定日期范围：

- 无指定 → 读取 `~/.wx-cli/lead-scanner-state.json` 中的 `last_scan_time`，从该时间点到现在
- 首次运行（无 state 文件）→ 默认扫今天
- 用户指定了日期（如"最近3天""这周"）→ 按指定范围

### Step 2: 获取目标会话

```bash
wx sessions --json --limit 1000
```

从返回结果中筛选目标：

**私聊**：`is_group = false` 且 `chat_type = "private"`。排除以下 chat ID：
- `gh_` 开头（公众号）
- `filehelper`（文件传输助手）
- 含 `@openim`（企业微信机器人）
- 含 `@qy_`（企业微信）
- `brandservicesessionholder`、`notifymessage` 等系统号

**小群**：`is_group = true`。对每个群执行 `wx members "<chatroom_id>" --json`，只保留成员数 ≤ `max_group_size`（默认20）的群。

只保留扫描范围内有新消息的会话（按 `time` 字段判断）。

同时保存每条会话的 `summary` 和 `last_sender` 字段，作为后续 fallback 数据源。

### Step 2.5: 刷新 daemon 索引

在拉消息之前，先重启 daemon 确保索引最新：

```bash
wx daemon stop
wx daemon status  # 确认已停止，daemon 会在下次 wx 命令时自动重启
```

wx-cli daemon 有缓存，长时间运行后可能漏掉新消息所在的数据库 shard。不刷新会导致部分联系人消息拉不到。

### Step 3: 批量拉取消息

**两个数据源并行使用**，确保不遗漏：

**数据源 A：wx history 拉完整消息**

对每个目标会话：

```bash
wx history "<chat_id>" --since YYYY-MM-DD --until YYYY-MM-DD -n 200 --json
```

如果 `wx history` 报错"找不到联系人"，不要跳过，而是用 sessions 里的 `summary` 字段作为 fallback（见数据源 B）。

从返回的 `messages` 数组中，**排除自己发的消息**（sender 匹配 `self_wxid_prefix`）。

也排除系统消息（`type` 含"系统"）和撤回消息。

**数据源 B：sessions summary 兜底**

对于 wx history 拉取失败的会话，直接用 summary 内容做关键词匹配。summary 虽然只有最后一条消息，但对于"刚发来需求"的场景已经足够识别线索。

### Step 4: 识别线索

对剩余消息做两层筛选：

**第一层：关键词快筛**

扫描消息 content，命中以下任一关键词组的视为候选：

| 关键词组 | 场景 |
|---------|------|
| 培训、企培、内训、课程、讲师、授课 | 培训需求 |
| GEO、搜索优化、AI可见、品牌诊断 | GEO 需求 |
| 合作、渠道、分成、代理、转介绍 | 渠道合作 |
| 报价、方案、收费、价格、预算、签约 | 商务意向 |
| 需求、想了解、能不能做、帮我们 | 主动咨询 |
| 约时间、见面、吃饭、聊聊、对接 | 约见 |
| 你已添加了、打招呼 | 新好友 |

用户可通过 EXTEND.md 的 `extra_keywords` 追加自定义关键词。

**第二层：语义判断**

对候选消息做上下文判断，排除以下噪音：
- 自己群发的自我介绍（多个不同人收到相同内容）
- 已有客户的日常工作沟通（非新需求）
- 广告群发、课程推广、非定向消息
- 纯社交寒暄无商业意图

保留的才是真正的线索。每条线索提取：
- **来源**：私聊 / 群名
- **对方**：sender 昵称
- **时间**：消息时间
- **内容摘要**：关键消息原文（≤200字）
- **线索类型判断**：客户商机 or 渠道

**判断进哪张表的规则：**
- 有明确的品牌/公司/产品，且表达了服务需求 → **客户商机表**
- 是转介绍人、渠道合作伙伴、平台方、协会资源 → **渠道表**
- 新加好友但意图不明 → **暂不录入**，只在报告中提及

### Step 5: 去重

从飞书 Base 拉取已有记录用于比对：

```bash
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <opportunity_table_id> \
  --field-id "客户名称" --field-id "关键人物" \
  --limit 200

lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <channel_table_id> \
  --field-id "渠道名称" --field-id "关键人物" \
  --limit 200
```

去重规则：
- 客户名称/渠道名称完全相同 → 跳过，但如果有新的跟进信息，追加到该记录的"跟进记录"字段
- 关键人物相同且业务线相同 → 跳过
- 模糊匹配 → 标记为疑似重复，在报告中提示但仍录入

### Step 6: 写入飞书 Base

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <target_table_id> \
  --json '<record_json>'
```

**客户商机表字段映射（参考）：**

| 字段 | 填写规则 |
|------|---------|
| 客户名称 | 品牌/公司名（必填） |
| 业务线 | 根据需求判断 |
| 阶段 | 新线索统一填"线索" |
| 关键人物 | 对方姓名+角色 |
| 联系方式 | 手机号/微信号（如有） |
| 简介 | 一句话描述需求背景 |
| 跟进记录 | 消息原文摘要 + 日期 |
| 下次动作 | 建议的下一步 |
| 优先级 | S=明确需求+决策人 / A=明确需求 / B=有意向 / C=待观察 |

单次写入一条，串行执行，批次间隔 0.5 秒。

### Step 7: 更新状态 & 输出报告

写入完成后：

1. 更新 `~/.wx-cli/lead-scanner-state.json`：

```json
{
  "last_scan_time": "2026-05-17",
  "last_scan_stats": {
    "sessions_scanned": 150,
    "leads_found": 5,
    "leads_written": 3,
    "duplicates_skipped": 2
  }
}
```

2. 输出中文摘要报告：

```
线索扫描完成（YYYY-MM-DD）

扫描范围：X 个私聊 + Y 个小群
发现线索：N 条
  → 客户商机：M 条（已录入飞书）
  → 渠道线索：K 条（已录入飞书）
  → 跳过重复：J 条

高优先级线索：
1. [客户名] — 需求描述 — 建议动作
2. ...

新增好友（未录入，待观察）：
- XX（身份描述）
- ...
```

## 常见问题

| 问题 | 处理 |
|------|------|
| wx 命令报错 permission denied | 检查 `~/.wx-cli` 目录权限：`sudo chown -R $(whoami) ~/.wx-cli` |
| wx history 返回"找不到联系人" | 先 `wx daemon stop`，再重试 |
| wx history 返回空 | daemon 需要重启：`wx daemon stop && wx daemon start` |
| lark-cli 写入失败 | 检查 auth 状态：`lark-cli auth status` |
| 消息量太大（>500条） | 按天分批处理 |

## 适用场景

- 销售型业务（to B）每日线索梳理
- 渠道合作管理
- 社群运营中的商机识别
- 任何需要从微信消息中提取结构化信息到 CRM 的场景

## 技术依赖

- [wx-cli](https://github.com/jackwener/wx-cli) — 微信本地数据 CLI 工具
- [lark-cli](https://github.com/nicepkg/lark-cli) — 飞书命令行工具
- Claude Code — AI 驱动的语义识别和工作流编排
