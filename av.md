# AdaptiveVuls（AV）项目理解

> 本文记录当前对 AdaptiveVuls 研究问题、方法对象、执行闭环、Runtime 边界、会话记忆和论文定位的统一理解。V1.35.6 是形成这一设计的历史实现；截至 2026-07-29，V1.36 已将三个科学主体、Sticky Case Investigation、单次 Fresh Reviewed Delta 和确定性 Finding 投影实现为可运行协议。

## 1. 一句话定义

AdaptiveVuls 是一种面向开放式仓库审计的证据约束调查方法。它以具有稳定身份、但可随证据修订的 Investigation Case 作为长期调查单元，用 Operational Evidence State（OES）表示当前安全判断、竞争解释和决定性证据义务。具有 Case 局部记忆的 Coding-Agent Investigator 根据 OES 持续获取 Evidence；Fresh Reviewer 独立限定 Evidence 的实际证明范围，并提出 OES 更新、Case Revision、Branch 或有界处置；Runtime 将合法的 Reviewed Delta 原子提交到 Canonical State，随后重新构建调查组合并选择下一项行动。

最简闭环：

```text
Case + OES
    ↓
Goal-scoped Sticky Case Investigation
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

AV 的核心不是多 Agent 编排、更多阶段、更多字段或固定工具流水线，而是：

> Evidence 如何改变正在调查的安全问题，并进一步改变整个调查 Portfolio 的后续行为。

V1.36 的三个 lane 是权限与视角边界，不是并行流水线；主循环保持顺序执行，同一时刻最多运行一个 Coding-Agent 会话。Controller 和 Reviewer 每次 Fresh，Case Investigator 在同一 Case Revision 内 Sticky，Revision/Branch 后使用新会话。Capability Advisor 是本地 advisory-only JavaScript，只向承担对应 Obligation 的 Case Investigator交付建议，不是第四个 Worker。

V1.36 不接受生产 Controller 轮数上限，也不把时间、Token、Episode 或剩余运行时间写入 Packet/Prompt。正式实验的统一墙钟只作为外部停止边界。代码中的 `testOnlyMaxControllerDecisions` 仅用于有限 mock simulation，不属于方法或正式实验配置。

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

---

## 3. 核心设计原则

### 3.1 开放探索，约束提升

```text
Permissive Exploration
+
Constrained Claim Promotion
```

Agent 可以自由阅读、搜索、使用工具、提出 Observation、Frontier、Case proposal 和 Evidence proposal；但 Case admission、Claim support/refutation、Case Revision、Negative Closure 和 Report admission 必须满足相应来源、审查和状态迁移条件。

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

Case 是稳定身份、内容可修订的可证伪安全命题：

```text
Frame(c) = <
  Security Proposition,
  Supporting Observations,
  Candidate Property/Boundary,
  Attacker Model,
  Supporting Vulnerability Explanation,
  Alternative Vulnerability Explanation,
  Alternative Non-vulnerability Explanation,
  Initial Scope
>
```

Case admission 要求：

```text
Grounded
∧ Security-Relevant
∧ Falsifiable
∧ Alternative-Aware
∧ Scoped
```

Case 可以在 Boundary、Impact、Ownership、Scope 未知时存在，不要求 PoC、严重性、完整攻击链或 Report Evidence。无目标依据、不可证伪的纯叙述不能形成 Case。

### 4.5 Claim：C/M/B/I/Own/S

- **C — Control：** 攻击者、调用者、管理员或环境能控制什么，入口是否可达。
- **M — Mechanism：** 目标行为或状态变化是否发生，以及发生条件。
- **B — Boundary/Property：** 目标拥有或承诺的安全边界、不变量或接口约定是否被违反。
- **I — Impact：** 目标机制增加了什么攻击能力或安全损失。
- **Own — Ownership：** 责任属于目标、调用者、部署、Dependency 还是其他组件。
- **S — Scope：** 适用版本、配置、入口、参与者、部署和环境。

必须防止 M supported 自动扩张为 I supported 或 Report。

### 4.6 Competing Explanations

每个 Case 至少保留：

1. Supporting Vulnerability Explanation；
2. strongest Alternative Vulnerability Explanation；
3. strongest Alternative Non-vulnerability Explanation。

非漏洞解释包括 Expected Behavior、Caller Misuse、Dependency Ownership、Deployment Responsibility、Missing Attacker Control、No Incremental Capability、Invalid Security Boundary 和 Impact Inflation。

### 4.7 Operational Evidence State（OES）

```text
OESₜ(c) = <Qₜ(c), Altₜ(c), Obligationsₜ(c)>
```

其中：

- `Qₜ(c)`：Claim 状态、Scope、Supporting/Refuting Artifacts 和 residual unknowns；
- `Altₜ(c)`：竞争解释及当前判别状态；
- `Obligationsₜ(c)`：仍可能改变处置的 Evidence Obligations。

OES 不是完成度评分或漏洞分类器。它的作用是把当前未知转换成可执行调查，并在 Evidence 变化后重新控制行动。

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
  subject Case/Claim,
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
merge
```

终态 Disposition：

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

负责从最新 Canonical State 和 Eligible Action Board 选择下一项调查行动。不运行漏洞实验，不修改科学真值，不重新研究整个仓库，也不使用固定数值算法。

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

两者使用相同工具权限、Evidence Contract 和 Case Session。

Fresh Review 一次完成：

```text
Evidence Review
+
Case Transition
```

原 Reviewer 与 Adjudicator 不再作为两个常规 Coding-Agent Worker。常规 `adjudicate_case` 应被删除；Reviewer 的合法 transition 由 Runtime Gate 提交。

---

## 6. 会话记忆

推荐会话模型：

| 角色/动作 | 会话模式 |
|---|---|
| Portfolio Controller | Fresh per decision |
| Cognition / Security Map Revision | Fresh |
| Frontier Explorer | Sticky per Frontier ID |
| Case Formation | Fresh |
| Case Investigator | Sticky within current Case proposition |
| Fresh Case Reviewer | Fresh per Evidence Checkpoint |
| Reporter | Fresh 或确定性 |
| Capability Advisor | Stateless |

### 6.1 Local OES Update

Claim 状态、Evidence、Obligation、局部 Scope 和 competing explanation 的变化不创建 Case Revision，继续原 Sticky Investigator Session。

### 6.2 True Case Revision

Security Proposition、attacker model、root-cause frame、Boundary/Property、Ownership question 或主解释结构发生变化时，创建真正 Revision，并使用 Fresh Investigator on New Revision。

### 6.3 Branch/Sibling Case

不同安全目标、Boundary、根因或独立影响路径使用新 Case ID 和 Fresh Investigator。

```text
OES局部成熟 → Sticky
Case命题修订 → Fresh
Sibling/New Case → Fresh
```

Sticky Session 只是效率优化。Session 丢失后必须能从 Case、OES、Artifacts、Attempts、Negative Knowledge 和 Reviewed Delta 重建；否则外部化不完整。

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

每次 Review 必须满足 **Case Transition Completeness**：如果选择 `local_update` 或 `continue`，提交后的当前 Revision 必须仍有至少一个明确、可行动、会改变处置的 Evidence Obligation；如果已经没有决定性未知，就必须依据现有证据选择有界终态；如果调查命题发生实质变化，则使用 Revision/Branch。禁止形成“继续，但不知道继续验证什么”的成熟 Case 滞留状态。Runtime 只机械检查后继结构是否存在，不替 Reviewer 决定具体处置。

如果 Reviewer 引入或替换新的 attacker model、root cause、Boundary、Ownership、影响族或关键 Claim，则必须 `revise/branch + continue`。新 Claim 初始为 unknown，不能在同一次 Review 中授权 Report。

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

Report 至少要求 exact current Revision、Fresh Reviewed Basis、required Claims 在 admitted Scope 下获得支持、无 Report-blocking Obligation，并明确 attacker path、target-owned Property、incremental Impact、Ownership 和 Scope。Runtime 只检查这些判断是否经过合法 Review 和引用链获得授权，不解释科学文本是否正确。

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

把同一 Security Case 方法作为调查手册注入 Coding Agent：理解目标、记录 Observation、探索 Frontier、形成 Case、使用 OES、获取 Evidence、挑战解释、处置 Case，并继续整个 Portfolio。Agent自行维护文件和调查连续性，也可以使用子 Agent。Skill 不模拟 Runtime，也不规定固定 Worker拓扑。

### AV-Enforced

使用外部 Runtime 强制 Canonical IDs、exact Revisions、Artifact lineage、Fresh Review、stale rejection、CAS、exactly-once、Action Board、Finding Gate 和 crash/resume。

```text
Skill：告诉Agent应该如何调查
Full：保证Evidence、状态和Transition在长期执行中不会失控
```

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

当前 V1.35.6 仍然是旧四角色实现，不能把本文目标拓扑误写成已经完成的代码状态。

---

## 18. 论文核心命题

AV 最终应验证：

### H1：Case Formation

带来源 Observation 是否能更稳定地形成可证伪、可裁决的安全问题。

### H2：Evidence-bounded Adjudication

Fresh Review 是否减少 Mechanism→Impact 扩张、Caller Misuse误报、Ownership错误、Scope inflation 和 Unsupported Report。

### H3：Evidence-conditioned Control

Reviewed Evidence 是否实际导致：

```text
OES变化
→ Action Board变化
→ 后续Action变化
→ 调查结果变化
```

### H4：Evolving Case必要性

对于 Evidence 会改变攻击者、Boundary、根因、Ownership 或 Scope 的调查，完整 Case lifecycle 是否比冻结 Candidate Pool 更有效。

AV 的贡献不能表述为“更多 Worker 更可靠”，而应表述为：

> 一个长期 Investigator 根据 OES 获取 Evidence；一个 Fresh Reviewer让 Evidence 获得有界科学语义；更新后的 Canonical State重新控制整个调查 Portfolio。

---

## 19. 最终对象、主体和动作

### 科学对象

```text
Security Map
Observation
Frontier
Investigation Case
Claim
Competing Explanation
OES
Evidence Obligation
Artifact
Reviewed Delta
Disposition
Finding
```

### 科学主体

```text
Portfolio Controller
Case Investigator
Fresh Case Reviewer
```

### 逻辑动作

```text
target_cognition
explore_frontier
form_case
investigate_case
acquire_deciding_artifact
review_case_checkpoint
update/revise/branch/merge/dispose
external_finalize
```

### 控制平面

```text
Canonical State
Eligible Action Board
Reducer
CAS
Exactly-once
Lineage
Freshness
Transition Gates
Recovery
FindingIndex
```

---

## 20. 最终总结

```text
Grounded Observations
        ↓
Security Frontiers
        ↓
Investigation Case + OES
        ↓
Sticky Case Investigation
        ↓
Evidence Checkpoint
        ↓
Fresh Case Review
        ↓
 ┌────────────┬───────────┬───────────┬────────────┐
 │Local Update│ Revision  │  Branch   │Disposition │
 └────────────┴───────────┴───────────┴────────────┘
        ↓
Canonical State + Action Board Rebuild
        ↓
Fresh Portfolio Selection
        ↺
```

最终最准确的一句话是：

> AdaptiveVuls 将开放式漏洞调查组织为一个由 Evidence 持续驱动的 Investigation Case lifecycle：带来源的 Observation 形成可证伪 Case，OES 将未决判断和竞争解释转换为调查问题，具有局部连续记忆的 Investigator 获取有意义的 Evidence Checkpoint，Fresh Reviewer 独立限定 Evidence 并授权 Case 变化，更新后的 Canonical State 随后重新控制整个调查 Portfolio。
