# OP08 装摄像头螺丝工序 — PLC GRAPH 程序

## 项目概述

工业 4.0 产线 OP08 工位：**装摄像头螺丝工序**。

CPU: Siemens S7-1500T (1511T-1 PN)  
编程软件: TIA Portal V21  
编程语言: GRAPH (顺序控制) + SCL/LAD (辅助逻辑)

---

## 程序架构

```
OB1 (LAD 直接版)
 ├── Network 0: 仿真桥接 (M100→I114/I100/...)
 ├── Network 1: 调用 FB_OP08_Receive
 ├── Network 2: 调用 FB_OP08_Screw
 ├── Network 3: 调用 FB_OP08_SendOut
 └── Network 4+: 面板指示灯 + 三色灯 + 蜂鸣器

FB_OP08_Receive  [FB0] — 接收工件 (GRAPH, 19步 + ERR)
  └── 载具到达 → 阻挡 → 定位 → 移载取料 → 二次定位 → 压摄像头 → 发拧螺丝请求

FB_OP08_Screw    [FB1] — 拧螺丝 (GRAPH, 12步 + ERR)
  └── 收到请求 → 回原点 → 循环(取螺丝→拧螺丝) ×3颗 → 发完成信号

FB_OP08_SendOut  [FB2] — 送出工件 (GRAPH, 13步 + ERR)
  └── 等拧完+压头松开 → 移载取成品 → 放回载具 → 释放 → 传送带送出 → 发完成信号
```

---

## FB 间握手协议

通过 `握手信号 [DB4]` 传递三个 Bool 信号：

```
FB0 (接收) ──bReq_ScrewStart──→ FB1 (拧螺丝)
FB1 (拧螺丝) ──bAck_ScrewDone──→ FB2 (送出)
FB2 (送出) ──bAck_OutDone──→ FB0 (接收)

             ┌──────────────────────────────────┐
             │  FB0: 压摄像头压下                │
             │       → bReq_ScrewStart=1         │
             │  FB1: 三轴回原点→取螺丝→拧×3     │
             │       → bAck_ScrewDone=1          │
             │  FB0: 压头松开                    │
             │  FB2: 移载取成品→放出→送出        │
             │       → bAck_OutDone=1            │
             │  FB0: 回到待机                    │
             └──────────────────────────────────┘
```

---

## FB101 双版本说明

FB101 拧螺丝有两个版本，选一个用：

| | MC 版 (FB_OP08_Screw_GRAPH.md) | IO 版 (FB_OP08_Screw_IO版.md) |
|--|------|------|
| 伺服控制 | TO_PositioningAxis + MC_MoveAbsolute | FB_ServoGotoPos 直接写 Q0.0~Q0.6 |
| 回零 | MC_Home | FB_ServoGotoHome |
| 工艺对象 | 需要 3 个 (AxisX/Y/Z) | 不需要 |
| TM Pulse 模块 | 需要 (或虚拟轴) | 不需要 |
| 前固定指令 | 15 个 MC 网络 | 8 个 (2 个坐标选择 + 6 个 FB 调用) |
| 适用场景 | 有 PROFINET 伺服驱动器 | 只有脉冲+方向接线 |
| PLCSIM 仿真 | 需要配置虚拟轴 | 可直接跑 |

**GRAPH 步序 (S1~S12 + ERR) 在两个版本中逻辑完全一致**，区别只在前固定指令的伺服控制方式。

---

## 文件清单

```
OP08_PLC_程序/
│
├── README.md                           ← 本文件
│
├── Docs/
│   ├── OP08_变量分配.md                 ← IO 变量→FB 分配映射表
│   ├── OP08_变量分配.xlsx               ← Excel 版变量分配表 (可直接导入)
│   └── OP08_仿真M位分配表.md            ← PLCSIM 仿真用 M 位分配
│
├── GRAPH/
│   ├── FB_OP08_Receive_GRAPH.md        ← FB100 接收工件步序表 (19步)
│   ├── FB_OP08_Screw_GRAPH.md          ← FB101 拧螺丝步序表 (MC版)
│   ├── FB_OP08_Screw_IO版.md           ← FB101 拧螺丝步序表 (IO版)
│   └── FB_OP08_SendOut_GRAPH.md        ← FB102 送出工件步序表 (13步)
│
└── SCL_Helpers/
    ├── FB_ServoGotoPos.scl             ← 伺服绝对定位 FB (IO版用)
    ├── FB_ServoGotoHome.scl            ← 伺服回原点 FB (IO版用)
    └── OB1_LAD_Direct.scl              ← OB1 主程序 (LAD+SCL混合)
```

---

## TIA Portal 操作步骤

### 1. 导入变量表
- 项目树 → PLC 变量 → 从 Excel 导入 `op8_tags.xlsx`

### 2. 创建 GRAPH 功能块
- 添加新块 → 类型: 函数块 → 语言: GRAPH
- 分别创建 FB0 (接收工件)、FB1 (拧螺丝)、FB2 (送出工件)
- 按 GRAPH/*.md 文档录入步序、转移、Supervision

### 3. 导入 SCL 源文件
- 外部源文件 → 添加:
  - `FB_ServoGotoPos.scl` → 从源生成块 → 创建背景 DB
  - `FB_ServoGotoHome.scl` → 从源生成块 → 创建背景 DB
- 在 FB1 的前固定指令中拖入调用，引脚参见 IO 版文档

### 4. 创建 OB1
- 添加 OB1 → 语言: LAD
- 按 `OB1_LAD_Direct.scl` 录入 Network 0 (仿真桥接) + Network 1~8 (FB 调用 + 指示灯)

### 5. (MC 版专用) 配置工艺对象
- 工艺对象 → 新增 TO_PositioningAxis ×3 (AxisX/Y/Z)
- 配置脉冲发生器或 PROFIdrive
- 在 FB1 前固定指令中拖入 MC_Power / MC_MoveAbsolute / MC_Home 等

### 6. 编译下载
- 项目 → 编译 → 软件 (全部重建)
- 无错误后 → 下载到 PLC 或 PLCSIM

---

## PLCSIM 仿真指南

### 方式一：PLCSIM Sequence（推荐）

1. 启动 PLCSIM → 下载到仿真器
2. 打开 PLCSIM → Sequence → Sequence_1
3. 按 `Docs/OP08_PLCSIM_Sequence.md` 填写时间序列
4. 点播放，程序自动运行
！！！注意示例程序三个序列表，手速要快，亮了马上点

### 方式二：M 位手动模拟

1. OB1 Network 0 保留仿真桥接代码（M100→I114）
2. 监控表中按 `Docs/OP08_仿真M位分配表.md` 添加 M 地址
3. 手动逐个改 M 位值，模拟传感器信号

---

## 硬件配置

| 组件 | 说明 |
|------|------|
| CPU | 1511T-1 PN (S7-1500T) |
| 伺服 | 3 轴 (X/Y/Z)，脉冲+方向 (Q0.0~Q0.6) |
| 移载 | 1 号移载 (平移+升降+夹爪) |
| 压摄像头 | 升降气缸 + 旋转气缸 |
| 电批 | 真空吸螺丝 + 刹车信号反馈 |
| 螺丝 | 3 颗，3 个取料位 + 3 个拧紧位 |
| 载具 | 前阻挡 + 工位阻挡 + 定位 + 二次定位 + 回流线 |
| 面板 | 准备/启动/停止/急停/复位/模式 + 三色灯 + 蜂鸣器 |
