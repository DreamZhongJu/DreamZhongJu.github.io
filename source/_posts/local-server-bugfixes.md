---
title: 某PVZ二游的本地服务端Bug修复记录
date: 2026-08-20 21:05:00
updated: 2026-08-20 21:05:00
categories:
  - 技术实践
tags:
  - 本地服务端
  - Bug修复
  - Java
  - 类型解析
  - 防御性编程
  - 逆向工程
description: 记录部署本地服务端时遇到的 6 个 Bug（类型不匹配、空值处理缺失、数据结构不一致三类），以及「现象 → 根因 → 修复」的完整排查过程。
---

# 某PVZ二游的本地服务端Bug修复记录

## 前言

在部署某款二次元塔防游戏的本地服务端时，遇到了不少 Bug。大部分是开源项目自身的代码质量问题，少部分是版本兼容性问题。本文记录了修复过程，作为技术参考。

**声明**：本文分析的是开源社区项目代码，不涉及对官方软件的逆向工程。

---

## Bug 1：掉落类型解析错误

### 现象

打完关卡后，服务端返回 HTTP 500，战斗结算失败。

### 根因

游戏关卡掉落表（stage table）中，`dropType` 字段是字符串枚举：

```
"dropType": "ONCE"      // 首次通关掉落
"dropType": "NORMAL"    // 常规掉落
"dropType": "ADDITIONAL" // 额外掉落
"dropType": "COMPLETE"   // 通关奖励
```

但服务端代码使用了 `getIntValue()` 将其解析为整数：

```java
// 错误代码
int dropType = reward.getIntValue("dropType");
```

当遇到 `"ONCE"` 时，解析器抛出 `NumberFormatException`。

### 修复

将整数解析改为字符串解析，并建立映射关系：

```java
String dropType = reward.getString("dropType");
// 转为内部编码
int dropTypeCode = "NORMAL".equals(dropType) ? 2
    : "COMPLETE".equals(dropType) ? 2
    : "ADDITIONAL".equals(dropType) ? 4
    : 0;
```

---

## Bug 2：掉落概率解析错误

### 现象

同上，也是战斗结算 500 错误，但这次是另一个字段。

### 根因

同样的数据表中，`occPercent` 字段也是字符串枚举：

```
"occPercent": "ALWAYS"     // 必定掉落
"occPercent": "SOMETIMES"  // 概率掉落
```

服务端代码同样使用了 `getIntValue()`：

```java
int occPercent = reward.getIntValue("occPercent");  // 错误！
```

### 修复

```java
String occPercentStr = reward.getString("occPercent");
int occPercent = "ALWAYS".equals(occPercentStr) ? 0 : 2;
```

---

## Bug 3：战斗回放保存失败阻塞流程

### 现象

打完关卡后，保存战斗回放时服务端崩溃，导致客户端无法继续。

### 根因

战斗回放数据是 Base64 编码 + GZip 压缩的 JSON。服务端解密失败时返回 `null`，后续代码未做空值检查，直接访问 `null` 的字段导致 NPE。

### 修复

在解密失败时，返回最小成功响应，不阻塞后续流程：

```java
try {
    String stageId = BattleData.getJSONObject("journal")
        .getJSONObject("metadata").getString("stageId");
    // ... 正常处理
} catch (Exception ex) {
    // 返回空成功响应，不阻塞战斗流程
    return createEmptySuccessResponse();
}
```

---

## Bug 4：主线进度指向教学关卡

### 现象

通关主线第一关后，主进度被设置为 `tr_01`（教学关卡），客户端误以为需要进入教学关，反复触发"数据已更新，即将同步"提示。

### 根因

服务端的关卡解锁逻辑中，主线关卡（`main_00-01`）的下一关在数据表里指向了教学关卡（`tr_01`），`mainStageProgress` 被错误更新：

```java
String nextStageId = mainStage.getJSONObject(stageId).getString("next");
// nextStageId = "tr_01"  ← 教学关
UserSyncData.getJSONObject("status").put("mainStageProgress", nextStageId);
```

### 修复

只对 `main_` 前缀的主线关卡更新进度：

```java
if (nextStageId.startsWith("main_")) {
    UserSyncData.getJSONObject("status").put("mainStageProgress", nextStageId);
}
```

---

## Bug 5：干员数据字段缺失

### 现象

在基建界面查看干员休息状态时，客户端崩溃，反复提示"数据已更新，即将同步"。

### 根因

服务端在生成干员数据时，缺少 `bubble.private` 和 `privateRooms` 字段。客户端读取干员疲劳数据时访问不存在的字段，触发异常。

正确格式：
```json
{
  "bubble": {
    "normal": {"add": -1, "ts": 0},
    "assist": {"add": -1, "ts": 0},
    "private": {"add": -1, "ts": 0}  // ← 缺少这个
  },
  "privateRooms": []  // ← 缺少这个
}
```

### 修复

补充完整字段定义，确保所有干员都有正确的基建数据结构。

---

## Bug 6：抽卡系统空指针异常

### 现象

抽卡时服务端返回 500，无法完成抽卡。

### 根因

抽卡逻辑从 `gacha.normal` 中读取当前卡池的抽卡次数，但新卡池在 `gacha.normal` 中没有对应的条目，返回 `null`：

```java
JSONObject normal = gacha.getJSONObject("normal");
int cnt = normal.getJSONObject(poolId).getIntValue("cnt");  // poolId 不存在 → NPE
```

### 修复

在玩家存档中预置所有卡池的计数条目，初始值设为 0。

---

## 小结

这些 Bug 的本质原因可以归纳为三类：

1. **类型不匹配**：字符串枚举当整数解析（Bug 1、2）
2. **空值处理缺失**：解密失败等场景未做防御性编程（Bug 3、6）
3. **数据结构不一致**：服务端生成的数据与客户端期望的格式有差异（Bug 4、5）

修复这些 Bug 的经验可以推广到其他类似项目的开发中：**服务端实现必须严格遵循客户端的数据格式约定**，尤其是在处理从游戏资源文件中读取的数据时，应先确认字段类型。
