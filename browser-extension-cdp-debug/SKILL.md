---
name: browser-extension-cdp-debug
description: 浏览器插件开发的 CDP 调试工作流。双实例对比取样 + 生命周期感知读源码 + Git 安全锚点 + 排错限时。适用于 MV3 插件开发中「页面被插件改了 DOM」的场景。附带多个 B站插件实战案例（轮播替换、placeholder 拆分等）。
triggers:
  - 浏览器插件调试
  - CDP 读页面源码
  - 双份对比
  - 扩展加载后取样
---

# 浏览器插件 CDP 调试工作流

## 适用场景

浏览器扩展（MV3）改了页面 DOM 后，需要对比「原生页面」和「扩展后页面」的差异来定位 bug。

## 核心原则

1. **启动 + 连通性 = `edge-cdp-connector` skill**：浏览器启动和 CDP 验证由专用 skill 处理，本 skill 只负责调试工作流
2. **取样必须覆盖生命周期**：页面加载完成 + 扩展注入完成，缺一不可
3. **改代码前先 commit**：每次修复独立一个 commit，方便回滚
4. **排错限时 1 小时**：AI 独立排查超过 1h 无进展，停下来汇报给用户

---

## 工作流

### Phase 0：启动 CDP 实例

**职责分离：启动 + 连通性验证交给 `edge-cdp-connector` skill 处理。**

AI 收到"启动 CDP"请求时，先加载 `edge-cdp-connector` SKILL.md 执行其工作流（探活 → 启动 → 验证），连通后再回到本 skill 的 Phase 1 继续。

连通后记录端口号和 Browser 版本，后续阶段通过该端口访问 CDP。

### Phase 1：生命周期感知读源码

#### 1.1 导航到目标页面

```bash
curl -s -X PUT "http://127.0.0.1:9222/json/new?https://target.site/"
```

#### 1.2 轮询等待扩展注入完成

**Python 脚本：**

```python
import websocket, json, time, urllib.request

tabs = json.load(urllib.request.urlopen('http://127.0.0.1:9222/json/list'))
target = [t for t in tabs if 'target.site' in t.get('url','') and t['type']=='page'][0]

ws = websocket.create_connection(target['webSocketDebuggerUrl'], suppress_origin=True, timeout=60)

# 轮询等待扩展特征出现
for attempt in range(20):
    time.sleep(1.5)
    ws.send(json.dumps({"id": attempt+1, "method": "Runtime.evaluate", "params": {
        "expression": "document.querySelectorAll('img[data-mini-cover]').length"
    }}))
    # 必须按 id 匹配读取，不能假定下一条就是响应
    msg = recv_id(ws, attempt+1)
    val = msg.get("result",{}).get("result",{}).get("value",0)
    if val > 0:
        print(f"Extension active: {val} covers found")
        break
```

**关键：CDP 的 DOM.enable 会产生大量事件消息，必须按 msg.id 匹配响应，不能用 `ws.recv()` 假定下一条就是自己要的。**

#### 1.3 读取完整 DOM

```python
# Enable DOM → getDocument → getOuterHTML
# suppress_origin=True 绕过 Edge 的 Origin 校验（否则 403）
ws.send(json.dumps({"id": 100, "method": "DOM.enable"}))
recv_id(ws, 100)

ws.send(json.dumps({"id": 101, "method": "DOM.getDocument", "params": {"depth": -1, "pierce": True}}))
doc = recv_id(ws, 101)
root_id = doc["result"]["root"]["nodeId"]

ws.send(json.dumps({"id": 102, "method": "DOM.getOuterHTML", "params": {"nodeId": root_id}}))
html = recv_id(ws, 102)["result"]["outerHTML"]
```

### Phase 2：双份对比

需要两份源码才能对比：
- **原生**：未装扩展的 Edge 实例 → 页面加载完成 → 读源码
- **扩展后**：装扩展的 Edge 实例 → 页面加载完成 + 扩展注入完成 → 读源码

**时机判断标准：**

| 状态 | 检测方法 |
|---|---|
| 页面 DOM 就绪 | `document.readyState === "complete"` |
| 扩展注入完成 | 轮询检测到扩展自定义的 DOM 标记（如 `data-mini-cover` 数量 > 0） |
| 数据加载完成 | console 中出现扩展的完成日志 |

### Phase 3：根因分析

**对比维度（按优先级）：**

1. **DOM 结构差异**：目标区域的 HTML 标签层级是否变化
2. **图片/资源 src**：`<img src>` 和 `<source srcset>` 是否被替换
3. **CSS 计算样式**：`getComputedStyle()` 的 position/display/height/visibility
4. **事件监听**：`getEventListeners()` 检查绑定
5. **Store 状态**：通过 `__vue_app__` 检查 Pinia store 数据

**常见陷阱：**

- `<picture>` 包裹 `<img>` 时，浏览器优先加载 `<source srcset>`，忽略 `<img src>`
- 绝对定位的图片不撑父容器 → 父容器 height:0 → 不可见
- Vue 不响应某些 store 变化 → 需要直接改 DOM
- PNA 阻断 fetch 127.0.0.1 → 图片必须用 data URL

### Phase 4：Git 安全锚点

```bash
# 改之前
git add <files>
git commit -m "fix: [描述]"

# 改之后验证
node --check inject.js  # 语法检查
# 用户验收 → 提交
```

### Phase 5：修复与迭代

1. AI 汇报根因 + 方案 A/B/C，等用户确认
2. 改代码 → `node --check` → 汇报结果
3. 用户验收 → commit
4. 验收不通过 → git checkout 回滚 → 重新分析
5. 进入下一个问题

---

## 排错限时规则

- AI 独立排查 **30 分钟** 无进展 → 停下来汇报当前发现 + 请求用户介入
- 总计超过 **1 小时** → 必须升级给用户，给出 A/B/C 选项等决策

---

## 辅助脚本

### `recv_id()` — CDP 响应读取

```python
def recv_id(ws, expected_id, timeout=10):
    """按 msg.id 匹配响应，跳过事件消息。"""
    import time
    start = time.time()
    while time.time() - start < timeout:
        try:
            ws.settimeout(timeout)
            msg = json.loads(ws.recv())
            if msg.get("id") == expected_id:
                return msg
        except:
            break
    return None
```

## 案例库

本 skill 积累了多个 B站插件开发的实战案例，位于 `references/`：

| 案例 | 内容 | 何时参考 |
|---|---|---|
| `references/case-bilibili-carousel.md` | 轮播图替换：改B站代码到崩溃 vs 自写组件半小时搞定 | 需要替换 B站组件时 |
| `references/case-bilibili-searchbox-placeholder.md` | 搜索框 placeholder 替换：独立模块拆分 + 语义统一 + MutationObserver 防守 | 需要改搜索框/placeholder 时 |

每个案例都包含：背景 → 方案对比 → 踩坑 → 可复用结论。

**工作流入口**：收到调试任务后，先读对应案例，再按 Phase 0-5 执行。

### 双实例启动模板

**实例1（带扩展，端口 9222）：**

```bash
msedge.exe --remote-debugging-port=9222 --user-data-dir=profile-ext --no-first-run --no-default-browser-check --disable-first-run-ui
```

**实例2（纯净，端口 9223）：**

```bash
msedge.exe --remote-debugging-port=9223 --user-data-dir=profile-pure --disable-extensions --no-first-run --no-default-browser-check --disable-first-run-ui
```

**注意：两个实例必须用不同端口。**
