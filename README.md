# wx-lead-scanner

微信销售线索自动扫描器 — Claude Code Skill

每天从微信私聊和小群消息中自动提取销售线索，去重后录入飞书多维表格 CRM。

## 效果

- **以前**：每天花 20-30 分钟翻微信找线索，手动录入 CRM，还经常漏
- **现在**：说一句"扫线索"，30 秒出结果，自动录入飞书

## 工作原理

```
微信本地数据(wx-cli) → AI识别线索 → 飞书CRM去重 → 自动写入
```

1. 通过 [wx-cli](https://github.com/jackwener/wx-cli) 读取 Mac 微信本地数据库（数据不出本机）
2. AI 做两层筛选：关键词快筛 + 语义判断（排除广告、闲聊、已有客户日常沟通）
3. 对比飞书多维表格已有记录去重
4. 自动写入对应表（客户商机表 / 渠道表）
5. 输出扫描报告

## 前置依赖

| 工具 | 用途 | 安装 |
|------|------|------|
| [wx-cli](https://github.com/jackwener/wx-cli) | 读取微信本地数据 | `npm install -g @jackwener/wx-cli` |
| [lark-cli](https://github.com/nicepkg/lark-cli) | 操作飞书多维表格 | `npm install -g lark-cli` |
| Claude Code | AI 语义识别 + 工作流编排 | [claude.ai/code](https://claude.ai/code) |

## 快速开始

### 1. 安装 wx-cli 并初始化

```bash
npm install -g @jackwener/wx-cli

# 对微信重签名（仅需一次）
sudo codesign --force --deep --sign - /Applications/WeChat.app

# 重启微信，登录后初始化
killall WeChat && open /Applications/WeChat.app
sudo wx init
```

### 2. 安装此 skill

将 `SKILL.md` 放到 Claude Code 的 skills 目录：

```bash
cp -r wx-lead-scanner ~/.claude/skills/wx-lead-scanner
```

### 3. 配置 EXTEND.md

```bash
mkdir -p ~/.wx-lead-scanner
cat > ~/.wx-lead-scanner/EXTEND.md << 'EOF'
# wx-lead-scanner 配置

## 飞书 Base
base_token: 你的飞书多维表格token
opportunity_table_id: tblXXXXXX
channel_table_id: tblYYYYYY

## 微信身份
self_wxid_prefix: wxid_你的前缀
self_display: 你的显示名

## 自定义关键词（可选）
extra_keywords: 你的产品名, 你的行业术语
EOF
```

### 4. 使用

在 Claude Code 中说：

- "扫线索" — 扫描今天的新消息
- "扫一下最近3天的线索" — 指定时间范围
- "这周有什么新客户" — 自然语言触发

## 飞书表结构参考

### 客户商机表

| 字段 | 类型 | 说明 |
|------|------|------|
| 客户名称 | 文本 | 品牌/公司名 |
| 业务线 | 单选 | 你的业务分类 |
| 阶段 | 单选 | 线索/需求确认/报价/成交/交付/流失 |
| 关键人物 | 文本 | 对接人姓名+角色 |
| 联系方式 | 文本 | 手机号/微信号 |
| 优先级 | 单选 | S/A/B/C/D |
| 跟进记录 | 文本 | 消息摘要+日期 |
| 下次动作 | 文本 | 建议的下一步 |
| 简介 | 文本 | 需求背景描述 |
| 来源渠道 | 关联 | 关联渠道表 |

### 渠道表

| 字段 | 类型 | 说明 |
|------|------|------|
| 渠道名称 | 文本 | 渠道方名称 |
| 业务线 | 单选 | 业务分类 |
| 类型 | 单选 | 渠道合作/直销/协会平台/人脉圈 |
| 关键人物 | 文本 | 对接人 |
| 分成模式 | 文本 | 合作分成规则 |
| 联系方式 | 文本 | 手机号/微信号 |
| 跟进记录 | 文本 | 消息摘要+日期 |

## 注意事项

- wx-cli 所有数据读取都在本地完成，不经过任何第三方服务器
- 需要 macOS + Mac 版微信（WeChat 4.x）
- 微信需保持登录状态
- 首次使用建议先手动跑一次 `wx sessions` 确认 wx-cli 工作正常

## License

MIT
