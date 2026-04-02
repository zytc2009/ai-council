# 会议纪要

- **议题**：多智能体协作系统的最佳实践设计
- **阶段**：review
- **时间**：2026-04-02 08:45
- **参会者**：claude-sonnet
- **轮次**：2 / 2

---

## 讨论摘要

### 共识点

1. 方案整体质量较高，渐进式演进策略方向正确。
2. 风险表存在明显缺口：Orchestrator 单点故障风险和 ContextDelta 合并冲突处理策略均未覆盖。
3. 方案中以智能体数量（10个、15个）作为架构演进门槛，缺乏可观测的客观依据，应替换为可度量的触发指标。
4. `temperature=0` 场景下的重试熔断机制是被低估的高优先级问题，相同输入+相同 temperature 必然产生相同错误，无效重试只会消耗 token。
5. 风险条目必须对应具体的实现变更（接口定义、数据结构或配置约束），不能停留在文字描述层面。

### 分歧点

本次为单人 review 阶段，无多方分歧。以下记录方案原文与评审意见之间的立场差异：

| 议题 | 方案原文立场 | 评审意见立场 |
|------|------------|------------|
| 动态注册表引入门槛 | 智能体数量 > 10 个 | 路由配置变更频率 > 1次/周，且有运行时动态发现需求 |
| Coordinator 层引入门槛 | 智能体数量 < 15 个用两层 | 跨团队独立部署需求出现，或不同业务域需要不同调度策略 |
| Orchestrator 故障应对 | 未列入风险表 | 高影响风险，需状态持久化至外部存储（Redis） |
| ContextDelta 合并策略 | 未定义 | 必须在 Phase 0 以接口字段形式显式约定 |

### 关键论据

**claude-sonnet（第1轮）**
- Orchestrator 是系统唯一调度核心，其崩溃与"性能瓶颈"是两个不同风险维度，方案混淆了两者。
- 并行智能体同时写入同一 ContextDelta 字段是 Phase 2 必然触发的场景，合并策略应在 Phase 0 设计时明确，而非留给实现阶段自行决定。
- 数量阈值是结果，不是原因——真正触发架构升级的是行为模式（变更频率、部署隔离需求），数量只是副产品。

**claude-sonnet（第2轮）**
- Redis Streams + Consumer Groups 的故障转移语义可直接复用于 Orchestrator 状态恢复，无需自研检查点机制，实现成本远低于方案暗示的复杂度。
- 合并策略若不在数据结构层面显式约束（`merge_strategy` 字段），Phase 2 必然出现分叉实现，架构一致性只能靠约定维持。
- `RetryPolicy` 熔断逻辑若散落在各处重试代码中，会形成重复且难以统一调整，应提取为独立装饰器集中注入。

---

## 下一步建议

### 风险表补充

- [ ] 在风险表（第5节）新增条目："Orchestrator 进程崩溃导致全系统停摆"，标注为高影响风险，应对措施为状态持久化至 Redis。

### Phase 0 交付物更新

- [ ] 在 `ContextDelta` 数据类定义中增加 `merge_strategy` 字段，默认值为 `"reject"`，强制每个 Delta 自声明合并意图：
  ```python
  @dataclass
  class ContextDelta:
      field: str
      value: Any
      merge_strategy: Literal["lww", "reject", "field_merge"] = "reject"
      agent_id: str
      timestamp: float
  ```

### 架构门槛可观测化

- [ ] 将动态注册表引入条件修改为：路由配置变更频率 > 1次/周，或存在运行时动态发现需求。
- [ ] 在 CI pipeline 或 pre-commit hook 中加入路由变更计数检测：
  ```bash
  git log --oneline --since="7 days ago" -- config/routes.yaml | wc -l
  # 结果 > 1 时输出 WARNING，提示进行架构评审
  ```
- [ ] 新建 `docs/adr/002-coordinator-layer.md`，固化 Coordinator 层引入的触发 checklist（满足任一即引入）：
  - [ ] 存在至少2个团队独立部署的智能体集群
  - [ ] 不同业务域需要不同的超时/重试策略
  - [ ] 跨域调度日志需要隔离审计

### Orchestrator 可用性实现

- [ ] 将任务调度状态从进程内存迁移至 Redis Stream，每个任务对应一条 Stream 条目。
- [ ] Orchestrator 重启后通过 `XAUTOCLAIM` 接管未完成任务，复用 Consumer Group 故障转移语义。

### 重试熔断模块化

- [ ] 提取 `RetryPolicy` 为独立装饰器，统一注入所有 LLM 调用点：
  ```python
  class RetryPolicy:
      def __init__(self, max_retries: int = 3,
                   circuit_break_threshold: int = 3,
                   alert_callback: Callable = None): ...
  ```
- [ ] 熔断条件（连续同类错误 > N 次触发告警并停止重试）在配置文件中集中管理，全局生效。
- [ ] 在风险表补充注记：`temperature=0` 场景下连续重试失败时，优先排查 Prompt 设计而非增加重试次数。