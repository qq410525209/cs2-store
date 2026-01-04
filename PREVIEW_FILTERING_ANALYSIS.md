# CS2-Store Preview功能过滤机制分析

## 概述

本文档详细说明CS2-Store插件中预览（Inspect）功能如何实现过滤，确保只有预览使用者能够看到预览模型，而其他玩家无法看到该模型。

## 核心组件

### 1. InspectList 字典存储
**位置**: [`Store/src/cs2-store.cs:30`](Store/src/cs2-store.cs:30)

```csharp
public Dictionary<CBaseModelEntity, CCSPlayerController> InspectList { get; set; } = [];
```

**作用**:
- 存储预览实体（3D模型）与其所有者（玩家）的映射关系
- 键：`CBaseModelEntity` - 预览的3D模型实体
- 值：`CCSPlayerController` - 拥有该预览的玩家

### 2. 预览菜单选项
**位置**: [`Store/src/menu/menu.cs:165-179`](Store/src/menu/menu.cs:165)

```csharp
public static void AddInspectOption(this IMenu menu, CCSPlayerController player, Dictionary<string, string> item)
{
    if (item["type"] is not ("playerskin" or "customweapon"))
        return;

    menu.AddMenuOption(player, (p, o) =>
    {
        o.PostSelectAction = PostSelectAction.Nothing;

        if (Instance.InspectList.ContainsValue(p))
            return;

        InspectAction(p, item, item["type"]);
    }, "menu_store<inspect>");
}
```

**功能**:
- 仅为 `playerskin`（玩家皮肤）和 `customweapon`（自定义武器）类型的物品添加预览选项
- 检查玩家是否已经在预览其他物品（`ContainsValue(p)`）- 每个玩家同时只能预览一个物品
- 调用 `InspectAction` 来启动预览

### 3. 预览实体创建
**位置**: [`Store/src/item/items/playerskin.cs:147-173`](Store/src/item/items/playerskin.cs:147)

```csharp
public static void Inspect(CCSPlayerController player, string model, string? skin)
{
    CBaseModelEntity? entity = Utilities.CreateEntityByName<CBaseModelEntity>("prop_dynamic");
    if (entity == null || !entity.IsValid || player.PlayerPawn.Value is not CCSPlayerPawn playerPawn)
        return;

    Vector _origin = GetFrontPosition(playerPawn.AbsOrigin!, playerPawn.EyeAngles);
    QAngle modelAngles = new(0, playerPawn.EyeAngles.Y + 180, 0);

    entity.Spawnflags = 256u;
    entity.Collision.SolidType = SolidType_t.SOLID_VPHYSICS;
    entity.Teleport(_origin, modelAngles, playerPawn.AbsVelocity);
    entity.DispatchSpawn();

    Server.NextFrame(() =>
    {
        if (entity.IsValid)
        {
            entity.SetModel(model);
            if (skin != null)
                entity.AcceptInput("Skin", null, entity, skin);
        }
    });

    Instance.InspectList[entity] = player;
    Instance.AddTimer(1.0f, () => RotateEntity(player, entity, 0.0f));
}
```

**执行流程**:
1. 创建一个 `prop_dynamic` 实体（3D动态模型）
2. 计算玩家面前的位置（距离100单位）
3. 设置模型朝向玩家（角度 +180°）
4. 在游戏中生成实体
5. **关键步骤**: 将实体与玩家关联 → `Instance.InspectList[entity] = player;`
6. 启动旋转动画（持续4秒）

### 4. 实体旋转动画
**位置**: [`Store/src/item/items/playerskin.cs:175-195`](Store/src/item/items/playerskin.cs:175)

```csharp
public static void RotateEntity(CCSPlayerController player, CBaseModelEntity entity, float elapsed)
{
    if (!entity.IsValid)
        return;

    float totalTime = 4.0f;
    float totalRotation = 360.0f;
    float interval = 0.04f;
    float rotationStep = (interval / totalTime) * totalRotation;

    QAngle currentAngles = entity.AbsRotation!;
    entity.Teleport(null, new QAngle(currentAngles.X, currentAngles.Y + rotationStep, currentAngles.Z), null);

    if (elapsed < totalTime)
        Instance.AddTimer(interval, () => RotateEntity(player, entity, elapsed + interval));
    else
    {
        Instance.InspectList.Remove(entity);
        entity.Remove();
    }
}
```

**功能**:
- 每 0.04 秒旋转模型一次
- 总旋转时间：4 秒
- 总旋转角度：360°（完整旋转一圈）
- 4秒后自动移除实体并从 `InspectList` 中清除

## 核心过滤机制 - OnCheckTransmit

**位置**: [`Store/src/event/event.cs:263-290`](Store/src/event/event.cs:263)

这是实现预览过滤的**关键机制**：

```csharp
public static void OnCheckTransmit(CCheckTransmitInfoList infoList)
{
    if (Instance.InspectList.Count == 0 && Item_Trail.TrailList.Count == 0)
        return;

    foreach ((CCheckTransmitInfo info, CCSPlayerController? player) in infoList)
    {
        if (player is not { IsValid: true, IsBot: false })
            continue;

        ulong playerSteamId = player.SteamID;

        foreach ((CBaseModelEntity? entity, CCSPlayerController? owner) in Instance.InspectList)
        {
            if (owner.IsValid && owner.SteamID != playerSteamId)
                info.TransmitEntities.Remove(entity);
        }

        // ... Trail过滤逻辑（类似机制）
    }
}
```

### 工作原理详解

#### 1. **OnCheckTransmit 监听器**
- 这是 CounterStrikeSharp 提供的特殊监听器
- 在服务器向每个客户端发送实体更新之前调用
- 允许插件控制哪些实体应该发送给哪些玩家

#### 2. **遍历所有玩家**
```csharp
foreach ((CCheckTransmitInfo info, CCSPlayerController? player) in infoList)
```
- 对每个正在接收实体更新的玩家进行处理
- `info` 包含将要发送给该玩家的实体列表

#### 3. **过滤预览实体**
```csharp
foreach ((CBaseModelEntity? entity, CCSPlayerController? owner) in Instance.InspectList)
{
    if (owner.IsValid && owner.SteamID != playerSteamId)
        info.TransmitEntities.Remove(entity);
}
```

**关键逻辑**:
- 遍历所有预览实体
- 检查预览实体的所有者（owner）是否是当前接收数据的玩家
- 如果 **所有者的SteamID ≠ 当前玩家的SteamID**：
  - 从传输列表中移除该实体（`Remove(entity)`）
  - 结果：该玩家的客户端不会收到这个实体的数据

#### 4. **效果**
- ✅ **预览所有者**：可以看到自己的预览模型（因为 `owner.SteamID == playerSteamId`，不会被移除）
- ❌ **其他玩家**：完全看不到该预览模型（实体数据未传输到他们的客户端）

## 技术优势

### 1. **服务器端过滤**
- 过滤在服务器端完成，客户端根本不会收到不应该看到的实体数据
- 相比客户端渲染控制更安全、更可靠

### 2. **性能优化**
- 不传输不必要的实体数据，减少网络带宽占用
- 客户端不需要处理额外的渲染逻辑

### 3. **防作弊**
- 即使客户端使用作弊工具也无法看到其他玩家的预览
- 数据在网络层面就被过滤掉了

### 4. **扩展性**
- 同样的机制也用于 Trail（拖尾效果）的过滤
- 可以轻松扩展到其他需要玩家私有显示的功能

## 预览生命周期

```
1. 玩家打开商店菜单
   ↓
2. 选择物品并点击"预览"（Inspect）
   ↓
3. 检查是否已有预览（InspectList.ContainsValue）
   ↓
4. 创建 prop_dynamic 实体
   ↓
5. 设置模型和位置（玩家面前100单位）
   ↓
6. 添加到 InspectList[entity] = player
   ↓
7. OnCheckTransmit 开始过滤
   │  └─ 只传输给所有者
   ↓
8. 启动旋转动画（4秒，360°）
   ↓
9. 动画结束后：
   - 从 InspectList 移除
   - 删除实体
```

## 清理机制

### 回合开始时清理
**位置**: [`Store/src/item/items/playerskin.cs:86-91`](Store/src/item/items/playerskin.cs:86)

```csharp
public static HookResult OnRoundStart(EventRoundStart @event, GameEventInfo info)
{
    Instance.InspectList.Clear();
    ForceModelDefault = false;
    return HookResult.Continue;
}
```

- 每个回合开始时清空所有预览
- 防止预览实体残留影响新回合

## 武器预览的差异

对于 `customweapon` 类型的预览：

**位置**: [`Store/src/item/items/customweapon.cs:405-429`](Store/src/item/items/customweapon.cs:405)

```csharp
public static void Inspect(CCSPlayerController player, string weapon)
{
    if (player.PlayerPawn.Value?.WeaponServices?.ActiveWeapon.Value is not CBasePlayerWeapon activeWeapon) 
        return;

    if (!Weapon.TryParseWeaponSpec(weapon, out string weaponBase, out string weaponSubclass))
        return;

    if (Weapon.GetDesignerName(activeWeapon) != weaponBase)
    {
        player.PrintToChatMessage("You need correct weapon", weaponBase);
        return;
    }

    Weapon.SetSubclass(activeWeapon, weaponBase, weaponSubclass);

    Instance.AddTimer(3.0f, () =>
    {
        if (player.IsValid && player.PlayerPawn.Value.WeaponServices.ActiveWeapon.Value == activeWeapon)
        {
            Weapon.ResetSubclass(activeWeapon);
        }
    });
}
```

**特点**:
- 不创建新实体，直接修改玩家当前武器的子类（subclass）
- 预览时间：3秒（而非皮肤的4秒）
- 要求玩家手持对应类型的武器
- 不需要 OnCheckTransmit 过滤（因为是玩家自己的武器）

## 总结

CS2-Store 的预览过滤机制巧妙地利用了 Source 引擎的实体传输系统：

1. **状态追踪**: 通过 `InspectList` 字典记录预览实体与所有者的映射
2. **传输控制**: 在 `OnCheckTransmit` 监听器中根据所有者身份过滤实体
3. **安全可靠**: 服务器端过滤确保数据不会发送到非所有者的客户端
4. **自动清理**: 动画结束或回合开始时自动清理预览实体

这种设计确保了预览功能的私密性、性能和安全性，是游戏服务器插件开发中实体可见性控制的优秀实践。
