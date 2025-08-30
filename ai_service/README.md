# AI NPC集成指南

## 📋 功能特性

### 🔍 **智能对话系统**
- **长期记忆**: 支持无限对话积累，无历史限制
- **自动优化**: 智能摘要触发，减少API调用
- **关系记忆**: 完整记录玩家互动历史和关系变化

### ⚡ **性能优化**
- **向量缓存**: 查询向量缓存机制，提升响应速度
- **缓存优化**: MoonPalace缓存键稳定，命中率显著提升
- **内存高效**: 4KB/查询向量，游戏运行期间持续有效

### 🎮 **使用配置**
- **记忆容量**: 每npc可配置memory_capacity条对话（默认100条）
- **摘要机制**: 自动触发，无需手动干预
- **实现位置**: 核心逻辑在`src/udp_server.py`和`src/history_manager.py`

## 系统文件说明

### 核心文件功能
- **udp_server.py** - 网络网关：接收游戏客户端请求，返回AI回复
- **npc_manager.py** - AI大脑：加载配置、生成回复、管理记忆
- **memory_store.py** - 长期记忆：存储NPC对玩家的关系数据
- **history_manager.py** - 对话记录：永久保存所有聊天记录，查询时返回最近指定数量作为上下文
- **knowledge_qwen.py** - 语义搜索：向量知识检索（千问）
- **knowledge_basic.py** - 关键词搜索：基础文本匹配回退方案

### 数据流向
游戏客户端 → udp_server.py → npc_manager.py → [记忆+历史+知识库] → 大模型 → 回复

## 快速创建新的AI NPC

### 1. 配置NPC角色
编辑 `ai_service/config/npc_roles.json`，添加新NPC配置：

```json
{
  "your_npc_id": {
    "name": "NPC显示名称",
    "title": "NPC称号",
    "role": "角色类型",
    "personality": "性格描述",
    "background": "背景故事",
    "greeting": "初次见面问候语",
    "topics": ["话题1", "话题2", "话题3"],
    "speech_style": "说话风格",
    "knowledge_base": ["知识领域1", "知识领域2"],
    "knowledge_threshold": 0.4,  // 知识库匹配阈值，0-1之间，0=匹配所有，1=禁用匹配，0.5-0.7为推荐范围
    "memory_capacity": 100,  // 记忆容量：0=禁用历史记忆，1-9=自动调整为10，≥10=使用指定值，推荐100-200
    "relationship_tips": {
      "gifts": ["门派秘籍", "稀有装备", "银两"],  // 赠送礼物提升关系
      "topics": ["各门派特色", "武功搭配技巧"],  // 喜欢的话题
      "taboos": ["询问外挂", "索要管理员权限"]  // 禁忌话题
    }
  }
}
```

### 2. 创建NPC文件
复制模板文件并修改配置：

```bash
cp u/mudren/npc/ai_npc_template.c clone/npc/你的npc名.c
```

### 3. 修改NPC配置
在你的NPC文件中修改以下内容：

```lpc
// 修改这些配置项
set_name("张三", ({"zhang san", "zhangsan", "zhang"}));
set("title", "江湖游侠");
set("long", "这里写NPC描述...");
set("ai_npc_id", "zhang_san");  // 必须与json中的键匹配
set("ai_topics", ({"武功", "江湖", "美食"}));
```

### 4. 放置NPC
将NPC放置到游戏世界中，在房间文件中添加：

```lpc
// 方法一：在房间create函数中设置
void create() {
    ::create();
    set("short", "房间名称");
    set("long", "房间描述...");
    set("objects", ([
        "/clone/npc/你的npc名" : 1,     // 单个NPC
        "/clone/npc/另一个npc" : 2,     // 多个相同NPC
    ]));
    setup();
}

// 方法二：管理员动态添加
// 管理员可以使用：clone /clone/npc/你的npc名
```

## 改造现有NPC为AI

### 1. 添加AI配置
在现有NPC的create()函数中添加：
```lpc
set("ai_npc_id", "zhou butong");  // 对应json中的键名
```

### 2. 添加AI对话入口
在现有NPC中添加：
```lpc
int accept_talk(object me, string topic) {
    string player_id = me->query("id");
    string player_name = me->name();

    mapping context = ([
        "time": NATURE_D->game_time(),
        "location": environment(this_object())->query("short"),
        "weather": NATURE_D->outdoor_room_description()
    ]);

    if (!topic || topic == "") {
        topic = "你好";
    }

    AI_CLIENT_D->send_chat_request(
        query("ai_npc_id"),
        player_id,
        player_name,
        topic,
        context
    );

    return 1;
}
```

## 配置示例

编辑文件：`ai_service/config/npc_roles.json`

### 1. 江湖侠客
```json
{
  "li bai": {
    "name": "李白",
    "title": "诗剑双绝",
    "personality": "豪爽仗义，嫉恶如仇",
    "background": "行走江湖二十载，剑下斩尽不平事",
    "topics": ["武功", "江湖", "正义", "剑客"],
    "memory_capacity": 150  // 记忆容量：0=禁用历史记忆，1-9=自动调整为10，≥10=使用指定值
  }
}
```

### 2. 商人NPC
```json
{
  "wang zhanggui": {
    "name": "王掌柜",
    "title": "商会会长",
    "personality": "精明能干，善于经商",
    "background": "经营百年老店，见多识广",
    "topics": ["生意", "商品", "行情", "各地特产"],
    "memory_capacity": 80  // 商人NPC记忆较短，主要关注当前交易
  }
}
```

## 相关文档
- **AI_CLIENT_D**：`/adm/daemons/ai_client_d.c` - 游戏内AI客户端守护程序，处理NPC与Python服务端的通信
- **talk指令**：`/cmds/std/talk.c` - 玩家talk命令实现，用于与AI NPC对话

## 测试命令

游戏内使用以下命令测试：
```
talk npc_id 测试消息
```

## 注意事项
1. **ai_npc_id** 必须与json配置中的键完全一致
2. 重启AI服务后配置生效


## 服务端使用

### 1. 配置环境变量
```bash
cd ai_service
echo "OPENAI_API_KEY=你的月之暗面API密钥" > .env
echo "OPENAI_BASE_URL=https://api.moonshot.cn/v1" >> .env
echo "OPENAI_MODEL=moonshot-v1-auto" >> .env
echo "DASHSCOPE_API_KEY=你的千问API密钥" >> .env
```

### 2. 初始化知识库
```bash
# 基础关键词搜索
python scripts/setup_basic.py

# 千问向量搜索（推荐）
python scripts/setup_qwen.py
```

### 3. 启动服务端
```bash
python main.py          # 正常模式
python main.py -d       # 调试模式（显示详细日志）
```
