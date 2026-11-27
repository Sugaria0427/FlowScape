# 实验启动与重跑时序图

## 载入与启动

```mermaid
---
config:
  theme: base
  fontSize: 20
  themeVariables:
    fontSize: "20px"
    actorBkg: "#eef4fb"
    actorBorder: "#8399b1"
    actorTextColor: "#102a43"
    actorLineColor: "#a7b6c7"
    signalColor: "#41546a"
    signalTextColor: "#102a43"
    labelBoxBkgColor: "#eaf3ff"
    labelBoxBorderColor: "#879aaf"
  themeCSS: |
    .messageText, .loopText > tspan {
      paint-order: stroke fill;
      stroke: #ffffff;
      stroke-width: 4px;
      stroke-linejoin: round;
    }
  sequence:
    actorFontWeight: 600
    actorMargin: 22
    width: 94
    height: 92
    diagramMarginX: 12
    diagramMarginY: 26
    boxMargin: 12
    boxTextMargin: 6
    messageMargin: 14
    mirrorActors: true
---
sequenceDiagram
    actor U as <br/>实验者
    participant V as M10<br/>控制台
    participant G as M9<br/>服务网关
    participant X as M6<br/>实验管理
    participant L as M1<br/>输入校验
    participant S as 实验目录
    participant R as M7<br/>运行管理

    U->>V: 预设或手工输入
    V->>G: 请求校验
    G->>X: 请求预览
    X->>L: 标准化并校验
    L-->>X: 预览或错误
    X-->>G: 校验结果
    G-->>V: 预览或字段错误
    U->>V: 核对预览、选策略
    V->>G: 请求启动
    G->>X: create（配置）
    X->>L: 校验整批输入
    L-->>X: 通过或错误
    alt 校验通过
        X->>X: 冻结输入并计算 SHA-256
        X->>S: 保存清单和输入
        X->>R: create_runs（冻结清单）
        R-->>X: 各策略的 run_id
        X-->>G: 标识与清单哈希
        G-->>V: 202 已接受
        Note over S,R: M7 推进内核<br/>M5 追加事件
        R-->>G: 已提交水位
        G-->>V: WebSocket<br/>进度通知
    else 校验失败
        X-->>G: 字段级报告
        G-->>V: 422 整批拒绝
    end
```

场景校验使用 `POST /api/scenarios/validate`，确认启动使用 `POST /api/experiments`。

## 同源对照与导出

```mermaid
---
config:
  theme: base
  fontSize: 20
  themeVariables:
    fontSize: "20px"
    actorBkg: "#eef4fb"
    actorBorder: "#8399b1"
    actorTextColor: "#102a43"
    actorLineColor: "#a7b6c7"
    signalColor: "#41546a"
    signalTextColor: "#102a43"
    labelBoxBkgColor: "#eaf3ff"
    labelBoxBorderColor: "#879aaf"
  themeCSS: |
    .messageText, .loopText > tspan {
      paint-order: stroke fill;
      stroke: #ffffff;
      stroke-width: 4px;
      stroke-linejoin: round;
    }
  sequence:
    actorFontWeight: 600
    actorMargin: 22
    width: 94
    height: 92
    diagramMarginX: 12
    diagramMarginY: 26
    boxMargin: 12
    boxTextMargin: 6
    messageMargin: 14
    mirrorActors: true
---
sequenceDiagram
    participant V as M10<br/>控制台
    participant G as M9<br/>服务网关
    participant X as M6<br/>实验管理
    participant R as M7<br/>运行管理
    participant M as M8<br/>指标与导出
    participant S as 实验目录

    V->>G: 请求窗口指标
    G->>X: 比对输入哈希
    alt 输入一致
        X-->>G: 共同输入哈希
        G->>R: 读取各运行提交边界
        R-->>G: 水位、提交序号及终止状态
        G->>M: metrics（窗口、提交边界）
        M->>S: 可结算时<br/>读取已提交数据
        M-->>G: 指标、未就绪或运行失败
        G-->>V: 指标查询结果
    else 输入不一致
        X-->>G: 差异字段与各运行哈希
        G-->>V: 409 禁止对照
    end
    V->>G: 请求导出实验
    G->>M: export（实验标识）
    M->>S: 读取清单、输入和已提交事件
    M-->>G: 重跑输入与结果
    G-->>V: 返回下载
```

查询使用 `GET /api/experiments/{id}/metrics?a=&b=`，导出使用 `GET /api/experiments/{id}/export`。

## 凭清单重跑

```mermaid
---
config:
  theme: base
  fontSize: 20
  themeVariables:
    fontSize: "20px"
    actorBkg: "#eef4fb"
    actorBorder: "#8399b1"
    actorTextColor: "#102a43"
    actorLineColor: "#a7b6c7"
    signalColor: "#41546a"
    signalTextColor: "#102a43"
    labelBoxBkgColor: "#eaf3ff"
    labelBoxBorderColor: "#879aaf"
  themeCSS: |
    .messageText, .loopText > tspan {
      paint-order: stroke fill;
      stroke: #ffffff;
      stroke-width: 4px;
      stroke-linejoin: round;
    }
  sequence:
    actorFontWeight: 600
    actorMargin: 22
    width: 94
    height: 92
    diagramMarginX: 12
    diagramMarginY: 26
    boxMargin: 12
    boxTextMargin: 6
    messageMargin: 14
    mirrorActors: true
---
sequenceDiagram
    actor U as <br/>实验者
    participant V as M10<br/>控制台
    participant G as M9<br/>服务网关
    participant X as M6<br/>实验管理
    participant S as 实验目录
    participant R as M7<br/>运行管理
    participant M as M8<br/>指标与导出

    U->>V: 载入导出的输入
    V->>G: 请求重跑
    G->>X: 清单与期望哈希
    X->>S: 读取清单及输入
    X->>X: 校验哈希及引擎、策略版本
    alt 清单与运行环境匹配
        X->>S: 建目录并保存输入
        X->>R: 按清单为各策略建运行
        R-->>X: 新运行标识列表
        X-->>G: 新旧实验引用
        G-->>V: 202 已接受
        Note over S,R: M7 推进重跑<br/>M5 追加事件
        R-->>X: 终止状态与提交边界
        X->>M: 用原窗口和提交边界计算
        M->>S: 读取冻结输入和已提交事件
        M-->>X: 逐作业结果与窗口汇总
        X->>X: 原记录可用时生成一致性报告
    else 清单或版本校验失败
        X-->>G: 具体差异
        G-->>V: 拒绝重跑
    end
```

重跑使用 `POST /api/experiments/rerun`。缺少关键配置返回 422，哈希不符或指定的引擎、策略版本不可用返回 409，均附具体差异。

**覆盖需求** UC-06～08、FR-EXP-01～04、NFR-EXP-01，对应概要设计 5.1、5.4 和 7.2 节。

- 创建与重跑使用 `Idempotency-Key`。启动校验失败时整批拒绝。M6 比较参与对照的运行，只放开 `policy_spec` 的差异。错误响应的 `details` 给出字段路径、运行标识及相应哈希。同源通过后，M8 按 M7 发布的水位与提交序号结算；正常完成可将已知终态延伸至 `b`。
- 导出包含清单和全部引用输入，以及 `jobs.csv` 和 `summary.json`。带着清单及其输入文件即可重跑。只导入输入时可以生成新运行，有随导出保留的原结果时可以同时核对起止时刻、卡位和窗口指标。有原事件记录时可按 `event_seq` 做逐事件核对。

**EXP-A 走查（AC-EXP-01、02）**。AC-EXP-01 用 `seed=42` 和稳定 FIFO 生成一份单策略清单。按原清单重跑后，清单哈希、已知作业起止时刻、卡位、完成数 1、未完成数 2、分配量 10 GPU·秒应一致。改动 `truth.json` 中的时长后，M6 定位对应输入并拒绝创建运行。AC-EXP-02 另建双策略实验，两次运行使用相同平台、负载与种子。窗口 `[0,8)` 内，稳定 FIFO 与可行即运行的吞吐分别为 0.125、0.25 作业/秒，占用率为 62.5%、81.25%，Jain 指数为 `25/26`、`169/170`。错配样例将其中一份运行的 `random_seed` 改为不同值，M6 返回该字段的差异。

**与概要设计对齐**。第 5.4、6.3 节补充清单引用文件的重跑条件，第 7.2 节补充同源失败和导出内容。
