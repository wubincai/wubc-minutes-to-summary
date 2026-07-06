---
name: wubc-minutes-to-summary
description: >
  飞书妙记录音 → 智能纪要自动化流水线。自动检索指定时间范围的录音（默认前一天），
  按 meeting-minutes-pro 工作流生成带开篇画板的结构化纪要文档，
  最后将文档作为子文档写入飞书"工作区"知识库下的"智能纪要"节点。
agent_created: true
read_when:
  - 用户说"把昨天的录音整理成纪要"、"检索录音生成纪要"、"把录音转成智能纪要"
  - 用户说"整理一下最近几天的会议录音"、"把这几天的录音写成纪要"
  - 用户说"帮我把XX日的录音整理成智能纪要"、"检索一下XX的录音"
---

# wubc-minutes-to-summary — 录音→智能纪要自动化

## 核心能力

三阶段流水线：

1. **检索** — 按时间范围搜索飞书妙记（默认前一天），逐条循环处理
2. **生成** — 按 meeting-minutes-pro 工作流生成带画板的纪要文档
3. **归档** — 写入飞书"工作区"知识库"智能纪要"节点

## 完整工作流

### Step 1：检索录音

```bash
lark-cli minutes +search --start <YYYY-MM-DD> --end <YYYY-MM-DD> --as user --format pretty
```

**时间规则**：
- 用户指定日期 → 按指定日期
- 用户指定范围 → 按范围
- 用户未指定 → 默认前一天

**结果处理（循环入口）**：

取出所有结果，逐条执行 Step 2 → Step 3。0 条时直接告知用户结束。

> **分页处理**：`minutes +search` 单次最多 200 条，如有 `has_more` 需翻页。按一天以内的场景一般不会超额。

### Step 2：生成智能纪要

完全遵循 meeting-minutes-pro 工作流：

#### 2.1 获取逐字稿

```bash
lark-cli vc +notes --minute-tokens <minute_token> --as user --format json
```

读取 `artifacts.transcript_file` 路径中的文件。

#### 2.2 深度分析

识别 3-5 个核心议题，提取所有关键数字，选择画板类型：
- **方案比较** → 五区布局 + 底部对比表
- **下一步行动** → 流程步骤卡 + 箭头
- **多方关系** → 关系卡，无表格

#### 2.3 构建文档 XML 并创建

按 meeting-minutes-pro 的固定结构：

```
标题 → blockquote(时间) → blockquote(参会人) → p空行 → h1核心概述 → <whiteboard type="blank"/> → p概述 → ul分主题 → p空行 → h1待办 → checkbox列表
```

每个主题的首个 `<li>` 用 `<b>` 粗体，保留所有关键数字。

```bash
xml_content="..." && lark-cli docs +create --content "$xml_content" --as user --format json
```

从 `data.document.new_blocks` 提取 `block_type == "whiteboard"` 的 `block_token`。

#### 2.4 绘制开篇画板

DSL v2 格式，五区布局：

```
┌─ 大标题（fontSize:28, 居中, 粗体）───────────────────┐
├─ 💡 核心结论区（#FFFBE6 黄色背景, 3条判断句）─────────┤
├─ 三栏卡片：[蓝 #EAF1FB] [紫 #EAE2FE] [黄绿 #F0F9E8]┤
│  每张 = 彩色标题栏(icon+标题) + 白底要点列表           │
├─ 底部对比表（有方案对比时必加, 表头 #1F2329 深色）──────┤
└──────────────────────────────────────────────────────┘
```

用 Python `json.dumps(ensure_ascii=False)` 生成 DSL JSON，避免字符冲突。

```bash
npx -y @larksuite/whiteboard-cli@^0.2.11 -i <diagram.json> -t openapi -F json \
  | lark-cli whiteboard +update --whiteboard-token <board_token> \
      --source - --input_format raw --idempotent-token <MMdd_topic_序号> --overwrite --as user
```

**关键经验**：用 `--as user` 写入画板（实战验证 user 可写）。



---

## 循环处理流程

```
检索 → 共N条录音，开始逐条处理
  │
  ├─ [第1条] Step 2 生成 → Step 3 归档 → 完成
  ├─ [第2条] Step 2 生成 → Step 3 归档 → 完成
  ...
  └─ [第N条] Step 2 生成 → Step 3 归档 → 完成

全部完成 → 输出汇总（成功/失败统计 + 所有文档链接）
```

**进度汇报规则**：
- 第一条开始前告知用户共找到几条录音，即将逐条处理
- 每条完成后输出该条标题和文档链接
- 全部完成后输出汇总统计

### Step 3：归档到智能纪要节点

#### 3.1 查找智能纪要节点（首次执行时查找，后续复用已找到的 node_token）

```bash
lark-cli wiki nodes list --params '{"space_id":"7490053072341843987","page_size":50}' --as user --format json
```

找到 `title == "智能纪要"` 的节点，提取 `node_token`。

#### 3.2 如不存在则创建

```bash
lark-cli wiki +node-create --params '{"space_id":"7490053072341843987","parent_node_token":"","title":"智能纪要","obj_type":"docx"}' --as user --format json
```

#### 3.3 在节点下创建子文档

```bash
lark-cli docs +create --content "<XML_CONTENT>" --parent-token <智能纪要node_token> --as user --format json
```

#### 3.4 写入新文档的画板

同上 2.4 步骤，提取新的 `board_token` 后写入。

## 环境信息

| 项目 | 值 |
|------|-----|
| 工作区知识库 space_id | `7490053072341843987` |
| lark-cli 身份 | `--as user`（已验证） |
| 画板写入身份 | `--as user`（已验证可行） |
| whiteboard-cli | `@larksuite/whiteboard-cli@^0.2.11` |

## 质量检查清单

- [ ] 妙记已检索，日期范围正确
- [ ] 逐字稿已获取
- [ ] 标题：`智能纪要：{主题} {YYYY年M月D日}`
- [ ] 含 blockquote(时间) + blockquote(参会人) + whiteboard 占位
- [ ] 画板：大标题 + 黄色结论区 + 三栏彩色卡片 + 底部对比表（可选）
- [ ] 原文数字/金额全部保留
- [ ] 待办有明确责任人
- [ ] 文档创建在"智能纪要"节点下（非独立文档）
- [ ] 画板已写入新文档
