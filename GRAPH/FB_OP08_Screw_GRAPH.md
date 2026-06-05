# FB_OP08_Screw (FB101) — 拧螺丝 GRAPH 顺序图 (S7-1500T / MC版)

## 变量区

```
IN_OUT (传感器+执行器)
    bFeederReady     : Bool;     // I101.4  螺丝振盘到位
    bScrewVacuum     : Bool;     // I101.5  真空检知
    bScrewRunning    : Bool;     // I105.1  电批启动反馈
    bScrewBrake      : Bool;     // I105.2  刹车(扭矩到达)
    bScrewStart       : Bool;    // Q107.2  电批启动
    bScrewVacuumValve : Bool;    // Q106.5  真空阀

STATIC
    ── MC触发 ──
    bExec_HomeX/Y/Z   : Bool;
    bExec_MoveX/Y/Z   : Bool;
    ── MC反馈 ──
    bDone_HomeX/Y/Z   : Bool;
    bDone_MoveX/Y/Z   : Bool;
    bErr_MoveX/Y/Z    : Bool;
    ── 实时目标 (MC块读取) ──
    rX_Target         : Real;
    rY_Target         : Real;
    rZ_Target         : Real;
    rVelocity         : Real := 50.0;
    ── 螺丝盘三个取料位 ──
    rX_Feeder1        : Real := 0.0;
    rY_Feeder1        : Real := 80.0;
    rZ_Feeder1        : Real := -15.0;
    rX_Feeder2        : Real := 30.0;
    rY_Feeder2        : Real := 80.0;
    rZ_Feeder2        : Real := -15.0;
    rX_Feeder3        : Real := 60.0;
    rY_Feeder3        : Real := 80.0;
    rZ_Feeder3        : Real := -15.0;
    ── 螺丝孔三个拧紧位 ──
    rX_Screw1         : Real := 15.0;
    rY_Screw1         : Real := 10.0;
    rZ_Screw1         : Real := -25.0;
    rX_Screw2         : Real := 45.0;
    rY_Screw2         : Real := 10.0;
    rZ_Screw2         : Real := -25.0;
    rX_Screw3         : Real := 75.0;
    rY_Screw3         : Real := 10.0;
    rZ_Screw3         : Real := -25.0;
    ── 原点 ──
    rX_HomePos        : Real := 0.0;
    rY_HomePos        : Real := 0.0;
    rZ_HomePos        : Real := 0.0;
    ── 计数 ──
    iScrewCount       : Int := 3;
    iCurrentScrew     : Int := 0;
```

---

## 前固定指令

### 网络 1~5: X轴 MC块

```
1: MC_POWER_X       Axis→AxisX  Enable→1
2: MC_MOVEABS_X     Axis→AxisX  Execute→#bExec_MoveX  Position→#rX_Target
                    Velocity→#rVelocity  Done→#bDone_MoveX  Error→#bErr_MoveX
3: MC_HOME_X        Axis→AxisX  Execute→#bExec_HomeX  Mode→0  Done→#bDone_HomeX
4: MC_RESET_X       Axis→AxisX  Execute→0
5: MC_HALT_X        Axis→AxisX  Execute→0
```

### 网络 6~10: Y轴（同上）
### 网络 11~15: Z轴（同上）

### 网络 16: 螺丝孔坐标选择

根据 `#iCurrentScrew` 更新 `#rX_Target / #rY_Target / #rZ_Target` 为对应孔位坐标。

梯形图写法 — 3组 CMP== 并联 MOVE：

```
┌──[ EQ ]──┐    ┌──────────┐
│iCurrent  │    │  MOVE    │    ┌──────────┐
│Screw     ├───→│rX_Screw1 │    │  MOVE    │    ┌──────────┐
│   ==1    │    │→rX_Target│    │rY_Screw1 │    │  MOVE    │
└──────────┘    └──────────┘    │→rY_Target│    │rZ_Screw1 │
                                └──────────┘    │→rZ_Target│
                                                └──────────┘

┌──[ EQ ]──┐    ┌──────────┐
│iCurrent  │    │  MOVE    │    ┌──────────┐
│Screw     ├───→│rX_Screw2 │    │  MOVE    │    ┌──────────┐
│   ==2    │    │→rX_Target│    │rY_Screw2 │    │  MOVE    │
└──────────┘    └──────────┘    │→rY_Target│    │rZ_Screw2 │
                                └──────────┘    │→rZ_Target│
                                                └──────────┘

┌──[ EQ ]──┐    ┌──────────┐
│iCurrent  │    │  MOVE    │    ┌──────────┐
│Screw     ├───→│rX_Screw3 │    │  MOVE    │    ┌──────────┐
│   ==3    │    │→rX_Target│    │rY_Screw3 │    │  MOVE    │
└──────────┘    └──────────┘    │→rY_Target│    │rZ_Screw3 │
                                └──────────┘    │→rZ_Target│
                                                └──────────┘
```

### 网络 17: 螺丝盘坐标选择

结构与网络16完全相同，但源改为 `rX_Feeder1/2/3`、`rY_Feeder1/2/3`、`rZ_Feeder1/2/3`。

⚠️ **注意**：网络16和网络17都往 `rX_Target` 写值。靠 GRAPH 步骤里的 `bSelFeeder` 标志决定用哪个。当 `bSelFeeder=1` 时网络16被屏蔽、网络17生效；`bSelFeeder=0` 时反之。

实现方式：在每组的 EQ 条件上串联一个互锁条件。

- 网络16 每组条件：`iCurrentScrew==N AND bSelFeeder=0`
- 网络17 每组条件：`iCurrentScrew==N AND bSelFeeder=1`

STATIC 区加一个：`bSelFeeder : Bool := 0;`

---

## GRAPH 步序表

```
┌──────────────────────────────────────────────────────────────────────┐
│  S1: 待机                                                             │
│  动作:                                                                  │
│   N  #bScrewStart := 0                                                 │
│   N  #bScrewVacuumValve := 0                                           │
│   N  "握手信号".bAck_ScrewDone := 0                                     │
│   N  #iCurrentScrew := 0                                               │
│   N  #bSelFeeder := 0         // 默认指向孔位坐标                       │
│   N  #bExec_HomeX := 0, #bExec_HomeY := 0, #bExec_HomeZ := 0           │
│   N  #bExec_MoveX := 0, #bExec_MoveY := 0, #bExec_MoveZ := 0           │
│   N  #bDone_HomeX := 0, #bDone_MoveX := 0  (Y/Z同)                     │
│                                                                         │
│  T1: "握手信号".bReq_ScrewStart = 1                                     │
│      ↓                                                                  │
│  S2: 三轴回原点                                                         │
│  动作:                                                                  │
│   S  #bExec_HomeX := 1, #bExec_HomeY := 1, #bExec_HomeZ := 1           │
│                                                                         │
│  T2: #bDone_HomeX=1 AND #bDone_HomeY=1 AND #bDone_HomeZ=1              │
│      ↓                                                                  │
│                                                                         │
│  ╔══════════════════ 每颗螺丝循环: 取 → 拧 ═══════════════════╗        │
│                                                                         │
│  S3: 计数+1, 选螺丝盘坐标 → X/Y到螺丝盘                                │
│  动作:                                                                  │
│   R  #bExec_HomeX,Y,Z := 0                                             │
│   S1 #iCurrentScrew := #iCurrentScrew + 1      // ★ 只在步激活时执行一次
│   N  #bSelFeeder := 1                          // ★ 指向螺丝盘坐标     │
│   S  #bExec_MoveX := 1, #bExec_MoveY := 1      // X/Y→螺丝盘           │
│                                                                         │
│  T3: #bDone_MoveX=1 AND #bDone_MoveY=1          // 到达螺丝盘上方       │
│      ↓                                                                  │
│  S4: Z降到螺丝盘取螺丝                                                   │
│  动作:                                                                  │
│   R  #bExec_MoveX := 0, #bExec_MoveY := 0                              │
│   S  #bExec_MoveZ := 1                         // Z下降                │
│                                                                         │
│  T4: #bDone_MoveZ=1                             // Z到取螺丝深度        │
│      ↓                                                                  │
│  S5: 真空吸螺丝                                                         │
│  動作:                                                                  │
│   R  #bExec_MoveZ := 0                                                 │
│   N  #bScrewVacuumValve := 1                                            │
│                                                                         │
│  T5: bScrewVacuum=1 AND bFeederReady=1 AND TON 300ms                   │
│      ↓                                                                  │
│  S6: Z上升(持螺丝)                                                      │
│  动作:                                                                  │
│   S  #bExec_MoveZ := 1                                                 │
│   N  #rZ_Target := #rZ_HomePos               // 手动覆盖为安全高度      │
│                                                                         │
│  T6: #bDone_MoveZ=1                                                      │
│      ↓                                                                  │
│  S7: 切换到孔位坐标 → X/Y定位到螺丝孔                                   │
│  动作:                                                                  │
│   R  #bExec_MoveZ := 0                                                 │
│   N  #bSelFeeder := 0                          // ★ 切换为孔位坐标     │
│   S  #bExec_MoveX := 1, #bExec_MoveY := 1      // X/Y→螺丝孔           │
│                                                                         │
│  T7: #bDone_MoveX=1 AND #bDone_MoveY=1                                  │
│      ↓                                                                  │
│  S8: Z降到拧螺丝深度                                                    │
│  动作:                                                                  │
│   R  #bExec_MoveX := 0, #bExec_MoveY := 0                              │
│   S  #bExec_MoveZ := 1                                                 │
│                                                                         │
│  T8: #bDone_MoveZ=1                                                      │
│      ↓                                                                  │
│  S9: 电批拧螺丝                                                         │
│  动作:                                                                  │
│   R  #bExec_MoveZ := 0                                                 │
│   N  #bScrewStart := 1                                                 │
│                                                                         │
│  T9: bScrewBrake=1                              // 刹车=扭矩到达       │
│      ↓                                                                  │
│  S10: 停电批 + 关真空                                                    │
│  动作:                                                                  │
│   N  #bScrewStart := 0                                                 │
│   N  #bScrewVacuumValve := 0                                            │
│                                                                         │
│  T10: bScrewRunning=0 AND TON 200ms                                     │
│      ↓                                                                  │
│  S11: Z上升                                                            │
│  动作:                                                                  │
│   S  #bExec_MoveZ := 1                                                 │
│   N  #rZ_Target := #rZ_HomePos               // 手动覆盖为安全高度      │
│                                                                         │
│  T11: #bDone_MoveZ=1                                                     │
│       ↓                                                                 │
│       ├── #iCurrentScrew < #iScrewCount  →  跳转 S3 (下一颗)           │
│       └── #iCurrentScrew >= #iScrewCount →  S12                         │
│                                                                         │
│  S12: 拧螺丝完成                                                        │
│  动作:                                                                  │
│   R  #bExec_MoveZ := 0                                                 │
│   N  #bSelFeeder := 0                                                  │
│   N  "握手信号".bAck_ScrewDone := 1                                      │
│                                                                         │
│  T12: "握手信号".bReq_ScrewStart = 0                                    │
│      ↓                                                                  │
│      → 回到 S1                                                          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## bSelFeeder 的作用

```
bSelFeeder = 1  →  网络17生效  →  rX_Target = 螺丝盘坐标  →  用于 S3~S6
bSelFeeder = 0  →  网络16生效  →  rX_Target = 螺丝孔坐标  →  用于 S7~S11
```

切换时机：
- S3 开头置 1（要去螺丝盘了）
- S7 开头置 0（要去螺丝孔了）

---

## 一颗螺丝的完整路径

```
S3: bSelFeeder=1, iCurrentScrew+1, X/Y→螺丝盘 N
S4: Z↓ 取螺丝深度
S5: 真空吸
S6: Z↑
S7: bSelFeeder=0, X/Y→螺丝孔 N
S8: Z↓ 锁螺丝深度
S9: 电批拧 → 等刹车
S10: 停电批+关真空
S11: Z↑
→ 还有下一颗? 跳S3 / 全部完成? S12
```


## Supervision 超时监控

每个关键步设置超时，超时后跳转 ERR 步。

| 步骤 | 监控条件 | 超时时间 | 说明 |
|------|----------|----------|------|
| S2 | #bDone_HomeX=1 AND #bDone_HomeY=1 AND #bDone_HomeZ=1 | T#15S | 三轴回零超时 |
| S3 | #bDone_MoveX=1 AND #bDone_MoveY=1 | T#10S | X/Y到螺丝盘定位超时 |
| S4 | #bDone_MoveZ=1 | T#5S | Z到取螺丝深度超时 |
| S5 | bScrewVacuum=1 AND bFeederReady=1 | T#3S | 真空吸螺丝超时 |
| S6 | #bDone_MoveZ=1 | T#5S | Z上升超时 |
| S7 | #bDone_MoveX=1 AND #bDone_MoveY=1 | T#10S | X/Y到螺丝孔定位超时 |
| S8 | #bDone_MoveZ=1 | T#5S | Z到锁螺丝深度超时 |
| S9 | bScrewBrake=1 | T#8S | 电批拧紧超时(刹车未触发) |
| S11 | #bDone_MoveZ=1 | T#5S | Z上升超时 |
| 全局 | #bErr_MoveX=1 OR #bErr_MoveY=1 OR #bErr_MoveZ=1 | — | 伺服报警 |

在 GRAPH 编辑器中：选中步骤 → Supervision `-(v)-` → 设时间 → 跳转目标选 **ERR**。


## ERR 步

```
ERR 步动作:
  N  #bScrewStart := 0            // 停电批
  N  #bScrewVacuumValve := 0      // 关真空
  N  #bExec_MoveX := 0            // 停所有运动
  N  #bExec_MoveY := 0
  N  #bExec_MoveZ := 0
  N  #bExec_HomeX := 0
  N  #bExec_HomeY := 0
  N  #bExec_HomeZ := 0
  N  #bSelFeeder := 0

ERR → S1 转移条件:
  "bBtn_Reset" = 1                // I114.4 复位按钮
```

三色灯和蜂鸣器由 OB1 根据 `bError` 统一处理，ERR 步不需重复写。
