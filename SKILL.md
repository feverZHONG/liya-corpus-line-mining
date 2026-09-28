---
name: corpus-line-mining
tier: T2  # T分级: T2=直接做 / T1=先请示 / T0=一律拒
description: 要从本地语料/旧稿堆批量挖可复用素材（原句、场景卡）时用。会话库直取+拆句筛选+旧稿判笔分级+人审落库。
---

# 本地语料挖句 · corpus-line-mining

> 从「自己/角色的历史发言」里捞可复用原句。**原句优先于编造**——能溯源到真实语句的，绝不用现编的凑。

## 什么时候用

- 给台词库/角色卡/桌宠填素材，要找角色真说过的话
- 收语录、找证据句、回溯「我当初怎么说的」
- 写东西要找语气样本（不是靠记忆回忆，是拿库里的原句）
- **一堆旧稿／AI 生成的稿要变成能用的东西**（判谁写的 → 素材分级卡片 → 重排大纲）：走 `skill: draft-archaeology`（旧稿考古正本，2026-09-25 合并后本 skill 不再自持该档）

## 数据根是什么

本 skill 说的**数据根**＝放 `skills/` 与 `state.db` 的那一层目录：脚本按 `LIYA_DATA_ROOT` 环境变量取，
没设就从脚本位置往上找含 `skills/` 的那层，再兜底家目录。下面的配方里 `<数据根>` 照自己的环境替换。

## 语料源优先级

| 优先级 | 源 | 特点 |
| --- | --- | --- |
| 1 | **本机会话库** `<数据根>/state.db` | 最富最真，可批量；历史对话全在 |
| 2 | 日记 / 短篇 / 作品全文 | 第一人称，语气真但多非对白，拆成短句当语气样本 |
| 3 | 现成语录档 | 多为「值得思考的句子」，**先甄别是不是角色台词**再采 |
| 4 | 现编 | 最后手段；编出来的一句就是一份污染 |

## 会话库直取配方

sqlite **只读打开**（`file:<数据根>/state.db?mode=ro`）。表：`messages`（role / content / tool_calls / timestamp）、`sessions`（source / chat_type / title）。

```python
con = sqlite3.connect("file:<数据根>/state.db?mode=ro", uri=True)   # 换环境改成你的数据根
rows = con.execute("""
 select m.content, s.chat_type from messages m join sessions s on s.id=m.session_id
 where m.role='assistant' and m.content is not null
   and length(m.content) between 6 and 2000
   and (m.tool_calls is null or m.tool_calls in ('','[]'))
   and m.content not like '%```%' and m.content not like '%http%'
   and s.source='qqbot'
""").fetchall()
```

- **别把整条消息当一句话**：金句藏在长回复里 → 先按 `。！？!?\n` **拆句**，再按长度筛（6~34 字最像可复用句子）。
- **两轮过滤**：先滤任务型回复（推送/提交/脚本/文件/日志/索引/配置/归档/接口/编号…）与 `Operation interrupted` 类系统残句；再按口吻标志（自称、对用户的称呼、语气词）筛一遍。
- **`chat_type` 分 dm / group**：群聊是对外口气，与私聊音色不同，先按 dm 出稿。
- **规模参考**：10 万条消息 → 拆句约 1.4 万 → 口吻筛选 300 余条 → 首批入库 20~40 条。**机器筛到几百条就够，最终采用的每一条必须人过。**
- 候选池落盘 JSON（`cache/<主题>-candidates.json`），便于二次筛和复现。

## 标注与落库

- 素材行两种形态：**带目标格标注**（`[场景|情绪] 台词`，key 顺序必须与配置的维度顺序一致）＝可自动入库；**不带前缀**＝落进「待归类」清单。
- 不确定归属的句子**不加前缀**，别硬塞格子——硬塞比留空更贵。
- 用 `#` 开头的行写来源与说明（解析器跳过 `#`，注释不会污染入库内容）。
- 分批落文件（`素材/<日期>-<来源>.md`），后续批次分文件追加，互不覆盖；**同一批来源要可回溯**（哪个源、哪次筛选）。
- **空格子不硬凑**：真实语料没覆盖的格子留空并如实上报，格子有洞是数据现状。

## 改配置类产物：先沙盒，后正式

素材入库一类操作会**改配置**（如 `--apply` 写 `台词配置.json`，自动备份 `.bak`），照旧先复制副本跑完整链路：

```bash
cp -r <项目目录> <数据根>/cache/<项目>-sandbox
cd <数据根>/cache/<项目>-sandbox && <入库命令 --apply> \
  && <质量闸命令> && <生成命令> && <覆盖率/验证命令>
```

报告固定给三个数字：**产物条数变化 + 质量闸结果 + 覆盖率**，用户点头再对正式库跑同一串。

## 坑

- **`session_search` 能找会话、能翻窗口，但不能数数、不能批量**——要全量/要统计一律回 `state.db` 跑 SQL。
- **`messages.timestamp` 是 UTC epoch**——出口先 +8。
- **过滤词表按用途调，不要一套通吃**：任务型词表用来「排噪」、口吻词表用来「提精」，混用会同时误杀和漏网。
- **候选池 ≠ 采用的句子**：中间必须有人审那一步，别把筛选结果直接当成品交。

## 相关

- 台词系统本身（画格子/生成引擎/覆盖率）：`dialogue-system-builder`（**用户自有**，只读参考，改动前问用户）
- 会话库做「声音漂移审计」（称呼率／自称率／断点）：`voice-drift-audit`
- 语录入档流程与注释双层规范：`quotes-archive`
- 旧稿堆／共创对话的判笔、素材卡分级、大纲重排、**现稿写到一半回头挖料的四档产出**：姊妹 skill `draft-archaeology`（原 `references/legacy-draft-cards.md` 已并入它的 `ai-hand-fingerprints.md`／`outline-confirmation.md`／`mining-and-yield.md`）

> 上列姊妹 skill 都是**作者环境**的另一套工具（未单独公开）；本仓自包含的部分是上方「语料源 / 会话库直取 / 标注与落库 / 沙盒」那几节。
