---
name: wx-lead-scanner
description: 微信销售线索自动扫描器。每天从微信私聊和小群消息中提取新销售线索，去重后自动录入飞书多维表格（客户商机表/渠道表）。当用户说"扫线索"、"今天线索"、"微信线索"、"录线索"、"扫一下线索"、"scan leads"、"有什么新线索"、"看看今天有没有客户"、"帮我录一下线索"时触发。也适用于用户说"扫一下最近几天"、"这周线索"等指定时间范围的场景。
---

# 微信销售线索扫描器

从微信本地消息中自动识别销售线索，去重后写入飞书 CRM 多维表格。

核心理念：**不依赖关键词，用全量语义扫描识别线索**。关键词匹配会遗漏大量真实线索（约见、渠道转介绍、展会后跟进等场景往往不含业务关键词）。

## 前置条件

- [wx-cli](https://github.com/jackwener/wx-cli) 已安装且 `wx init` 已完成（微信保持登录）
- [lark-cli](https://github.com/larksuite/cli) 已认证（`lark-cli auth login`）
- 所有 `wx` 命令必须使用 `dangerouslyDisableSandbox: true`
- 用户需在 `EXTEND.md` 中配置自己的参数（见下方配置说明）

## 配置（EXTEND.md）

首次使用前，在以下路径之一创建 `EXTEND.md`（优先级从高到低）：

1. `.wx-lead-scanner/EXTEND.md`（项目目录）
2. `~/.wx-lead-scanner/EXTEND.md`（用户目录）

参考 `EXTEND.md.example` 填写。支持多 CRM 路由（按业务线分流到不同 Base）。

## 工作流

按以下步骤执行，不要跳步。

### Step 1: 确定扫描范围

解析用户输入，确定日期范围：

- 无指定 → 读取 `~/.wx-cli/lead-scanner-state.json` 中的 `last_scan_time`，从该时间点到现在
- 首次运行（无 state 文件）→ 默认扫今天
- 用户指定了日期（如"最近3天""这周"）→ 按指定范围

### Step 2: 获取全量会话 + 刷新索引

先刷新 daemon，再拉会话列表：

```bash
wx daemon stop
wx daemon status  # 确认已停止，daemon 会在下次 wx 命令时自动重启
```

然后获取全量会话：

```bash
wx sessions --json --limit 1000
```

排除系统账号：
- `gh_` 开头（公众号）
- `filehelper`（文件传输助手）
- 含 `@openim`（企业微信机器人）
- 含 `@qy_`（企业微信）
- `brandservicesessionholder`、`brandsessionholder`、`notifymessage`
- `@placeholder_foldgroup`（折叠群聊）

只保留扫描范围内有新消息的会话（按 `timestamp` 字段判断）。

**此步不做关键词筛选，保留全量会话进入下一步。**

### Step 3: 全量语义初筛（Session 级）

这是整个流程最关键的步骤：**不用关键词，用语义判断全量会话。**

将所有会话的 `chat`（联系人名/群名）+ `summary`（最后一条消息）+ `time` 作为整体列表，做语义初筛。

**判断信号（任一命中即为候选）：**

| 信号类型 | 示例 |
|---------|------|
| 明确业务意图 | 提到品牌、产品、报价、预算、合同、付款 |
| 约见/跟进 | 约时间、面聊、发定位、"下周找你" |
| 渠道/转介绍 | 介绍客户、分成、帮推、拉群对接 |
| 对方身份暗示 | 名字含公司名/职位/行业（如"XX网络推广部""XX品牌CEO"） |
| 新好友 | "你已添加了""打招呼" |
| 间接线索 | 发PDF/PPT/报告、语音通话后跟文字确认 |

**排除噪音（不需要拉消息）：**
- 纯社交寒暄（"哈哈""好的""[表情]"且联系人名无业务标识）
- 社群运营通知（分红、积分、签到）
- 广告群发
- 电商/外卖/物业等生活服务

初筛结果分两类：
- **A 类（高可能）**：直接拉完整消息
- **B 类（联系人名有业务相关性但 summary 无明确意图）**：只看 summary，不拉消息

**群聊优化**：只对 A 类群聊检查成员数 `wx members "<chatroom_id>" --json`，保留 ≤ `max_group_size` 的小群。避免对几百个群逐个查成员数。

### Step 4: 拉取消息 + 深度识别

对 Step 3 的 A 类会话拉取完整消息：

```bash
wx history "<chat_id>" --since YYYY-MM-DD --until YYYY-MM-DD -n 200 --json
```

如果报错"找不到联系人"，用 sessions 的 `summary` 字段兜底。

**私聊消息注意事项**：wx-cli 在私聊模式下 `from_wxid` 和 `from_nickname` 均返回 null，无法直接区分自己和对方的消息。判断自发消息的方法：
- 消息内容完全匹配自我介绍模板 → 自己发的
- 相同内容出现在多个不同私聊中 → 群发的自我介绍
- 内容是诊断链接/报价截图/公司介绍文件 → 自己发的

对每条会话的消息做语义判断，识别真正的线索。每条线索提取：
- **来源**：私聊 / 群名
- **对方**：联系人名 或 sender 昵称
- **时间**：消息时间
- **内容摘要**：关键消息原文（≤200字）
- **线索类型**：客户商机 or 渠道
- **阶段判断**：已签约/已付款 → "成交"；其他 → "线索"

**路由规则：**

如果 EXTEND.md 配置了多个 Base（多业务线），按消息内容判断业务线分流。业务线不明确的走默认 Base。

判断客户还是渠道：
- 有明确品牌/公司/产品+服务需求 → **客户表**
- 转介绍人、渠道合作伙伴、平台方 → **渠道表**
- 新好友意图不明 → **暂不录入**，只在报告中提及

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

参考 lark-base skill 的 `references/lark-base-record-upsert.md` 和 `references/lark-base-shortcut-record-value.md`。

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
| 阶段 | 新线索填"线索"，已签约填"成交" |
| 关键人物 | 对方姓名+角色 |
| 联系方式 | 手机号/微信号（如有） |
| 简介 | 一句话描述需求背景 |
| 跟进记录 | 消息原文摘要 + 日期 |
| 下次动作 | 建议的下一步 |
| 优先级 | S=已签约或明确需求+决策人 / A=明确需求 / B=有意向 / C=待观察 |

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
    "leads_updated": 1,
    "channels_written": 2,
    "duplicates_skipped": 1
  }
}
```

2. 输出中文摘要报告：

```
线索扫描完成（YYYY-MM-DD）

扫描范围：X 个私聊 + Y 个小群
发现线索：N 条
  → 新建客户：M 条
  → 新建渠道：K 条
  → 更新已有记录：U 条
  → 跳过重复：J 条

高优先级线索：
1. [客户名]（业务线）— 需求描述 — 建议动作
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
| 私聊消息 from_wxid 为 null | wx-cli 私聊不返回 sender，用内容特征区分自发消息 |

## 为什么用全量语义扫描而不是关键词匹配

关键词匹配在实际使用中会漏掉大量真实线索：

| 被遗漏的线索类型 | 为什么关键词抓不到 |
|----------------|------------------|
| 渠道转介绍 | 对方说"帮你拉个群聊聊"，不含业务关键词 |
| 展会/活动后跟进 | "约下周二面聊""给我个定位" |
| 对方身份即线索 | 联系人名"XX品牌推广部"但 summary 只有"好的好的" |
| 间接商务意图 | 发了公司介绍 PDF 后语音通话，文字只有"看了PPT感觉可以" |
| 已签约但未录入 | 合同/付款/发票等签约场景 |

全量语义扫描的 token 消耗很低：136 个会话的 `chat名+summary` 加起来不到 1 万字，一次语义初筛即可完成。只对筛出的候选会话（通常 10-20 个）拉完整消息。

## 技术依赖

- [wx-cli](https://github.com/jackwener/wx-cli) — 微信本地数据 CLI 工具
- [lark-cli](https://github.com/larksuite/cli) — 飞书命令行工具
- Claude Code — AI 驱动的语义识别和工作流编排
