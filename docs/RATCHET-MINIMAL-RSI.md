# 学习笔记：kayba-ai/recursive-improve 的最小 RSI 循环实现

> 来源：[kayba-ai/recursive-improve](https://github.com/kayba-ai/recursive-improve)（287 ⭐ · Apache-2.0 · Python）
> 我们已 fork：https://github.com/cyberspace-cs/recursive-improve
> 本文是学习笔记，重点拆解其 `recursive_improve/ratchet/` 目录——**一个手撸最小 Agent RSI 循环的教科书实现**。

---

## 1. 这个项目解决什么问题

你的 Agent 每次运行都是"无状态"的——从零开始，唯一提升它的方式是手动改代码，改进无法累积。

**recursive-improve 闭环了这件事：**

```
Agent 运行 → 每次 LLM 调用被捕获（trace）→ 分析失败模式 → 应用针对性修复 → 再跑 → 更好了
```

90% 的 Claude 代码由 Claude 自己写。这个项目就是让**你自己的 Agent** 也能递归自改进。

---

## 2. 整体架构（5 个组件）

| 组件 | 作用 | 类型 |
| --- | --- | --- |
| `ri.patch()` | monkey-patch OpenAI/Anthropic/LiteLLM 客户端，捕获每次调用 | Python |
| `ri.session()` | 上下文管理器，把 trace 写成结构化 JSON 到 `eval/traces/` | Python |
| `/recursive-improve` | Claude Code / Codex skill：分析 traces 并应用修复 | Skill |
| `recursive-improve benchmark` | 快照指标质量、存储、跨时间对比 | CLI |
| `/ratchet` | **自主循环**：improve → run → eval → keep or revert → repeat | Skill + CLI |

```
your agent ──> ri.patch() + ri.session() ──> eval/traces/*.json
                                                  │
                                                  ▼
                                            /recursive-improve
                                                  │
                                                  ▼
                                        improved agent code ──> repeat
                                                  │
                                                  ▼
                                             benchmark ──> dashboard
```

---

## 3. 核心：ratchet（棘轮）最小 RSI 循环

`/ratchet` 是 autoresearch 式的自主循环，`recursive_improve/ratchet/` 只有 **6 个文件**，却实现了完整的 keep-or-revert 闭环。这是最值得学习的地方。

### 3.1 文件职责

| 文件 | 职责 |
| --- | --- |
| `__init__.py` | 导出 5 个入口：eval / commit / revert / log / status |
| `config.py` | 把 `program.md`（Markdown 配置）解析为 `RatchetConfig` |
| `engine.py` | 循环编排：评估、提交、回滚、记日志、查状态 |
| `scorer.py` | `composite_score`：多指标加权合成一个标量 |
| `git_ops.py` | git 操作：建分支、提交、回滚、脏检查 |
| `log.py` | JSONL 结构化日志 + Markdown 人类可读摘要 |

### 3.2 循环流程（keep-or-revert 棘轮）

```
Step 1  Configure   读 program.md：目标 / 运行命令 / 指标 / 停止条件（必须用户确认）
Step 2  Branch      创建 ri/ratchet-<时间戳> 专用分支
Step 3  Baseline    ratchet eval → 得起始 composite score
Step 4  LOOP（直到停止条件）:
        ├─ 4a  improve   调用 /recursive-improve 全流程（自动批准所有修复）
        ├─ 4b  run      跑 Agent 生成新 traces
        ├─ 4c  eval      ratchet eval → 新 score
        ├─ 4d  decide    score 提升 ? keep : revert
        │       ├─ keep   → ratchet commit（git add -A + 带分数 commit）
        │       └─ revert → git checkout -- . + git clean -fd（回到上次 commit）
        └─ 4e  log      JSONL 追加本次迭代 + 更新 Markdown summary
Step 5  Stop      max_iterations / max_duration / plateau_patience 任一触发
```

### 3.3 scorer.py：复合评分（最精妙的一行逻辑）

```python
def composite_score(metrics: dict, config: RatchetConfig) -> float:
    score = 0.0
    total_weight = 0.0
    for name, spec in config.metrics.items():
        if name not in metrics:
            continue
        value = metrics[name]["value"]
        if spec.direction == "minimize":
            value = 1.0 - value          # 关键：最小化指标反转
        score += value * spec.weight
        total_weight += spec.weight
    if total_weight == 0:
        return 0.0
    return round(score / total_weight, 4)
```

**设计要点：** 把 `minimize` 类指标（如错误率）反转为 `1 - value`，让**所有指标统一为"分数越高越好"**，一个标量就能做 keep/revert 决策。这是整个棘轮能简单化的根基。

### 3.4 config.py：配置即代码（program.md）

用 Markdown 当配置文件，`##` 分节、`- key: value` 解析：

```markdown
# Improvement Goals

## Objective
减少 Agent 在长任务中的放弃率

## Agent Run Command
uv run python examples/technova_agent.py

## Metrics
- clean_success_rate: maximize (weight: 2.0)
- error_rate: minimize (weight: 1.0)

## Stopping Conditions
- max_iterations: 20
- max_duration_hours: 8
- plateau_patience: 3
```

解析器支持：Objective / Agent Run Command / Traces Directory / Metrics（方向+权重）/ Stopping Conditions / Time Budget / Improve Command / Evolution。

### 3.5 git_ops.py：keep 与 revert 的物理实现

```python
def commit_iteration(iteration, score, prev_score=None):
    # git status --porcelain 判断有无变更 → 无变更返回 None
    # git add -A
    # commit -m "ratchet: iteration {i}, score {s} ({prev} -> {new})"
    # 返回短 commit hash

def revert_to_last_commit():
    # git checkout -- .   丢弃已跟踪文件改动
    # git clean -fd       删除改进步骤产生的未跟踪文件
```

**棘轮语义：** 只有提升被保留（commit 成为新的"棘齿"），失败被完全抹除——**只进不退**。

### 3.6 log.py：可审计的迭代历史

- `ratchet_log.jsonl`：每次迭代一行 JSON（时间戳、耗时、基线分、新分、决策、commit、指标、trace 数）
- `ratchet_summary.md`：人类可读的 Markdown 表格（Iter | Score | Delta | Decision | Commit | Duration），自动找 best score

---

## 4. 停滞检测（plateau）

```python
plateau = 0
for e in reversed(entries):        # 从最新往回数
    if e["decision"] == "revert":
        plateau += 1
    else:
        break
```

连续 `plateau_patience`（默认 3）次 revert 就停止——防止无限循环烧 token。这是自主循环里**必须**有的刹车机制。

---

## 5. 与 RSIAgent 的对比（我们仓库）

| 维度 | recursive-improve / ratchet | RSIAgent（AetherLabsAI） |
| --- | --- | --- |
| 定位 | **最小**单 Agent 自改进闭环 | 多智能体框架（actor/verifier/curriculum） |
| 改进对象 | Agent 代码 + 提示词 | 新环境中的自主探索 + 可复用记忆 |
| 决策依据 | 运行 traces 的复合指标评分 | broad-then-deep 探索 + verifier 分级 |
| 复杂度 | 6 个文件，可 1 小时读懂 | 完整框架（core/explore/llm/env） |
| 基础设施 | git 分支即版本控制 | QEMU VM + checkpoint 回滚 |

**互补关系：** ratchet 提供了"改进决策循环"的最小可读实现；RSIAgent 提供了"在多环境中自主进化"的完整框架。理解 ratchet 是理解 RSIAgent 的极佳入门。

---

## 6. 我们能借鉴的设计模式

1. **单一标量决策**：多指标 → 加权合成一个分数 → 简单阈值判断，不要用 LLM 做 keep/revert 判断
2. **方向统一**：minimize 指标反转，全系统"越高越好"，避免比较逻辑里到处写 if
3. **配置驱动**：Markdown 当配置文件（program.md），Agent 能读能改，人也能看懂
4. **git 即回滚机制**：不自己写快照，用 git 分支 + checkout 实现 keep/revert，零额外复杂度
5. **JSONL 日志 + MD 摘要**：机器可读（JSONL 追加写）与人类可读（Markdown 表格）分离
6. **停滞刹车**：plateau_patience 防止死循环烧钱
7. **HITL 边界**：配置阶段必须用户确认，改进阶段自动批准——把人的介入放在"定目标"而不是"看细节"

---

## 7. 最小复刻（30 行核心）

```python
# minimal_ratchet.py —— ratchet 循环的最小抽象
import subprocess, json
from pathlib import Path

def run_eval(traces_dir):        # 1. 评估 traces
    metrics = {"error_rate": {"value": 0.1}, "success": {"value": 0.9}}
    return metrics

def composite(metrics, specs):   # 2. 合成单一分数（minimize 反转）
    s = w = 0.0
    for k, spec in specs.items():
        v = metrics[k]["value"]
        v = 1 - v if spec["dir"] == "minimize" else v
        s += v * spec["weight"]; w += spec["weight"]
    return s / w if w else 0.0

def loop(program):
    cfg = parse(program)          # 3. 读配置（目标/指标/停止条件）
    baseline = composite(run_eval(), cfg["metrics"])
    plateau = 0
    for i in range(cfg["max_iterations"]):
        improve()                                  # 4. 调 LLM 改代码
        new = composite(run_eval(), cfg["metrics"])
        if new > baseline:                         # 5. keep or revert
            git_commit(i, new, baseline); baseline = new; plateau = 0
        else:
            git_revert(); plateau += 1
        log_iteration(i, new, baseline, plateau)   # 6. 记日志
        if plateau >= cfg["plateau_patience"]: break
```

**一句话总结：** ratchet = 配置驱动的「评估 → 改进 → 复评 → git keep/revert」循环，核心智慧是把一切决策简化为单一标量分数 + 一个停滞计数器。
