# MEME 数据集文件内容说明

这是一个评估 LLM 长时记忆系统的 benchmark（**MEME: Multi-Entity and Evolving Memory Evaluation**），从「实体范围（单实体/多实体）」×「时间动态（静态/演化）」两个正交维度出题。整个仓库分两层：`dataset/` 是**发布用的最终数据集**，`code/data/` 是**生成数据集的可复现管道 + 中间产物**。

## 一、`dataset/`（最终交付数据集）

三个 JSON 文件是**同一 schema 的三个变体**，区别只在「干扰填充 token」的多少（见 [README.md](file:///e:/Projects/MEME-benchmark/dataset/README.md)）：

| 文件 | episode 数 | 填充 token | 用途 |
|------|-----------|-----------|------|
| [meme_nofiller.json](file:///e:/Projects/MEME-benchmark/dataset/meme_nofiller.json) | 100 | 无 | 只保留证据会话 |
| [meme_filler32k.json](file:///e:/Projects/MEME-benchmark/dataset/meme_filler32k.json) | 100 | ~32K | 默认设置 |
| [meme_filler128k.json](file:///e:/Projects/MEME-benchmark/dataset/meme_filler128k.json) | 40 | ~128K | 重噪声压力测试（filler32k 的子集） |

每个文件是一个 **episode 数组**，单个 episode 结构如下（对照 [meme_nofiller.json 第 1 个 episode](file:///e:/Projects/MEME-benchmark/dataset/meme_nofiller.json#L1-L258)）：

- `episode_id`：`pl_001`…`pl_050`（个人生活）或 `sw_001`…`sw_050`（软件项目）
- `domain`：`personal_life` 或 `software_project`
- `root`：级联链的根实体；`root_change` 是它 before→after 的变化（触发级联）
- `chain_entities` / `filler_entities` / `entities`：链上实体、无关实体、以及带 `{before, after, task, hop}` 的完整实体值表
- `tasks`：评测题列表，见下方任务类型
- `total_sessions` / `evidence_sessions` / `filler_sessions`：会话数统计
- `total_tokens` / `evidence_tokens` / `filler_tokens`：token 统计（tiktoken cl100k_base）
- `sessions`：按时间顺序排列的对话列表（`evidence` 型带 `gold_facts`，`filler` 型不带）
- `evidence_session_indices`：证据会话在 `sessions` 中的位置
- `before_questions` / `after_questions`：变化事件前/后的问题（用于 trivial-pass 过滤）

### 六种任务类型（`tasks[].type`）

| 标签 | 全称 | 象限 | 考察点 |
|------|------|------|--------|
| `ER` | Exact Recall | 单实体·静态 | 逐字复述静态事实 |
| `Agg` | Aggregation | 多实体·静态 | 汇总分散在多个会话里的多个事实 |
| `Tr` | Tracking | 单实体·演化 | 重建同一实体的多次修订历史（3 个有序值） |
| `Del` | Deletion | 单实体·演化 | 用户显式删除后不再报告该事实 |
| `Cas` | Cascade | 多实体·演化 | 沿依赖规则传播更新（`hop`=1 或 2） |
| `Abs` | Absence | 多实体·演化 | 无替换规则时识别「答案已不可信」（`hop`=1 或 2） |

其中 `Cas`/`Abs` 的 `hop` 字段表示依赖链深度（1 跳/2 跳），`Deletion`/`Cas`/`Abs` 的评分依赖 before_questions 的 trivial-pass 过滤。

## 二、`code/data/`（生成管道）

这是一个五步可复现管道，外加两个中间数据目录。核心脚本：

**[generate_episode.py](file:///e:/Projects/MEME-benchmark/code/data/generate_episode.py)** — 步骤 3：从 DAG 知识图谱中**采样子图 + 赋值 + 分配任务**。内部任务代号 A~H 与最终标签的对应：`A`=Cas、`B`=Abs、`C`=根实体更新、`E`=Del、`F`=Agg、`G`=ER、`H`=Tr。包含一致性规则（`CONSISTENCY_RULES`）、软件域的派生值（`DERIVED_VALUES`）。

**[generate_gold_facts.py](file:///e:/Projects/MEME-benchmark/code/data/generate_gold_facts.py)** — 步骤 4：把 episode 骨架转成**逐阶段的黄金事实种子**（`phase1_fact_introduction` / `phase2_before_questions` / `phase3_change_and_deletion` / `phase4_after_questions`）。含事实模板、依赖句式、if-then 句式（Cas 的前置声明）、删除句式。

**[generate_conversations.py](file:///e:/Projects/MEME-benchmark/code/data/generate_conversations.py)** — 步骤 4（verbalization）：用 **gpt-4o 自我对话**（User LLM / Assistant LLM 交替），把 third-person 事实转成自然多轮对话，并对每个论元做质量校验（annotate + if-then 规则校验 + Gemini 2.5 Flash 语义审计）。依赖/if-then/逐字事实走模板直出保证准确。

**[assemble_episodes.py](file:///e:/Projects/MEME-benchmark/code/data/assemble_episodes.py)** — 步骤 5：把证据会话与 filler 会话**交错组装**成最终 episode，按 token 预算填充。结构为 `[filler×N] 事实介绍 → before_questions → 变更/删除事件 → after_questions`。

**[filter_fillers.py](file:///e:/Projects/MEME-benchmark/code/data/filter_fillers.py)** — filler 池预过滤：用 **BM25 + text-embedding-3-small 混合检索 + GPT-4o-mini 裁判**，剔除与证据事实冲突（矛盾/替代/实体混淆）的 filler，防止污染。

**[verify_assembled.py](file:///e:/Projects/MEME-benchmark/code/data/verify_assembled.py)** — 本地校验组装结果（顺序、时间戳递增、问题位置、gold_facts 存在性等）。

### 中间数据目录

- **[entity_pools/](file:///e:/Projects/MEME-benchmark/code/data/entity_pools)** — 两个域的实体池。每个实体有 `id / definition / role / values`，`role` 分 `root`（根）、`middle`（1 跳中间节点）、`leaf`（叶子）、`orphan`（游离/可做 filler）。个人生活 39 个实体、软件项目 51 个，值全部虚构防参数化污染。
- **[dependency_maps/](file:///e:/Projects/MEME-benchmark/code/data/dependency_maps)** — 两个域的手工 DAG 依赖边（`source → target + pattern + declaration_example`）。个人生活 34 条边、软件项目 27 条。

## 三、`filler32k_pl/` 和 `filler32k_sw/`

这是**组装后的中间产物**（`assemble_episodes.py` 的输出，尚未合并成发布 JSON 的逐 episode 文件）：

- `filler32k_pl/`：50 个 **Personal Life** episode（`episode_001.json`~`episode_050.json`）
- `filler32k_sw/`：50 个 **Software Project** episode（`episode_001.json`~`episode_050.json`）

每个文件结构与最终 `meme_filler32k.json` 里单个 episode 一致，只是 `episode_id` 还是数字（1~50），而发布版给它们加上了 `pl_`/`sw_` 前缀并合并为一个数组（见 [filler32k_pl/episode_001.json](file:///e:/Projects/MEME-benchmark/code/data/filler32k_pl/episode_001.json#L1-L120) 与 [meme_nofiller.json](file:///e:/Projects/MEME-benchmark/dataset/meme_nofiller.json#L1-L4) 的对照）。

## 整体数据流

```
entity_pool.json ─┐
                  ├─→ generate_episode.py ─→ episode_XXX.json（骨架）
dependency_map.json┘                              │
                                                  ▼
                                  generate_gold_facts.py → gold_facts_XXX.json
                                                  │
                                                  ▼
                          generate_conversations.py（LLM self-chat）→ conversations_XXX.json
                                                  │
                  filler 池（外部下载）──filter_fillers.py──┘
                                                  ▼
                                  assemble_episodes.py → filler32k_pl/sw/*.json
                                                  ▼
                              合并/加前缀 → dataset/meme_*.json（发布）
```


# 填充 token

「填充 token」（filler tokens）指的是 benchmark 里**与评测无关的干扰对话**所占据的 token 数量。

## 为什么需要它

MEME 评测的核心是"长上下文中的记忆检索"。如果评测数据只包含用户明确陈述过的证据事实（evidence），那么任何模型只要把对话读完就能轻易回答，测不出"记忆能力"的差异。

所以生成器在证据会话（evidence sessions）之间**穿插了大量无关的普通闲聊会话**（filler sessions），用 token 把关键事实"淹没"在长长的上下文里，模拟真实场景：AI 助手要在一堆和本次提问无关的历史对话中，准确记住哪几条才是有用的。

## 在数据里的体现

每个 episode 的 token 统计分三类（见 [assemble_episodes.py](file:///e:/Projects/MEME-benchmark/code/data/assemble_episodes.py#L505-L514)）：

- `evidence_tokens`：证据会话的 token —— 和评测问题直接相关，是"针"
- `filler_tokens`：填充会话的 token —— 无关闲聊，是"海"
- `total_tokens` = 两者之和

```
[填充×N] → 事实介绍 → [填充×N] → 变更/删除事件 → [填充×N]
  ↑ 干扰           ↑ 证据              ↑ 干扰
```

## 三个变体就是靠它区分的

| 配置 | filler tokens | 含义 |
|------|--------------|------|
| `nofiller` | 0 | 只有证据，没有干扰（最易） |
| `filler32k` | ~32K | 默认，中等干扰 |
| `filler128k` | ~128K | 大量干扰，压力测试 |

`filler128k` 只有 40 个 episode，是因为 128K 上下文的评测成本太高，所以从 filler32k 里取子集并加大干扰量。

## 一个关键约束：填充不能"污染"

填充会话不是随便塞的。如果某个 filler 恰好也聊到了"我爱人在哪上班"这种和证据实体相关的内容，就会给模型制造混淆甚至矛盾答案。所以管道里有专门的 [filter_fillers.py](file:///e:/Projects/MEME-benchmark/code/data/filter_fillers.py) 做过滤（BM25 + embedding 混合检索 + LLM 裁判），确保填充会话与证据事实**无冲突、无替代答案、无实体混淆**。

简单说：**filler tokens = 用来增加检索难度的"噪声" token 总量，是 benchmark 控制任务难度的核心旋钮。**


# 数据使用流程

README 里的「运行过程」是指**评测（eval）流程**：先把发布数据集裁剪成评测器能识别的形状，再让记忆系统去"读对话 → 答题"，最后用 LLM 裁判判分。数据集的三个字段（`sessions`、`before/after_questions`、`tasks`）分别扮演不同角色。

## 完整数据流

```
dataset/meme_filler32k.json  (flat 数组, episode_id="pl_001")
        │  unpack_dataset.py 拆分
        ▼
code/data/filler32k_pl/episode_001.json  (episode_id=1)
        │  run_agent.py 逐 episode 读取
        ▼
output/<system>/agent_XXX.json  (agent 答案)
        │  judge.py 裁判
        ▼
output/<system>/judge/  (分数)
```

## 第 1 步：下载 + 拆分（[unpack_dataset.py](file:///e:/Projects/MEME-benchmark/code/dataset_tools/unpack_dataset.py)）

HF 发布的是**扁平数组**（一个 JSON 里几百个 episode，`episode_id` 带 `pl_`/`sw_` 前缀）。eval 器要的是"一个 episode 一个文件、按域分目录"，所以先拆：

```bash
python code/dataset_tools/unpack_dataset.py --input dataset/meme_filler32k.json --output code/data
# → code/data/filler32k_pl/episode_001.json ... episode_050.json
# → code/data/filler32k_sw/episode_001.json ... episode_050.json
```

它只做两件事：按 `episode_id` 的正则 `pl_/sw_NNN` 分流到 `{condition}_{domain}/` 目录，并把 `episode_id` 从 `"pl_001"` 还原成数字 `1`（见 [unpack_dataset.py L28-L44](file:///e:/Projects/MEME-benchmark/code/dataset_tools/unpack_dataset.py#L28-L44)）。

## 第 2 步：跑 agent（[run_agent.py](file:///e:/Projects/MEME-benchmark/code/eval/run_agent.py)）

这是数据集被"消费"的核心环节。读入一个 episode JSON 后，[run_episode()](file:///e:/Projects/MEME-benchmark/code/eval/run_agent.py#L138) 按 **before/after 问题的位置**把 `sessions` 切成四段喂给记忆系统：

```
Phase 1  喂 sessions[0 .. before_pos]       → 这些是"事实介绍 + 填充"
Phase 2  问 before_questions                → 变化前状态对不对（trivial-pass 依据）
Phase 3  喂 sessions[before_pos .. after_pos] → 这段含 Change+Delete 事件
Phase 4  问 after_questions                 → 最终的评测题
```

关键字段用法：

- **`episode["sessions"]`**：一整条按时间排序的对话流（evidence+filler 混在一起），逐条调用 `agent.ingest_session(sess)` 喂进去。`type=="evidence"` 的会话还带 `gold_facts`（用来拼 `gold_facts`，也供 golden_memory 基线用）。
- **`before_questions["position_after_session"]` / `after_questions["position_after_session"]`**：这两个数字就是切分点 `before_pos`/`after_pos`（[L158-159](file:///e:/Projects/MEME-benchmark/code/eval/run_agent.py#L158-L159)）。它们保证 before 问题一定问在"变更之前"，after 问题一定问在"变更之后"——这是测演化记忆的前提。
- 每段喂完调用 `agent.finalize_ingest()`，并抓记忆快照（`memory_snapshot`）。

最后输出 `agent_XXX.json`，含 `before_answers`、`after_answers`、`gold_facts`、token 用量等（[L213-L227](file:///e:/Projects/MEME-benchmark/code/eval/run_agent.py#L213-L227)）。

> 注意：`run_agent.py` 读的是**拆分后的逐 episode 文件**，不是 `dataset/` 里那个大 JSON；所以必须先跑 unpack。

## 第 3 步：judge 判分（[judge.py](file:///e:/Projects/MEME-benchmark/code/eval/judge.py)）

读取 agent 输出，对每个问题用**任务特定 prompt** 的 LLM 裁判判断对错，任务标签决定用哪套规则（[JUDGE_PROMPTS L207-214](file:///e:/Projects/MEME-benchmark/code/eval/judge.py#L207-L214)）：

| 任务 | 判分点 |
|------|--------|
| `ER` | 确定性子串匹配（不调 LLM）|
| `Tr` | 3 个历史值是否按序都出现 |
| `Agg` | 一次 LLM 调用核对全部实体值 |
| `Cas` | 是否只报新值、不并报新旧、不用将来时 |
| `Abs` | 是否表达"不确定"，而非自信报旧值 |
| `Del` | 是否没泄露被删的值 |

`Cas`/`Abs`/`Del` 还叠加 **trivial-pass 过滤**：只有当同一个实体在 `before_questions` 里也答对了（证明系统真记住了原始事实），after 的正确答案才计为"真实通过"，否则归为"侥幸/从未记住"。

## 一句话总结

- `dataset/*.json` = 发布态的扁平数据；
- `unpack_dataset.py` 把它切成评测器能吃的样子；
- `run_agent.py` 用 `sessions` + 两个 question 位置字段，把"读对话→变化前答题→继续读→变化后答题"的时间线跑一遍；
- `judge.py` 用 `tasks` 里的 `gold_answer`/`entity_values` 对答案做任务化裁判。

需要我具体看某个 agent（比如 BM25 怎么把 sessions 索引起来）或 golden_memory 基线是怎么只用 gold_facts 的，我可以继续拆。