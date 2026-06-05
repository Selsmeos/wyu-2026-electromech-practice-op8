# FB_OP08_SendOut (FB102) — 送出工件 GRAPH 顺序图

## 接口

```
VAR_INPUT
    bEstop         : Bool;      // 急停 (I114.3)
    bReset         : Bool;      // 复位 (I114.4)
END_VAR

VAR_IN_OUT
    // === 一号移载 (取成品) ===
    bTrans1Lat_Ret    : Bool;  // I104.0 平移缩回位
    bTrans1Lat_Ext    : Bool;  // I104.1 平移伸出位
    bTrans1Lift_Up    : Bool;  // I104.2 升降上限位
    bTrans1Lift_Dn    : Bool;  // I104.3 升降下限位
    bTrans1Grip_Rel   : Bool;  // I104.4 夹爪松开
    bTrans1Grip_Clp   : Bool;  // I104.5 夹爪夹紧
    bTrans1ProdDet    : Bool;  // I104.6 产品检测
    bTrans1Lat_ExtDrv : Bool;  // Q106.0
    bTrans1Lat_RetDrv : Bool;  // Q106.1
    bTrans1Lift_DnDrv : Bool;  // Q106.2
    bTrans1Lift_UpDrv : Bool;  // Q106.3
    bTrans1Grip_ClpDrv: Bool;  // Q106.4
    
    // === 压摄像头安全互锁 (只读，不驱动) ===
    bPressCam_Up       : Bool;  // I115.1 ★移载前必须确认已上升
    bPressCamRot_Home  : Bool;  // I115.3 ★移载前必须确认已回原位
    
    // === 载具定位/阻挡 ===
    bStaBlkCyl_Start  : Bool;  // I100.2
    bStaBlkCyl_End    : Bool;  // I100.3
    bPosCyl_Start     : Bool;  // I100.4
    bPosCyl_End       : Bool;  // I100.5
    bPalletStaDetect  : Bool;  // I100.7 载具在位
    bStaBlkCyl_Drv    : Bool;  // Q102.1
    bPosCyl_Drv       : Bool;  // Q102.2
    
    // === 二次定位 ===
    b2ndPosLat_Start  : Bool;  // I101.0
    b2ndPosLat_End    : Bool;  // I101.1
    b2ndPosLon_Start  : Bool;  // I101.2
    b2ndPosLon_End    : Bool;  // I101.3
    b2ndPosLat_Drv    : Bool;  // Q102.4
    b2ndPosLon_Drv    : Bool;  // Q102.5
    
    // === 传送带 ===
    bConv_Main        : Bool;  // Q103.0
    bConv_OP8_9_1     : Bool;  // Q117.0 OP8→9#1
    bConv_OP8_9_1R    : Bool;  // Q117.1 回流#1
    bConv_OP8_9_2R    : Bool;  // Q117.3 回流#2
    
    // === 回流检测 ===
    bRetDetect_Mid    : Bool;  // I114.6 二号移载位物料感应
    bRetDetect_Pos    : Bool;  // I114.7 回流位有料检测
    bRetStop_Detect   : Bool;  // I115.0 回流挡停有料检测
    bRetStopCyl_Drv   : Bool;  // Q113.6 回流挡停气缸
    
    // === 安全 ===
    bSafetyDoor_Trans : Bool;  // I105.3 移载位安全门
    bSafetyDoor1      : Bool;  // I115.6
    bSafetyDoor2      : Bool;  // I115.7
END_VAR_IN_OUT
```

---

## GRAPH 步序表

```
┌─────────────────────────────────────────────────────────────────────┐
│  S1: 待机 (等待拧螺丝完成)                    [初始步]               │
│  动作:                                                                 │
│   N  #bTrans1Lat_RetDrv := 1        // 平移缩回(载具侧)                │
│   N  #bTrans1Lift_UpDrv := 1        // 升降上升                        │
│   N  #bTrans1Grip_ClpDrv := 0       // 夹爪松开                        │
│   N  #bConv_Main := 1               // 主传送带运行                     │
│   N  #bConv_OP8_9_1 := 0            // 复位工位段传送带                 │
│   N  "握手信号".bAck_OutDone := 0    // 清除完成                         │
│                                                                        │
│  T1: "握手信号".bAck_ScrewDone=1                      // ★ FB101拧螺丝完成          │
│      AND bPressCam_Up=1                              // ★ 压摄像头已上升(不撞)      │
│      AND bPressCamRot_Home=1                          // ★ 旋转缸已回原位(不撞)      │
│      AND bPalletStaDetect=1                           // 载具仍在位                  │
│       ↓                                                                │
│  S2: 移载伸到拧螺丝位取成品                                            │
│  动作:                                                                 │
│   N  #bTrans1Lat_RetDrv := 0                                           │
│   N  #bTrans1Lat_ExtDrv := 1        // 平移伸出(到拧螺丝位)             │
│                                                                        │
│  T2: bTrans1Lat_Ext=1              // 伸到位                           │
│       ↓                                                                │
│  S3: 下降取成品                                                        │
│  动作:                                                                 │
│   N  #bTrans1Lift_UpDrv := 0                                           │
│   N  #bTrans1Lift_DnDrv := 1        // 下降                            │
│                                                                        │
│  T3: bTrans1Lift_Dn=1 AND bTrans1ProdDet=1  // 下降到位且有产品       │
│       ↓                                                                │
│  S4: 夹爪夹紧                                                          │
│  动作:                                                                 │
│   N  #bTrans1Grip_ClpDrv := 1       // 夹紧                            │
│                                                                        │
│  T4: bTrans1Grip_Clp=1 AND TON 200ms                                  │
│       ↓                                                                │
│  S5: 上升                                                             │
│  动作:                                                                 │
│   N  #bTrans1Lift_DnDrv := 0                                           │
│   N  #bTrans1Lift_UpDrv := 1        // 上升                            │
│                                                                        │
│  T5: bTrans1Lift_Up=1                                                  │
│       ↓                                                                │
│  S6: 移载缩回到载具位                                                  │
│  动作:                                                                 │
│   N  #bTrans1Lat_ExtDrv := 0                                           │
│   N  #bTrans1Lat_RetDrv := 1        // 缩回(回到载具上方)              │
│                                                                        │
│  T6: bTrans1Lat_Ret=1                                                  │
│       ↓                                                                │
│  S7: 下降放成品到载具                                                  │
│  动作:                                                                 │
│   N  #bTrans1Lift_UpDrv := 0                                           │
│   N  #bTrans1Lift_DnDrv := 1        // 下降                            │
│                                                                        │
│  T7: bTrans1Lift_Dn=1                                                  │
│       ↓                                                                │
│  S8: 松开夹爪                                                          │
│  动作:                                                                 │
│   N  #bTrans1Grip_ClpDrv := 0       // 松开                            │
│                                                                        │
│  T8: bTrans1Grip_Rel=1 AND TON 300ms                                  │
│       ↓                                                                │
│  S9: 上升                                                             │
│  动作:                                                                 │
│   N  #bTrans1Lift_DnDrv := 0                                           │
│   N  #bTrans1Lift_UpDrv := 1        // 上升                            │
│                                                                        │
│  T9: bTrans1Lift_Up=1                                                  │
│       ↓                                                                │
│  S10: 释放二次定位+定位气缸                                            │
│  动作:                                                                 │
│   N  #b2ndPosLat_Drv := 0           // 二次定位松开                     │
│   N  #b2ndPosLon_Drv := 0                                              │
│                                                                        │
│  T10: b2ndPosLat_Start=1 AND b2ndPosLon_Start=1  // 确认已松开        │
│       ↓                                                                │
│  S11: 释放载具定位+阻挡                                                │
│  动作:                                                                 │
│   N  #bPosCyl_Drv := 0              // 定位松开                         │
│   N  #bStaBlkCyl_Drv := 0           // 阻挡松开                         │
│                                                                        │
│  T11: bPosCyl_Start=1 AND bStaBlkCyl_Start=1  // 确认已松开           │
│       ↓                                                                │
│  S12: 送出载具                                                         │
│  动作:                                                                 │
│   N  #bConv_OP8_9_1 := 1            // OP8→9传送带运行                  │
│   N  #bConv_Main := 1               // 主传送带运行                     │
│                                                                        │
│  T12: bPalletStaDetect=0 AND TON 2s  // 载具离开+延时确认              │
│       ↓                                                                │
│  S13: 送出完成                                                        │
│  动作:                                                                 │
│   N  "握手信号".bAck_OutDone := 1    // ★ 通知FB100                     │
│                                                                        │
│  T13: "握手信号".bAck_OutDone = 0    // FB100收到后清除了               │
│       ↓                                                                │
│       → 回到 S1                                                        │
│                                                                         │
│  ERR: 故障步                                                            │
│  动作:                                                                  │
│   N  #bTrans1Lat_ExtDrv := 0                                           │
│   N  #bTrans1Lat_RetDrv := 0                                           │
│   N  #bTrans1Lift_DnDrv := 0                                           │
│   N  #bTrans1Lift_UpDrv := 0                                           │
│   N  #bTrans1Grip_ClpDrv := 0                                          │
│   N  #bConv_Main := 0                                                  │
│   N  #bConv_OP8_9_1 := 0                                               │
│   N  #bStaBlkCyl_Drv := 0                                              │
│   N  #bPosCyl_Drv := 0                                                 │
│   N  #b2ndPosLat_Drv := 0                                              │
│   N  #b2ndPosLon_Drv := 0                                              │
│   N  "握手信号".bAck_OutDone := 0                                       │
│                                                                         │
│  TERR: %I114.4 = 1                 // 复位按钮 → 回到 S1               │
│       ↓                                                                 │
│       → 回到 S1                                                        │
└─────────────────────────────────────────────────────────────────────┘
```


## Supervision 超时监控

| 步骤 | 监控条件 | 超时时间 | 说明 |
|------|----------|----------|------|
| S2 | bTrans1Lat_Ext=1 | T#3S | 平移伸出超时 |
| S3 | bTrans1Lift_Dn=1 | T#3S | 下降超时 |
| S3 | bTrans1ProdDet=1 | — | 无产品→ERR |
| S4 | bTrans1Grip_Clp=1 | T#2S | 夹紧超时 |
| S5 | bTrans1Lift_Up=1 | T#3S | 上升超时 |
| S6 | bTrans1Lat_Ret=1 | T#3S | 平移缩回超时 |
| S10 | b2ndPosLat_Start=1 AND b2ndPosLon_Start=1 | T#3S | 二次定位松开超时 |
| S11 | bPosCyl_Start=1 AND bStaBlkCyl_Start=1 | T#3S | 载具释放超时 |
| S12 | bPalletStaDetect=0 | T#5S | 载具未离开 |
| 全局 | 安全门开(bSafetyDoor1/2=0) | — | 紧急停止 |

在 GRAPH 编辑器中：选中步骤 → Supervision `-(v)-` → 设时间 → 跳转目标选 **ERR**。


## ERR 步

```
ERR 步动作:
  N  #bTrans1Lat_ExtDrv := 0       // 平移停
  N  #bTrans1Lat_RetDrv := 0
  N  #bTrans1Lift_DnDrv := 0       // 升降停
  N  #bTrans1Lift_UpDrv := 0
  N  #bTrans1Grip_ClpDrv := 0      // 夹爪松
  N  #bConv_Main := 0              // 传送带停
  N  #bConv_OP8_9_1 := 0
  N  #bStaBlkCyl_Drv := 0          // 阻挡松
  N  #bPosCyl_Drv := 0             // 定位松
  N  #b2ndPosLat_Drv := 0          // 二次定位松
  N  #b2ndPosLon_Drv := 0
  N  "握手信号".bAck_OutDone := 0    // 清信号

ERR → S1 转移条件:
  "bBtn_Reset" = 1                 // I114.4 复位按钮
```

三色灯和蜂鸣器由 OB1 统一处理。
