# CLAUDE.md — NEXUS 项目指南

## 项目概述

**NEXUS** — 单文件中文AI驱动的多世界穿书文字游戏。
文件：`nexus.html`（单文件，无框架，无构建工具，无依赖）。
AI充当主神/世界主持人：叙述环境，扮演所有NPC和穿越者，裁定规则，推进剧情。

---

## 架构：页面系统

SPA with **4 bottom-tab pages**:

| Tab | ID | Purpose |
|-----|-----|-----|
| **Story** | `page-story` | 主叙事界面：对话气泡、副本信息栏、AI消息流 |
| **空间** | `page-space` | 主神空间：积分商店 / 穿越者通讯录 / 副本记录 (3 sub-tabs) |
| **状态** | `page-status` | 玩家信息、当前副本、任务列表、道具 |
| **设置** | `page-settings` | API配置、主题切换、数据导入导出、重置、角色创建 |

### 消息类型

| Type | CSS Class | Use |
|------|-----------|-----|
| `narrative` | `.msg-narrative` | 旁白/环境叙述，左侧accent色竖线 |
| `npc` | `.msg-npc-row` / `.msg-npc-bubble` | 原住民对话，冷色玻璃气泡 |
| `traveler` | `.msg-traveler-row` / `.msg-traveler-bubble` | 穿越者对话，teal色调玻璃气泡 |
| `player` | `.msg-user-row` / `.msg-user-bubble` | 玩家对话，右侧accent色气泡 |
| `system` | `.msg-system` | 系统消息，居中灰色小字 |

---

## 数据层：`Store` 对象

**localStorage key**: `'NEXUS_V1'`

### 核心方法

| Method | Purpose |
|--------|---------|
| `Store.init()` | 加载或初始化_db |
| `Store.save()` | 写入localStorage |
| `Store.player()` | 返回玩家数据 |
| `Store.messages()` | 返回消息列表 |
| `Store.addMessage(msg)` | 添加消息（自动ID） |
| `Store.travelers()` | 返回穿越者列表 |
| `Store.startRun(runData)` | 开始新副本 |
| `Store.endRun(result, success)` | 结算副本到历史 |
| `Store.getOrCreateChat(travelerId)` | 获取/创建私聊 |
| `Store.hasReviveTicket()` | 检查复活券 |
| `Store.useReviveTicket()` | 消耗复活券 |
| `Store._persistImportantNPCs(run)` | 持久化重要NPC数据 |
| `Store._getImportantNPCData(name)` | 获取跨副本NPC数据 |
| `Store._extractMemories(npc, result)` | 提取副本记忆(3-5条) |
| `Store._compressMemories(npc)` | 压缩最早记忆(上限10条) |

### _db 结构

```
_db {
  version: 1,
  player: { name, gender, appearance, background, personality, points, level, completedRuns, inventory[], companions[] },
  currentRun: { id, type, title, worldDesc, playerRole, mainTask, sideTasks[], timeLimit, difficulty, baseReward, npcs[], companions[], currentLocation, currentTime, ... },
  runHistory: [Run],
  travelers: [{ id, name, gender, appearance, background, personality, points, level, online, contactUnlocked, isImportant, affection, affectionRate, longTermMemories[] }],
  messages: [{ id, type, content, characterName?, characterId?, timestamp, meta }],
  privateChats: { [travelerId]: { messages[], hasUnread } },
  importantNPCs: { [npcName]: { name, type, desc, affection, affectionRate, isImportant, longTermMemories[] } },
  shop: { purchased[] },
  saveSlots: [],
  settings: { theme, apiKey, model, baseUrl }
}
```

---

## 角色重要性分级系统

### 两级分类

| 等级 | `isImportant` | 好感度 | 记忆 | 主动消息 |
|------|--------------|--------|------|---------|
| **普通角色** | `false` (默认) | 仅副本内有效，出本清零 | 无 | 无 |
| **重要角色** | `true` (手动标记) | 跨副本持久化，独立增幅速度 | 跨副本长期记忆(≤10条) | 副本前/中/后主动联系 |

### 好感度系统

- 默认值：0（中立），范围 -100 ~ +100
- 增幅速度 `affectionRate`：
  - `'slow'` — 0.5x 乘数
  - `'normal'` — 1x 乘数（默认）
  - `'fast'` — 2x 乘数
- AI根据角色人设在 `[AFF]` 标记中给出原始Δ值，程序自动乘以速率
- 玩家可在角色卡中手动设置速率

### 记忆系统（仅重要角色）

- **短期**：对话上下文最近20轮（由AI._buildMessages自动管理）
- **长期**：每次副本结束时 `Store._extractMemories()` 自动提取3-5条：
  - 好感度≥60 → 5条；≥30 → 4条；默认3条
  - 格式：`{ time, event, emotionTag }`（emotionTag: positive/negative/neutral）
- **上限**：10条，超出时 `Store._compressMemories()` 保留最新5条+压缩摘要
- **注入**：下次[NPC]标记出场时，`Store._getImportantNPCData()` 恢复好感度+记忆
- **系统提示词注入**：`buildMainSystemPrompt()` 的[重要角色系统]段注入最近3条记忆

### 主动消息（仅重要角色）

通过 `[MSG name:content]` 标记触发，AI在适当时机使用：

| 时机 | 触发条件 | 内容示例 |
|------|---------|---------|
| 副本开始前 | 重要角色好感度≥30 | 情报、警告、世界背景提示 |
| 副本中途 | 好感度≥40 + 剧情关键点 | 关心、线索、远程建议 |
| 副本结束后 | 好感度≥50 + 副本成功 | 祝贺、邀请见面 |

程序侧：副本结束时自动检查重要角色好感度≥50，触发祝贺消息。

---

## AI 模块

### System Prompts

- **`buildMainSystemPrompt()`** — 主叙事提示词，包含全局规则、世界观、叙述风格、13种标记参考、NPC扮演细则、重要角色系统（记忆注入+主动消息规则）、玩家/副本状态
- **`buildTravelerChatPrompt(traveler)`** — 穿越者私聊提示词，包含身份定义、角色卡、性格演绎指南、对方副本状态、聊天规范

### Markers (13 types)

| Marker | Purpose |
|--------|---------|
| `[POINTS ±X]` | 积分变化 |
| `[AFF name:±X]` | 好感度变化（自动×affectionRate） |
| `[ITEM +name:desc]` / `[ITEM -name]` | 道具获得/失去 |
| `[TASK_UPDATE name:progress]` | 任务进度更新 |
| `[TASK_COMPLETE name]` | 任务完成 |
| `[RUN_END result:reason]` | 副本结束 |
| `[NPC name:type:desc]` | 原住民登场（自动恢复重要NPC跨副本数据） |
| `[TRAVELER name:desc]` | 穿越者出现 |
| `[MSG name:content]` | 重要角色主动消息/穿越者私信 |
| `[LOCATION name]` | 场景切换 |
| `[TIME value]` | 时间推进 |

### API

- `AI.call(messages, options)` — 非流式OpenAI格式调用，30s超时，返回`{ok,text}/{error,msg}`
- `AI.streamCall(messages, onChunk, onDone, onError, options)` — SSE流式调用，60s超时
- `AI.testConnection()` — 测试API连通性
- `AI.parseCommands(text)` — 正则提取13种标记→按位置排序返回
- `AI.applyCommands(cmds)` — 执行标记到Store（含好感度速率乘数+重要NPC数据恢复）
- `AI.parseResponse(text)` — 清洗标记返回纯文本
- `AI._buildMessages(systemPrompt, options)` — 上下文窗口组装

### 设置

支持自定义 Base URL / API Key / Model，兼容 OpenAI、Deepseek、Claude 等兼容接口。

---

## CSS 架构

- **Dark theme (default)**: `--bg:#06060B`, accent teal `#00A8A8`
- **Light theme**: `body.light` 覆盖CSS变量
- **Glass morphism**: `backdrop-filter: blur()` 用于dock、气泡、信息栏
- **Mobile-first**: max-width 430px, safe-area-inset
- **Font stack**: serif (Songti) 用于旁白, mono (SF Mono) 用于界面, UI (SF Pro) 用于正文

---

## 关键设计决策

1. **单文件**: 无构建步骤，即时迭代
2. **Store 单数据源**: 所有变更通过Store.save()持久化
3. **双层穿越者**: 主空间联系人列表 + 副本内同行穿越者
4. **副本数据独立**: currentRun 独立于 player，结算后归档 runHistory
5. **标记系统**: 参考VOID的parseCommands，13种正则标记，AI在叙述中嵌入
6. **双提示词系统**: 主叙事和穿越者私聊使用独立System Prompt
7. **角色重要性分级**: 普通角色好感度出本清零；重要角色好感度+记忆跨副本持久化，附主动消息+记忆提取
8. **复活券机制**: 副本死亡时自动检查inventory，消耗复活券→复活叙事→继续
