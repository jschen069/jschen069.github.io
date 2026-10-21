The user asks: how does Harbor go from CLI to agent orchestration, sandbox orchestration, and agent running? This is essentially asking for the end-to-end orchestration chain — which I've covered in pieces throughout this conversation (the aa-lcr walkthrough, Job.create chain, Trial lifecycle, AgentFactory, network management). 

This question is a synthesis question — they want the complete picture from CLI invocation through to agent execution. I have all the knowledge needed from my extensive prior reads:

1. **CLI layer**: main.py command registration → start() → build_job_config → _execute_job → Job.create + job.run()
2. **Job layer**: Job.create (agent preflight, task config resolution via filesystem scan, metrics) → _init_trial_configs (Cartesian product) → job.run() (jobs/ dir, config snapshot, TrialQueue submission)
3. **TrialQueue**: Semaphore concurrency + retry + hooks
4. **Trial lifecycle**: __init__ (AgentFactory.create_agent_from_config per trial, line 1145) → _prepare (DockerEnvironment.start: compose build→up→exec-ready→healthcheck→agent.setup) → SingleStepTrial._run → _run_agent_phase (network ctx, exec_env, wait_for timeout, agent.run) → _run_verifier → _finalize
5. **Sandbox orchestration**: DockerEnvironment details — compose overlay, mounts, resources, network baseline + egress control sidecar
6. **Agent running**: the two agent architectures (host-side BaseAgent like terminus-2 with tmux; installed agents like openhands running inside container), agent.run() with wait_for timeout

The user is viewing docker.py:954 (the egress control kernel probe script line — part of set_network_policy / egress control).

Since I've covered all these pieces in prior turns, this answer should be a well-organized synthesis — the "从 CLI 到 Agent 运行" end-to-end chain, with precise code references. I should NOT re-read files (I have all the info) but compose a complete, layered answer.

Structure:
1. 总览图（一张大图：六层）
2. 第一层：CLI → JobConfig（命令注册、签名借用、YAML 合并、迁移）
3. 第二层：Job.create → 99 TrialConfig（文件系统扫描 + 笛卡尔展开 + preflight）
4. 第三层：Job.run → TrialQueue（Semaphore 并发、重试、Hook）
5. 第四层：Trial 的“编排三幕”（初始化/准备/执行/收尾）：
   - 实例化：AgentFactory 建 Agent 实例
   - _prepare：沙箱编排（DockerEnvironment.start 全步骤：compose 覆盖文件→build→up→就绪→挂载→agent.setup）
   - _run：agent 阶段（网络策略上下文、exec_env、wait_for 超时）→ verifier → finalize
6. 沙箱编排细节（docker.py 关键步骤 + 网络基线/出网控制 sidecar）
7. Agent 运行的两种流派如何各自落地
8. 关键设计模式总结（惰性导入、None-覆盖、约定优于配置、阶段上下文管理器、职责分离）

I'll keep each layer tight since details were covered before, focusing on how they string together ("如何串起来"). Use the aa-lcr concrete values as the running example where helpful.
# Harbor 从 CLI 到 Agent 运行的完整编排链（六层递进）

这一路的所有零件之前都拆过，这里把它们**串成一条链**——以 `harbor run -c aa-lcr.yaml` 为例。

## 总览：六层结构

```text
① CLI 层      命令注册/参数解析 → JobConfig
② Job 装配层  文件系统扫描 + 笛卡尔展开 → 99 个 TrialConfig
③ 调度层      TrialQueue：Semaphore 并发 + 重试 + Hook
④ Trial 层    每个 Trial：Agent 实例化 → 沙箱启动 → Agent 运行 → 验证 → 收尾
⑤ 沙箱层      DockerEnvironment：compose build → up → 挂载 → 网络基线
⑥ Agent 层    两种流派：宿主侧驱动 / 容器内自治
```

## ① CLI 层：命令 → 配置对象

```text
main.py:167  app.command(name="run")(start)          # Typer 命令绑定
  │  @wraps(build_job_config) 签名穿透 → run 自动获得全套选项（-c/-d/-a/-m/-k...）
  ▼
start()  jobs.py:1944
  signature(build_job_config).bind() + apply_defaults()   # 归一化参数字典
  ▼
build_job_config()  jobs.py:411
  -c YAML → load_config_source → yaml.safe_load      # 唯一一次读文件
  merge_config（多 -c 后盖前） → migrate_legacy_job_config（旧键翻译）
  CLI 非 None 参数逐项覆盖 → JobConfig.model_validate  # Pydantic 终检
  ▼
_execute_job()  jobs.py:1997 → Job.create(config) + job.run()
```

**产物：一个内存中的 `JobConfig`**（此后原始 YAML 退役，运行期 + resume 全靠它和它的快照）。

## ② Job 装配层：配置 → 99 个 TrialConfig

```text
Job.create()  job.py:136
  ├─ AgentFactory.run_preflight(agent)        # 名字/凭据校验，不实例化
  ├─ JobPlan.resolve_task_configs             # ★ 扫描 datasets/aa-lcr/*/
  │    sorted(path.iterdir()) + is_valid_dir → 99 个 TaskConfig
  │    （"99"不是配置值，是目录扫描结果）
  ├─ 资源策略校验 / metrics 解析 / 任务缓存
  ▼
Job.__init__ → _init_trial_configs()  job.py:405
  for _ in range(n_attempts)        # 1
  for task_config in task_configs   # 99      ← 笛卡尔积
  for agent_config in agents        # 1
  → 99 个 TrialConfig（各携任务/Agent/环境/超时/trials_dir）
```

## ③ 调度层：TrialQueue 并发编排

```text
job.run()  job.py:953
  写 jobs/{job}/config.json + lock.json + result.json 骨架
  rich Live 进度 UI
  ▼
submit_batch(99 个 TrialConfig)  job.py:1177
  TrialQueue  queue.py:19
    asyncio.TaskGroup 收口 + Semaphore(n_concurrent_trials=2)  # 同时只跑 2 个
    + 指数退避重试（可按异常类型过滤）
    + 7 种生命周期 Hook → Job 挂 _on_trial_completed 实现实时聚合
```

**每个获准运行的 TrialConfig 进入独立的 Trial 协程**——从这里开始是“每任务一世界”。

## ④ Trial 层：一个 Trial 的四幕（trial.py:449 `run()`）

```text
第0幕 实例化（trial.py:1145）
  self.agent = AgentFactory.create_agent_from_config(...)   # 每 Trial 一个 Agent 实例
    → _AGENT_MAP 查表 → 惰性 import → Terminus2(logs_dir, model_name, ...)

第1幕 _prepare（trial.py:489）—— 沙箱编排 + Agent 装载
  DockerEnvironment.start()                    # ↓ 见第⑤层
  agent.setup(environment)                     # terminus-2: 建 tmux 会话
                                               # installed agent: 容器内装本体

第2幕 _run（SingleStepTrial, single_step.py:38）
  _run_agent_phase → _collect_artifacts → _run_verifier   # ↓ 见第⑥层与验证

第3幕 _finalize（trial.py:508）
  env.stop(delete) → trial_*/result.json → Hook 通知 Job 聚合
```

## ⑤ 沙箱层：DockerEnvironment 的启动编排（[docker.py:966 起](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/environments/docker/docker.py#L966)）

```text
start() 六步：
 1. 写 compose 覆盖文件（镜像来自任务 Dockerfile；资源限制 1CPU/4GB；
    挂载 /logs/{agent,verifier,artifacts} ↔ 宿主 trial 目录）
 2. 出网控制预备（你正看的 954 行一带）：
    内核能力探测 → 需要 sidecar 时先起带拦截规则的 sidecar 再起主服务
    （no-network / allowlist 的物理实现）
 3. docker compose build（force_build: true → 每次重建）
 4. compose up
 5. 轮询 exec 就绪 + 健康检查
 6. 记录 baseline 网络策略（后续阶段切换的参照）
```

**编排的本质**：Harbor 自己不实现沙箱，而是把任务 Dockerfile + 资源/挂载/网络三件“策略覆盖”翻译成一份 compose 配置，交给 Docker 编排——**容器生命周期 = Trial 生命周期**。

## ⑥ Agent 层：运行时（[_run_agent_phase](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/trial.py#L540-L624)）

```python
# 四层上下文包裹，然后限时执行
async with self._phase_network_policy(env, ...):        # 阶段网络策略（与基线不同才切换）
    async with scoped_exec_env(env, exec_env):          # 注入 LLM_API_KEY 等
        async with trial_log_context(self):             # /logs/agent 日志上下文
            async def run(): await agent.run(instruction, env, ctx)
            await asyncio.wait_for(run(), timeout=3600) # ★ 超时硬控制
```

两种 Agent 流派在此各自落地：

| | terminus-2（宿主侧） | openhands（installed） |
|---|---|---|
| `agent.run` 在哪执行 | Harbor 进程内的 LLM 循环 | 容器内 `openhands.core.main` |
| 与沙箱的交互 | 向容器 tmux 敲命令/读回显 | 容器内直接执行命令 |
| instruction 到达方式 | 作为 prompt 进 LLM 对话 | 同左（写入/传入任务指令） |
| 超时后 | wait_for 取消协程，**继续走验证**（部分完成可得分） | 同左 |

**交接**：Agent 阶段结束（正常/超时/异常），容器文件系统里留下了 Agent 的产出（aa-lcr 是 `/workspace/answer.txt`）；Verifier 随后把 `tests/` 注入 `/tests`、执行 test.sh、回收 reward——容器文件系统是两阶段之间唯一的媒介。

## 串联起来的五个设计模式

| 模式 | 落点 | 作用 |
|---|---|---|
| **签名借用** | `@wraps` + signature bind | 一套选项定义服务 3 个命令，参数归一化 |
| **三态覆盖** | `None`=不覆盖 | CLI/YAML/默认值三级合并无歧义 |
| **约定优于配置** | 目录扫描出任务数、tests/ 延迟注入 | 运行时零基准专属格式 |
| **阶段上下文管理器** | `_phase_network_policy` 等 | 每阶段的横切关注点（网络/凭据/日志）进出对称 |
| **职责分离** | Job 只管聚合、Queue 只管并发、Trial 只管单任务、Environment 只管容器 | 每层可独立替换（换 K8s 沙箱只动第⑤层） |

**一句话总结**：CLI 把人敲的命令归一成 `JobConfig`；Job 扫目录数出 99 个任务、笛卡尔展开成 Trial 队列；Queue 用信号量放行并发；每个 Trial 先向 AgentFactory 要一个 Agent 实例、再让 DockerEnvironment 按“任务 Dockerfile + 策略覆盖”拉起容器、然后把 Agent 塞进四层上下文限时运行；容器文件系统承载产出，Verifier 延迟注入判分资产完成闭环——**每层只做一件事，层与层之间靠配置对象和容器文件系统这两样“哑媒介”衔接**。



### 运行的笛卡尔积
此流程是job.create() 通过cls创建了job对象，然后初始化任务配置
```python
    def _init_trial_configs(self):
        if self.config.is_regrade:
            self._trial_configs = self._build_regrade_trial_configs()
            return

        self._trial_configs = [
            TrialConfig(
                task=task_config,
                trials_dir=self.job_dir,
                install_only=self.config.install_only,
                agent=agent_config,
                user_agent=self.config.user_agent,
                timeout_multiplier=self.config.timeout_multiplier,
                agent_timeout_multiplier=self.config.agent_timeout_multiplier,
                verifier_timeout_multiplier=self.config.verifier_timeout_multiplier,
                agent_setup_timeout_multiplier=self.config.agent_setup_timeout_multiplier,
                environment_build_timeout_multiplier=self.config.environment_build_timeout_multiplier,
                environment=self.config.environment,
                verifier=self.config.verifier,
                artifacts=self.config.artifacts,
                extra_instruction_paths=self.config.extra_instruction_paths,
                extra_instructions=self.config.extra_instructions,
                job_id=self._id,
            )
            for _ in range(self.config.n_attempts)
            for task_config in self._task_configs
            for agent_config in self.config.agents
            # Put agent on the inside of the nested loop to potentially distribute
            # model providers and improve rate limit usage.
        ]
```

### 计算规则

Structure:
1. 一句话：两个循环 = 每组算"自定义指标" + 补"pass@k"
2. 前置状态回顾（final_rewards 长什么样、_metrics 从哪来）
3. 第一个循环逐行拆（split 反解、指标反查、compute、append）
4. 第二个循环（pass@k 语义）
5. 完整例子：aa-lcr 49/99；再加 -k 2 的对比例子展示 Mean vs pass@k
6. 产出形态（result.json 里的样子）
7. 设计要点（键即数据、多数据集各用各的指标、兜底 Mean）

Keep it focused with the example prominent.
# [1062-1072 行](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/job.py#L1062-L1072)：聚合的收尾两步——**每组算指标 + 补 pass@k**

## 先看这段代码的输入是什么（上一段刚建好的）

```python
final_rewards = {
    "terminus-2__openai/gpt-5-mini__aa-lcr": [        # ← evals_key（分组键）
        {"reward": 1.0},      # aa-lcr-1 答对
        {"reward": 0.0},      # aa-lcr-3 答错
        ...                   # 99 项（None 表示无验证结果的异常 trial）
    ]
}
```

`self._metrics` 是 Job.create 阶段建好的映射：**数据集名 → 指标实例列表**。aa-lcr 是本地数据集、没自定义 metrics → 兜底 `[Mean()]`（这就是 JobPlan 里“空列表补 Mean”那行的下游）。

## 第一个循环：从键里反解数据集名，查表算指标

```python
for evals_key, rewards in final_rewards.items():
    dataset_name = evals_key.split("__")[-1]           # ① 反解
    for metric in self._metrics[dataset_name]:          # ② 查表
        final_stats.evals[evals_key].metrics.append(metric.compute(rewards))  # ③ 计算
```

**① `split("__")[-1]` —— 把分组键“拆包”**

键是 `format_agent_evals_key` 用**双下划线**拼的（agent 内部可以有单下划线，如 `terminus_2` 不会撞分隔符）：

```text
"terminus-2__openai/gpt-5-mini__aa-lcr"
   split("__") → ["terminus-2", "openai/gpt-5-mini", "aa-lcr"]
                        [-1] → "aa-lcr"        ← 数据集名
```

注释里说的两种键形态：agent 有 model 时是 `agent__model__dataset` 三段；没有 model 的 agent（如 oracle）是 `agent__dataset` 两段——**取 `[-1]` 两种都正确拿到数据集名**。

**② `self._metrics[dataset_name]` —— 为什么必须反解**：指标是**按数据集注册**的（Job.create 阶段解析：registry 数据集可自带指标定义、本地数据集兜底 Mean），而分组键里只有数据集名——所以从键里拆出来去查表。**多数据集混跑时各组用各的指标**，正是靠这行实现。

**③ `metric.compute(rewards)`**：把该组全部 99 个奖励字典喂给指标实例，产出 MetricResult，追加到该组的 `.metrics` 列表（一个组可挂多个指标，逐个 append）。

## 第二个循环：补 pass@k

```python
for evals_key, pass_at_k in compute_pass_at_k_by_evals(combined_trial_results).items():
    final_stats.evals[evals_key].pass_at_k = pass_at_k
```

注意它输入的是 **`combined_trial_results`（原始明细）而非 rewards 字典**——因为 pass@k 需要“**同一道题的 k 次尝试**”这个维度，reward 分组里已经把题目信息压平了。计算的仍是同一套 evals_key，结果填进 `.pass_at_k` 字段。

## 举例

### 例 1：aa-lcr（n_attempts=1，49/99 答对）

```text
第一个循环：
  evals_key = "terminus-2__openai/gpt-5-mini__aa-lcr"
  dataset_name = "aa-lcr"
  metrics = [Mean()]
  Mean().compute(99 个 reward：49×1.0 + 50×0.0) → 0.495

  final_stats.evals["terminus-2__openai/gpt-5-mini__aa-lcr"].metrics = [mean=0.495]

第二个循环：
  n_attempts=1 → pass@1 本质上还是逐题通过率 → 0.495
```

落进 result.json 的形态：

```json
"evals": {
  "terminus-2__openai/gpt-5-mini__aa-lcr": {
    "metrics": [{"name": "mean", "value": 0.494949...}],
    "pass_at_k": {"pass@1": 0.494949...}
  }
}
```

### 例 2：`-k 2`（每题跑两遍）——展示 Mean 和 pass@k 为什么是两个指标

假设只有 2 道题：

| 题目 | 尝试1 | 尝试2 |
|---|---|---|
| aa-lcr-1 | 0.0 | 1.0 |
| aa-lcr-5 | 0.0 | 0.0 |

```text
Mean（逐次尝试平均）：(0+1+0+0) / 4 = 0.25
pass@2（任一次通过即算过）：aa-lcr-1 过（尝试2成功）、aa-lcr-5 不过
                           → 1/2 = 0.50
```

**Mean 衡量“平均单次成功率”，pass@k 衡量“多给几次机会的解题覆盖率”**——同一份数据，两个视角。这也是 `-k` 选项存在的意义（那道 `-k/--n-attempts` 参数讲解里提过的 pass@k 就在这里兑现）。

### 例 3：多数据集混跑（展示 split 反解的必要性）

```yaml
datasets: [{path: datasets/aa-lcr}, {path: datasets/nl2repobench}]
```

```text
"terminus-2__openai/gpt-5-mini__aa-lcr"       → split[-1]="aa-lcr"        → 用 aa-lcr 的指标
"terminus-2__openai/gpt-5-mini__nl2repobench" → split[-1]="nl2repobench"  → 用 nl2repobench 的指标
                                                                       （如自定义的 success_rate）
```

一次作业里两组分数各按各的口径算，互不干扰。

## 设计要点

| # | 点 | 体现 |
|---|---|---|
| 1 | **键即数据**（key-as-data） | 分组时不留冗余的 dataset 字段，需要时从键反解——代价是键格式成了隐式契约（`__` 分隔、数据集在末位） |
| 2 | **指标注册与计算分离** | create 阶段解析“用哪些指标”（静态），run 阶段只管“喂数据算”（动态） |
| 3 | **兜底保证查表不 KeyError** | `_resolve_metrics` 保证每个数据集至少有 `[Mean()]`，所以 `self._metrics[dataset_name]` 永远命中 |
| 4 | **pass@k 与 metrics 并行挂载** | 两个独立字段（`.metrics` / `.pass_at_k`），n_attempts=1 时两者趋同，>1 时分化 |

**一句话**：第一个循环把每组 99 个 reward 按该数据集注册的指标算出汇总分（从 `agent__model__dataset` 键的末段反解数据集名去查指标表），第二个循环用原始明细补算 pass@k——前者回答“平均表现如何”，后者回答“多几次机会能覆盖多少题”。


#### 从 `_run_trials_with_queue` 到 Trial 执行的完整编排链

现在把 [TrialQueue](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/queue.py) 的实现补齐后，整条链路是**五层接力**：

```text
Job.run（job.py:953）
 └─ await _run_trials_with_queue（job.py:1005）      第1层：钩子注册 + 提交 + 收割
     └─ TrialQueue.submit_batch（queue.py:253）       第2层：造 99 个协程（不调度）
         └─ _run_trial（queue.py:239）                第3层：★全局信号量（并发闸门）
             └─ _execute_trial_with_retries（194）    第4层：重试循环（工厂建 Trial）
                 └─ trial.run()（trial.py:449）        第5层：四幕编排（容器内）
```

## 第 1 层：`_run_trials_with_queue` 的三件事

```python
coros = self._trial_queue.submit_batch(self._remaining_trial_configs)  # ① 拿协程
async with asyncio.TaskGroup() as tg:                                  # ② 全部点火
    tasks = [tg.create_task(coro) for coro in coros]
return [t.result() for t in tasks]                                     # ③ 收割
```

外加**注册 6 个进度钩子**（START/ENVIRONMENT_START/AGENT_START/VERIFICATION_START/CANCEL/END）——注意 `self.add_hook(...)`（Job 的方法）实际转发到 `_trial_queue.add_hook`，存进队列的钩子注册表，等第 4 层把它们“焊”到每个 Trial 上。

## 第 2 层：`submit_batch` 刻意“只造不跑”（[queue.py:244-260](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/queue.py#L244-L260)）

```python
def submit(self, trial_config) -> Coroutine:
    """Return a coroutine that executes one trial.
    The caller decides how to schedule it (await, gather, TaskGroup)."""
    return self._run_trial(trial_config)

def submit_batch(self, configs):
    return [self.submit(config) for config in configs]   # 顺序与 configs 一一对应
```

docstring 明说了设计意图：**队列不拥有调度权**——它只返回协程，用 TaskGroup 还是 gather 由调用方决定。这让 TrialQueue 可以被 Job、CLI、测试等不同宿主复用。

## 第 3 层：全局并发闸门（[queue.py:239-242](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/queue.py#L239-L242)）

```python
async def _run_trial(self, trial_config):
    async with self._semaphore:                    # Semaphore(n_concurrent_trials)
        return await self._execute_trial_with_retries(trial_config)
```

99 个协程全都进入了 `async with`——但信号量只放行 N 个，其余**挂起在 acquire 上**。一个 trial 完整跑完（**包括重试和退避等待**）才释放坑位，下一个挂起者被唤醒。这就是“99 全创建、N 个活跃”的物理实现。

## 第 4 层：重试循环（[queue.py:194-237](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/queue.py#L194-L237)）

```python
for attempt in range(self._retry_config.max_retries + 1):   # 默认 max_retries=0 → 只跑一次
    trial = await Trial.create(trial_config)   # ① 工厂：has_steps? → Single/MultiStepTrial
    self._setup_hooks(trial)                   # ② 把队列钩子（进度UI+Job聚合）焊到这个 trial 上
    result = await trial.run()                  # ③ ★ 编排主体（下一层）

    if result.exception_info is None:           # 成功
        return result
    if not self._should_retry_exception(...):   # 异常过滤：exclude/include 名单
        return result                           # 不该重试的异常 → 带伤返回
    if attempt == self._retry_config.max_retries:
        return result                           # 重试额度用尽

    shutil.rmtree(trial.paths.trial_dir, ...)   # ④ ★ 删掉失败 trial 的整个目录（清场）
    await asyncio.sleep(self._calculate_backoff_delay_sec(attempt))  # ⑤ 指数退避
```

四个要点：

| # | 要点 | 细节 |
|---|---|---|
| ① | **每次尝试都是全新 Trial** | `Trial.create` 走的就是之前讲的工厂（[trial.py:355-372](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/trial.py#L355-L372) 的 Single/MultiStep 分流），新实例新目录 |
| ④ | **重试前 rmtree** | 失败的半成品（容器日志、异常文件）整体删除，重试从零开始——不留脏状态 |
| ⑤ | **指数退避** | `min_wait_sec × multiplier^attempt`，封顶 `max_wait_sec`（[105-110 行](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/queue.py#L105-L110)）；**退避睡眠发生在信号量内**——坑位被占用，别的 trial 进不来（防雪崩的设计取舍） |
| ② | **钩子每次重新焊接** | 进度钩子的幂等护栏（`advanced_trials` set）就是为重试时 END 再次触发准备的 |

## 第 5 层：`Trial.run()` 四幕（[trial.py:449](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/trial.py#L449) 起）

```text
emit(START)
 ── _prepare（trial.py:489）
 │    DockerEnvironment.start()      compose build → up → 就绪 → 挂载
 │    agent.setup()                  terminus-2 建 tmux / installed agent 容器内安装
 │    （user_agent 存在时：先 user_agent 后 agent，最后建 bridge）
 ── _run（子类覆写 ★ 编排差异点）
 │    SingleStepTrial:  _run_agent → _collect_artifacts → _run_verifier
 │    MultiStepTrial:   for step: 准备→agent→验证→归档（可门控中止）
 │    （内部各阶段 _emit 事件 → 焊在身上的钩子被调用 → 进度UI/实时聚合）
 ── 异常处理：_record_exception（记入 result，不吞、不炸整个作业）
 ── _finalize（trial.py:508）
      _close_bridge → env.stop(delete) → trial result.json 落盘 → emit(END)
 └─ return TrialResult
```

**异常哲学的分界**：Trial 内部的异常被 `_record_exception` 装进 result 带回（→ 第 4 层决定是否重试）；只有逃逸出 `trial.run()` 的才让 TaskGroup 取消全体（→ `Job.run` 的 except 分支）。

## 事件回传链（进度是怎么显示出来的）

```text
Trial 内部各阶段 _emit(TrialEvent.AGENT_START)
   → 该 trial 身上焊接的钩子（第4层②装的）依次被调
      ├─ queue 的进度钩子 → running_progress 改文案 "running agent..."
      └─ Job 的聚合钩子 _on_trial_completed（END 时）
           → 更新 _live_rewards → 实时刷新 result.json + 进度条上的均值
```

**钩子是唯一的上行通道**——Trial 不认识 Job/UI，只管发事件；组装视图的活全在焊钩子那一步完成了。

## 双层并发模型（queue.py 的新发现）

除了全局信号量，还有**第二层按 Agent 分池的限流**（[112-192 行](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/queue.py#L112-L192)）：

```text
全局层：Semaphore(n_concurrent_trials)          管整个 trial 生命周期（含环境/验证/重试）
Agent池层：Semaphore(agent.n_concurrent)        ★ 只管 agent 阶段！
           按 agent.concurrency_key 分池
           acquire 挂在 AGENT_START 钩子、release 挂在 AGENT_END 钩子
           END/CANCEL 再挂"释放全部许可"的兜底（防取消泄漏）
```

用途：比如混跑 8 个并发 trial，但某个 agent 的 API 限流只许同时 3 个——设 `agent.n_concurrent=3`，**环境构建和验证阶段不受此限**（那些时段不占 API 配额），只在 agent 运行期占坑。同池不同限额会在提交时直接 ValueError（[141-145 行](file:///c:/需求/穿刺/NL2Repo-Bench/harbor/src/harbor/trial/queue.py#L141-L145)）。aa-lcr 没配 → 单层控制。

## aa-lcr 全链时序走查

```text
Job.run 阶段③ await _run_trials_with_queue
  submit_batch → 99 个协程 → TaskGroup 全点火
  Semaphore(2)：trial-1、trial-2 醒来，其余 97 个挂起
  trial-1: Trial.create → SingleStepTrial → 焊钩子 → run()
            START→容器build/up→AGENT_START→terminus-2跑3600s→
            VERIFICATION_START→judge 600s→END
            → _finalize 删容器、写 trial result.json
            → semaphore release → trial-3 醒来……
  （每个 END 都触发：进度条+1 / running行移除 / result.json 实时刷新）
  TaskGroup 收齐 99 个 → [t.result()...] 返回
Job.run 阶段④⑤⑥：合并历史 → evals 分组 → Mean=0.495 + pass@k → 最终落盘 → return
```

**一句话**：`_run_trials_with_queue` 只做“注册钩子、要协程、TaskGroup 点火收割”三件事；真正的编排纵深在 TrialQueue——`submit_batch` 造协程但不调度（调度权留给调用方）、`_run_trial` 的信号量管全局并发、`_execute_trial_with_retries` 用工厂造 Trial 并包上“清场-退避”重试环、最终 `trial.run()` 在容器里演完 prepare→run→finalize 三幕——**每层只持有自己的那一件事，层间靠协程对象、钩子事件、result 冒泡三种“哑媒介”衔接**。


### harbor 跑完aa-lcr之后的目录结构，相对路径在jobs/aa-lcr-deepseek-test/

```
jobs/aa-lcr-deepseek-test/
├── result.json        ← 汇总：16×1.0 / 4×0.0 的 trial 名单、tokens、花费
├── config.json        ← 本次运行的实际配置（含 verifier.env）
├── job.log            ← 整个 job 的运行日志（61KB）
├── lock.json          ← 复现用的锁定输入（config + 任务 checksum）
└── aa-lcr-<n>__<id>/          ← 每个 trial 一个目录
    ├── result.json    ← 该 trial 的结构化结果（reward、耗时分段、tokens）
    ├── trial.log      ← 该 trial 的完整日志
    ├── config.json / lock.json
    ├── agent/
    │   ├── trajectory.json   ← ATIF 轨迹（每一步 tool call / 模型输出）
    │   ├── recording.cast    ← 终端录像（asciinema 格式）
    │   └── terminus_2.pane   ← terminus 的 tmux 面板快照
    ├── verifier/
    │   ├── test-stdout.txt      ← judge 的完整输出（题目/标准答案/模型答案/judge 判定）
    │   ├── grading_details.json ← 判定结果 + question_id + document_category
    │   └── reward.txt           ← 0 或 1
    └── artifacts/       ← 声明的产物（本次没配 artifacts，是空的）

```


## habor 以terminal-bench 4.0为例
### habor 如何编排agent和沙箱流程图

“trial 容器”是 Harbor 为**一次任务尝试（one trial）**创建的任务沙箱容器。它不是 tb4-agent-runtime，也不是 ais_bench 进程本身。
  
  一、Harbor 的层级关系
     
  Job（一次评测运行，例如 63 个任务）
   └─ Task（一个基准任务，例如 wdm-design）
       └─ Trial（该任务的一次尝试，例如第 1 次 attempt）
           ├─ Agent Environment（trial 容器：agent 在里面干活）
           │    └─ 单容器任务：main
           │    └─ Compose 任务：main + sidecar（api/redis/postgres...）
           └─ Verifier Environment（另一个 trial 容器：用独立 verifier 镜像跑测试）

  - -k 1 → 每个 task 1 个 trial；
  - -k 5 → 每个 task 5 个 trial（pass@k）；
  - 每个 trial 都从干净的任务镜像重新创建，互不污染。

  二、本次运行中的实际例子

  你现在的 wdm-design：

  docker ps --filter name=wdm-design

  会看到：

  wdm-design__gfpecv6__env-main-1
    image: harborframework/terminal-bench:wdm-design-environment-407d1d42601ed972@sha256:...
    status: Up

  拆解名字：

  ┌────────────┬────────────────────────────────────────────────────┐
  │    部分    │                        含义                        │
  ├────────────┼────────────────────────────────────────────────────┤
  │ wdm-design │ 任务名                                             │
  ├────────────┼────────────────────────────────────────────────────┤
  │ gfpecv6    │ 本次 trial 的随机后缀（区分不同 attempt）          │
  ├────────────┼────────────────────────────────────────────────────┤
  │ env        │ agent 环境（不是 verifier）                        │
  ├────────────┼────────────────────────────────────────────────────┤
  │ main       │ compose 服务名；多容器任务还会有 api-1、redis-1 等 │
  ├────────────┼────────────────────────────────────────────────────┤
  │ -1         │ compose 副本编号                                   │
  └────────────┴────────────────────────────────────────────────────┘

  对应的 Trial 产物目录：

  trials/outputs/default/20261021_075601/results/.../details/wdm-design__gFpecv6/
    ├─ agent/claude-code.txt      # agent 轨迹
    ├─ trial.log
    ├─ result.json
    └─ verifier/reward.txt        # verifier 结果（稍后出现）

  三、trial 容器里有什么

  以 wdm-design 为例：

  - 基础镜像：harborframework/terminal-bench:wdm-design-environment-...，即官方预构建 environment 镜像（digest 固定）。
  - 任务工作目录：通常是 /app，由镜像和 compose 挂载决定。
  - agent：Harbor 把离线 bundle claude-code-ubuntu-20.04-x86_64.tar.gz 解包进去；所以容器里有 claude、node 等。
  - agent 指令：Harbor 把 task 的 instruction 传给 claude。
  - 资源限制：来自 task.toml 的 cpus、memory_mb、storage_mb、gpus；wdm-design 是 2 CPU / 16GiB。
  - 网络：由 Harbor 的 compose 模板决定；当前 prebuilt 模板是 network_mode: host，所以容器直接使用宿主网络。
  - 日志/产物：/logs、artifacts 路径由 Harbor 挂载并回收。

  四、Verifier 容器是另一个 trial 容器

  TB4 的任务几乎都是 environment_mode = "separate"，所以：

  1. agent trial 容器先启动，claude 在里面执行任务；
  2. agent 结束后，Harbor 停止/回收 agent 环境；
  3. 再用 task.toml 里固定的 verifier 镜像启动另一个 trial 容器：
  <task>__<随机>__verifier__trial-main-1
  image: harborframework/terminal-bench:<task>-verifier-...@sha256:...
  4. verifier 容器挂载 /tests，运行 pytest，写 reward.txt；
  5. 最终 result.json 汇总 reward。

  这就是为什么每个任务有 environment + verifier 两张镜像（157 = 66 + 66 + 25 sidecar）。

  五、生命周期

    ├─ 启动 main（+ sidecars）
  trial start
    ├─ 创建 compose 项目 <task>__<rand>__env
    ├─ 启动 main（+ sidecars）
    ├─ 安装离线 agent-deps（本次：claude-code bundle）
    ├─ 运行 agent（claude）
    ├─ 收集 artifacts / 停止 main
    ├─ 创建 verifier 项目 <task>__<rand>__verifier__trial
    ├─ 安装离线 agent-deps（本次：claude-code bundle）
    ├─ 运行 agent（claude）
    ├─ 收集 artifacts / 停止 main
    ├─ 创建 verifier 项目 <task>__<rand>__verifier__trial
    ├─ 安装离线 agent-deps（本次：claude-code bundle）
    ├─ 运行 agent（claude）
    ├─ 收集 artifacts / 停止 main
    ├─ 创建 verifier 项目 <task>__<rand>__verifier__trial
    ├─ 运行 verifier，写 reward
    └─ 清理（--delete 请求；即使未删除，Harbor 通常也会 down 环境）

  trial 结束后：

  - 容器通常被删除；
  - 但 trial 目录（agent 日志、verifier 输出、result.json）保留在 trials/outputs/.../details/ 下；
  - 如果加了 --no-delete，容器会保留用于排查。

  六、和我们其他容器的区别

  ┌─────────────────────────────────────────┬─────────────────────────────────────────────────────────┐
  │                  容器                   │                          角色                           │
  ├─────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ tb4-agent-runtime                       │ 编排器：跑 ais_bench / Harbor，挂 docker.sock；长期运行 │
  ├─────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ wdm-design__...__env-main-1             │ trial 容器：agent 实际干活的沙箱；每个 trial 一个       │
  ├─────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ wdm-design__...__verifier__trial-main-1 │ verifier trial 容器：跑测试打分                         │
  ├─────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ aisbench-session-*                      │ 你之前其他 AISBench 会话的外层容器，与本轮 trial 无关   │
  └─────────────────────────────────────────┴─────────────────────────────────────────────────────────┘

  七、一句话总结

  trial 容器 = 某次任务尝试的临时 Docker 沙箱：用官方 environment 镜像启动，agent 在里面执行任务，结束后用独立 verifier 镜像再起一个沙箱打分；两者都由 Harbor 通过宿主 Docker daemon 创建，用完即清理。


### sidecar 容器
一、精确分布
     
  ┌───────────────────────────┬────────────┬──────────────────────────────────────────────────────────────────────────────────────────────────┐
  │           任务            │ sidecar 数 │                                           sidecar 服务                                           │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ heat-pump-warranty        │          6 │ warranty-portal、asset-ledger、document-vault、returns-ledger、compliance-ledger、warranty-inbox │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ intrastat-meldung         │          5 │ odoo、compliance-hub、idev、services、dms                                                        │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ live-database-cutover     │          3 │ mysql-db、postgres-db、customer（另有外部 redis:7-alpine）                                       │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ legacy-utility-triage     │          2 │ legacy-workstation、legacy-app                                                                   │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ medical-claims-processing │          2 │ playwright-mcp、workspace                                                                        │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ payments-pipeline-fix     │          2 │ seeder、customer（另有外部 apache/kafka-native:4.3.1）                                           │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ ctr-optimization          │          1 │ api                                                                                              │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ cumulative-layout-shift   │          1 │ barber-shop-data-backend                                                                         │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ freight-dispatch-shift    │          1 │ event-feed                                                                                       │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ kv-live-surgery           │          1 │ loadgen                                                                                          │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ nextjs-performance        │          1 │ warehouse-api                                                                                    │
  ├───────────────────────────┼────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ 合计                      │         25 │ 11 个任务                                                                                        │
  └───────────────────────────┴────────────┴──────────────────────────────────────────────────────────────────────────────────────────────────┘

  - 25 是镜像数量，不是“每个任务都跑 25 个容器”。
  - 只有这 11 个任务的 environment/docker-compose.yaml 定义了 sidecar；其余 55 个任务只有 main。
  - 两个外部镜像也只被特定任务使用：redis:7-alpine → live-database-cutover；apache/kafka-native:4.3.1 → payments-pipeline-fix。

  二、sidecar 的作用

  sidecar 是任务环境所需的辅助服务，让 agent 面对一个接近真实的系统，而不是一个空容器。大致分几类：

  1. 业务后端 / 数据服务
     - nextjs-performance 的 warehouse-api：给前端提供数据。
     - cumulative-layout-shift 的 barber-shop-data-backend：提供页面数据。
     - freight-dispatch-shift 的 event-feed：提供事件流。
  2. 模拟企业系统
     - heat-pump-warranty 的 6 个服务：保修门户、资产台账、文档库、退货台账、合规台账、保修收件箱。
     - intrastat-meldung 的 odoo/DMS/compliance-hub 等：ERP、文档管理、合规系统。
  3. 数据库迁移 / 基础设施
     - live-database-cutover：MySQL、PostgreSQL、Redis、customer 服务，模拟不停机数据库切换。
  4. 浏览器 / 工具服务
     - medical-claims-processing 的 playwright-mcp + workspace：浏览器自动化和工作区。
  5. 事件管道
     - payments-pipeline-fix：Kafka + seeder + customer，模拟支付事件流。
  6. 负载/压测
     - kv-live-surgery 的 loadgen：持续产生负载。
  7. 验证辅助
     - ctr-optimization 的 api：持有 campaign snapshot；agent 查询它，verifier 通过 collect hook 从它收集证据。

  三、它们什么时候启动

  Harbor 为每个 trial 创建 Compose 项目：

  <task>__<随机>__env

  然后 docker compose up 会启动该任务 compose 文件里定义的所有服务，例如：

  ctr-optimization__xxx__env-main-1
  ctr-optimization__xxx__env-api-1

  - 只有对应任务的 trial 运行时，它的 sidecar 才会启动；
  - trial 结束（--delete 或 Harbor 正常 teardown）时一起清理；
  - 它们不常驻，也不是全量 63 任务时“同时跑 25 个”。

  四、与 verifier 的关系

  sidecar 不只是给 agent 用，verifier/collect 阶段也可能依赖它们：

  - ctr-optimization：[[verifier.collect]] service = "api"，从 api sidecar 收集健康检查/快照；
  - live-database-cutover：从 postgres/redis sidecar 导出 dump/rdb；
  - payments-pipeline-fix：从 kafka sidecar 收集快照；
  - medical-claims-processing：从 workspace sidecar 收集产物。

  这也是为什么 verifier.collect 里会指定 service，而不是只在 main 里跑命令。

  五、资源与并发影响

  - 单容器任务：1 个 main；
  - heat-pump-warranty：main + 6 个 sidecar，共 7 个容器；
  - live-database-cutover：main + 3 个 sidecar + redis，共 5 个；
  - 全量并发时，如果同时跑多个 Compose 任务，容器数量、内存、端口占用会明显上升。

  所以我们在小批量阶段建议先用 -n 2，并观察 docker stats、内存和磁盘；不要一上来用 8 并发跑这些 Compose 任务。

  六、一句话总结

  25 个 sidecar 是 11 个 Compose 任务的辅助服务镜像，不是全局常驻、也不是每个任务都有；它们只在所属任务的 trial 中启动，用来提供业务后端、数据库、浏览器、事件流、负载等真实环境，并可能参与 verifier 的产物收集。

