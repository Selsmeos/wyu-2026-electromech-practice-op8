# FB_OP08_Receive (FB100) — 接收工件 GRAPH 顺序图

## 接口

```
VAR_INPUT
    bAutoMode   : Bool;     // 自动模式 (I114.5)
    bStart      : Bool;     // 启动按钮 (I114.1)
    bStop       : Bool;     // 停止按钮 (I114.2)
    bEstop      : Bool;     // 急停 (I114.3, NC→FALSE=触发)
    bReset      : Bool;     // 复位按钮 (I114.4)
END_VAR

VAR_IN_OUT
    // 气缸传感器 (地址见变量分配表)
    bPreBlkCyl_Start   : Bool;  // I100.0
    bPreBlkCyl_End     : Bool;  // I100.1
    bStaBlkCyl_Start   : Bool;  // I100.2
    bStaBlkCyl_End     : Bool;  // I100.3
    bPosCyl_Start      : Bool;  // I100.4
    bPosCyl_End        : Bool;  // I100.5
    bPalletPreDetect   : Bool;  // I100.6
    bPalletStaDetect   : Bool;  // I100.7
    b2ndPosLat_Start   : Bool;  // I101.0
    b2ndPosLat_End     : Bool;  // I101.1
    b2ndPosLon_Start   : Bool;  // I101.2
    b2ndPosLon_End     : Bool;  // I101.3
    bTrans1Lat_Ret     : Bool;  // I104.0
    bTrans1Lat_Ext     : Bool;  // I104.1
    bTrans1Lift_Up     : Bool;  // I104.2
    bTrans1Lift_Dn     : Bool;  // I104.3
    bTrans1Grip_Rel    : Bool;  // I104.4
    bTrans1Grip_Clp    : Bool;  // I104.5
    bTrans1ProdDet     : Bool;  // I104.6
    bPressCam_Up       : Bool;  // I115.1
    bPressCam_Dn       : Bool;  // I115.2
    bPressCamRot_Home  : Bool;  // I115.3
    bPressCamRot_Press : Bool;  // I115.4
    bSafetyDoor1       : Bool;  // I115.6
    bSafetyDoor2       : Bool;  // I115.7
    // 输出
    bConv_Main          : Bool;  // Q103.0
    bPreBlkCyl_Drv      : Bool;  // Q102.0
    bStaBlkCyl_Drv      : Bool;  // Q102.1
    bPosCyl_Drv         : Bool;  // Q102.2
    b2ndPosLat_Drv      : Bool;  // Q102.4
    b2ndPosLon_Drv      : Bool;  // Q102.5
    bPressCam_UpDrv     : Bool;  // Q103.2
    bPressCam_DnDrv     : Bool;  // Q103.3
    bPressCamRot_PressDrv: Bool; // Q103.4
    bPressCamRot_HomeDrv : Bool; // Q103.5
    bTrans1Lat_ExtDrv   : Bool;  // Q106.0
    bTrans1Lat_RetDrv   : Bool;  // Q106.1
    bTrans1Lift_DnDrv   : Bool;  // Q106.2
    bTrans1Lift_UpDrv   : Bool;  // Q106.3
    bTrans1Grip_ClpDrv  : Bool;  // Q106.4
END_VAR_IN_OUT
```

---

## GRAPH 步序表

图例: N=步骤激活时输出, S=置位, R=复位, D=延时

```
┌─────────────────────────────────────────────────────────────────────┐
│  S1: 待机                                [初始步/启动后回到此步]       │
│  动作:                                                                 │
│   N  bConv_Main := 1            // 主传送带运行                        │
│   N  bTrans1Lat_RetDrv := 1     // 平移缩回(取料位)                    │
│   N  bTrans1Lift_UpDrv := 1     // 升降上升                           │
│   N  bTrans1Grip_ClpDrv := 0    // 夹爪松开                           │
│   N  bPreBlkCyl_Drv := 0        // 前阻挡松开                          │
│   N  bStaBlkCyl_Drv := 0        // 工位阻挡松开                        │
│   N  bPosCyl_Drv := 0           // 定位松开                           │
│   N  b2ndPosLat_Drv := 0        // 二次定位松开                        │
│   N  b2ndPosLon_Drv := 0       //                                    │
│   N  bPressCam_UpDrv := 1       // 压摄像头升(安全位)                  │
│   N  bPressCamRot_HomeDrv := 1  // 旋转缸回原位                        │
│                                                                        │
│  T1: bPalletPreDetect=1 AND "握手信号".bAck_OutDone=1                  │
│      ↓                                                                 │
│  S2: 前阻挡                                                           │
│  动作:                                                                 │
│   N  bPreBlkCyl_Drv := 1        // 前阻挡伸出                           │
│   N  bConv_Main := 0            // 主传送带停                           │
│                                                                        │
│  T2: bPreBlkCyl_End=1 AND TON 500ms                                    │
│      ↓                                                                 │
│  S3: 载具导入                                                          │
│  动作:                                                                 │
│   N  bConv_Main := 1             // 工位段传送带运行(导入载具)          │
│                                                                        │
│  T3: bPalletStaDetect=1          // 工位载具检知                        │
│      ↓                                                                 │
│  S4: 工位阻挡+定位                                                     │
│  动作:                                                                 │
│   N  bStaBlkCyl_Drv := 1        // 工位阻挡伸出                         │
│   N  bConv_Main := 0            // 工位传送带停                         │
│   N  bPreBlkCyl_Drv := 0        // 前阻挡松开(放下一件到等待位)          │
│                                                                        │
│  T4: bStaBlkCyl_End=1           // 阻挡到位                            │
│      ↓                                                                 │
│  S5: 定位                                                             │
│  动作:                                                                 │
│   N  bPosCyl_Drv := 1           // 定位气缸伸出                         │
│                                                                        │
│  T5: bPosCyl_End=1 AND TON 300ms                                       │
│      ↓                                                                 │
│  S6: 移载下降取料                                                      │
│  动作:                                                                 │
│   N  bTrans1Lift_UpDrv := 0                                            │
│   N  bTrans1Lift_DnDrv := 1     // 升降缸下降                          │
│                                                                        │
│  T6: bTrans1Lift_Dn=1 AND bTrans1ProdDet=1  // 下降到位且有产品        │
│      ↓                                                                 │
│  S7: 夹爪夹紧                                                          │
│  动作:                                                                 │
│   N  bTrans1Grip_ClpDrv := 1    // 夹爪夹紧                             │
│                                                                        │
│  T7: bTrans1Grip_Clp=1 AND TON 200ms                                   │
│      ↓                                                                 │
│  S8: 移载上升                                                          │
│  动作:                                                                 │
│   N  bTrans1Lift_DnDrv := 0                                            │
│   N  bTrans1Lift_UpDrv := 1     // 上升                                │
│                                                                        │
│  T8: bTrans1Lift_Up=1           // 上升到位                            │
│      ↓                                                                 │
│  S9: 移载平移到拧螺丝位                                                │
│  动作:                                                                 │
│   N  bTrans1Lat_RetDrv := 0                                            │
│   N  bTrans1Lat_ExtDrv := 1     // 平移伸出(送到拧螺丝工位)             │
│                                                                        │
│  T9: bTrans1Lat_Ext=1           // 平移到位                            │
│      ↓                                                                 │
│  S10: 移载下降放料                                                     │
│  动作:                                                                 │
│   N  bTrans1Lift_UpDrv := 0                                            │
│   N  bTrans1Lift_DnDrv := 1     // 下降                                │
│                                                                        │
│  T10: bTrans1Lift_Dn=1          // 下降到位                            │
│      ↓                                                                 │
│  S11: 松开夹爪                                                        │
│  动作:                                                                 │
│   N  bTrans1Grip_ClpDrv := 0    // 松开夹爪                             │
│                                                                        │
│  T11: bTrans1Grip_Rel=1 AND TON 300ms                                  │
│      ↓                                                                 │
│  S12: 移载上升退回                                                     │
│  动作:                                                                 │
│   N  bTrans1Lift_DnDrv := 0                                            │
│   N  bTrans1Lift_UpDrv := 1     // 上升                                │
│                                                                        │
│  T12: bTrans1Lift_Up=1          // 上升到位                            │
│      ↓                                                                 │
│  S13: 移载缩回                                                        │
│  动作:                                                                 │
│   N  bTrans1Lat_ExtDrv := 0                                            │
│   N  bTrans1Lat_RetDrv := 1     // 平移缩回                             │
│                                                                        │
│  T13: bTrans1Lat_Ret=1          // 缩回到位                            │
│      ↓                                                                 │
│  S14: 二次定位                                                         │
│  动作:                                                                 │
│   N  bPosCyl_Drv := 1           // 定位气缸保持                         │
│   N  b2ndPosLat_Drv := 1        // 二次定位横向伸出                     │
│   N  b2ndPosLon_Drv := 1        // 二次定位纵向伸出                     │
│                                                                        │
│  T14: b2ndPosLat_End=1 AND b2ndPosLon_End=1  // 二次定位到位           │
│      ↓                                                                 │
│  S15: 旋转缸转到压位                                                   │
│  动作:                                                                 │
│   N  bPressCamRot_HomeDrv := 0                                         │
│   N  bPressCamRot_PressDrv := 1 // 旋转到压位                           │
│                                                                        │
│  T15: bPressCamRot_Press=1      // 旋转缸到压位                        │
│      ↓                                                                 │
│  S16: 压摄像头下降 (★ 下一步开始全程保持压下)                           │
│  动作:                                                                 │
│   N  bPressCam_UpDrv := 0                                              │
│   N  bPressCam_DnDrv := 1       // 压摄像头下降(压紧摄像头)             │
│                                                                        │
│  T16: bPressCam_Dn=1 AND TON 500ms  // 压紧到位+稳定                   │
│      ↓                                                                 │
│  S17: 等待拧螺丝 (★ 压摄像头保持压下状态)                               │
│  动作:                                                                 │
│   N  bPressCam_DnDrv := 1       // ← 保持压下！                        │
│   N  "握手信号".bReq_ScrewStart := 1    // ★ 通知FB101拧螺丝            │
│                                                                        │
│  T17: "握手信号".bAck_ScrewDone=1  // ★ FB101拧螺丝完成→立即松开压头   │
│      ↓                                                                 │
│  S18: 压摄像头上升 (拧完了，松开)                                       │
│  动作:                                                                 │
│   N  bPressCam_DnDrv := 0                                              │
│   N  bPressCam_UpDrv := 1       // 压摄像头上升                         │
│                                                                        │
│  T18: bPressCam_Up=1            // 上升到位                            │
│      ↓                                                                 │
│  S19: 旋转缸回原位 + 等送出完成                                        │
│  动作:                                                                 │
│   N  bPressCamRot_PressDrv := 0                                        │
│   N  bPressCamRot_HomeDrv := 1  // 旋转回原位                           │
│                                                                        │
│  T19: bPressCamRot_Home=1 AND "握手信号".bAck_OutDone=1                │
│      ↓                                                                 │
│      → 回到 S1                                                         │
│                                                                         │
│  ERR: 故障步                                                            │
│  动作:                                                                  │
│   N  bConv_Main := 0                                                   │
│   N  bPreBlkCyl_Drv := 0                                               │
│   N  bStaBlkCyl_Drv := 0                                               │
│   N  bPosCyl_Drv := 0                                                  │
│   N  b2ndPosLat_Drv := 0                                               │
│   N  b2ndPosLon_Drv := 0                                               │
│   N  bTrans1Lat_ExtDrv := 0                                            │
│   N  bTrans1Lat_RetDrv := 0                                            │
│   N  bTrans1Lift_DnDrv := 0                                            │
│   N  bTrans1Lift_UpDrv := 0                                            │
│   N  bTrans1Grip_ClpDrv := 0                                           │
│   N  bPressCam_DnDrv := 0                                              │
│   N  bPressCam_UpDrv := 1            // 压头升到安全位                  │
│   N  bPressCamRot_PressDrv := 0                                        │
│   N  bPressCamRot_HomeDrv := 1       // 旋转回原位                      │
│   N  "握手信号".bReq_ScrewStart := 0                                    │
│   N  "握手信号".bAck_OutDone := 0                                       │
│                                                                         │
│  TERR: %I114.4 = 1                 // 复位按钮 → 回到 S1               │
│      ↓                                                                  │
│      → 回到 S1                                                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键时序 (压摄像头 + 三FB协作)

```
FB100:
S15: 旋转到压位
  │
  ▼
S16: 压下降 (压紧摄像头)
  │
  ▼
S17: ┌─ bReq_ScrewStart=1 ──────────────────┐
     │  ★ 压头保持压下 (bPressCam_DnDrv=1)   │
     │                                         │
     │  FB101: S2回零→S3定位→S4降→S5吸→       │
     │         S6拧→S7停→S8升 (螺丝1)         │
     │         S3定位→S4降→S5吸→               │
     │         S6拧→S7停→S8升 (螺丝2)         │
     │         S9 bAck_ScrewDone=1             │
     │  ← bAck_ScrewDone=1 ──────────────────┘
  │
  ▼
S18: 压上升 (拧完，松开)
  │
  ▼
S19: 旋转回位 + 等送出完成
     │
     │  FB102: T1确认压头↑+旋转回位           │
     │         S2~S8 移载取成品放回载具       │
     │         S10~S12 释放载具送出            │
     │         S13 bAck_OutDone=1              │
     │  ← bAck_OutDone=1 ─────────────────┘
  │
  ▼
S1: 待机
```

## 为什么不会死锁

| 步骤 | 事件 |
|------|------|
| FB101完成 | → `bAck_ScrewDone=1` |
| FB100 T17触发 | → S18压上升 → S19旋转回位 (压头物理释放) |
| FB102 T1 | 检查 `bAck_ScrewDone=1 AND bPressCam_Up=1 AND bPressCamRot_Home=1` |
| | → 此时压头已释放，条件全部满足，开始取成品 |
| FB102完成 | → `bAck_OutDone=1` |
| FB100 T19触发 | → 回到S1, 准备接收下一件 |

---

## 异常处理

| 步骤 | 监控条件 | 超时 | 故障处理 |
|------|----------|------|----------|
| S2 | bPreBlkCyl_End=1 | 3s | 跳转ERR |
| S5 | bPosCyl_End=1 | 3s | 跳转ERR |
| S6 | bTrans1Lift_Dn=1 | 3s | 跳转ERR |
| S6 | bTrans1ProdDet=1 | — | 无产品→ERR |
| S7 | bTrans1Grip_Clp=1 | 2s | 跳转ERR |
| S13 | bTrans1Lat_Ret=1 | 3s | 跳转ERR |
| S16 | bPressCam_Dn=1 | 3s | 跳转ERR |
| S17 | — | 60s | 拧螺丝+送出总超时 |

**ERR步**: 停止所有输出，红灯(Q116.2)闪烁，蜂鸣(Q116.3)响，等待复位按钮。
复位条件: bReset=1 → 所有气缸回原位 → 跳转S1
