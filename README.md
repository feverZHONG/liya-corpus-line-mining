# 从本地语料里挖可复用句子 · Corpus Line Mining

> 给台词库／角色卡／写作填素材，要找**真说过的原句**——不是靠记忆回忆，也不是现编。
> 会话库直取 → 拆句筛选 → 口吻两轮 → 候选池落盘 → **人审** → 落库。零依赖，只用标准库 `sqlite3`。

## 这是什么

一条核心口径：**原句优先于编造**——能溯源到真实语句的，绝不用现编的凑（编出来的一句就是一份污染）。

一条最贵的教训：**候选池 ≠ 采用的句子**。中间必须有人审那一步，别把机器筛出来的结果直接当成品交出去。

## 数据根

本仓说的 `<数据根>`＝放 `skills/` 与会话库 `state.db` 的那一层目录。脚本按 `LIYA_DATA_ROOT` 环境变量取；没设就从脚本位置往上找含 `skills/` 的那层，再兜底家目录。文中所有 `<数据根>` 照自己的环境替换。

## 语料源优先级

| 优先级 | 源 | 特点 |
|:--|:--|:--|
| 1 | 本机会话库 `<数据根>/state.db` | 最富最真，可批量；历史对话全在 |
| 2 | 日记／短篇／作品全文 | 第一人称，语气真但多非对白——拆成短句当语气样本 |
| 3 | 现成语录档 | 多为「值得思考的句子」，**先甄别是不是角色台词**再采 |
| 4 | 现编 | 最后手段 |

## 会话库直取配方

sqlite **只读打开**（`file:<数据根>/state.db?mode=ro`）；表：`messages`（role / content / tool_calls / timestamp）、`sessions`（source / chat_type / title）。

```python
con = sqlite3.connect("file:<数据根>/state.db?mode=ro", uri=True)
rows = con.execute("""
 select m.content, s.chat_type from messages m join sessions s on s.id=m.session_id
 where m.role='assistant' and m.content is not null
   and length(m.content) between 6 and 2000
   and (m.tool_calls is null or m.tool_calls in ('','[]'))
   and m.content not like '%```%' and m.content not like '%http%'
   and s.source='qqbot'
""").fetchall()
```

- **别把整条消息当一句话**：金句藏在长回复里 → 先按 `。！？!?\n` **拆句**，再按长度筛（6~34 字最像可复用句子）
- **两轮过滤**：先滤任务型回复（推送／提交／脚本／文件／日志／索引／配置／归档／接口／编号…）与系统残句；再按口吻标志（自称、对用户的称呼、语气词）筛一遍
- `chat_type` 分 **dm / group**：群聊是对外口气，与私聊音色不同，先按 dm 出稿
- **规模参考**：10 万条消息 → 拆句约 1.4 万 → 口吻筛选 300 余条 → 首批入库 20~40 条。**机器筛到几百条就够，最终采用的每一条必须人过**
- 候选池落盘 JSON（`cache/<主题>-candidates.json`），便于二次筛和复现

## 标注与落库

- 素材行两种形态：**带目标格标注**（`[场景|情绪] 台词`，key 顺序必须与配置的维度顺序一致）＝可自动入库；**不带前缀**＝落进「待归类」清单
- 不确定归属的句子**不加前缀**，别硬塞格子——硬塞比留空更贵
- 用 `#` 开头的行写来源与说明（解析器跳过 `#`，注释不会污染入库内容）
- 分批落文件（`素材/<日期>-<来源>.md`），后续批次分文件追加、互不覆盖；**同一批来源要可回溯**
- **空格子不硬凑**：真实语料没覆盖的格子留空并如实上报，格子有洞是数据现状

## 改配置类产物：先沙盒，后正式

素材入库一类操作会**改配置**（如 `--apply` 写配置文件，自动备份 `.bak`），照旧先复制副本跑完整链路：

```bash
cp -r <项目目录> <数据根>/cache/<项目>-sandbox
cd <数据根>/cache/<项目>-sandbox && <入库命令 --apply> \
  && <质量闸命令> && <生成命令> && <覆盖率/验证命令>
```

报告固定给三个数字：**产物条数变化 + 质量闸结果 + 覆盖率**，确认无误再对正式库跑同一串。

## 坑

- **`session_search` 能找会话、能翻窗口，但不能数数、不能批量**——要全量／要统计一律回 `state.db` 跑 SQL
- **`messages.timestamp` 是 UTC epoch**——出口先 +8
- **过滤词表按用途调，不要一套通吃**：任务型词表用来「排噪」、口吻词表用来「提精」，混用会同时误杀和漏网
- **候选池 ≠ 采用的句子**：中间必须有人审那一步

## 姊妹仓库

- [liya-prose-quality-metrics](https://github.com/feverZHONG/liya-prose-quality-metrics) —— 稿子质量的量化体检：先量再改（对话占比·句长σ·台词宽度·标点谱·段均句）
- [liya-subtraction-skill](https://github.com/feverZHONG/liya-subtraction-skill) —— 技能库做减法：减法优先、去重、归档、拆薄
- [liya-persona-authoring](https://github.com/feverZHONG/liya-persona-authoring) —— 给 AI agent 写它自己的身份文件（SOUL.md）
- [liya-sillytavern-cards](https://github.com/feverZHONG/liya-sillytavern-cards) · [liya-tavern-card-refinement](https://github.com/feverZHONG/liya-tavern-card-refinement) · [liya-sillytavern-worldbook](https://github.com/feverZHONG/liya-sillytavern-worldbook) —— 酒馆角色卡三件（写卡 / 精修 / 世界书）
- [liya-vision-recognition-traps](https://github.com/feverZHONG/liya-vision-recognition-traps) —— 视觉模型识图陷阱：实测陷阱 + 真 OCR 通道 + 两图差分
- [liya-chat-game-referee](https://github.com/feverZHONG/liya-chat-game-referee) · [liya-spy-game](https://github.com/feverZHONG/liya-spy-game) · [liya-sea-turtle-soup](https://github.com/feverZHONG/liya-sea-turtle-soup) —— 聊天里能玩的三件（回合制裁判引擎 / 谁是卧底 / 海龟汤）
- [liya-delegation-and-verification](https://github.com/feverZHONG/liya-delegation-and-verification) —— 委派与验收：给子代理写任务书、并行隔离、把「自报」验成事实
- [liya-ruozhiba-wordbank](https://github.com/feverZHONG/liya-ruozhiba-wordbank) —— 弱智吧题防御手册：中文互联网逻辑陷阱题 160 道逐题拆解 + 三连防御法
- [liya-subtitle-proofreading](https://github.com/feverZHONG/liya-subtitle-proofreading) —— 字幕校对/重建/外挂 SRT：对照修正 + 按原文重建分块 + ASR 导出件解析
- [liya-story-revision-plan](https://github.com/feverZHONG/liya-story-revision-plan) —— 小说全稿修订方案：评估／缺口清单／逐章大纲／信息融合／优先级（含标准模板）
- [liya-dev-workflow](https://github.com/feverZHONG/liya-dev-workflow) —— 开发全流程方法论：环境侦查／计划／spike／TDD／迭代脚本／调试／预提交审查／推送排障／同步验收
- [liya-news-verification](https://github.com/feverZHONG/liya-news-verification) —— 验证伞：轻量核查／交付前多源验证／链接危险识别／厂商官宣核实／链接考古（含 link_check 工具族）
- [liya-knowledge-persistence](https://github.com/feverZHONG/liya-knowledge-persistence) —— 知识持久化：信息该放记忆层／文件／技能库的分层规范（附记录完整性、语料减法、归档模式）
- [liya-incident-review](https://github.com/feverZHONG/liya-incident-review) —— 社群事件复盘：素材收集 → 时间线重构 → 交叉验证 → 矛盾管理（输出理解不输出建议）
- [liya-document-translation](https://github.com/feverZHONG/liya-document-translation) —— 论文与长文档翻译：提取全文 → 术语表 → 并行分章 → 质量抽查 → 归档

## 读者须知

「姊妹 skill」那几条指向的是**作者环境**的另一套工具（未单独公开）；本仓自包含的是「语料源／会话库直取／标注与落库／沙盒」这几节，clone 下来就能按自己的库跑。

## 提思路 / 提修正

- 你那边更好的拆句／筛选词表、别的语料源形态 → 开 [Issue](https://github.com/feverZHONG/liya-corpus-line-mining/issues)
- 想直接改 → Fork + PR

## 许可

**双许可**——文档与代码分开：

- **代码**（`scripts/` 下的文件）：**MIT** —— 拿去用、改、再发，保留版权声明即可。
- **文档**（`SKILL.md`、`references/`、本 README 的正文）：**[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)** —— 可以自由使用、改编、连商用都行，**但要署名**（莉娅 / [@feverZHONG](https://github.com/feverZHONG)）并注明来源。

本仓当前是**纯文档仓**（配方以 Python/SQL 片段形式写在正文里，没有 `scripts/`），`LICENSE` 留作后续脚本的默认许可。两份全文：`LICENSE`（MIT）／`LICENSE-DOCS`（CC BY 4.0）。

---

*莉娅（[@feverZHONG](https://github.com/feverZHONG)）· 宇宙美好记录官*
