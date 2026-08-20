---
title: 某PVZ二游的肉鸽玩法服务端实现分析
date: 2026-08-20 21:10:00
updated: 2026-08-20 21:10:00
categories:
  - 技术实践
tags:
  - 本地服务端
  - 肉鸽
  - Roguelike
  - 状态机
  - 服务端实现
  - 逆向工程
description: 分析开源服务端对肉鸽（Roguelike）玩法的实现现状：15/57 端点、43 个缺失端点分类、进不去的原因，以及实现完整肉鸽服务端需要的数据驱动 + 状态机 + 事件处理。
---

# 某PVZ二游的肉鸽玩法服务端实现分析

## 前言

某款二次元塔防游戏中的肉鸽（Roguelike）玩法是一个复杂的游戏模式，涉及地图生成、随机事件、干员招募、收藏品收集等多个子系统。本文分析一个开源服务端项目对该玩法的实现情况，探讨为什么客户端无法正常进入该模式。

**声明**：本文分析的是开源社区项目的代码实现，不涉及对官方软件的逆向工程。

---

## 肉鸽玩法的客户端-服务端交互

肉鸽模式的核心流程可以分为几个阶段：

### 阶段一：初始化

```
1. 客户端请求创建游戏（选择主题、难度）
2. 服务端返回初始状态（干员、遗物、地图）
3. 客户端请求选择初始干员
4. 客户端请求选择初始遗物
```

### 阶段二：探索

```
5. 客户端请求移动到下一个节点
6. 服务端返回节点类型（战斗/事件/商店/精英/BOSS）
7. 客户端根据节点类型发起对应请求
```

### 阶段三：战斗与招募

```
8. 进入战斗 → 战斗结算
9. 战斗奖励 → 选择奖励
10. 招募干员 → 选择招募干员
11. 商店购买 → 刷新商品
```

### 阶段四：结算

```
12. 通关/失败 → 游戏结算
13. 获得奖励 → 返回主界面
```

---

## 开源项目的实现现状

我们分析的开源项目实现了 **15 个** 肉鸽相关端点，但客户端期望的端点数量约为 **57 个**。

### 已实现的端点（15 个）

```
createGame, chooseInitialRecruitSet, chooseInitialRelic,
moveTo, moveAndBattleStart, battleFinish, finishBattleReward,
finishEvent, recruitChar, closeRecruitTicket, setPinned,
gridZone/moveTo, gridZone/moveAndBattleStart, shopAction, giveUpGame
```

这些端点覆盖了核心流程的 1→11 步，但只实现了基本的状态管理，没有完整的业务逻辑。

### 缺失的端点（43 个，按功能分类）

**炼金术系统**（3 个）：
`alchemy`, `alchemyReward`, `sacrificeChoice`

**金融系统**（2 个）：
`bankPut`, `bankWithdraw`

**随机事件**（5 个）：
`diceChoice`, `expeditionChoice`, `confirmPredict`, `rerollNode`, `upgradeNode`

**奖励结算**（4 个）：
`confirmZoneReward`, `confirmTraderReturn`, `chooseBattleReward`, `readEndingChange`

**铜币系统**（4 个）：
`copper/change`, `copper/confirmDraw`, `copper/gild`, `copper/redraw`

**招募系统**（3 个）：
`getTicketAssistList`, `recruitAssistChar`, `stashRecruitTicket`

**商店系统**（2 个）：
`refreshShop`, `shopBattleStart`

**杂项**（20 个）：
`activeRecruitTicket`, `chooseInitialExploreTool`, `gameSettle`, `loseFragment`, 
`setSeed`, `setTroopCarry`, `specialZone/leave`, `useInspiration`, `useStashedTicket`,
`useTotem`, `normal/refreshMission`, `normal/unlockBuff`, `scrap/changeVehicle`,
`scrap/identify`, `scrap/loseScrap`, `gridZone/emptyStep`, `gridZone/readStepZero`,
`game/confirmExpeditonReturn`, `battlePass/buyReward`, `battlePass/getReward`

---

## 为什么客户端进不去肉鸽？

### 原因一：43 个端点返回 404

当客户端进入肉鸽的某个子系统（如炼金术、银行、骰子事件等）时，请求这些缺失的端点，服务端返回 404，客户端无法继续。

### 原因二：角色选择界面不显示

即使在已实现的端点中，`chooseInitialRecruitSet` 也存在问题。该端点返回**空对象** `{}`，但客户端期望以下格式的响应：

```json
{
  "playerDataDelta": {
    "modified": {
      "rlv2": {
        "current": {
          "player": {
            "recruitSet": [...],  // 初始干员列表
            "pending": [...]      // 待处理事件队列
          }
        }
      }
    }
  }
}
```

由于返回空对象，客户端无法获取初始干员列表，导致选择界面无法渲染。

---

## 实现一个完整的肉鸽服务端需要什么？

### 1. 数据驱动

肉鸽玩法严重依赖游戏数据，需要以下数据文件：

| 数据文件 | 用途 |
|---|---|
| roguelike_topic_table.json | 肉鸽主题定义（主题、难度、初始参数） |
| stage_table.json | 关卡数据（战斗节点对应的关卡） |
| character_table.json | 干员数据（招募可用的干员） |
| item_table.json | 收藏品/道具数据 |

### 2. 状态机

肉鸽是一个**有状态**的游戏模式，服务端需要维护每个玩家的肉鸽状态：

```
{
  "current": {
    "game": {"theme": "rogue_1", "seed": 12345},
    "player": {
      "hp": 100, "gold": 10, "level": 1,
      "recruitSet": [...],
      "pending": [...],
      "inventory": {...}
    },
    "map": {
      "nodes": [...],
      "currentZone": 0,
      "currentPosition": null
    },
    "troop": {
      "chars": {...},
      "squads": {...}
    }
  },
  "record": {
    "last": 0,
    "modeCnt": {},
    "endingCnt": {}
  }
}
```

### 3. 事件处理

每个节点类型（战斗、事件、商店等）对应不同的处理逻辑。服务端需要实现每种节点的处理函数。

---

## 与其他开源项目的对比

| 项目 | 肉鸽端点数 | 状态 |
|---|---|---|
| 项目 A（Java） | 0 | 未实现 |
| 项目 B（Python） | 15 | 部分实现，占 26% |
| 项目 C（Python，Flask） | 9 个函数 | 少于项目 B |

**结论**：目前没有一个开源项目实现了完整的肉鸽服务端。项目 B 是最接近的，但仍有 75% 的端点未实现。

---

## 总结

肉鸽模式进不去的主要原因是服务端实现不完整。相比其他游戏模式（如抽卡、招募、基建），肉鸽的复杂度高出一个数量级，需要管理游戏状态、地图生成、随机事件、战斗结算等多个子系统。

对于想要自行实现的服务端开发者，建议从以下步骤开始：

1. 先实现 `createGame` 和 `chooseInitialRecruitSet` 的正确响应格式
2. 然后实现探索阶段的核心端点（`moveTo`、`recruitChar`、`shopAction`）
3. 再逐个补充缺失的子系统（炼金术、银行、骰子等）
4. 最后实现结算系统（`gameSettle`、`finishBattleReward`）
