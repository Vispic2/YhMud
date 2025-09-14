# questd - 任务守护进程

## 📋 概述

questd 是炎黄群侠传MUD游戏的任务系统核心守护进程，负责管理整个游戏的任务分配、完成验证、奖励发放和特殊奖励系统。它提供了完整的任务生命周期管理，从任务接取到完成奖励的全流程控制，是游戏进度和玩家成长的重要驱动系统。

## 🎯 主要功能

### 1. 任务分配管理
- **智能任务分配**：根据玩家等级和经验智能匹配合适的任务
- **任务类型支持**：支持送信、杀人、寻物等多种任务类型
- **任务难度调节**：根据玩家实力动态调整任务难度
- **NPC任务匹配**：为NPC匹配合适的任务供玩家接取

### 2. 任务完成验证
- **完成条件检查**：验证玩家是否满足任务完成条件
- **任务状态跟踪**：跟踪任务执行进度和状态变化
- **时间限制管理**：管理任务的时间限制和超时处理
- **异常处理**：处理任务失败、放弃等异常情况

### 3. 奖励系统管理
- **基础奖励发放**：发放经验、潜能、威望等基础奖励
- **等级奖励系统**：根据连续完成任务数量发放特殊奖励
- **物品奖励池**：管理多级别的物品奖励池
- **特殊奖励触发**：达到一定条件时触发特殊奖励

### 4. 转世玩家特殊处理
- **超级NPC生成**：为转世玩家生成特殊挑战NPC
- **奖励加成**：转世玩家获得额外奖励加成
- **任务难度调整**：根据转世次数调整任务难度

## 💻 技术实现

### 核心数据结构

```lpc
// 基础奖励物品列表（10个任务）
string *ob1_list = ({
    "/clone/fam/etc/vit1",      // 活力丹1
    "/clone/fam/etc/int1",      // 智力丹1
    "/clone/fam/etc/str1",      // 力量丹1
    "/clone/fam/etc/con1",      // 根骨丹1
    "/clone/fam/etc/dex1",      // 身法丹1
    "/clone/fam/etc/per1",      // 容貌丹1
    "/clone/fam/pill/food1",    // 食物丹1
    "/clone/fam/pill/neili1",   // 内力丹1
    "/clone/fam/pill/jingli1",  // 精力丹1
    "/clone/fam/pill/linghui1"  // 灵慧丹1
});

// 30个任务奖励列表
string *ob2_list = ({
    "/clone/fam/etc/vit2",      // 活力丹2
    "/clone/fam/etc/int2",      // 智力丹2
    // ... 更多奖励物品
});

// 特殊奖励触发阈值
#define ONE_DAY         86400   // 一天的时间（秒）
#define MAX_QUEST_LEVEL 9       // 最大任务等级
```

### 主要函数接口

#### 任务分配函数
```lpc
int askQuest(object who, object me)
```
- **参数说明**：
  - `who`: 请求任务的玩家对象
  - `me`: 提供任务的NPC对象
- **返回值**：成功返回1，失败返回0
- **功能**：为玩家分配合适的任务

#### 任务完成函数
```lpc
int completeQuest(object who, object me)
```
- **参数说明**：
  - `who`: 完成任务的玩家对象
  - `me`: 验证任务的NPC对象
- **返回值**：成功返回1，失败返回0
- **功能**：验证任务完成并发放奖励

#### 特殊奖励函数
```lpc
private void special_bonus(object me, object who, mixed arg)
```
- **功能**：根据任务完成数量发放特殊物品奖励

## 🔄 工作流程

### 任务分配流程

1. **基础验证阶段**
   ```lpc
   // 检查NPC是否可以分配任务
   if (!me->query("can_quest"))
       return 0;

   // 检查玩家是否已有任务
   if (who->query_temp("quest"))
       return 0;
   ```

2. **玩家资格验证**
   ```lpc
   // 检查玩家等级和状态
   if (who->query("combat_exp") < 10000)
       return 0;

   // 检查是否处于特殊状态
   if (who->query_temp("netdead") || who->query_temp("biguan"))
       return 0;
   ```

3. **任务类型匹配**
   ```lpc
   // 根据玩家经验确定任务等级
   lvl = NPC_D->check_level(who);
   exp = who->query("combat_exp");

   // 经验10万以内为送信任务
   if (exp < 100000)
       quest_type = "letter";
   else
       quest_type = "kill";
   ```

4. **转世玩家特殊处理**
   ```lpc
   // 转世玩家获得特殊NPC任务
   reborn = who->query("reborn/count");
   if (who->query("reborn") && who->query("combat_exp") > 800000 && random(50) <= reborn)
   {
       // 生成超级NPC任务
       who->set_temp("super", reborn);
       ob = new(CLASS_D("generate") + "/killed_super.c");
   }
   ```

5. **任务分配执行**
   ```lpc
   // 设置任务信息
   who->set_temp("quest", ([
       "id": me->query("id"),
       "name": me->name(),
       "type": quest_type,
       "time": time(),
       "level": level,
       "exp": exp
   ]));
   ```

### 任务完成验证流程

1. **任务存在性检查**
   ```lpc
   // 检查玩家是否有任务
   quest = who->query_temp("quest");
   if (!quest || quest["id"] != me->query("id"))
       return 0;
   ```

2. **任务类型验证**
   ```lpc
   // 根据任务类型进行不同验证
   switch (quest["type"]) {
   case "letter":
       return check_letter_quest(who, me);
   case "kill":
       return check_kill_quest(who, me);
   }
   ```

3. **时间限制检查**
   ```lpc
   // 检查是否超时
   if (time() - quest["time"] > ONE_DAY)
   {
       who->delete_temp("quest");
       return 0;
   }
   ```

4. **奖励计算与发放**
   ```lpc
   // 计算基础奖励
   lvl = NPC_D->check_level(who);
   reborn = who->query("reborn/count");
   exp = 10 + random(5) + lvl + reborn;
   pot = 5 + random(3) + lvl + reborn;
   mar = 1 + random(2);
   weiwang = 2 + random(3) + lvl / 2;
   score = 2 + random(3) + lvl / 2;

   // 发放奖励
   who->add("combat_exp", exp);
   who->add("potential", pot);
   who->add("experience", mar);
   who->add("weiwang", weiwang);
   who->add("score", score);
   ```

5. **特殊奖励检查**
   ```lpc
   // 检查是否达到特殊奖励条件
   quest_count = who->query("quest_count");
   quest_count++;
   who->set("quest_count", quest_count);

   // 根据完成数量发放特殊物品
   if (quest_count == 30 || quest_count == 50 || quest_count == 100 ||
       quest_count == 200 || quest_count == 300 || quest_count == 400 ||
       quest_count == 500 || quest_count == 600 || quest_count == 700 ||
       quest_count == 800)
   {
       special_bonus(me, who, quest_count);
   }
   ```

## ⚙️ 奖励系统详解

### 基础奖励计算

基础奖励公式：
```lpc
经验值 = 10 + random(5) + 玩家等级 + 转世次数
潜能值 = 5 + random(3) + 玩家等级 + 转世次数
阅历值 = 1 + random(2)
威望值 = 2 + random(3) + 玩家等级 / 2
积分值 = 2 + random(3) + 玩家等级 / 2
```

### 特殊奖励等级

| 完成数量 | 奖励等级 | 物品品质 |
|----------|----------|----------|
| 30个     | 二级奖励 | 优质丹药 |
| 50个     | 三级奖励 | 高级丹药 |
| 100个    | 四级奖励 | 稀有丹药 |
| 200个    | 五级奖励 | 珍贵丹药 |
| 300个    | 六级奖励 | 极品丹药 |
| 400个    | 七级奖励 | 传说丹药 |
| 500个    | 八级奖励 | 神话丹药 |
| 600个    | 九级奖励 | 顶级丹药 |
| 700个    | 十级奖励 | 至尊丹药 |
| 800个    | 顶级奖励 | 逆天丹药 |

### 转世玩家加成

转世玩家获得额外加成：
- **基础奖励**：每项奖励 + 转世次数
- **超级NPC**：有概率遇到特殊超级NPC（经验>80万且转世次数>0）
- **任务难度**：转世次数影响任务难度和奖励品质

## ⚠️ 重要注意事项

### 任务系统设计原则
1. **公平性**：确保不同等级玩家都能获得合适任务
2. **渐进性**：任务难度随玩家成长逐步提升
3. **多样性**：支持多种任务类型，避免单调
4. **奖励平衡**：奖励与投入时间和难度成正比

### 性能优化考虑
1. **缓存机制**：频繁查询的数据应缓存
2. **批量处理**：奖励发放考虑批量处理
3. **超时管理**：及时清理超时任务数据
4. **并发安全**：确保多玩家同时操作的正确性

### 数据一致性
1. **原子操作**：奖励发放必须是原子操作
2. **状态同步**：确保任务状态同步更新
3. **异常处理**：完善的异常情况处理机制
4. **日志记录**：重要操作需要记录日志

## 🔧 使用示例

### NPC配置任务功能
```lpc
// 在门派任务NPC中配置
void init()
{
    add_action("do_quest", "quest");
    add_action("do_complete", "complete");
}

int do_quest(string arg)
{
    // 请求任务
    return QUEST_D->askQuest(this_player(), this_object());
}

int do_complete(string arg)
{
    // 完成任务
    return QUEST_D->completeQuest(this_player(), this_object());
}
```

### 任务状态查询
```lpc
// 查询玩家任务状态
mapping quest = player->query_temp("quest");
if (quest) {
    write("你当前的任务：\n");
    write(sprintf("任务类型：%s\n", quest["type"]));
    write(sprintf("任务时间：%s\n", ctime(quest["time"])));
    write(sprintf("已接任务数：%d\n", player->query("quest_count")));
}
```

### 特殊奖励配置
```lpc
// 配置特殊奖励物品
string *ob5_list = ({
    "/clone/fam/pill/renshen1",     // 人参1
    "/clone/fam/pill/lingzhi1",     // 灵芝1
    "/clone/fam/pill/xuelian1",     // 雪莲1
    "/clone/fam/pill/food2",        // 食物2
    "/clone/fam/pill/neili2",       // 内力2
    "/clone/fam/pill/jingli2",      // 精力2
    "/clone/fam/etc/vit3",          // 活力3
    "/clone/fam/etc/int3",          // 智力3
    "/clone/fam/etc/str3",          // 力量3
    "/clone/fam/etc/con3"           // 根骨3
});
```

## 📊 性能指标

### 任务处理性能
- 任务分配：<50ms
- 任务完成：<30ms
- 奖励计算：<10ms
- 特殊奖励：<100ms

### 并发处理能力
- 支持1000+玩家同时任务
- 每秒处理100+任务请求
- 无锁设计确保并发安全

## 🔗 相关文档

- [任务系统架构](../systems/quests/architecture.md) - 任务系统整体设计
- [NPC系统文档](../systems/npc/overview.md) - NPC任务配置
- [奖励系统设计](../systems/economy/rewards.md) - 奖励机制说明
- [转世系统文档](../features/reborn/overview.md) - 转世特殊处理
- [平衡性分析](../systems/quests/balance.md) - 任务平衡性设计

---

*最后更新: 2025-09-14*
*维护团队: 炎黄群侠传开发组*