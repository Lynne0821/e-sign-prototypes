# 核心业务流程设计 (Core Process Design)

本文档描述了**采购单与对账单 e签宝对接**的核心业务流程、系统交互以及状态流转。

## 1. 核心流程泳道图 (Cross-functional Flowchart)

本图展示了从上游获取数据、生成单据，到交付系统发起、e签宝平台内管控以及最终归档的完整主干流程。

```mermaid
flowchart TD
  subgraph upstream ["上游系统 (金蝶 / 携客云)"]
    K[金蝶采购单 \n 状态: 已审核]
    X[携客云对账单 \n 状态: 已确认]
  end

  subgraph delivery ["交付系统"]
    A[拉取/同步单据数据]
    B[校验数据与组装 PDF 文件]
    C{上游状态满足 \n 且 配置完善?}
    D[匹配默认内部组织\n与供应商签署人]
    F[自动调用 e签宝API创建签署任务\n更新状态: 签署中]
    Err[标记状态: 配置异常/待发起\n(需人工介入补充)]
    E[人工补充配置/确认无误后\n手动点击 发起签署]
    H[接收 Webhook 回调或主动查询]
    I[更新状态: 已签署/已拒签\n归档生效 PDF]
  end

  subgraph esign ["e签宝平台"]
    G1[流转用印与审批\n发送签署通知]
    G2[供应商实名与确认意愿\n进行电子盖章]
    G3[流程完结]
  end

  K -.-> A
  X -.-> A
  
  A --> B
  B --> C
  C -->|否| Err
  Err --> E
  E --> D
  C -->|是| D
  D --> F
  
  F --> G1
  G1 --> G2
  G2 --> G3
  
  G3 -- 触发 Webhook --> H
  H --> I
```

## 2. 系统交互时序图 (System Sequence Diagram)

本图展示了具体系统间的 API 调用时序和交互，重点体现了数据获取、自动发起签署请求和异常人工处理机制。

```mermaid
sequenceDiagram
  autonumber
  participant Upstream as 上游系统<br>(金蝶/携客云)
  participant Operator as 采购员/结算员
  participant Delivery as 交付系统
  participant eSign as e签宝开放平台
  participant Supplier as 供应商用户

  Delivery->>Upstream: 获取已审核/确认单据数据
  Upstream-->>Delivery: 返回 JSON 原始数据
  Delivery->>Delivery: 根据 Excel 模板规则组装 PDF
  Delivery->>Delivery: 读取系统配置 (组织、供应商联系人)
  
  alt 配置完善且上游状态正常
    Delivery->>Delivery: 自动匹配签署人并组装参数
    Delivery->>eSign: 1. 自动上传文件 (PDF)
    eSign-->>Delivery: 返回 fileId
    Delivery->>eSign: 2. 自动发起签署任务
    eSign-->>Delivery: 返回 taskId (流程ID)
    Delivery->>Delivery: 更新单据状态为「签署中」
  else 配置异常或缺失
    Delivery->>Delivery: 更新单据状态为「配置异常/待发起」
    Delivery-->>Operator: 列表页/详情页提示需人工处理
    Operator->>Delivery: 补充配置并点击“发起签署”
    Delivery->>eSign: 发起签署请求...
  end
  
  Note over eSign,Supplier: 签署及审批管控在 e签宝端内闭环
  eSign->>Supplier: 发送短信/邮件通知
  Supplier->>eSign: 登录、实名认证并盖章
  
  alt 签署完成
    eSign-->>Delivery: Webhook 回调 (状态: 签署完成)
    Delivery->>eSign: 下载已盖章生效文件
    Delivery->>Delivery: 归档文件，更新状态为「已签署」
  else 拒签/作废
    eSign-->>Delivery: Webhook 回调 (状态: 拒签/作废)
    Delivery->>Delivery: 更新状态为「拒签/作废」
  end
```

## 3. 状态机流转 (State Machine)

对交付系统内**采购单/对账单**在当前对接范畴下的状态流转进行定义。

```mermaid
stateDiagram-v2
  [*] --> 签署中: 同步组装完成，配置完善，自动发起
  [*] --> 待发起: 同步组装完成，但配置异常(需人工介入)
  
  待发起 --> 签署中: 补充配置后手动发起
  
  签署中 --> 已签署: Webhook 回调完成
  签署中 --> 拒签: Webhook 回调拒签
  签署中 --> 已作废: 平台/手动主动撤销
  
  拒签 --> 签署中: 操作员重置/重发
  已作废 --> 签署中: 操作员重置/重发
  
  已签署 --> [*]
```
