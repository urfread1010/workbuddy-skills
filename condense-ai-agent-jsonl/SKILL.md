---
name: condense-ai-agent-jsonl
description: 把 AI 编码 agent（Codex rollout、WorkBuddy 等）的原始会话 jsonl 洗成人类可读的「USER 问 / AI 答」纯文本对话流，每条带原始行号便于回溯。适用于人工标注、人工检视聊天记录的场景。当用户抱怨"jsonl 太长/太多字段看不到聊天内容"时使用。
agent_created: true
---

# 洗 AI agent 会话 jsonl 为可读对话

## 何时用
- 原始 rollout / 会话 jsonl 里 90% 是噪声字段（`token_count`、`reasoning`、`function_call`、`session_meta`、`turn_context`），人眼无法直接阅读
- 用户需要人工标注、人工回溯某段对话，或把聊天记录交给下游工具/人阅读
- 明确目标：只留「谁说了什么」+ 可回原文件定位的行号

## 现成脚本（优先复用）
`C:\Users\31310\Documents\注意力监控系统\ai_sessions\scripts\condense_codex_jsonl.py`

```bash
PY="C:/Users/31310/AppData/Local/Programs/Python/Python312/python.exe"

# 洗整个 codex sessions 目录（默认 src/out 已写好）
"$PY" scripts/condense_codex_jsonl.py

# 只洗前 3 个（试跑）
"$PY" scripts/condense_codex_jsonl.py --limit 3

# 洗指定文件
"$PY" scripts/condense_codex_jsonl.py --files A.jsonl B.jsonl

# 自定义源/输出目录
"$PY" scripts/condense_codex_jsonl.py --src <目录> --out <目录>
```

默认：src=`C:\Users\31310\.codex\sessions`，out=`ai_sessions\out\condensed`（镜像 `YYYY/MM/DD/*.md` 日期结构 + `_index.md` 索引表）。

## Codex rollout jsonl 的格式映射（关键知识）

**保留**（4 类真实对话）：
| 来源 | 路径 | 角色 |
|---|---|---|
| `event_msg/item_completed` | `payload.item.type=="UserMessage"` → `item.content[].text` | USER |
| `event_msg/item_completed` | `payload.item.type=="AgentMessage"` → `item.content[].text` | AI |
| `event_msg/task_complete` | `payload.last_agent_message` | AI |
| `response_item/message` | `payload.role in (user, assistant)` → `content[].input_text/output_text` | USER/AI |

**丢弃**：`session_meta`（含巨型 `base_instructions`）、`turn_context`、`event_msg/task_started`、
`event_msg/token_count`、`response_item/reasoning`（`encrypted_content`，无明文）、
`response_item/function_call(_output)`、`response_item/custom_tool_call(_output)`、
以及 `response_item/message` 中 `role in (developer, system)` 的注入。

## 踩过的坑（必看）
1. **重复**：同一段对话会同时出现在 `event_msg` 和 `response_item` 里 → 必须按 `(role, text)` 去重。
2. **系统注入**：用户消息里混着 `<environment_context>`、`<system-reminder>`、`<user-context>` 等块，是机器注入不是人话 → 正则剔除；若去掉所有 `<tag>` 后可见文字 <2 字符则整条丢弃。
3. **行号是刚需**：用户标注/回溯时要跳回原 jsonl，所以每条必须带 `[L<原始行号>]`。不要在提取时重置计数。
4. **空会话**：有的 jsonl 只有 `session_meta` + developer 注入（无真实对话），提取出 0 条是正常的，不是 bug。
5. **图片**：`@image#N` / `<image_local_path>` 替换为 `[图片]` 占位。
6. **量级**：368 文件 / 725MB → 输出约 11MB，全量约 1-2 分钟。

## 扩展到别的来源
- WorkBuddy 自己的会话 jsonl（`C:\Users\31310\.workbuddy\projects\*.jsonl`）格式不同：
  `{"type":"message","role":"user","content":[{"type":"input_text","text":...}]}`，
  在 `extract_messages` 里加一个 `t=="message"` 分支即可复用其余逻辑。
- ChatGPT 自采数据是 `ai_sessions/data/chatgpt_messages.jsonl`（已是规范化格式，另有各源自采脚本）。
