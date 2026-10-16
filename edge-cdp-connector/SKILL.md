---
name: edge-cdp-connector
description: AI 启动带 CDP 调试端口的 Edge 浏览器实例并验证连通性。适用于需要 CDP 连接进行浏览器自动化调试的场景。
triggers:
  - 启动 Edge CDP
  - CDP 连通性验证
  - Edge 调试端口
---

# Edge CDP 连接器

## 适用场景

需要通过 CDP（Chrome DevTools Protocol）连接 Edge 浏览器进行调试时，AI 自动启动实例并验证连通性。

## 核心原则

1. **先探后启**：启动前检查端口是否已被占用
2. **profile 隔离**：不同实例用不同 `--user-data-dir`
3. **失败即报**：启动失败立即报告错误，不自行换方案

---

## 标准启动命令

### 实例1（带扩展 / 日常 profile）

```bash
"C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe" --remote-debugging-port=9222 --user-data-dir="C:/urfread/edge-instance" --no-first-run --no-default-browser-check --disable-first-run-ui
```

### 实例2（纯净对比）

```bash
"C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe" --remote-debugging-port=9223 --user-data-dir="C:/urfread/edge-instance2" --no-first-run --no-default-browser-check --disable-first-run-ui
```

---

## AI 工作流

### Step 1：探活

启动前先检查端口是否已被占用：

```bash
curl -s --max-time 2 http://127.0.0.1:9222/json/version
```

- **通** → 已有实例在跑，直接验证连通
- **不通** → 进入 Step 2

### Step 2：启动 Edge

按上方标准命令启动，**超时设 15 秒**等 DevTools WebSocket 就绪。

成功标志：stdout 出现 `DevTools listening on ws://127.0.0.1:9222/...`

### Step 3：验证连通

```bash
curl -s http://127.0.0.1:9222/json/version
```

- **通** → 返回 JSON（Browser / Protocol-Version / webSocketDebuggerUrl）
- **不通** → 报告错误，检查 stdout/stderr

### Step 4：失败处理

| 错误 | 原因 | 处理 |
|---|---|---|
| "无法在现有会话中打开" | 同 profile 已有 Edge 在跑 | 报告用户关闭冲突实例 |
| 端口被占用 | 其他程序占端口 | 换端口或关闭占用程序 |
| 超时无 DevTools 输出 | Edge 启动失败 | 报告 stderr，等用户决策 |

---

## 端口分配

| 用途 | 端口 | profile |
|---|---|---|
| 日常扩展调试 | 9222 | `C:/urfread/edge-instance` |
| 纯净对比 | 9223 | `C:/urfread/edge-instance2` |

---

## 禁止行为

- ❌ 启动前不探活
- ❌ 失败后自行换端口/换 profile
- ❌ 启动后不验证连通
