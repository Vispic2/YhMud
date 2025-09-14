# masterd - 门派师父守护进程

## 📋 概述

masterd 是炎黄群侠传MUD游戏中专门处理门派师父功能的守护进程，负责管理NPC师父向玩家传授武功和发放物品的核心逻辑。基于实际代码实现，确保门派技能的合理传授和门派资源的分配。

## 🎯 主要功能

### 1. 武功传授管理
- **门派武功传授** - 处理门派师父向弟子传授武功的完整流程
- **绝招技能授予** - 管理武功绝招的传授条件和执行
- **技能等级验证** - 验证弟子是否满足学习各项武功的前提条件
- **基础技能提升** - 在传授主技能的同时提升相关基础技能等级

### 2. 物品发放管理
- **门派物品分配** - 处理师父向弟子发放门派特有物品
- **奖励物品管理** - 管理完成特定条件后的物品奖励
- **门派资源控制** - 确保门派资源的合理分配和使用

### 3. 门派权限控制
- **门派关系验证** - 严格验证传授者和学习者的门派归属
- **贡献值管理** - 管理和验证门派贡献值要求
- **权限等级控制** - 根据门派地位控制可传授的武功等级

### 4. 学习条件验证
- **多维度条件检查** - 综合检查神值、贡献值、技能等级、内力修为等
- **前置技能验证** - 确保弟子具备学习所需的前置技能
- **修为要求验证** - 验证内力和精力修为是否达标

## 💻 技术实现

### 核心数据结构

```lpc
// 支持传授的基础技能类型（可激发技能）
string *valid_types = ({
    "force", "dodge", "unarmed", "cuff", "strike", "hand", "finger",
    "claw", "sword", "blade", "staff", "hammer", "club", "whip",
    "dagger", "throwing", "parry", "magic", "medical", "poison",
    "chuixiao-jifa", "tanqin-jifa", "guzheng-jifa", "cooking"
});
```

### 主要函数接口

#### 武功传授函数
```lpc
public mixed teach_pfm(object who, object ob, mapping b)
```
- **参数**:
  - `who`: 学习者（弟子/玩家对象）
  - `ob`: 传授者（师父/NPC对象）
  - `b`: 传授配置映射，包含所有条件和要求
- **返回值**: 成功返回1，失败返回错误信息字符串
- **功能**: 执行完整的武功传授逻辑，包括条件验证和技能授予

#### 物品发放函数
```lpc
public mixed give_item(object who, object ob, mapping b)
```
- **参数**: 同上
- **功能**: 执行物品发放逻辑，验证获得条件

## 🔄 工作流程

### 武功传授完整流程

#### 1. 门派关系验证阶段
```lpc
// 检查是否为同门派（除非设置为free）
if (!b["free"] && who->query("family/family_name") != ob->query("family/family_name"))
    return "你和" + ob->name() + "并非同门，看起来" + ob->name() + "并不打算传授什么给你。";
```

#### 2. 侠义正气值检查
```lpc
// 验证侠义正气值是否达标
if (b["shen"] && who->query("shen") < b["shen"])
    return "你的侠义正气值太低，恐怕难以领悟其中精要。";
```

#### 3. 门派贡献值验证
```lpc
// 检查门派贡献值是否足够
if (b["gongxian"] && who->query("gongxian") < b["gongxian"])
    return "你在门派中的贡献太低，恐怕难以领悟其中精要。";
```

#### 4. 前置技能条件验证
```lpc
// 验证最多5项前置技能及其等级
for (i = 1; i <= 5; i++) {
    if (b["sk" + i] && who->query_skill(b["sk" + i], 1) < b["lv" + i])
        return "你的" + to_chinese(b["sk" + i]) + "火候不够，恐怕难以领悟其中精要。";
}
```

#### 5. 内功轻功要求检查
```lpc
// 检查基础内功和轻功等级
if (b["force"] && who->query_skill("force", 1) < b["force"])
    return "你的内功修为太差，恐怕难以领悟其中精要。";

if (b["dodge"] && who->query_skill("dodge", 1) < b["dodge"])
    return "你的轻功修为太差，恐怕难以领悟其中精要。";
```

#### 6. 内力精力修为验证
```lpc
// 验证内力上限是否达标
if (b["neili"] && who->query("max_neili") < b["neili"])
    return "你的内力修为太差，恐怕难以领悟其中精要。";

// 验证精力上限是否达标
if (b["jingli"] && who->query("max_jingli") < b["jingli"])
    return "你的精力修为太差，恐怕难以领悟其中精要。";
```

#### 7. 武功传授执行
```lpc
// 计算技能提升等级
lvl = who->query_skill(name, 1) / 2 + 1;
if (lvl > 200) lvl = 200;

// 授予主要武功技能
who->set_skill(name, lvl);

// 提升相关基础技能（如果是可激发类型）
for (i = 1; i <= 5; i++) {
    if (b["sk" + i] && member_array(b["sk" + i], valid_types) != -1)
        who->set_skill(b["sk" + i], who->query_skill(b["sk" + i], 1) + lvl/2);
}
```

#### 8. 门派贡献值扣除
```lpc
// 扣除门派贡献值（如果配置了要求）
if (b["gongxian"])
    who->add("gongxian", -b["gongxian"]);
```

## ⚙️ 配置参数详解

### 武功传授配置参数

| 参数名 | 类型 | 必须 | 说明 | 示例 |
|--------|------|------|------|------|
| `name` | string | 是 | 武功技能名称 | "sword", "force", "dodge" |
| `perform` | string | 否 | 绝招名称（预留） | "pi", "qiao" |
| `free` | int | 否 | 是否无视门派限制（1=是） | 1 |
| `shen` | int | 否 | 侠义正气值最低要求 | 10000 |
| `gongxian` | int | 否 | 门派贡献值要求 | 500 |
| `sk1-sk5` | string | 否 | 前置技能1-5名称 | "force", "sword" |
| `lv1-lv5` | int | 否 | 前置技能1-5最低等级 | 100, 120 |
| `force` | int | 否 | 基础内功最低等级 | 120 |
| `dodge` | int | 否 | 基础轻功最低等级 | 120 |
| `neili` | int | 否 | 内力上限最低要求 | 1000 |
| `jingli` | int | 否 | 精力上限最低要求 | 500 |

### 消息配置参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `msg1` | string | 传授失败时显示的消息 |
| `msg2` | string | 传授成功时显示的消息 |
| `temp1-temp3` | string | 临时消息模板变量 |
| `tmsg1-tmsg3` | string | 临时消息内容 |

## ⚠️ 开发注意事项

### 安全检查机制
1. **门派严格验证** - 默认情况下必须同门派才能传授（除非设置free=1）
2. **多条件综合验证** - 所有条件必须同时满足，缺一不可
3. **技能等级上限控制** - 传授后的技能等级最高不超过200级
4. **贡献值原子操作** - 贡献值扣除必须确保足够余额

### 性能考虑
1. **循环效率** - 技能验证循环最多5次，避免过度计算
2. **数组查找优化** - 使用member_array进行技能类型判断
3. **错误提前返回** - 任何条件不满足立即返回，避免无效计算

### 扩展性考虑
1. **配置驱动** - 所有条件都通过配置参数控制，便于调整
2. **技能类型可扩展** - valid_types数组易于维护和扩展
3. **消息模板化** - 支持自定义成功失败消息

## 🔧 实际使用示例

### 掌门传授高级武功
```lpc
// 在门派掌门NPC中配置
void init()
{
    add_action("do_skills", "skills");
}

int do_skills(string arg)
{
    mapping teach;

    // 配置传授本门高级剑法的要求
    teach = ([
        "name": "sword",           // 传授剑法技能
        "shen": 50000,             // 需要5万侠义正气值
        "gongxian": 2000,          // 需要2000门派贡献
        "sk1": "force",            // 前置技能1：内功
        "lv1": 150,                // 内功需要150级
        "sk2": "sword",            // 前置技能2：剑法
        "lv2": 120,                // 基础剑法需要120级
        "sk3": "dodge",            // 前置技能3：轻功
        "lv3": 120,                // 轻功需要120级
        "force": 150,              // 基础内功需要150级
        "neili": 5000,             // 内力上限需要5000
        "jingli": 2000,            // 精力上限需要2000
        "msg1": "你的修为还差得远，回去再练练吧。",
        "msg2": "很好！本门绝学今日传于你，望你好自为之！"
    ]);

    return MASTER_D->teach_pfm(this_player(), this_object(), teach);
}
```

### 师父发放门派物品
```lpc
// 在门派师父NPC中配置
int do_gift(string arg)
{
    mapping gift;

    // 配置发放门派服饰的要求
    gift = ([
        "shen": 10000,             // 需要1万侠义正气值
        "gongxian": 500,           // 需要500门派贡献
        "sk1": "force",            // 前置技能：内功
        "lv1": 80,                 // 内功需要80级
        "force": 80,               // 基础内功需要80级
        "msg1": "你对本门的贡献还不够，继续努力吧。",
        "msg2": "这是本门特制服饰，你拿去吧。"
    ]);

    return MASTER_D->give_item(this_player(), this_object(), gift);
}
```

---
*基于 adm/daemons/masterd.c 实际代码实现*
*维护团队：炎黄群侠传开发组*"file_path"："C:\Users\oiuv\home\mud\docs\daemons\masterd.md"