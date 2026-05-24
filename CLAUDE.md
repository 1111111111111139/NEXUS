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
| **设置** | `page-settings` | API配置、主题切换、数据导入导出、重置 |

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
| `Store.endRun(result)` | 结算副本到历史 |
| `Store.getOrCreateChat(travelerId)` | 获取/创建私聊 |

### _db 结构

```
_db {
  version: 1,
  player: { name, gender, appearance, background, personality, points, level, completedRuns, inventory[], companions[] },
  currentRun: { id, type, title, worldDesc, playerRole, mainTask, sideTasks[], timeLimit, difficulty, baseReward, npcs[], companions[], currentLocation, currentTime, ... },
  runHistory: [Run],
  travelers: [{ id, name, gender, appearance, background, personality, points, level, online, contactUnlocked }],
  messages: [{ id, type, content, characterName?, characterId?, timestamp, meta }],
  privateChats: { [travelerId]: { messages[], hasUnread } },
  shop: { purchased[] },
  settings: { theme, apiKey, model, baseUrl }
}
```

---

## AI 模块

### System Prompts

- **`buildMainSystemPrompt()`** — 主叙事提示词，包含全局规则、世界观、叙述风格、标记参考、玩家状态、当前副本信息
- **`buildTravelerChatPrompt(traveler)`** — 穿越者私聊提示词，包含角色设定、聊天规则、独立人格

### Markers (13 types)

| Marker | Purpose |
|--------|---------|
| `[POINTS ±X]` | 积分变化 |
| `[AFF name:±X]` | 好感度变化 |
| `[ITEM +name:desc]` / `[ITEM -name]` | 道具获得/失去 |
| `[TASK_UPDATE name:progress]` | 任务进度更新 |
| `[TASK_COMPLETE name]` | 任务完成 |
| `[RUN_END result:reason]` | 副本结束 |
| `[NPC name:type:desc]` | 原住民登场 |
| `[TRAVELER name:desc]` | 穿越者出现 |
| `[MSG name:content]` | 穿越者私信 |
| `[LOCATION name]` | 场景切换 |
| `[TIME value]` | 时间推进 |

### API

- `AI.call(messages, options)` — 非流式OpenAI格式调用
- `AI.streamCall(messages, onChunk, onDone, onError, options)` — SSE流式调用
- `AI.testConnection()` — 测试API连通性
- `AI.parseCommands(text)` — 正则提取标记 → 排序返回
- `AI.applyCommands(cmds)` — 执行标记到Store
- `AI.parseResponse(text)` — 清洗标记返回纯文本

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
