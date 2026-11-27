# 同时钟快照查询时序图

```mermaid
sequenceDiagram
    autonumber
    actor U as 用户
    participant V as M10 Web 控制台
    participant G as M9 服务网关
    participant R as M7 运行管理与快照查询
    participant S as 实验目录
    participant K as M3 仿真内核
    U->>V: 拖动共同游标到 t
    V->>V: 更新 selection_version，保留共同窗口 [a,b)
    V->>G: GET snapshot?t=t（request_id）
    G->>R: snapshot(experiment_id,t)
    R->>R: 校验实验、同源结果与 t 范围
    loop 实验内每个 run_id
        R->>R: 原子读取已提交水位及 committed_event_seq
        alt 水位达到 t 或正常完成可保持终态
            R->>S: 取不晚于 t 的完整同刻检查点
            S-->>R: 状态与 checkpoint_event_seq（无则初始状态）
            R->>S: 读取检查点之后且 sim_time≤t 的已提交事件
            S-->>R: 按 event_seq 有序返回
            R->>R: 重建状态，标记 snapshot_time=t
        else 计算水位尚未达到 t
            R->>R: 记录 NOT_READY（水位，无该时刻状态）
        else 运行异常中止且 t 超过最后有效水位
            R->>R: 记录 RUN_FAILED（不外推）
        end
    end
    R-->>G: 各运行结果、watermark、event_seq、comparison_ready
    G-->>V: HTTP 200（允许部分运行 NOT_READY）
    V->>V: 仅接纳当前 request_id；展示共同 t 与未就绪提示
    Note over V,R: 不用不同时间的快照比较；不按各自第 N 个事件对齐
    K->>S: 追加 t 的全部事件并提交
    K->>R: 同刻固定点结束，发布水位与提交序号
    R->>G: watermark 更新通知
    G-->>V: WS progress（各 run 独立水位）
    V->>G: 当前游标仍为 t 时重新查询 snapshot
```

覆盖 FR-OBS-01、FR-OBS-04、FR-JOB-04、NFR-EXP-02。检查点只在完整同刻处理后生成；若事件数达到保存间隔，也必须等同刻固定点结束。水位与提交序号成对读取，不读取内核正在修改的对象；查询不驱动调度。水位可在确认下一事件晚于目标时推进至该目标，无须伪造业务事件。正常完成可展示终态到实验窗口末尾，并附实际结束时间；异常中止不能如此处理。

**AC-OBS-05 走查**：A 水位 100 秒、B 水位 80 秒，定位 t=90。A 返回 90 秒状态，B 返回 NOT_READY 和水位 80，comparison_ready=false；B 水位达到 90 后重新查询，两列 snapshot_time 都为 90，才允许解释同时刻差异。快速拖到 95 后，迟到的 90 秒响应按 request_id 丢弃。

**EXP-A 走查**：t=2 固定点后，稳定 FIFO 的 J3 排队且原因「等待优先级」（阻塞项 J2）；可行即运行的 J3 在第二张 A100 运行。t=8 两侧 J2 均运行，FIFO 的 J3 仍排队，可行即运行的 J3 已完成。共同窗口 [0,8) 的数值由 M8 提供，见概要设计 7.2；窗口与状态游标分别传递。

与文字对齐：第 5.3 节补充无事件时刻、水位发布、终态和检查点边界，第 7.2 节明确部分就绪响应，不要求把所有运行锁步计算。多尝试历史回放属于 P1。
