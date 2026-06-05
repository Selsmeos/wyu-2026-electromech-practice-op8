# FB_OP08_Screw (FB101) — 路二版：IO 脉冲直控

## 与 MC 版的区别

| | MC 版 (旧) | IO 版 (本文件) |
|--|-----------|--------------|
| 伺服控制 | TO_PositioningAxis + MC_MoveAbsolute | FC_ServoGotoPos 直接写 Q 脉冲 |
| 回零 | MC_Home | FC_ServoGotoHome |
| 前固定指令 | 15 个 MC 网络 + 1 个 SCL | 无 |
| TO 工艺对象 | 需要 3 个 | 不需要 |
| TM Pulse 模块 | 需要 | 不需要 |
| IO 地址 | TO 内部管理 | 直接操作 Q0.0~Q0.6, I0.2~I2.4 |

---

## 变量区

```
IN_OUT
    ── 螺丝刀传感器 ──
    bFeederReady     : Bool;     // I101.4 螺丝振盘到位
    bScrewVacuum     : Bool;     // I101.5 真空检知
    bScrewRunning    : Bool;     // I105.1 电批启动反馈
    bScrewBrake      : Bool;     // I105.2 电批刹车(扭矩到达)
    ── 螺丝刀执行器 ──
    bScrewStart       : Bool;    // Q107.2 电批启动
    bScrewVacuumValve : Bool;    // Q106.5 真空阀

STATIC
    ── FC 调用触发/反馈 ──
    bGo_HomeX        : Bool;     // 触发 X 回零
    bGo_HomeY        : Bool;
    bGo_HomeZ        : Bool;
    bGo_MoveX        : Bool;     // 触发 X 定位
    bGo_MoveY        : Bool;
    bGo_MoveZ        : Bool;
    bDone_HomeX      : Bool;     // X 回零完成
    bDone_HomeY      : Bool;
    bDone_HomeZ      : Bool;
    bDone_MoveX      : Bool;     // X 定位完成
    bDone_MoveY      : Bool;
    bDone_MoveZ      : Bool;
    bErr_MoveX       : Bool;     // X 伺服报警
    bErr_MoveY       : Bool;
    bErr_MoveZ       : Bool;
    ── 目标位置 ──
    rX_Target        : Real;
    rY_Target        : Real;
    rZ_Target        : Real;
    rVelocity        : Real := 30.0;
    ── 螺丝盘坐标 (3个取料位) ──
    rX_Feeder1       : Real := 0.0;
    rY_Feeder1       : Real := 80.0;
    rZ_Feeder1       : Real := -15.0;
    rX_Feeder2       : Real := 30.0;
    rY_Feeder2       : Real := 80.0;
    rZ_Feeder2       : Real := -15.0;
    rX_Feeder3       : Real := 60.0;
    rY_Feeder3       : Real := 80.0;
    rZ_Feeder3       : Real := -15.0;
    ── 螺丝孔坐标 ──
    rX_Screw1        : Real := 15.0;
    rY_Screw1        : Real := 10.0;
    rZ_Screw1        : Real := -25.0;
    rX_Screw2        : Real := 45.0;
    rY_Screw2        : Real := 10.0;
    rZ_Screw2        : Real := -25.0;
    rX_Screw3        : Real := 75.0;
    rY_Screw3        : Real := 10.0;
    rZ_Screw3        : Real := -25.0;
    ── 原点 ──
    rX_HomePos       : Real := 0.0;
    rY_HomePos       : Real := 0.0;
    rZ_HomePos       : Real := 0.0;
    ── 计数 ──
    iScrewCount      : Int := 3;
    iCurrentScrew    : Int := 0;
    ── 螺丝盘/孔位切换 ──
    bSelFeeder       : Bool;     // 1=螺丝盘坐标, 0=孔位坐标
```

---

## 前固定指令（共 8 个网络）

### Network 1: 螺丝孔坐标选择

```
bSelFeeder ─┤/├── iCurrentScrew==1 ──→ MOVE rX_Screw1→rX_Target
                                       MOVE rY_Screw1→rY_Target
                                       MOVE rZ_Screw1→rZ_Target
bSelFeeder ─┤/├── iCurrentScrew==2 ──→ MOVE rX_Screw2→rX_Target
                                       MOVE rY_Screw2→rY_Target
                                       MOVE rZ_Screw2→rZ_Target
bSelFeeder ─┤/├── iCurrentScrew==3 ──→ MOVE rX_Screw3→rX_Target
                                       MOVE rY_Screw3→rY_Target
                                       MOVE rZ_Screw3→rZ_Target
```

### Network 2: 螺丝盘坐标选择

```
bSelFeeder ─┤ ├── iCurrentScrew==1 ──→ MOVE rX_Feeder1→rX_Target
                                       MOVE rY_Feeder1→rY_Target
                                       MOVE rZ_Feeder1→rZ_Target
bSelFeeder ─┤ ├── iCurrentScrew==2 ──→ MOVE rX_Feeder2→rX_Target
                                       MOVE rY_Feeder2→rY_Target
                                       MOVE rZ_Feeder2→rZ_Target
bSelFeeder ─┤ ├── iCurrentScrew==3 ──→ MOVE rX_Feeder3→rX_Target
                                       MOVE rY_Feeder3→rY_Target
                                       MOVE rZ_Feeder3→rZ_Target
```

### Network 3: FB_ServoGotoHome X轴

```
FB_ServoGotoHome (背景 DB: FB_ServoGotoHome_DB_X)
  Axis      := 1
  bExecute  := #bGo_HomeX
  bDone     => #bDone_HomeX
  bError    => #bErr_MoveX
```

### Network 4: FB_ServoGotoHome Y轴

```
FB_ServoGotoHome (背景 DB: FB_ServoGotoHome_DB_Y)
  Axis      := 2
  bExecute  := #bGo_HomeY
  bDone     => #bDone_HomeY
  bError    => #bErr_MoveY
```

### Network 5: FB_ServoGotoHome Z轴

```
FB_ServoGotoHome (背景 DB: FB_ServoGotoHome_DB_Z)
  Axis      := 3
  bExecute  := #bGo_HomeZ
  bDone     => #bDone_HomeZ
  bError    => #bErr_MoveZ
```

### Network 6: FB_ServoGotoPos X轴

```
FB_ServoGotoPos (背景 DB: FB_ServoGotoPos_DB_X)
  Axis      := 1
  TargetPos := #rX_Target
  Velocity  := #rVelocity
  bExecute  := #bGo_MoveX
  bDone     => #bDone_MoveX
  bError    => #bErr_MoveX
```

### Network 7: FB_ServoGotoPos Y轴

```
FB_ServoGotoPos (背景 DB: FB_ServoGotoPos_DB_Y)
  Axis      := 2
  TargetPos := #rY_Target
  Velocity  := #rVelocity
  bExecute  := #bGo_MoveY
  bDone     => #bDone_MoveY
  bError    => #bErr_MoveY
```

### Network 8: FB_ServoGotoPos Z轴

```
FB_ServoGotoPos (背景 DB: FB_ServoGotoPos_DB_Z)
  Axis      := 3
  TargetPos := #rZ_Target
  Velocity  := #rVelocity
  bExecute  := #bGo_MoveZ
  bDone     => #bDone_MoveZ
  bError    => #bErr_MoveZ
```

### 注意事项

- Network 1~2 做坐标选择（根据 bSelFeeder 和 iCurrentScrew 更新 rX/Y/Z_Target）
- Network 3~8 做伺服控制（每周期调用 FC，由 GRAPH 步的 bGo_* 触发）
- Network 顺序必须是 1→2→3→8，坐标选择在前，FC 调用在后，这样 FC 读到的 Target 是最新的

---

## GRAPH 步序表

与 MC 版逻辑完全一致，只是**触发/反馈信号改为 FC 变量**。

```
┌──────────────────────────────────────────────────────────────────────┐
│  S1: 待机                                                             │
│  动作:                                                                  │
│   N  #bScrewStart := 0                                                 │
│   N  #bScrewVacuumValve := 0                                           │
│   N  "握手信号".bAck_ScrewDone := 0                                     │
│   N  #iCurrentScrew := 0                                               │
│   N  #bSelFeeder := 0                                                  │
│   N  #bGo_HomeX := 0; #bGo_HomeY := 0; #bGo_HomeZ := 0                 │
│   N  #bGo_MoveX := 0; #bGo_MoveY := 0; #bGo_MoveZ := 0                 │
│   N  #bDone_HomeX := 0; #bDone_HomeY := 0; #bDone_HomeZ := 0           │
│   N  #bDone_MoveX := 0; #bDone_MoveY := 0; #bDone_MoveZ := 0           │
│                                                                         │
│  T1: "握手信号".bReq_ScrewStart = 1                                     │
│      ↓                                                                  │
│  S2: 三轴回原点                                                         │
│  动作:                                                                  │
│   S  #bGo_HomeX := 1                                                   │
│   S  #bGo_HomeY := 1                                                   │
│   S  #bGo_HomeZ := 1                                                   │
│   // 前固定指令: FC_ServoGotoHome(Axis:=1) → bDone_HomeX               │
│   // 前固定指令: FC_ServoGotoHome(Axis:=2) → bDone_HomeY               │
│   // 前固定指令: FC_ServoGotoHome(Axis:=3) → bDone_HomeZ               │
│                                                                         │
│  T2: #bDone_HomeX=1 AND #bDone_HomeY=1 AND #bDone_HomeZ=1              │
│      ↓                                                                  │
│  ╔══════════════ 每颗螺丝循环: 取 → 拧 ═══════════════╗                │
│                                                                         │
│  S3: 计数+1, 选螺丝盘坐标 → X/Y 到螺丝盘                                │
│  动作:                                                                  │
│   R  #bGo_HomeX := 0; #bGo_HomeY := 0; #bGo_HomeZ := 0                 │
│   S1 #iCurrentScrew := #iCurrentScrew + 1          // ★ 只在步激活时执行一次
│   N  #bSelFeeder := 1                                                  │
│   S  #bGo_MoveX := 1                                                   │
│   S  #bGo_MoveY := 1                                                   │
│   // rX_Target / rY_Target 已由前固定指令更新                            │
│                                                                         │
│  T3: #bDone_MoveX=1 AND #bDone_MoveY=1                                  │
│      ↓                                                                  │
│  S4: Z 降到螺丝盘取螺丝                                                 │
│  动作:                                                                  │
│   R  #bGo_MoveX := 0; #bGo_MoveY := 0                                  │
│   S  #bGo_MoveZ := 1                                                   │
│   // rZ_Target 已由前固定指令更新                                        │
│                                                                         │
│  T4: #bDone_MoveZ=1                                                     │
│      ↓                                                                  │
│  S5: 真空吸螺丝                                                         │
│  动作:                                                                  │
│   R  #bGo_MoveZ := 0                                                   │
│   N  #bScrewVacuumValve := 1                                            │
│                                                                         │
│  T5: bScrewVacuum=1 AND bFeederReady=1 AND TON 300ms                   │
│      ↓                                                                  │
│  S6: Z 上升 (持螺丝)                                                    │
│  动作:                                                                  │
│   S  #bGo_MoveZ := 1                                                   │
│   // ★ bSelFeeder 还是 1，前固定指令取的是 Feeder 坐标                   │
│   //    需要在动作中手动覆盖 rZ_Target 为安全高度                         │
│   N  #rZ_Target := 0.0           // 覆盖为安全高度                       │
│                                                                         │
│  T6: #bDone_MoveZ=1                                                     │
│      ↓                                                                  │
│  S7: 切换孔位坐标 → X/Y 到螺丝孔                                        │
│  动作:                                                                  │
│   R  #bGo_MoveZ := 0                                                   │
│   N  #bSelFeeder := 0                                                  │
│   S  #bGo_MoveX := 1                                                   │
│   S  #bGo_MoveY := 1                                                   │
│                                                                         │
│  T7: #bDone_MoveX=1 AND #bDone_MoveY=1                                  │
│      ↓                                                                  │
│  S8: Z 降到锁螺丝深度                                                    │
│  动作:                                                                  │
│   R  #bGo_MoveX := 0; #bGo_MoveY := 0                                  │
│   S  #bGo_MoveZ := 1                                                   │
│                                                                         │
│  T8: #bDone_MoveZ=1                                                     │
│      ↓                                                                  │
│  S9: 电批拧螺丝                                                         │
│  动作:                                                                  │
│   R  #bGo_MoveZ := 0                                                   │
│   N  #bScrewStart := 1                                                  │
│                                                                         │
│  T9: bScrewBrake=1                                                      │
│      ↓                                                                  │
│  S10: 停电批 + 关真空                                                    │
│  动作:                                                                  │
│   N  #bScrewStart := 0                                                  │
│   N  #bScrewVacuumValve := 0                                            │
│                                                                         │
│  T10: bScrewRunning=0 AND TON 200ms                                     │
│      ↓                                                                  │
│  S11: Z 上升                                                            │
│  动作:                                                                  │
│   S  #bGo_MoveZ := 1                                                   │
│   N  #rZ_Target := 0.0           // 覆盖为安全高度                       │
│                                                                         │
│  T11: #bDone_MoveZ=1                                                     │
│       ↓                                                                 │
│       ├── #iCurrentScrew < #iScrewCount  →  跳转 S3 (下一颗)           │
│       └── #iCurrentScrew >= #iScrewCount →  S12                         │
│                                                                         │
│  S12: 完成                                                              │
│  动作:                                                                  │
│   R  #bGo_MoveZ := 0                                                   │
│   N  #bSelFeeder := 0                                                  │
│   N  "握手信号".bAck_ScrewDone := 1                                      │
│                                                                         │
│  T12: "握手信号".bReq_ScrewStart = 0                                    │
│      ↓                                                                  │
│      → 回到 S1                                                          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Supervision 超时监控

| 步骤 | 监控条件 | 超时 |
|------|----------|------|
| S2 | #bDone_HomeX=1 AND #bDone_HomeY=1 AND #bDone_HomeZ=1 | T#15S |
| S3 | #bDone_MoveX=1 AND #bDone_MoveY=1 | T#10S |
| S4 | #bDone_MoveZ=1 | T#5S |
| S5 | bScrewVacuum=1 AND bFeederReady=1 | T#3S |
| S6 | #bDone_MoveZ=1 | T#5S |
| S7 | #bDone_MoveX=1 AND #bDone_MoveY=1 | T#10S |
| S8 | #bDone_MoveZ=1 | T#5S |
| S9 | bScrewBrake=1 | T#8S |
| S11 | #bDone_MoveZ=1 | T#5S |
| 全局 | #bErr_MoveX=1 OR #bErr_MoveY=1 OR #bErr_MoveZ=1 | — |

## ERR 步

```
ERR 步动作:
  N  #bScrewStart := 0
  N  #bScrewVacuumValve := 0
  N  #bGo_MoveX := 0; #bGo_MoveY := 0; #bGo_MoveZ := 0
  N  #bGo_HomeX := 0; #bGo_HomeY := 0; #bGo_HomeZ := 0
  N  #bSelFeeder := 0
  // Q 脉冲点在 FC 内部随 bDone/置 0 自动清零

ERR → S1 转移条件:
  "bBtn_Reset" = 1    // I114.4
```

---

## 不需要的东西

- ~~TO_PositioningAxis~~
- ~~TM Pulse~~
- ~~MC_POWER / MC_MOVEABSOLUTE / MC_HOME / MC_RESET / MC_HALT~~
- ~~15 个 MC 网络~~

只需要：
- FB_ServoGotoPos (3 个调用，每轴一个背景 DB)
- FB_ServoGotoHome (3 个调用，每轴一个背景 DB)
- 2 个梯形图网络 (坐标选择)
- 总共 8 个前固定指令网络
