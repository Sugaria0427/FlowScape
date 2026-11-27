# 同刻处理与调度轮次时序图

```mermaid
sequenceDiagram
    autonumber
    participant Q as 确定性事件队列
    participant E as EventLoop
    participant L as 作业生命周期
    participant S as Scheduler
    participant P as Strategy
    participant M as Matcher
    participant A as AllocationLedger
    participant R as EventRecorder
    participant W as RunManager
    participant V as ReadOnlyView

    Note over Q,V: 每个 t 依次处理完成、节点变化、到达和调度
    Q->>E: 取出 (t, phase, enqueue_seq)
    E->>L: 处理 t 时完成作业并释放卡组
    L->>A: 原子释放已完成作业
    E->>L: 处理 t 时节点上下线
    E->>L: 处理 t 时到达作业
    L->>R: 追加 job_finished / node_state_changed / job_arrived

    loop 直到一轮无状态或原因变化
        E->>V: 构造当前已到达作业和资源状态
        V-->>E: 不含 JobTruth 或未来事件
        E->>S: run_round(view)
        S->>P: candidate_order(view)
        P-->>S: 稳定候选 job_id 顺序
        loop 按候选顺序，使用最新账本
            alt 策略此前已返回 STOP
                S->>M: classify(request) (只读，不分配)
                M-->>S: None (自身可匹配) 或 WaitReason
                S->>R: scheduling_result=blocked_by_policy，另记 physical_match
                Note over S,R: 自身可匹配记等待优先级及 blocker_job_id；否则记资源原因
            else 尚未 STOP
                S->>M: feasible(request)
                alt 找到可行卡组
                    M-->>S: 确定性第一组 CardGroup
                    S->>A: allocate_atomically(job_id, group)
                    A-->>S: 分配成功，更新账本
                    S->>L: 将作业转为运行
                    S->>R: 追加 job_started (含 round_id)
                    L->>Q: 按 JobTruth 登记未来完成事件
                else 当前候选无可行卡组
                    M-->>S: 空卡组
                    S->>M: classify(request)
                    M-->>S: WaitReason
                    S->>P: on_match_failure(request, reason)
                    alt CONTINUE (可行即运行)
                        P-->>S: CONTINUE
                        S->>R: 记录 physical_wait 和原因变化
                    else STOP (稳定 FIFO)
                        P-->>S: STOP
                        S->>S: 设置 stopped=true 并记录 blocker_job_id
                        S->>R: 记录队首 physical_wait 和原因变化
                    end
                end
            end
        end
        S->>R: 追加逐候选 schedule_round
        S-->>E: 返回本轮状态变化标志
    end
    E->>W: 推进水位至 t 并发送完成时刻快照
```

需求覆盖 `FR-SCH-01`、`FR-SCH-05`、`NFR-SCH-01`～`NFR-SCH-03`。事件入队使用 `(sim_time, phase, enqueue_seq)`；事件追加时另分配单调递增的 `event_seq`。轮次以每个运行内唯一且递增的 `round_id` 标识，不另存重复轮次号。`schedule_round` 区分物理匹配结果和调度结果：FIFO 返回 `STOP` 后，后续项仍只读检查物理可匹配性，但不获分配机会。自身可匹配时记录「等待优先级」和阻塞它的作业标识；自身不可匹配时保留资源原因。`reason_changed` 和 `job_started` 可记录来源 `round_id`，各自通过 `event_seq` 定位。原因依据只含稳定字段，不含时刻和 `round_id`。终止轮也写入。策略视图不含真值和未来事件；物理状态由独立事件重建。

手算核对 `SCH-A`。t=2 时 R 占一卡，L 已在 t=1 到达且需两卡，S 于 t=2 到达。FIFO 因 L 不可匹配而停止，S 自身可匹配，所以记录「等待优先级」，依据为 L 的 `job_id`；S 不启动。可行即运行检查 S 并使其于 `[2,4)` 运行。t=10 时先处理 R 完成并释放一卡。FIFO 为 L 分配两卡，S 与 T 的物理匹配分类均为「暂缺资源」，两者均不启动。可行即运行中 T 已于 t=4 启动并占用一卡，L 在 t=10 仍暂缺一卡，待 T 于 t=16 完成后再启动，区间为 `[16,26)`。队首保护回填是 P1 候选，不属于首版策略接口。
