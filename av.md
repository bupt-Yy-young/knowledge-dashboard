# AdaptiveVuls（AV）项目理解

> 本文记录当前对 AdaptiveVuls 研究问题、方法对象、执行闭环和论文定位的统一理解。V1.42 根据已有负面/混合实验进一步收缩：OES 不再是属性补全表，深挖被重新定义为“漏洞定位、假设形成、修订与验证”的连续过程。V1.43 在不改变科学状态和裁决合同的前提下，将初始 Survey 与所有普通 Case 调查合并为一条目标级 Sticky Investigator 工作链；Controller、Reviewer 与覆盖挑战仍保持科学上的 Fresh。完整设置运行后又补齐了 Claim-free 方向首次形成非终态 Revision 的机械载体规则（preserve→revise、branch→branch）及协议失败分类。旧版本和原始实验结果保持冻结。

## 1. 一句话定义

AdaptiveVuls 是一种面向开放式仓库的认知引导、证据约束的持续漏洞调查方法。Agent 先建立部分且可修订的目标安全认知，结合相关漏洞知识形成一组有目标依据的调查方向；Coding Agent 在持续阅读、追踪、实验和 PoC 中共同完成漏洞定位、假设形成、修订与验证；当前最能改变安全判断的未知决定下一项证据行动，新证据只允许结论达到它实际覆盖的条件和范围。Fresh Review 负责独立限定证据和提出后继，Runtime 仅负责合法提交。

最简闭环：

```text
Target Cognition + Vulnerability Knowledge
    ↓
Breadth-first Security Survey
    ↓
Grounded Investigation Direction
    ↓
Continuous Coding-Agent Investigation
    ↓
Meaningful Evidence Checkpoint
    ↓
Fresh Adversarial Review
    ↓
Reviewed Update / Revision / Branch / Disposition
    ↓
Canonical State Update
    ↓
Fresh Portfolio Replanning
    ↺
```

AV 的核心不是多 Agent 编排、更多阶段、更多字段、显式记忆或固定工具流水线，而是：

> 目标认知和漏洞知识能否引导 Agent 形成更有价值的调查方向，以及证据能否在持续调查中真正改变漏洞假设、下一步行动和最终结果。

V1.43 的三个 lane 是权限与视角边界，不是并行流水线；主循环保持顺序执行，同一时刻最多运行一个 Coding-Agent 会话。初始 Survey、所有普通 Case、Revision 和 Branch 调查共享一条目标级 Sticky Investigator 工作链；Controller 每次决策 Fresh，Reviewer 每个 Evidence Checkpoint Fresh，Refresh Survey 和 Saturation Challenge 也作为独立覆盖挑战 Fresh。Capability Advisor 是 advisory-only 实现组件，不是第四个科学主体。

V1.43 不接受生产 Controller 轮数上限，也不把时间、Token、Episode 或剩余运行时间写入 Packet/Prompt。正式实验的统一墙钟只作为外部停止边界。代码中的 `testOnlyMaxControllerDecisions` 仅用于有限 mock simulation，不属于方法或正式实验配置。上下文窗口阈值只属于 Harness 容量保护，不参与科学选择。

---

## 2. AV 试图解决的问题

现代 Coding Agent 已经能够：

- 阅读大型仓库；
- 搜索调用链和数据流；
- 使用静态、动态工具；
- 编写测试和 PoC；
- 运行构建、调试和差分实验；
- 形成漏洞假设。

因此 AV 不是要证明“Agent 会不会读代码或调用工具”。真正困难的是：

1. 零散代码事实何时足以形成一个稳定、可证伪的安全问题；
2. 技术机制成立以后，是否真的存在攻击者控制、安全边界、增量影响和目标责任；
3. 新证据改变攻击者、根因、Boundary、Ownership 或 Scope 时，如何保留问题连续性；
4. 一个 Case 的 Evidence 是否能改变其他 Case、Frontier 和下一项调查行动；
5. 如何避免 Mechanism reproduced 被直接扩张为 Vulnerability Report。

开放式审计中的调查对象不是预先给定、形成后也不是永久不变的。它可能经历：

```text
初始Observation
→ Frontier
→ Case R1
→ Evidence反驳部分解释
→ Case R2或Sibling Frontier
→ 新Evidence
→ Report / Caller Misuse / Expected Behavior / Refuted
```

AV 关注的就是这个持续演化的 Investigation Case lifecycle。

### 2.1 当前论文主线：从假设验证到 Coding-Agent 假设调查

已有 LLM 漏洞检测工作已经提出假设验证。以 VulAgent 为例，其在给定代码单元上先定位敏感操作，再构造触发条件和候选路径，随后利用检索到的项目上下文检查假设条件与路径上的防护。它回答的主要是：一个已经构造的漏洞假设是否与现有程序上下文一致。

仓库级开放审计不是简单地对每个函数重复这一过程。可疑位置、漏洞类型、攻击者模型、安全边界、相关上下文和验证方式都没有预先给定，而且相互依赖：需要假设才能决定读取和执行什么，也需要进一步理解和实验才能形成正确假设。跨函数状态、对象归属、组件责任、部署条件和运行环境也很难被一个预先固定的代码单元完整表示。

Coding Agent 提供了主动解决这一问题的能力。它可以自由浏览整个仓库、追踪调用链、运行静态和动态工具、编译目标、编写 PoC 和对照实验。强模型在宽泛提示下发现历史遗留漏洞的案例说明这种基础能力已经存在。AV 研究的不是如何让模型第一次拥有读代码和写 PoC 的能力，而是：

> **如何将 Coding Agent 的自由仓库探索能力组织成一个长期保持调查连续性、主动获取证据，并随证据形成和修订安全问题的开放式漏洞调查过程？**

因此 AV 的中心不是“构造完整假设后再验证”，而是从有依据的初步安全怀疑开始，在主动证据调查中逐渐恢复攻击条件、机制、安全属性、增量影响、目标责任和适用范围。Evidence 不仅决定保留或删除原假设，还可以收窄、修订或拆分问题，并将新事实反馈到后续发现空间。

最简区别是：

```text
函数/代码单元级假设验证：
敏感操作 → 完整条件/候选路径 → 上下文验证 → 保留或删除

原生 Coding Agent：
开放任务 → Agent在会话内部自行探索、调查和判断

AdaptiveVuls：
开放仓库探索 → 有依据的初步Case
→ Coding Agent主动获取Artifact
→ Evidence限定并修改安全问题
→ 继续、修订、分支或结束
→ Reviewed新认识反馈后续调查
```

原生 Coding Agent 理论上也可以完成这些动作，因此 AV 不能预设它不会检查边界、修订假设或建立对照。显式状态、会话恢复、来源记录和多 Worker 都只是实现支撑，不是方法收益。AV 真正需要验证的是：目标认知与漏洞知识是否改变了可达的漏洞方向，以及证据是否改变了后续行动、漏洞假设和正确结果。

AV 也不声称普遍优于广扫后验证。对于稳定、局部、已有明确 source/sink 或确定性崩溃的候选，批量发现后由强 Agent 分别验证可能更高效。AV 的预期优势集中在初始安全问题不完整，且攻击者、Boundary、Impact、Ownership 或 Scope 会随调查证据改变的情况。

因此正式主对比至少包括：

```text
Native Coding Agent
Batch-and-Confirm（高召回扫描/发现 → 冻结候选池 → 强Agent逐个验证）
AdaptiveVuls
可运行的仓库级Agent方法（优先选择能公平复用模型和工具的OpenAnt等）
```

主实验既要测正确漏洞、误报、漏报和成本，也要测 Scope/Impact/Ownership 夸大、决定性 Artifact，以及初始假设不准确时能否恢复出正确安全问题。最关键的独特结果不是“Case发生过Revision”，而是 Revision 或调查反馈是否实际产生了固定候选验证和原生会话没有得到的正确结论或新证据路径。

V1.37 对这条论文主线的实现对齐检查单独记录在：

```text
adaptivevuls/ADAPTIVEVULS_V1_37_MAINLINE_IMPLEMENTATION_ALIGNMENT_20260731.md
```

---

## 3. 核心设计原则

### 3.1 开放探索，约束提升

```text
Permissive Exploration
+
Constrained Claim Promotion
```

Agent 可以自由阅读、搜索、使用工具、提出 Observation、Frontier、Case proposal 和 Evidence proposal；但一个方向要成为 Case，至少要有来源明确的事实、具体的安全怀疑和可证伪的下一步问题。Claim support/refutation、Case Revision、Negative Closure 和 Report admission 仍必须满足相应来源、审查和状态迁移条件。这里约束的是“不能把空泛猜测直接升级成 Case”，而不是要求 Formation 提前完成整份漏洞证明。

### 3.2 Case 是长期调查对象

Case 不是一次任务或一个 Worker 调用：

```text
CASE-1-R1
→ Evidence
→ CASE-1-R2
→ New Obligation
→ Evidence
→ Disposition
```

Case ID 维持调查连续性；Revision 保存问题发生的实质变化；OES 保存当前可执行的科学状态。

### 3.3 Session Memory 不是科学真值

```text
Session Memory = 工作记忆
Canonical State = 科学权威
```

Session 可以记住构建环境、命令、失败实验、临时假设、仓库导航和工具经验；但当前 Revision、Claim、Reviewed Evidence、Active Obligation、Disposition 和 Report authority 必须来自 Canonical State。

### 3.4 Evidence 只能更新实际覆盖范围

```text
Mechanism reproduced
≠ Boundary violated
≠ Impact established
≠ Target ownership established
≠ Report allowed
```

同一 Artifact 可以支持 M，却不足以支持 C、B、I 或 Own。

### 3.5 任务活动不等于安全进展

读取更多文件、运行更多工具、测试通过、crash、PoC 执行成功、Frontier 被遍历都不自动构成安全进展。只有形成可证伪 Case、改变 Claim、区分竞争解释、产生新 Frontier、解决 Obligation 或获得有界处置，才算科学进展。

---

## 4. 科学对象

科学对象、科学角色和逻辑动作不能一一对应。

### 4.1 Security Map

目标安全语境的组织化模型：

```text
Actors
Assets
Entry Points
Trust Boundaries
State Transitions
Security-sensitive Capabilities
Contracts
Responsibilities
Material Unknowns
```

Security Map 是部分且可更新的，不自动生成漏洞，也不声称覆盖全部攻击面。

### 4.2 Observation

Observation 是从源码、运行结果、文档、配置、历史或工具输出中得到的、带有来源的局部描述性事实：

```text
ω = <fact, provenance, target/environment, scope>
```

例如：

```text
Session.__setstate__ iterates over state.items()
and assigns each entry through setattr.
```

Observation 不是 Worker，也不是漏洞结论。它与 Artifact、Evidence 的区别是：

```text
Artifact：原始源码、脚本、输出、trace、文档
Observation：这些材料显示了什么事实
Evidence：该材料如何改变指定Case Claim
```

任何探索或调查动作都可以产生 Observation，不存在专门的 Observation Worker。不是每个 grep 输出都要成为 Canonical Observation，只有安全相关、可追溯、可能影响 Target Model、Frontier 或 Case 的事实才需要保存。

### 4.3 Frontier

Frontier 是高召回探索阶段的安全关系问题：

```text
f = <observation refs, security relation, open question,
     scope, unresolved links>
```

它可以模糊、错误、合并、重开或 Park，不要求完整攻击模型、PoC 或 C/M/B/I/Own/S，也没有报告权限。完成一个 Frontier 只说明当前缺少有价值行动，不表示目标攻击面已经覆盖。

### 4.4 Investigation Case

Case 是稳定身份、内容可修订、值得持续深挖的初步安全问题。它在刚建立时可以只是一个有代码依据的安全怀疑，而不必已经是一条完整漏洞命题：

```text
Frame(c) = <
  Supporting Observations,
  Suspected Security Relation or Attacker Effect,
  Currently Grounded Claims,
  Falsifiable Deciding Question,
  Cheapest Useful First Investigation,
  Known Alternatives and Unknowns
>
```

Case admission 要求：

```text
Grounded
∧ Security-Relevant
∧ Falsifiable
∧ Investigable
```

Case 可以在攻击者模型、Boundary、Impact、Ownership、Scope 和竞争解释尚未完全形成时存在，不要求 PoC、严重性、完整攻击链或 Report Evidence。它至少要说明：哪些目标事实引出了怀疑、怀疑的安全关系是什么、什么问题可以推翻或推进它、先做哪项调查。无目标依据、不可证伪或没有可执行下一步的纯叙述不能形成 Case。

V1.37 要求 Case Formation 给出决定后续调查方向的核心问题、最便宜且最有区分力的第一项调查，以及至少一个当前确有依据的 Claim。初始 Case 可以只有 M，也可以只有 C+M 或 M+B；尚未形成的维度直接缺省，不提前创建内容空泛的占位 Claim。随着代码追踪、PoC、对照实验和文档证据出现，Investigator 可以提出新增或替换 Claim，Fresh Reviewer 通过 Revision 决定是否进入规范状态。只有形成 `report` 时，C/M/B/I/Own/S 才必须全部具有经过审查、范围明确的证据。

### 4.5 Claim：C/M/B/I/Own/S

- **C — Control：** 攻击者、调用者、管理员或环境能控制什么，入口是否可达。
- **M — Mechanism：** 目标行为或状态变化是否发生，以及发生条件。
- **B — Boundary/Property：** 目标拥有或承诺的安全边界、不变量或接口约定是否被违反。
- **I — Impact：** 目标机制增加了什么攻击能力或安全损失。
- **Own — Ownership：** 责任属于目标、调用者、部署、Dependency 还是其他组件。
- **S — Scope：** 适用版本、配置、入口、参与者、部署和环境。

这些维度是 Report 前的安全结论检查角度，也可以在某一维确实改变当前判断时成为调查 Claim；它们不是 Formation 的六格准入表，也不是调查进度表。必须防止 M supported 自动扩张为 I supported 或 Report，也要防止仅因某一维缺失就继续消耗调查。

### 4.6 Competing Explanations

Case 调查过程中应逐渐形成并保留：

1. Supporting Vulnerability Explanation；
2. strongest Alternative Vulnerability Explanation；
3. strongest Alternative Non-vulnerability Explanation。

非漏洞解释包括 Expected Behavior、Caller Misuse、Dependency Ownership、Deployment Responsibility、Missing Attacker Control、No Incremental Capability、Invalid Security Boundary 和 Impact Inflation。Formation 时已经可见的解释应记录；当时尚无依据的解释不必为了完整格式而编造，后续 Evidence 和 Review 可以补入。

### 4.7 Operational Evidence State（OES）

```text
OESₜ(c) = <Qₜ(c), Altₜ(c), Obligationsₜ(c)>
```

其中：

- `Qₜ(c)`：当前最可信的安全解释、已形成的 Claim、Artifact 实际覆盖范围和 residual unknowns；
- `Altₜ(c)`：竞争解释及当前判别状态；
- `Obligationsₜ(c)`：仍可能改变处置的 Evidence Obligations。

OES 不是完成度评分、漏洞分类器或六维补全表。调查时的最小 OES 只需回答：当前最可信的解释是什么、最强竞争解释是什么、哪个未知最可能改变判断、什么 Artifact 能区分它们，以及已有 Artifact 实际测到了什么条件和范围。

### 4.8 Evidence Obligation

不是每个 unknown 都生成 Obligation。一个 Canonical Obligation 必须：

- disposition-relevant；
- actionable；
- 有明确 deciding question；
- 有 support condition；
- 有 falsifier/contrary condition；
- 有 expected Artifact；
- 解决后可能改变 Claim、Case 或 Disposition；
- 没有等价 Active Obligation。

```text
Obligation = {
  subject Case/optional Claim,
  deciding question,
  competing explanations,
  support condition,
  falsifier,
  expected Artifact,
  expected state effects,
  status
}
```

状态使用 `active / resolved / parked`。`blocked / timeout / repair-pending` 属于 Operational State。

### 4.9 Artifact

Artifact 是调查活动与正式 Claim 变化之间的证据载体。动态 Artifact 应保留命令、cwd、依赖、输入、退出状态、原始输出、标记、controls 和 Scope；静态 Artifact 应保留目标版本、代码位置、查询/追踪方法、路径和条件。计划中的实验、未执行脚本和自然语言风险描述不是 Evidence Artifact。

V1.37 不再用一个全局 E1/E2/E3 给整个 Case 打分，而是为每个 Artifact 和每次 Reviewed Claim 更新记录：

```text
Evidence layer:
source / bounded_execution / target_runtime / end_to_end

Target fidelity:
source_only / extracted_model / surrogate /
instrumented_target / exact_target

另外单独记录：
tested scope / positive control / negative control /
limitations / independent reproduction
```

源码推理、抽取函数重放、替代模型、真实目标运行和端到端影响不能互相冒充。Fresh Review 也不等于独立复现。E1/E2/E3 只可作为实验统计时从逐 Claim 记录派生出的简写：源码约为 E1，有界执行约为 E2，可信的目标运行或端到端复现约为 E3；它不是全局 Case 状态，也不是所有 Report 都必须统一达到 E3。

### 4.10 Reviewed Delta

Investigator 的解释只是无权限 proposal。Fresh Reviewer 产生：

```text
ReviewedDelta = {
  artifact scope,
  reviewed Claim effects,
  competing explanation updates,
  obligation updates,
  derived Observation/Frontier proposals,
  Case transition proposal
}
```

只有 Runtime 提交后的 Reviewed Delta 才能改变 Canonical OES。

### 4.11 Disposition 与 Operational State

非终态 Transition：

```text
continue
local_update
revise
branch
```

终态 Transition 为 `terminal` 或 `merge`；其中 `merge` 只能产生 `duplicate_merged`。终态 Disposition：

```text
report
hold
expected_behavior
caller_misuse
dependency_ownership
refuted
duplicate_merged
```

Operational State：

```text
blocked
timeout
repair-pending
handoff-failed
```

三类语义不能混用。

### 4.12 Finding

Finding 是 disposition=`report` 且通过机械准入 Gate 的 Case 投影。它必须绑定 exact Case Revision、Reviewed Claims、Artifact Basis、Fresh Review、admitted Scope、target-owned Property、attacker path、incremental Impact 和 Ownership。Reporter 不能创造 Finding，只能读取 FindingIndex。

---

## 5. 科学角色与动作

### 5.1 三个核心科学主体

#### Portfolio Controller

负责从最新 Canonical State 和 Eligible Action Board 选择下一项调查行动。不运行漏洞实验，不修改科学真值，不重新研究整个仓库，也不使用固定数值算法。每次选择还要说明机会成本：继续当前 Case 时，哪个重要方向会暂时得不到调查，以免系统只闭合成熟 Case 或只追逐新 Frontier。

#### Case Investigator

根据 Case/OES 获取能够改变 Case 的 Evidence。它可以读代码、查文档、使用工具、写脚本和 PoC、建立正负对照、检查竞争解释，并发现新的 Observation/Frontier。它不能直接修改 Canonical Claim 或发布 Finding。

#### Fresh Case Reviewer

独立判断 Artifact 的实际覆盖范围，检查 attacker baseline、target-added capability、counterfactual effect、Boundary、Ownership 和 Impact/Scope inflation，并提出 Claim/OES 更新、Obligation、Revision/Branch 或有界 Disposition。Reviewer 不运行工具、不补实验，也不直接写 Canonical State。

### 5.2 动作而不是额外角色

Discovery Actions：

```text
target_cognition
explore_frontier
form_case
```

Investigation Actions：

```text
investigate_case
acquire_deciding_artifact
```

原 Producer 与 Materializer 不再是两个科学角色，而是同一个 Case Investigator 面对两种 Action Scope：

- 无优先 Obligation：自主选择决定性未知；
- 有 Active Obligation：优先获取指定判别 Artifact。

两者使用相同工具权限、Evidence Contract 和目标级 Investigator Session。

Fresh Review 一次完成：

```text
Evidence Review
+
Case Transition
```

原 Reviewer 与 Adjudicator 不再作为两个常规 Coding-Agent Worker。常规 `adjudicate_case` 应被删除；Reviewer 的合法 transition 由 Runtime Gate 提交。

---

## 6. 会话记忆（V1.43）

V1.43 的会话模型是：

| 角色/动作 | 科学上的会话语义 | Harness 实现 |
|---|---|---|
| 初始 `survey_repository` | 主 Investigator 连续链起点 | Sticky：`v1-43:target-investigator` |
| `investigate_case` / `acquire_deciding_artifact` | 继续同一目标调查链 | 同一 Sticky Key |
| Case Revision / Branch / Sibling 调查 | 保留目标级仓库工作记忆 | 同一 Sticky Key |
| `refresh_survey` | Fresh coverage challenge | 唯一事务 Key；仅同事务 repair 可恢复 |
| `saturation_challenge` | Fresh coverage challenge | 唯一事务 Key；与 refresh 和主链隔离 |
| Portfolio Controller | Fresh per decision | `v1-43:controller:<actionId>` |
| Fresh Case Reviewer | Fresh per Evidence Checkpoint | `v1-43:review:<submissionId>` |
| Reporter / Capability Advisor / Runtime | 不调用科学模型或无会话状态 | Stateless |

这里的核心区分是：

```text
Session Memory = 目标仓库的工作连续性
Canonical State = Case、Revision、Evidence、Claim 和处置的唯一权威
```

主 Investigator 可以跨 Case 记住目录结构、构建环境、工具设置、失败实验和跨组件关系，但每次调用必须先读最新 Scientific Packet。当前 exact Case/Revision、最新 Review、active Obligation 和文件化 Artifact 覆盖旧会话记忆。Evidence、Scope、Claim 状态或处置不能因为同一 Agent 记得它们，就从一个 Case 静默转移到另一个 Case。

### 6.1 为什么 Revision 和 Branch 不再换 Session

V1.42 将 Sticky Key 绑定到 Case Revision，导致初始方向成熟、Case 切换和 Revision 后反复恢复仓库上下文。V1.43 将 Session 身份改为目标级工作链，因此局部更新、重大 Revision、Branch 和 Sibling Case 都不更换逻辑 Key。科学身份仍由 Case/Revision 记录，确认偏误由 Fresh Review、exact binding 和 Canonical State 优先规则控制。

### 6.2 科学 Fresh 与底层事务恢复

Controller、Reviewer 和覆盖挑战为每个独立科学任务分配从未共享的唯一 Key。Harness 使用 `sticky` 机制只是为了让同一个 handoff 格式修复，或同一 Checkpoint 的 transition repair，能够恢复刚才的 native Session。不同决策、不同 Checkpoint 和不同 coverage challenge 的 Key 不同，所以它们仍然是科学上的 Fresh root。

### 6.3 上下文容量轮换

主 Investigator 使用 Hybrid continuity。若 Runner 提供真实 context window，Harness 在警告区提示紧凑交接，在达到容量区后先允许 native compression；再次达到阈值且已观察到足够压缩后，Harness执行：

```text
seal → digest → bootstrap → 新 native Session
```

逻辑 Key 仍是 `v1-43:target-investigator`。这属于容量保护，不是 AV 的时间、Token 或 Episode 预算。若自定义 Provider 不提供真实窗口，正式启动配置应显式给出 `contextWindowTokens`，V1.43 不猜测一个未经确认的模型上限。

完整实现说明见：

```text
/data/yy/720.txt
```

---

## 7. Case Investigation 与 Evidence Checkpoint

Investigator 输入：

```text
Current Case Revision
Current OES
Competing Explanations
Optional Priority Obligation
Existing Reviewed Evidence
Negative Knowledge
Applicable Capability Advice
Scientific Objective
```

不向 Investigator 提供 Episode 时间预算、Token 预算、剩余运行时间、固定工具或 Provisional Disposition。

一次 Investigation 会话可以内部迭代：

```text
读代码
→ 形成判别问题
→ 写测试
→ 运行失败
→ 修正实验
→ 增加control
→ 检查替代解释
→ 再追调用链
→ 获取Artifact
```

只有出现有意义的科学 Checkpoint 才交接，例如：

1. 关键 Claim 实质变化；
2. 竞争解释被区分；
3. Case 需要 Revision/Branch；
4. 发现独立 Observation/Frontier；
5. 形成明确 unresolved Obligation；
6. 当前方向科学饱和；
7. 遇到真实 operational blocker。

Workflow 不机械规定最短/最长时间、工具调用数、Evidence 文件数或 Token 数量。

Investigator 输出：Attempts、Artifacts、Raw Results、Controls、Limitations、Proposed Claim Effects、Residual Alternatives、Suggested Revision/Branch、Remaining Unknown 和 Derived Objects。所有解释都标记为 `UNREVIEWED INVESTIGATOR PROPOSAL`，不输出 Recommend Report 或 Final Finding。

Investigator 只能对当前 Revision 已签发且被 Artifact 实际覆盖的 Claim 提出更新。如果深挖过程中才恢复出新的攻击者条件、Boundary、Impact、Ownership 或 Scope，不能把它硬塞进原有 Claim；应保留对应 Artifact，并提出新增 Claim 或改写命题。Fresh Reviewer 决定是否通过 Revision 接纳。V1.42 不再机械要求所有新 Claim 从 `unknown` 开始：如果它们只是首个持续调查中由已提交原始证据直接形成的初始命题，Reviewer 可以在同一次原子提交中建立并审查它们；实质性 Revision/Branch 仍不能同轮终态化。

---

## 8. Fresh Case Review

为减少锚定，Reviewer 输入顺序应为：

1. Authoritative Case/OES；
2. Raw Artifacts、命令、输出、controls 和 source/documentation anchors；
3. Attempt Record；
4. 最后才是 Unreviewed Investigator Proposal。

Reviewer 应先独立回答：

```text
Artifact观察到了什么？
测试了哪一层、没有测试哪一层？
攻击者初始能力是什么？
目标增加了什么能力？
移除目标机制后结果是否仍存在？
目标是否拥有声称的安全属性？
```

然后选择：

```text
local_update
continue
revise
branch
merge
terminal
```

每次 Review 必须满足 **Case Transition Completeness**：如果选择 `local_update` 或 `continue`，提交后的当前 Revision 必须仍有至少一个明确、可行动、会改变处置的 Evidence Obligation，或单独记录真实 operational blocker 及具体 resume condition；如果两者都没有，就必须依据现有证据选择有界终态；如果调查命题发生实质变化，则使用 Revision/Branch。禁止形成“继续，但没有科学或运行后继”的成熟 Case 滞留状态。Runtime 只机械检查后继结构是否存在，不替 Reviewer 决定具体处置。

如果 Reviewer 引入或替换新的 attacker model、root cause、Boundary、Ownership、影响族或关键 Claim，则必须 `revise/branch + continue`，不能在同一次 Review 中授权 Report。唯一例外是：初始调查方向尚无 Claim，但首个持续调查的原始 Artifact 已经完整覆盖一个窄化命题，Reviewer 可以在同轮形成、审查并有界终态化这个初始命题。

如果只是删除未经支持的宽泛表述，并且窄化后的必要 Claim 已被当前 Evidence 覆盖，例如 `RCE→DoS`、`all versions→tested version`，可以在同一 Review 中 qualification 后终态处置。

---

## 9. State → Action Control

Canonical Investigation State：

```text
Sₜ = {
  Security Map,
  Observations,
  Frontier Portfolio,
  Cases and Revisions,
  OES per Case,
  Evidence and Attempts,
  Reviewed Deltas,
  Active Obligations,
  Negative Knowledge,
  Operational Records,
  Action Lineage
}
```

方法内部不包含预算字段。

Eligible Action Board 至少包括：

```text
refresh_discovery
explore_frontier
form_case
investigate_case
acquire_deciding_artifact
```

`review_pending_checkpoint` 不是 Portfolio 选择项，而是 Evidence Checkpoint 后必须立即完成的协议延续。每个 Portfolio Action 绑定 subject、trigger state、可选 Obligation 和重复行动指纹。Action Board 是合法候选空间，不是固定任务队列。

如果当前 Board 中没有未提交的 Case、Frontier、Formation 或 Obligation Action，AV 不把它解释为目标审计完成，而是生成一次 fresh `refresh_discovery`：利用最新 Security Map、Reviewed Negative Knowledge、已完成处置和未覆盖攻击面重新进行安全关系综合。只有外部运行边界或用户停止能够结束本次审计；Portfolio 收敛没有“目标安全”的科学含义。

Frontier 的当前 revision 一旦实际形成 Case，就退出 Exploration Board，避免同一方向在探索与成熟阶段同时空转；后续新的 grounded update 会增加 Object revision 并重新打开该 Frontier。`hold` 则保留具体 resume condition，不自动占用调查循环；只有新 Observation/Frontier 实质改变该条件时，Fresh Formation 才能建立新 Revision 并重新激活 Case。

如果一次 Frontier 调查没有产生任何新的 grounded Object，Runtime 不会伪造 Object/Exploration revision 来反复恢复同一 sticky 方向；该精确 Action 被记录后，Portfolio 回到 fresh discovery。这不表示审计完成，只表示当前 Frontier 没有产生新的科学变化；只有后续新的 grounded Object 才重新打开它。

每次 Reviewed Delta Commit 后：

1. 更新 Claim/OES；
2. 更新 Case/Revision；
3. 更新 Obligations；
4. 更新 Negative Knowledge；
5. 注册 derived Observation/Frontier；
6. 重建 Action Board；
7. 保存 Board Delta；
8. 启动 Fresh Portfolio Selection。

完整因果链：

```text
Reviewed Delta
→ Canonical OES Change
→ Action Board Change
→ Fresh Planner Selection
→ Investigator Artifact
→ Fresh Review
→ New Reviewed Delta
```

---

## 10. 不设置任何内部预算

AV 不设置：

- 固定 15 分钟 Episode；
- 动态时间配额；
- Token 预算；
- 最大调查轮数；
- Case 资源额度；
- 固定探索/成熟比例；
- 对 Agent 暴露的剩余运行时间。

Portfolio Control 表示“下一次调查机会给谁”，不表示给每个 Case 分配多少分钟。

正式实验仍有统一的外部墙钟限制，但它属于实验 Harness，不属于 AV 方法，也不暴露给 Investigator。外部 deadline 到达只产生 operational interruption，保存当前 Case/OES，不把 timeout 解释为 refuted，也不表示目标安全。

---

## 11. Capability Advisor

Capability Advisor 是 stateless、advisory-only sidecar：

```text
Evidence Obligation
→ Expected Artifact
→ Required Capability
→ Candidate Tool/Approach
```

Investigator可以使用、替换或忽略建议。Advice 不是 Evidence，不修改 OES，不自动创建 Case，不自动选择 Action，也不能根据 C/M/B/I/Own/S 固定映射工具。

---

## 12. Thin Runtime

Runtime 只严格校验自己拥有的事实：

- Run/Action/Attempt/Role/nonce；
- Case/Revision/Claim/Obligation identity；
- stateVersion/stateHash 和 stale base；
- produced files、路径安全和 Artifact lineage；
- Fresh Review 和 exact Revision binding；
- exactly-once、CAS、duplicate suppression；
- Claim dimension binding；
- orphan Artifact isolation；
- crash/resume 和 accepted-uncommitted replay；
- Report Gate、Negative Closure Gate 和 FindingIndex binding。

Runtime 不判断：

- Boundary 是否合理；
- Caller Misuse 是否成立；
- 某种原语是否天然危险；
- 哪个工具最合适；
- Case 应 Revision 还是 Branch；
- 某类 Artifact 天然只能支持哪个 Claim；
- 目标是否已经安全；
- 如何分配时间或 Token。

### Report Gate

Report 至少要求 exact current Revision、Fresh Reviewed Basis、C/M/B/I/Own/S 在 admitted Scope 下获得支持或限定、每个 Claim 都绑定已审查的证据层次/目标保真度/测试范围、无 Report-blocking Obligation，并明确 attacker path、target-owned Property、incremental Impact、Ownership 和 Scope。Runtime 只检查这些判断是否经过合法 Review 和引用链获得授权，不解释科学文本是否正确。

### Negative Closure Gate

Refuted 至少要求 scoped proposition、explicit falsifier、observed attempt、decisive Artifact、Fresh Review、exact identity 和 bounded closure scope。严格禁止 `unknown→refuted`、`blocked/timeout→refuted` 和 Case refutation 自动关闭整个 Frontier。

---

## 13. Exploration 与 Maturation

Exploration 包括 Security Map 更新、Observation 发现、Frontier 探索、Case Formation 和 Sibling Frontier；Maturation 包括 Case Investigation、Obligation Resolution、Claim Update、Case Revision、Fresh Review 和 Disposition。

两者不是固定阶段，也没有轮次比例。Active Obligation 不自动压倒所有 Frontier，Frontier Exploration 也不能无限逃避已有决定性未知。Portfolio Controller 根据最新状态表达科学理由，Runtime只验证选择来自 Action Board。

---

## 14. Stable、Qualified 与 Evolving Case

### Stable Confirmation

```text
Case → Investigator强Artifact → Fresh Review → Report/Refuted
```

适合内存破坏、明确注入、路径穿越和强 sanitizer oracle。AV 在这里退化为严格候选验证。

### Qualification-heavy

```text
Case → Mechanism成立 → 限定C/B/I/Own/S → 补证 → Bounded Disposition
```

适合配置相关问题、Caller-conditional API、授权范围和 Ownership 复杂问题。

### Evolving Investigation

```text
Case R1
→ Evidence改变攻击者/边界/根因
→ Case R2或Branch
→ 新Obligation
→ 新Evidence
→ Disposition
```

适合 Caller Misuse/Contract、状态机、复杂业务逻辑、跨组件协议、固件更新链和 IPC/权限生命周期。

AV 不声称所有漏洞都必须经历复杂 Case Evolution。它的主张是：稳定候选可以退化为一次验证；当 Evidence 改变安全问题本身时，Case/OES 能保存并利用这种变化。

---

## 15. Skill 与 Full AV

### AV-Skill

Skill 形式把同一方法压缩成一个 Coding Agent 能直接执行的调查手册：

```text
仓库Survey
→ 调查方向列表
→ 选择一个方向长期深挖
→ 在调查中定位、形成或修改具体漏洞假设
→ PoC、对照和关键证据
→ 批判性检查证据实际证明的范围
→ 继续、收窄、修订、拆分、结束或反馈新方向
```

Skill 不要求 Agent 机械执行 `Observation → Frontier → Case Formation`，也不要求在初始方向中填写 C/M/B/I/O/S。但它明确加强目标认知：先恢复目标角色、参与者及其已有能力、资产和敏感能力、关键边界与状态转换、具体执行检查、默认配置、契约和责任假设，再把通用漏洞知识映射成与这些目标关系相连的调查方向。目标认知是可修订的安全视图，而不是README摘要或要求先完整理解整个项目。初始方向随后只需目标认知依据、代码锚点、风险信号、开放问题和有效的开始动作；C/M/B/I/O/S只在证据检查和最终报告时用于防止结论扩张。

Agent使用少量持久文件维护 `target-context.md`、`directions.md`、单个方向的调查记录、原始Artifact和最终报告。它可以调用子Agent进行独立面调查、PoC复现或证据挑战，但不规定固定Worker拓扑。

当前实现：

```text
adaptivevuls/skills/adaptivevuls-open-repository-audit/
```

用于对比实验的冻结副本：

```text
/data/yy/newtest/对比实验/02_Fixed_Workflow_Agent_V1_38/
```

### Full AV

Full AV执行相同的Survey、长期调查、证据检查和反馈思想，但使用外部Runtime强制稳定身份、exact Revision、Artifact来源、Sticky调查、Fresh Review、stale rejection、CAS、exactly-once、Portfolio选择、Finding Gate和crash/resume。

```text
Skill AV：方法说明由Agent自行执行和维护
Full AV：Runtime显式保存并约束长期调查
```

因此Skill AV与Full AV的比较可以回答：收益主要来自方法知识，还是来自外部状态、独立Review和反馈控制。

---

## 16. 与现有工作的边界

AV 不应声称所有漏洞都需要复杂 Case lifecycle。

- 对稳定内存破坏和强 oracle，OpenAnt/Revelio 式“广扫候选→逐个确认”可能更简单有效；
- 对稳定但需要范围限定的问题，AV 的主要价值是 OES 和 Evidence-bounded adjudication；
- 对攻击者、边界、Ownership、根因或 Scope 会随 Evidence 改变的问题，完整 Case lifecycle 的价值最强。

真正要证明的是：

> 在异构开放审计中，存在一部分高价值调查，其安全问题会在验证中发生实质变化；冻结 Candidate Pool 容易丢失这种变化，而可修订 Case/OES 能够保留变化并据此重新控制调查。

---

## 17. 最新实验带来的认识

V1.35.6 Requests 一小时运行证明了：

- stable Case identity 和 Revision 可工作；
- Evidence 可以绑定 exact Revision；
- no_proxy Case 的 Evidence 正确反驳了初始命题；
- Claim-local state 和 FindingIndex lineage 可工作。

但唯一 Pickle Finding 很可能是错误提升：攻击前提已经是恶意 Python pickle，而恶意 pickle 本身已有任意代码执行能力。Producer、Fresh Reviewer 和 Adjudicator都接受了“Requests 的 `__setstate__` 额外构成 RCE”的 Frame。它暴露了两个问题：

1. Mechanism Artifact 被扩张为 Boundary/Impact/Ownership；
2. 两个相似 Fresh Worker读取相同 Frame，并没有形成真正正交的科学判断。

最新运行的时间结构也显示旧拓扑开销较大：一个约 7–8 分钟的 Producer 后，又有约 4–5 分钟 Reviewer 和约 2–3 分钟 Adjudicator。两个不运行工具的 Worker接近再次消耗一个完整调查 Episode，却没有阻止错误报告。

因此目标设计收敛为：

```text
Producer + Materializer
→ 一个Sticky Case Investigator的两种Action Scope

Reviewer + Adjudicator
→ 一个Fresh Case Reviewer的两种输出路径
```

V1.36 已经实现三主体闭环，并在 curl/Django 长运行中暴露了“Reviewed 但没有科学后继”的 Case closure 缺口；该缺口已经通过 transition completeness 在 V1.36 中修复。随后 AVH-VG-0003 运行又暴露了两个执行保真问题：不完整命题不能合法 Report，以及源码、局部执行、真实目标运行和独立复现没有被稳定地区分。V1.37 因此保留严格的 Report 六维完整性和逐 Claim 证据元数据，但不再把 Report 完整性前移到 Formation。这个修正不增加 Worker、不增加内部预算，也不把该目标的已知 Oracle 写入发现策略。

---


## 18. 当前主线：开放仓库中的调查方向与长期深挖

AV 不主张全面替代高召回静态分析或 RepoAudit。对于已知漏洞类型、source/sink 或可枚举程序属性，系统扫描再验证通常更系统、更高效。AV 研究的是另一种开放场景：漏洞类型、位置、触发路径、相关上下文和验证方法都可能尚未给定。

当前流程是：

```text
仓库广度调查
→ 形成一组有目标依据的调查方向
→ 选择一个方向进行长期 Coding-Agent 调查
→ 在调查中定位、形成和修改具体漏洞假设
→ 编写 PoC、对照或取得其他关键证据
→ Fresh Review 限定证据和结论
→ 继续、修订、拆分、形成新方向或结束
```

调查方向不是完整漏洞报告。它只需要具体目标锚点、观察到的风险信号或可疑安全关系、一个开放问题以及一个有效的开始动作。已经足够具体的方向可以直接进入验证；较浅的方向则在长期调查中完成定位和假设形成。

## 19. 三个主要方法点

第一，AV 使用目标背景、通用漏洞知识和仓库广度阅读形成与目标相关的调查方向，而不是要求第一阶段对每个方向预先写出完整漏洞结论。它要解决的是开放仓库中“应该深入调查什么”。

第二，一个具有连续工作记忆的 Coding Agent 围绕单个方向持续读代码、追路径、形成或修改具体假设、编写 PoC 和建立对照。证据检查点出现后，Fresh Reviewer 只承认证据实际覆盖的行为、条件和范围，并决定下一步问题或有界结论。

第三，经审查的新认识可以修改当前方向，也可以形成新的调查方向并重新进入仓库调查组合。只有当这种反馈改变了后续问题、行动、证据路径或最终结论时，它才超出普通记忆保存。

## 20. V1.38 实现

```text
survey_repository / refresh_survey（Fresh）
        ↓
Investigation Directions（Runtime 内部仍使用 Case ID）
        ↓
Fresh Portfolio Controller
        ↓
investigate_case / acquire_deciding_artifact（Sticky）
        ↓
Meaningful Evidence Checkpoint
        ↓
Fresh Case Reviewer（强制，不经过 Controller 选择）
        ↓
继续 / Revision / Branch / Terminal / Reviewed New Direction
        ↓
Canonical Portfolio Rebuild
        ↺
```

V1.38 不再将 `Observation → Frontier → Case Formation` 实现为多个必须依次调用的 Agent 阶段。一次仓库 Survey 直接输出调查方向；Observation 和 Frontier 只保留为来源与内部记录。初始方向可以没有 C/M/B/I/O/S Claim，具体 Claim 在长期调查取得证据后通过 Fresh Review 进入正式状态。完整 C/M/B/I/O/S 只在 Report Gate 强制。

科学主体仍然只有：Portfolio Controller、Sticky Investigator 和 Fresh Reviewer。Runtime负责身份、来源、状态提交、恢复和报告门槛，不判断安全语义。系统不设置内部科学时间、Token 或 Episode 预算，只有外部运行边界。

仓库调查使用：

```text
adaptivevuls/skills/repository-security-survey/SKILL.md
adaptivevuls/skills/repository-security-survey/references/target-risk-patterns.md
```

其中按 native、Web/API、框架/库、服务端、文件归档、开发工具、JVM、Go/Rust、二进制、固件、授权、多租户、插件、网络访问、依赖和状态生命周期等目标形态提供常见风险信息。这些信息只帮助发现方向，不构成目标漏洞证据。

## 21. 当前最准确的一句话

> AdaptiveVuls 面向漏洞类型、位置和验证方法均未预先给定的开放式仓库审计：它先通过目标相关的仓库调查形成一组有依据但成熟度不同的调查方向，再让 Coding Agent 在持续的代码探索和实验中共同完成漏洞定位、假设形成与验证，并由 Fresh Review 限定证据、结论和后续反馈。

## 22. V1.39：加强目标认知，但不增加新阶段

V1.39 只加强仓库 Survey 中的目标认知，不改动后续长期调查、证据推进和 Fresh Review。目标认知不再只是“这个仓库有哪些模块”的概括，而是一个可被后续证据修订的安全背景，至少关注：目标在什么环境中使用、有哪些现实参与者及其已有能力、哪些资产或敏感能力需要保护、目标自己承诺维护哪些边界或接口约定、哪些状态和生命周期变化与安全有关、这些关系由哪些代码实际执行，以及还有哪些会改变调查方向的未知。

目标认知和仓库广度阅读仍在同一个 Fresh Survey 会话内完成，不增加 Cognition Worker，也不要求先完整理解仓库再开始找方向。Survey 一边建立部分安全背景，一边将其与漏洞知识和实际代码信号结合。每个调查方向必须说明四个目标联系：谁能控制什么、涉及什么资产或敏感能力、怀疑哪条边界/约定/生命周期关系，以及相关检查或使用代码在哪里。这个说明只是方向为何值得深入的依据，不是提前补齐完整漏洞命题。

因此 V1.39 的前半段可以概括为：

```text
部分且可修订的目标安全认知
+
漏洞知识与仓库代码信号
→
有目标依据的调查方向
```

随后仍然是 V1.38 已冻结的主循环：Sticky Investigator 围绕方向建立部分证据状态，优先处理最影响结论的缺口，取得有意义的证据检查点；Fresh Reviewer 限定证据实际说明的内容并决定继续、修订、拆分或结束。系统仍然没有内部科学预算，也没有新增 Worker 或强制 Formation 阶段。

## 23. V1.40：同轮新 Claim 与证据缺口绑定，以及反馈去重

V1.39 的真实测试说明了两个状态表达问题。第一，Reviewer 在同一轮创建新 Claim 时，还拿不到 Runtime 随后才会分配的 Claim ID，因此无法把已经明确的证据缺口绑定到新 Claim；V1.40 允许 Reviewer 使用本轮 `new_claims.local_key`，Runtime 在一次原子提交中将它解析为正式 Claim ID。这样，文字中提出的关键未知不会在规范状态里丢失。

第二，当前 Case 的下一步调查不再允许同时作为新方向反馈。它必须留在当前 Case 的 Revision 或 Evidence Obligation 中。只有根因、资产、边界、安全属性、参与者能力或组件确实不同的独立安全问题，才能成为 Case-derived Frontier；Reviewer 必须明确说明它与当前问题的区别。Runtime只校验这种显式关系和来源，不进行模糊语义相似度判断。

V1.40 同时明确所有文件引用字段必须写成 `analysis.md#limitations` 或 `artifacts/result.json` 这样的真实路径引用，减少科学内容正确但事务格式错误所造成的额外 Handoff Repair。该版本没有增加 Worker、调查阶段、内部预算或新的安全裁决逻辑。

## 24. V1.41：开放仓库调查的自然停止

AV 无法像固定漏洞类型或固定工作列表的方法一样证明仓库已经被完整检查，但可以判断当前调查组合是否达到稳定状态。V1.41 因此不把外层时钟当作唯一停止条件，而是在没有活动 Case、待审证据、未完成迁移和开放调查对象后，要求一次 Fresh Refresh Survey 明确说明是否产生了新方向、目标背景是否发生重要变化，以及还有哪些可行动或只能有界保留的未知。

第一次没有新方向只形成收敛候选。Runtime随后使用同一个 Investigator角色启动一次 Fresh saturation challenge，专门检查是否遗漏参与者、资产、边界、状态变化、执行检查、跨组件关系或被新证据改变的早期假设。只有连续两次都没有新对象、没有覆盖变化且没有可行动未知时，运行才以 `portfolio_converged` 自然结束。

`portfolio_converged` 只表示在当前目标认知、漏洞知识、证据和可用能力下，调查组合不再产生合法的新工作；它不表示仓库安全。外部时间到期仍单独记录为 `external_runtime_boundary`，不会自动产生 Hold、Refuted、Report 或收敛结论。该改动没有增加新 Worker、Lane、内部预算，也没有改变“目标认知与Survey形成方向—Sticky深挖—Fresh Review—证据反馈”的主线。

## 25. V1.42：从属性补全回到连续漏洞调查

V1.42 根据现有实验做了方法收缩。Part-B 证明了强 Agent 会产生真实机制与过强结论并存的问题；P0-C 只证明控制链能执行；Survey 消融尚未证明目标认知提高真实根因覆盖；ARVO-0002 又表明一个 Case 可以因为持续补范围和影响而推迟其他方向。因此，OES 字段、状态持久化和 Worker 编排不再被视为科学收益。

V1.42 将第二部分统一表述为：

> **基于证据约束的漏洞假设形成、修订与验证。**

其中“证据驱动”表示当前最能改变判断的未知决定下一项行动；“证据约束”表示结论不得超过 Artifact 实际测试的条件、层次和范围。定位、假设形成和验证在同一个 Sticky Coding-Agent 调查中共同发生，不再是“第一阶段生成完整候选、第二阶段只写 PoC”。

V1.42 的具体收缩是：

1. Investigator 在高成本复现前先检查目标行为、攻击者与安全属性关系、以及可区分实验是否具体；低信号方向可以早期提交 Checkpoint；
2. Evidence Obligation 可以先于 Claim 存在，不必为了继续调查创造占位 Claim；
3. 首个持续调查已经获得完整窄命题证据时，Fresh Reviewer 可以同轮形成并审查初始 Claim，避免为了身份分配强制第二次调查；
4. 新增 `not_substantiated` 用于有界关闭未形成具体安全命题、也没有决定性后继的低信号方向，不把它伪装成 Refuted 或 Hold；
5. Controller 不得因为缺少报告字段、想扩大 Scope/严重度或继续累积相同层次复现，而反复打磨一个 Case；只有能区分解释、改变命题或产生决定性 Artifact 的下一步才与其他方向竞争。

V1.42 依然是待验证的方法假设，不是已经证明有效的结论。后续必须以正确根因、正确处置、决定性 Artifact、错误关闭、覆盖损失和成本为结果，不能以 Claim 数、字段完整率或迁移次数作为效果证明。

## 26. V1.43：目标级 Investigator Session

V1.43 不改变 V1.42 的方法主线、科学对象、Transition completeness、Finding Gate 或停止规则，核心修正 Coding Agent 的工作记忆切分。初始目标认知与 Survey、所有普通 Case 调查、Case Revision、Branch 和跨 Case 切换现在共享一条目标级 Sticky Investigator Chain。这使 Agent 能保留仓库导航、构建、工具和跨组件理解，而不必在每个 Case 或 Revision 上重新恢复。

连续工作记忆不拥有科学权限。每次 Investigator 调用都显式声明当前 Canonical Packet 优先，绑定 exact Case/Revision、active Obligation 和最近 Reviewer transition，并禁止跨 Case 静默复用 Evidence、Claim、Scope 或 disposition。Fresh Reviewer仍然看文件化 Artifact 和规范状态，而不是主 Investigator 的隐藏对话。

Controller、Reviewer、Refresh Survey 和 Saturation Challenge 继续保持科学上的 Fresh。为同时支持同一事务的格式修复，Harness 给每个 Fresh 科学任务分配一个唯一 Sticky transaction key：不同 Portfolio 决策不共享，不同 Evidence Checkpoint 不共享，不同覆盖挑战也不共享；只有同一决策或同一 Checkpoint 的 repair 可以恢复原 Session。

主 Investigator 的 Hybrid continuity 可在获得真实 context-window telemetry 时允许 native compression，并在容量接近上限后执行 seal、digest、bootstrap 和 native Session 轮换。逻辑 Investigator Chain 不变。该机制是运行容量管理，不是内部科学预算，也不会改变 Case 结论。

V1.43 直接复用冻结的 V1.42 Handoff 校验和 Projection 模块，因为科学状态及其合法迁移没有变化；V1.42 文件及实验输出不回写。完整设置开发运行暴露出一个实现不一致：Claim-free 初始方向在第一次 Review 中形成 `case_revision` 后，Reviewer 先后选择了 `continue` 和 `local_update`，而该非终态 preserve Revision 的合法载体应为 `revise`。因此 V1.43 新增薄的 `security-case-handoff-v1-43.mjs` 适配层，明确并确定性执行 `preserve Revision→revise`、`branch Revision→branch`，同时保存修复前 Handoff；它不修改 Claims、Evidence、Obligations、Disposition 或分析。若 Mandatory Review 最终仍无法提交，Finalization 必须报告 `failed-review-handoff / protocol_failure / operationallyValid=false`，不再伪装成正常外部时间停止。

一次真实 OpenCode 开发性 smoke 已验证：初始 Survey 与后续 Case Investigation 在实际 Harness 中共享同一 `v1-43:target-investigator` Key、同一 Harness Session 和同一 OpenCode native Session ID；Controller 与 Reviewer 使用各自独立 Key。该运行仅是 Session 拓扑测试，不计入漏洞效果实验。详见 `adaptivevuls/evaluation/v1-43-target-sticky-live-smoke-20260731/SMOKE_RESULT.md`。
