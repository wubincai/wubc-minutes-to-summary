---
name: wubc-minutes-to-summary
description: >
  飞书妙记录音 → 智能纪要自动化流水线。自动检索指定时间范围的录音（默认前一天），
  逐条生成内容详尽、还原度高的结构化纪要（含开篇画板），然后根据纪要内容关键词
  自动匹配归档到"2026年工作计划"下对应的项目子节点，未匹配的归入"智能纪要"节点。
agent_created: true
read_when:
  - 用户说"把昨天的录音整理成纪要"、"检索录音生成纪要"、"把录音转成智能纪要"
  - 用户说"整理一下最近几天的会议录音"、"把这几天的录音写成纪要"
  - 用户说"帮我把XX日的录音整理成智能纪要"、"检索一下XX的录音"
---

# wubc-minutes-to-summary — 录音→智能纪要自动化（2026-07-12 优化版）

## 核心能力

四阶段流水线：

0. **前置准备** — 获取项目节点映射表
1. **检索** — 按时间范围搜索飞书妙记（默认前一天），逐条循环处理（含翻页）
2. **生成** — 生成内容详尽、还原度高的结构化纪要文档（含开篇飞书画板）
3. **归档** — 从纪要内容提取关键词，与"2026年工作计划"下的项目子节点匹配后归档

**匹配规则**：能匹配到项目子节点 → 作为子文档放到项目节点下；未匹配 → 作为子文档放到"智能纪要"节点下。

---

## 完整工作流

### Step 0：前置准备 — 获取项目节点映射表

在检索录音前，先获取"2026年工作计划"下所有子节点的列表，构建项目节点映射表，用于后续匹配归档。

#### 0.1 获取项目类节点列表

```bash
lark-cli wiki nodes list \
  --params '{"space_id":"7490053072341843987","parent_node_token":"XKQDwn4HaiTo9NkJz5RcYWMsnRg","page_size":50}' \
  --as user --format json
```

从返回的 `data.items` 中筛选出 `title` 以 `"项目："` 开头的节点：

| 节点 token | 节点标题 | 关键词区域 |
|:-----------|:---------|:----------|
| `J2scwL46GiVhbVkQWKScfDhhnwc` | 项目：定制商品开发 | 定制商品、选品、OEM、供应链、包装设计、开发 |
| `RK3dwcpdsi0xlVkFOb9cZHRonBh` | 项目：会员中心开发 | 会员、积分、权益、运营、会员系统、会员中心 |
| `NL6GwJ920iTgWZk7zFgcOWSenah` | 项目：车生态平台推广 | 车生态、养车、台州、洗车、维保、车服 |
| `IzPbwzTCJieUhgkfgcOcGqKMnha` | 项目：网红门店打造 | 网红、门店、线下、试点、陈列、网红项目 |
| `UafGwNwRUiZTWZkTncWcTnzunsc` | 项目：四大场景转化提升 | 场景、转化、加油站、流量、用户转化、场景 |
| `H7Gqw0aNIi88vXkO3fWc1r86nKM` | 项目：易捷速购探索提升 | 速购、线上、即时零售、便利店、O2O、易捷速购 |
| `FkBGwIDIOimhYQk53RsczREtnub` | 项目：分公司薪酬指标管理 | 薪酬、指标、绩效、考核、分公司、管理 |

**构建匹配索引**：用 Python 构建 `{节点token: 关键词列表}` 映射表。节点标题本身的关键词（如"定制商品"、"会员中心"、"车生态"等）也作为高优先级匹配词加入。此映射表在整个流水线中复用。

#### 0.2 获取"智能纪要"节点 token（兜底节点）

```bash
lark-cli wiki nodes list \
  --params '{"space_id":"7490053072341843987","page_size":50}' \
  --as user --format json
```

从返回中找到 `title == "智能纪要"` 的节点，提取 `node_token`（当前值为 `Nf9AwNeU8iKWIRk0rEbcNXPvn7b`）。不存在则创建：

```bash
lark-cli wiki +node-create \
  --params '{"space_id":"7490053072341843987","parent_node_token":"","title":"智能纪要","obj_type":"docx"}' \
  --as user --format json
```

**缓存机制**：将查到的 `fallback_node_token` 和项目映射表保存到 Python 变量或临时文件 `/tmp/minutes_node_map.json` 中，整个流水线过程中复用。

---

### Step 1：检索录音

```bash
lark-cli minutes +search --start <YYYY-MM-DD> --end <YYYY-MM-DD> --as user --format json
```

**时间规则**：
- 用户指定日期 → 按指定日期（start 和 end 相同）
- 用户指定范围 → 按范围
- 用户未指定 → 默认前一天

**输出格式**：

```json
{
  "ok": true,
  "data": {
    "has_more": false,
    "items": [
      {
        "create_time": "1735689600",
        "minute_token": "abc123",
        "obj_type": "minutes",
        "owner_id": "ou_xxx",
        "status": "2",
        "title": "XXX会议讨论",
        "url": "..."
      }
    ],
    "page_token": "xxxx"
  }
}
```

**分页处理**：当 `has_more == true` 时，传入上一页的 `page_token` 翻页：

```bash
lark-cli minutes +search --start <start> --end <end> --as user --format json \
  --params '{"page_token":"<上页page_token>"}'
```

循环直到 `has_more == false`，合并所有 `items`。

**结果处理（循环入口）**：

取出所有 `items`，逐条执行 Step 2 → Step 3。0 条时直接告知用户结束。

---

### Step 2：生成智能纪要

#### 2.1 获取逐字稿

```bash
lark-cli vc +notes --minute-tokens <minute_token> --as user --format json
```

**JSON 输出处理策略**：

- 优先检查 `code == 0` 或 `code == 200` 的状态码
- 递归查找 transcript 文本（可能在 `data.transcript_content`、`data.transcript` 或 `artifacts` 下）
- 元数据（标题、时长）在 `data.meta` 下
- 参会人信息在 `data.speakers` 或 `data.participants`

**提取的元数据**：会议主题、录音时间、参会人列表、逐字稿全文。

#### 2.2 深度分析

按以下步骤分析（不向用户展示过程）：

**① 主题拆分**：3-5 个核心议题，按决策点命名。每个主题包含背景、各方观点、分歧、结论。

**② 关键信息提取**：保留所有原始数据（金额、百分比、日期、公司名、产品名、人名、数量、单价、时长、频次）。不做四舍五入或模糊化。

**③ 参会人识别**：从 `speakers` 提取姓名/角色。无法识别时标注"说话人N（角色）"。

**④ 画板类型选择**：
- 方案比较 → 五区布局 + 底部对比表
- 下一步行动 → 五区布局，底部改为流程箭头
- 多方关系 → 五区，第三区改为关系卡，无底部表格

**⑤ 待办识别**：只提取有明确责任方且可执行的行动项。

**⑥ 匹配项目判断**

从逐字稿提取 3-5 个关键词，与 Step 0 的节点映射表匹配：

**匹配算法**（按优先级）：
1. **精确匹配**：关键词完全匹配节点关键词区域中的词
2. **包含匹配**：逐字稿中包含节点关键词区域中的词
3. **语义关联**：如"选品讨论"→ 匹配"定制商品开发"

匹配结果记入 `target_project_token`。

#### 2.3 构建内容详尽的纪要内容

**核心原则：还原度要高、内容要丰富**

- 正文用**自然语言叙述**，而非简单要点罗列
- 每个主题应有 **背景 → 讨论过程 → 分歧/关键决策 → 结论** 的完整逻辑链
- 保留原话措辞风格和语气，关键判断用引号标注
- 分主题段落用 3-5 句展开
- 所有数字、金额、比例原样保留
- 每个主题下 2-4 个子要点用 `<b>` 标注关键词

#### 2.4 构建文档 XML 并创建

使用 `lark-cli docs +create` 命令，`--api-version v2` 是默认值无需传入。

**注意：DocxXML 标签使用标准 HTML 标签名**，详细规则参考 `lark-cli skills read lark-doc references/lark-doc-xml.md`。

**文档结构顺序（固定）**：

```
<title> → blockquote(时间) → blockquote(参会人) → p空行 → h1(核心概述) → <whiteboard type="blank"/> → p概述 → p空行 → h1(会议要点) → h2主题1 → ul+li子要点 → h2主题2 → ... → p空行 → h1(待办事项) → checkbox列表
```

**✱ 关键标签对照表**：

| 你想用的标签 | 正确写法 | 说明 |
|:------------|:---------|:-----|
| `<heading1>` | `<h1>` | 一级标题用 HTML h1 |
| `<heading2>` | `<h2>` | 二级标题用 HTML h2 |
| `<bold>` | `<b>` | 加粗用 HTML b |
| `<document><head><body>` | **不要** | 直接写内容，无需包裹 |
| `<?xml ...>` | **不要** | 不写 XML 声明 |
| `<checkbox status="unchecked">` | `<checkbox done="false">` | 用 `done` 属性 |
| `--title "xxx"` | `<title>xxx</title>` | 标题放在内容中 |
| `<grid column_size="50|50">` | `<grid><column width-ratio="0.5">` | grid 分栏格式 |

**XML 模板**：

```xml
<title>智能纪要：{主题} {YYYY年M月D日}</title>

<blockquote>时间：{YYYY年M月D日 HH:mm}</blockquote>
<blockquote>参会人：{张三、李四、王五……}</blockquote>

<h1>一、核心概述</h1>
<whiteboard type="blank"></whiteboard>

<p>{3-5 句自然段落概述，包含会议背景、核心议题、总体结论}</p>

<h1>二、会议要点</h1>

<h2>1. {议题名称}</h2>
<p>{自然段落：背景→观点/分歧→结论，3-5 句}</p>
<ul>
  <li><b>{子要点1}</b>：{详细说明}</li>
  <li><b>{子要点2}</b>：{详细说明}</li>
</ul>

<h2>2. {议题名称}</h2>
<!-- 对比类议题嵌入 grid 双栏布局 -->
<grid>
  <column width-ratio="0.5">
    <p>{左栏文字要点}</p>
  </column>
  <column width-ratio="0.5">
    <table>
      <colgroup><col span="2" width="120"/></colgroup>
      <thead><tr><th>维度</th><th>方案A</th><th>方案B</th></tr></thead>
      <tbody>
        <tr><td>成本</td><td>{值}</td><td>{值}</td></tr>
        <tr><td>周期</td><td>{值}</td><td>{值}</td></tr>
      </tbody>
    </table>
  </column>
</grid>

<h1>三、待办事项</h1>
<checkbox done="false">责任人：张三 — {具体行动项}，截止：{日期}</checkbox>
<checkbox done="false">责任人：李四 — {具体行动项}，截止：{日期}</checkbox>
```

**创建命令**：

```bash
xml_content="..." && lark-cli docs +create --content "$xml_content" --as user --format json
```

**提取画板 token**：从返回的 `data.document.new_blocks` 中找 `block_type == "whiteboard"`，取 `block_token`。无需额外 fetch。

#### 2.5 绘制开篇画板

飞书画板 DSL v2 格式，五区布局（仅用文本节点实现）。

**✱ DSL v2 关键约束**（实测验证）：
- `"version"` 必须是数字 `2`，不是字符串 `"2.0"`
- 节点类型**只支持** `"type": "text"`，不支持 `rectangle` / `shape` / `group` 类型
- 画板用文本节点通过大小、颜色、加粗和 emoji 实现视觉分层
- 无法绘制背景矩形/卡片边框，靠文本定位和 emoji 分隔符实现视觉区域划分
- 用 Python `json.dumps(ensure_ascii=False)` 生成 DSL JSON

**标准结构（文本节点实现）**：

```
┌─ 大标题（fontSize:28, 居中, 粗体）───────────────────
│  💡 核心结论（fontSize:16, 黄色）
│  • 结论1（fontSize:13, 粗体）
│  • 结论2
│  • 结论3
├─ ═══ 当前方案问题 ═══（#5178C6 蓝色分隔标题）
│  3-4条文字要点（fontSize:12, 左对齐）
├─ ═══ 改进方案 ═══（#8569CB 紫色分隔标题）
│  3-4条文字要点（fontSize:12, 左对齐，用✅标记）
├─ ═══ 系统与转型 ═══（#509863 绿色分隔标题）
│  3-4条文字要点
├─ ═══════ 前后方案对比 ═══════（粗体居中分隔线）
│  对比维度 | 现行方案（红色）| 改进方案（绿色）
│  5行对比数据（文本对齐排列）
```

**内容原则**：
- 核心结论区必须写**判断句**（不是"讨论了XX"，而是"确定采用XX方案"）
- 保留所有关键数字，不模糊化
- 分隔标题用 `══════` 符号 + 颜色区分不同区域

**写入画板命令**：

```bash
npx -y @larksuite/whiteboard-cli@^0.2.11 -i diagram.json --to openapi --format json \
  | lark-cli whiteboard +update --whiteboard-token <board_token> \
      --source - --input_format raw --idempotent-token <MMDD_主题缩写_序号> --overwrite --as user
```

---

### Step 3：智能归档

使用 Step 2 中匹配到的 `target_project_token` 决定归档位置。

#### 3.1 在目标节点下创建子文档

注意：`--parent-token` 参数使用 `target_project_token`（可能是项目节点或智能纪要节点）。

```bash
lark-cli docs +create --content "<XML_CONTENT>" \
  --parent-token <target_project_token> --as user --format json
```

#### 3.2 写入新文档的画板

同上 2.5 步骤，提取新文档的 `board_token` 后写入。

---

## 循环处理流程

```
Step 0 → 构建项目节点映射表 → 完成
    |
    +-> Step 1 检索 -> 共N条录音
    |    |
    |    +- [第1条] Step 2 生成纪要(含匹配判断)
    |    |            -> Step 3 归档到对应节点 -> 完成
    |    +- [第2条] Step 2 生成纪要(含匹配判断)
    |    |            -> Step 3 归档到对应节点 -> 完成
    |    ...
    |    +- [第N条] Step 2 -> Step 3 -> 完成
    |
    +-> 全部完成 -> 输出汇总
```

**进度汇报规则**：
- 第一条开始前告知用户共找到几条录音，即将逐条处理
- 每条完成后输出：该条标题 + 匹配到的节点名称 + 文档链接
- 全部完成后输出汇总统计（成功/失败 + 每个节点的文档列表）

---

## 容错与重试机制

| 故障类型 | 处理策略 |
|:---------|:---------|
| 网络超时 | 重试 3 次，间隔 5 秒 |
| API 非 200 | 检查 error，临时错误重试 2 次 |
| 命令 exit 非零 | 输出 error 信息，跳过该条 |
| 逐字稿为空 | 标记"无有效内容"，跳过该条 |
| 画板写入失败 | 标记"画板未写入"，文档保留 |
| 节点不存在 | 回退到"智能纪要"节点 |

---

## 环境信息

| 项目 | 值 |
|:------|:-----|
| 工作区知识库 space_id | `7490053072341843987` |
| 项目类父节点（2026年工作计划） | `XKQDwn4HaiTo9NkJz5RcYWMsnRg` |
| 默认归档节点（智能纪要） | `Nf9AwNeU8iKWIRk0rEbcNXPvn7b` |
| lark-cli 身份 | `--as user` |
| 画板写入身份 | `--as user`（已验证可行） |
| whiteboard-cli | `@larksuite/whiteboard-cli@^0.2.11` |

---

## 质量检查清单

- [ ] Step 0 项目节点映射表已构建（7 个项目 + 1 个兜底）
- [ ] 妙记已检索，日期范围正确，分页已处理
- [ ] 逐字稿已获取，元数据已提取（主题、时间、参会人）
- [ ] 深度分析完成：3-5 个主题、所有关键数字、画板类型、匹配项目
- [ ] 匹配算法已执行，target_project_token 已确定
- [ ] 标题：`智能纪要：{主题} {YYYY年M月D日}`
- [ ] 内容还原度高：段落是自然语言叙述（背景→讨论→结论），非简单要点
- [ ] 所有原始数字/金额/公司名/人名已保留
- [ ] XML 标签使用正确：`h1`/`h2`/`b`（非 heading1/heading2/bold），`checkbox done="false"`
- [ ] 画板 DSL v2 只用 `type: "text"` 节点，`version` 为数字 2
- [ ] 待办有明确责任人 + 行动项 + 截止日期
- [ ] 文档创建在对应项目节点下（或智能纪要节点下）
- [ ] 画板已写入新文档
- [ ] 每条完成后输出了节点匹配信息 + 文档链接
