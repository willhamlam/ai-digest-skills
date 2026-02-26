---
name: ai-daily-digest-custom
description: "Fetches RSS feeds from 90 top Hacker News blogs (curated by Karpathy), uses AI to score and filter articles, and generates a daily digest in Markdown with Chinese-translated titles, category grouping, trend highlights, and visual statistics (Mermaid charts + tag cloud). Use when user mentions 'daily digest', 'RSS digest', 'blog digest', 'AI blogs', 'tech news summary', or asks to run /digest command. Trigger command: /digest."
---

# AI Daily Digest

从 Karpathy 推荐的 90 个热门技术博客中抓取最新文章，通过 AI 评分筛选，生成每日精选摘要。

## 命令

### `/digest`

运行每日摘要生成器。

**使用方式**: 输入 `/digest`，Agent 通过交互式引导收集参数后执行。

---

## 脚本目录

**重要**: 所有脚本位于此 skill 的 `scripts/` 子目录。

**Agent 执行说明**:

1. 确定此 SKILL.md 文件的目录路径为 `SKILL_DIR`
2. 脚本路径 = `${SKILL_DIR}/scripts/<script-name>.ts`

| 脚本                | 用途                                 |
| ------------------- | ------------------------------------ |
| `scripts/digest.ts` | 主脚本 - RSS 抓取、AI 评分、生成摘要 |

---

## 配置持久化

配置文件路径: `~/.hn-daily-digest/config.json`

Agent 在执行前**必须检查**此文件是否存在：

1. 如果存在，读取并解析 JSON
2. 询问用户是否使用已保存配置
3. 执行完成后保存当前配置到此文件

**配置文件结构**:

```json
{
  "anthropicApiKey": "",
  "anthropicApiBase": "",
  "timeRange": 48,
  "topN": 15,
  "language": "zh",
  "lastUsed": "2026-02-14T12:00:00Z"
}
```

---

## 交互流程

### Step 0: 加载环境变量

检查 `${SKILL_DIR}/.env` 是否存在，如果存在则读取其中的环境变量：

```bash
cat ${SKILL_DIR}/.env 2>/dev/null || echo "NO_ENV_FILE"
```

如果 `.env` 存在且包含 `ANTHROPIC_API_KEY`，在 Step 2 执行脚本时使用这些值。
如果 `.env` 中同时包含 `ANTHROPIC_API_BASE`，也一并使用。
如果 `.env` 中包含 `DIGEST_OUTPUT_DIR`，在 Step 2 中用作输出目录。

### Step 0b: 检查已保存配置

```bash
cat ~/.hn-daily-digest/config.json 2>/dev/null || echo "NO_CONFIG"
```

如果配置存在且有 `anthropicApiKey`，询问是否复用：

```
question({
  questions: [{
    header: "使用已保存配置",
    question: "检测到上次使用的配置：\n\n• 时间范围: ${config.timeRange}小时\n• 精选数量: ${config.topN} 篇\n• 输出语言: ${config.language === 'zh' ? '中文' : 'English'}\n\n请选择操作：",
    options: [
      { label: "使用上次配置直接运行 (Recommended)", description: "使用所有已保存的参数立即开始" },
      { label: "重新配置", description: "从头开始配置所有参数" }
    ]
  }]
})
```

### Step 1: 收集参数

使用 `question()` 一次性收集：

```
question({
  questions: [
    {
      header: "时间范围",
      question: "抓取多长时间内的文章？",
      options: [
        { label: "24 小时", description: "仅最近一天" },
        { label: "48 小时 (Recommended)", description: "最近两天，覆盖更全" },
        { label: "72 小时", description: "最近三天" },
        { label: "7 天", description: "一周内的文章" }
      ]
    },
    {
      header: "精选数量",
      question: "AI 筛选后保留多少篇？",
      options: [
        { label: "10 篇", description: "精简版" },
        { label: "15 篇 (Recommended)", description: "标准推荐" },
        { label: "20 篇", description: "扩展版" }
      ]
    },
    {
      header: "输出语言",
      question: "摘要使用什么语言？",
      options: [
        { label: "中文 (Recommended)", description: "摘要翻译为中文" },
        { label: "English", description: "保持英文原文" }
      ]
    }
  ]
})
```

### Step 1b: AI API Key（Anthropic）

如果 `.env` 中已包含 `ANTHROPIC_API_KEY`，或配置中已有 `anthropicApiKey`，跳过此步。

否则询问：

```
question({
  questions: [{
    header: "Anthropic API Key",
    question: "请提供 Anthropic API Key\n\n获取方式：访问 https://console.anthropic.com 获取 API Key",
    options: []
  }]
})
```

如果 `config.anthropicApiKey` 已存在，跳过此步。

### Step 2: 执行脚本

```bash
# 从 .env 读取输出目录，若未设置则默认 ./output
OUTPUT_DIR="${DIGEST_OUTPUT_DIR:-./output}"
mkdir -p "$OUTPUT_DIR"

# 从 ${SKILL_DIR}/.env 或 config 中获取的值
export ANTHROPIC_API_KEY="<key>"
# 可选：自定义 API 端点（从 .env 或 config 获取，默认 https://api.anthropic.com）
export ANTHROPIC_API_BASE="<base_url_if_set>"
export DIGEST_OUTPUT_DIR="$OUTPUT_DIR"

npx -y bun ${SKILL_DIR}/scripts/digest.ts \
  --hours <timeRange> \
  --top-n <topN> \
  --lang <zh|en> \
  --output ${OUTPUT_DIR}/digest-$(date +%Y%m%d).md
```

### Step 2b: 保存配置

```bash
mkdir -p ~/.hn-daily-digest
cat > ~/.hn-daily-digest/config.json << 'EOF'
{
  "anthropicApiKey": "<key>",
  "timeRange": <hours>,
  "topN": <topN>,
  "language": "<zh|en>",
  "lastUsed": "<ISO timestamp>"
}
EOF
```

### Step 3: 结果展示

**成功时**：

- 📁 报告文件路径
- 📊 简要摘要：扫描源数、抓取文章数、精选文章数
- 🏆 **今日精选 Top 3 预览**：中文标题 + 一句话摘要

**报告结构**（生成的 Markdown 文件包含以下板块）：

1. **📝 今日看点** — AI 归纳的 3-5 句宏观趋势总结
2. **🏆 今日必读 Top 3** — 中英双语标题、摘要、推荐理由、关键词标签
3. **📊 数据概览** — 统计表格 + Mermaid 分类饼图 + 高频关键词柱状图 + ASCII 纯文本图（终端友好） + 话题标签云
4. **分类文章列表** — 按 6 大分类（AI/ML、安全、工程、工具/开源、观点/杂谈、其他）分组展示，每篇含中文标题、相对时间、综合评分、摘要、关键词

**失败时**：

- 显示错误信息
- 常见问题：API Key 无效、网络问题、RSS 源不可用

---

## 参数映射

| 交互选项 | 脚本参数      |
| -------- | ------------- |
| 24 小时  | `--hours 24`  |
| 48 小时  | `--hours 48`  |
| 72 小时  | `--hours 72`  |
| 7 天     | `--hours 168` |
| 10 篇    | `--top-n 10`  |
| 15 篇    | `--top-n 15`  |
| 20 篇    | `--top-n 20`  |
| 中文     | `--lang zh`   |
| English  | `--lang en`   |

---

## 环境要求

- `bun` 运行时（通过 `npx -y bun` 自动安装）
- `ANTHROPIC_API_KEY`（必需）
- 可选：`ANTHROPIC_API_BASE`（自定义 API 端点，默认 `https://api.anthropic.com`）
- 可选：`DIGEST_OUTPUT_DIR`（自定义输出目录，默认 `./output`）
- 网络访问（需要能访问 RSS 源和 AI API）

---

## 信息源

90 个 RSS 源来自 [Hacker News Popularity Contest 2025](https://refactoringenglish.com/tools/hn-popularity/)，由 [Andrej Karpathy 推荐](https://x.com/karpathy)。

包括：simonwillison.net, paulgraham.com, overreacted.io, gwern.net, krebsonsecurity.com, antirez.com, daringfireball.net 等顶级技术博客。

完整列表内嵌于脚本中。

---

## 故障排除

### "ANTHROPIC_API_KEY not set"

需要提供 Anthropic API Key，可在 https://console.anthropic.com 获取。

### "Anthropic API error"

检查 API Key 是否有效、是否有足够配额。可通过设置 `ANTHROPIC_API_BASE` 环境变量使用自定义端点。

### "Failed to fetch N feeds"

部分 RSS 源可能暂时不可用，脚本会跳过失败的源并继续处理。

### "No articles found in time range"

尝试扩大时间范围（如从 24 小时改为 48 小时）。
