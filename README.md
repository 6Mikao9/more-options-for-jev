# More Options for Jev

Jev 原生接口一次最多稳定支持 255 个 options。这个仓库整理的是 **超过 255 选项时的可扩展方案**：把候选空间做成“虚拟化 + 分页”，让 Jev 每次只看一个有界 resident set，但可以按需翻页直到找到真正合适的工具或参数。

## 背景与目标

在真实 agent 场景里，工具、动作、参数候选会随着任务上下文快速膨胀，固定 255 options 会带来两个问题：

1. 候选被截断，正确项可能根本进不了当前决策帧。
2. 为了塞进上限而做硬裁剪，会显著降低覆盖率和可恢复性。

这个模块的目标是：

- 保留 Jev 决策接口的有界性；
- 通过分页把“逻辑上无限”的候选空间映射成“物理可驻留”的页；
- 用显式状态动作（如 `PAGE` / `EXPAND` / `REFINE`）支持可回放、可诊断的决策流程。

## 方案概览：Virtual Option Space + Resident Options

```text
Open-world candidate space
        ↓ (virtualization)
Virtual pages (logical)
        ↓ (materialize current page)
Resident options (physical, visible to Jev)
        ↓
Jev decision + state transition
```

核心思想：

- **物理页（physical pages）**：模型当前真实“看见”的 options 集合。
- **虚拟页（virtual pages）**：上下文里可寻址但未驻留的候选目录。
- **分页动作**：当当前页没有合适候选时，模型可以触发翻页，把其他虚拟页 materialize 成新的物理页。

这样，Jev 不需要一次处理全部候选；它只在小窗口内做高质量选择，同时保留对大空间的可达性。

## 运行时语义（建议）

可将以下动作作为显式状态迁移（便于 trace/replay）：

- `PAGE`：切换到另一候选页。
- `EXPAND`：扩大当前可见候选覆盖范围。
- `REFINE`：把粗粒度候选细化到字段/片段/token 级。
- `INVALIDATE`：在 revision 变化后使旧候选失效。
- `CLARIFY`：信息不足时请求澄清。
- `STOP`：完成或中止当前回合。

## 与 Jev Native Agent 项目的关系

本仓库是该能力模块的说明与提炼；完整运行时原型位于：

- https://github.com/6Mikao9/jev-native-agent-with-extended-options

建议结合该仓库中的 runtime 与实验文档理解完整上下文。

## Quick Start（来自 Jev Native Agent 项目，Core / 不走辅助模型）

下面是“**不依赖辅助模型**”的最小可运行路径，适合先验证 paging/option 协议与执行边界：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e .
jev-agent --workspace .\agent_workspace
```

也可以直接：

```powershell
python -m jev_agent.cli --workspace .\agent_workspace
```

REPL 内常用操作：

- `:tools` 查看可用工具
- `:plan file.search {"text":"TODO","path":"."}` 生成候选计划
- `:approve` 批准有副作用的执行

说明：

- Core 模式使用确定性的 scripted chooser，主要用于协议/流程 smoke test。
- 不需要 GPU、模型权重或 API key。
- 这条路径用于验证 runtime 机制，不代表真实 Jev 端到端质量上限。

## 后续扩展

如果需要接入 helper model 或 live Jev，可在上游仓库继续按 `.[models]` 与 `--live --key-stdin` 路径扩展；本仓库当前聚焦于“超 255 options 的虚拟化与分页能力说明”。
