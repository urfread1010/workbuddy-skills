# 案例：B站搜索框 placeholder 替换（独立模块 + 语义拆分 + 三管齐下防守）

## 背景

项目是一个 Chrome MV3 插件，替换 B站首页信息流。搜索框有两个需求：
1. **下拉面板**：展示长文本「每日思考」，挂载于搜索历史位置
2. **placeholder**：多条经典语录随机展示，每 3 分钟轮换

之前的状态：
- `quote_card.text`（长文本）抢了 placeholder 显示
- `quotes[]`（短名言）被挤到次要位置，没跑起来
- placeholder 替换逻辑散落在 560 行的 inject.js 里
- Observer 检测到 placeholder 变化 → 触发回调 → 但回调什么都没改

## 数据模型重构

### 旧模型（混乱）

| 字段 | 含义 | 用途 |
|---|---|---|
| `quote_card` | 长文本（如"消极体验造就积极体验..."） | 被当作 placeholder 内容 |
| `quotes` | 多条短名言（如"事缓则圆"） | 被 quote_card 挤掉 |

问题：两个都叫"quote"，优先级不清，quote_card 占着 quotes 的位置。

### 新模型（统一）

| 字段 | 含义 | 显示位置 | 显示名称 |
|---|---|---|---|
| `motto_card` | 长文本（每日思考） | 下拉面板长文本区 | 每日思考 |
| `quotes` | 多条短名言 | 搜索框 placeholder 随机 | 经典语录 |
| `tasks` | 待办事项 | 下拉面板列表 | 今日待办 |

### 全栈字段改名

- 前端 `inject.js`：`quoteCardCache` → `mottoCardCache`
- 前端 `searchbox.js`：渲染改用 `data.motto`（不再用 `data.quote`）
- 前端 `searchbox-placeholder.js`：新模块，独立处理 placeholder
- 前端 `bridge.js`：新增 `miniMotto` dataset
- 后端 `cover_server.py`：`quote_card` → `motto_card`，向后兼容
- 后端 `events.json`：`quote_card` → `motto_card`
- 中台 `admin.html`：Alpine.js `quoteCard` → `mottoCard`，标题更新

## 代码拆分

### 决策

把 placeholder 替换从 inject.js 里拆出来，独立成 `searchbox-placeholder.js`。

理由：
- inject.js 已经 560 行，越改越难维护
- placeholder 和搜索面板是两个不同的 UI 区域
- 后续想改 placeholder 行为（节奏/来源/随机策略）只动这一个文件

### 拆分后结构

```
inject.js（560→530行）
  ├── patchSearchGlobal()  — 左侧动态 icon + 右上动态按钮
  ├── patchDom()           — 轮播 + 残留卡 + 搜索面板数据
  └── publishSearchData()  — 拆分出 motto 和 quote 两条独立通道

searchbox.js（不变）
  └── 读 data.motto → 渲染「每日思考」

searchbox-placeholder.js（新建）
  └── 读 data.quotes → 随机取一条 → 写 placeholder
```

## 三管齐下防守机制

### 问题本质

B站 Vue 不断重写 placeholder（每次渲染都会重置），单次设置永远压不住。

### 方案

1. **启动即写**：200ms 轮询等 `__MINI_SEARCH__.quotes` 就绪，立刻写第一条
2. **持续防守**：MutationObserver 监听 `placeholder` 属性变化 → 立刻覆盖回我们的当前语录
3. **定时轮换**：`setInterval(180000)` 每 3 分钟随机换一条（用户聚焦时也换）

### 关键细节

- **启动轮询必须主动**：不等 B站 Vue 改触发，自己主动写
- **MutationObserver 必须区分自己和 B站**：检测 `cur === lastQuote` 跳过自己的修改，避免死循环
- **定时器 3 分钟**：太短（500ms）会闪烁，太长（10分钟）没变化

```javascript
// 核心逻辑
function restoreOnly() {
  const input = document.querySelector(INPUT_SEL);
  if (!input) return;
  const cur = input.getAttribute('placeholder') || '';
  if (cur === lastQuote) return; // 已被我们占，跳过
  if (!lastQuote) return;       // 没数据不覆盖
  input.setAttribute('placeholder', lastQuote);
}

function rotate() {
  const input = document.querySelector(INPUT_SEL);
  if (!input) return;
  const quote = getRandomQuote();
  if (!quote) return;
  input.setAttribute('placeholder', quote);
  lastQuote = quote;
}

setInterval(rotate, 180000);                    // 3分钟轮换
new MutationObserver(restoreOnly).observe(...);  // B站一改就覆盖
document.addEventListener('focusin', rotate);    // 聚焦也换
```

## 踩过的坑

| 问题 | 根因 | 修复 |
|---|---|---|
| 启动时显示 B站 placeholder | 被动等触发，启动时没有主动写 | 200ms 轮询主动写第一条 |
| placeholder 快速闪烁 | 定时器 500ms 太短 | 改为 3 分钟 |
| Observer 触发后什么都没改 | 回调函数和写入逻辑分离 | 合并到 restoreOnly() |
| 启动轮询写完后 B站立即覆盖 | 时机太早，数据还没写完 | 增加幂等判断（cur === lastQuote） |

## 可复用结论

1. **功能拆分原则**：inject.js 超过 200 行就该拆，按 UI 区域（搜索框/轮播/feed）拆，不要按技术层拆
2. **数据模型语义统一**：quote_card 和 quotes 语义不同，必须用不同字段名（motto_card vs quotes），优先级才不会乱
3. **和 Vue 打持久战**：单次改 DOM 必输，必须用 MutationObserver + 定时器持续防守
4. **启动即写**：不要等用户交互触发，数据就绪立刻写，否则用户看到原生内容
5. **Observer 防循环**：Observer 回调里要判断"是否自己的修改"，否则死循环

## 文件清单

| 文件 | 改动 |
|---|---|
| `service/cover_server.py` | `quote_card` → `motto_card`，向后兼容 |
| `service/covers/events.json` | `quote_card` → `motto_card` |
| `service/covers/admin.html` | Alpine.js 变量同步，标题更新 |
| `mvp/inject.js` | `motto_card` 读取；`publishSearchData` 拆通道 |
| `mvp/bridge.js` | 新增 `miniMotto` dataset |
| `mvp/searchbox.js` | 渲染改用 `data.motto` |
| `mvp/searchbox-placeholder.js` | 新建，随机展示经典语录 |
| `mvp/manifest.json` | 新增文件引用 |
