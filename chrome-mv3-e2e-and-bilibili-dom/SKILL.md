# Chrome MV3 扩展 E2E 测试 + B站信息流替换（本机实测）

本机调试 Chrome MV3 扩展的完整工作流，以及替换 B 站首页信息流的**正确方案（数据层注入）**。

## B站首页信息流替换 —— 正解：Pinia store 注入（2026-10-10 实测跑通）

**不要克隆 DOM，不要隐藏卡片**。B 站首页是一套 Vue3 + Pinia 应用，feed 是一个 store：

- 拿 store：`document.querySelector('#app').__vue_app__.config.globalProperties.$pinia._s.get('feed')`
  （`__vue_app__` 生产构建同样挂载；store 有 `index` / `feed` / `carousel` 三个）
- **卡片渲染源 = `feed.data.head.recommend`**（视图模型数组）。直接
  `arr.splice(0, arr.length, ...ourItems)` 即触发响应式重渲染，卡片由 B 站自己的组件画出，
  排版/hover/按钮全部原生。实测 21 张原生卡 → 8 张自定义卡， instantly 生效。
- **轮播图在独立的 `carousel` store（`recommendedSwipe`）**，DOM 是 `.recommended-swipe`，
  占网格第一格（2 列宽 [60,255,562,471]）。完全不碰它就原样保留，不需要"重画"。
- **item 字段必须全**（缺字段会渲染出红色 "undefined" 标签）：`track_id, freshIdx, goto, title,
  aid, cid, bvid, cover, pic, pic_4_3, pubdate, enable_vt, author{name,mid,face}, isAd, url,
  spaceUrl, stats{view,like,danmaku,vt,vt_display}, duration, spmInfo{spmid,mod,idx},
  isFollowed, rcmdReason, showInfo, dislike_switch, translated_title, translate_status, key, freshType`。
  **`rcmdReason` 必须给 `{}`**（运行时原生值），给 `{content:'undefined'}` 会渲染出红色 undefined。
- 自定义条目 `key`/`aid` 用负数（如 -10000-i），绝不和真实 aid 撞。
- 防 B 站后续刷新冲掉：`feed.noMoreFeed = true` 关滚动加载 + 每 3s 检查
  `arr[0].key` 是否还是我们的，不是就重新 splice。
- **content script 必须 `"world": "MAIN"`**（manifest content_scripts 里声明），
  否则 isolated world 摸不到页面 JS 对象上的 `__vue_app__`。MAIN world 的 console.log
  反而可以被 `page.on("console")` 抓到（见下面第 4 条的更正）。

**克隆 DOM 方案（已废弃）**：曾用 `cloneNode(true)` 改文字插网格头部，视觉能对但克隆节点
没有 Vue 组件实例——"稍后再看/不感兴趣/hover 预览"等 JS 交互全是死的，且必须隐藏原生
轮播图。属于逃课，被用户点破后重写为数据层注入。

## 本机 E2E 测试环境（踩坑结论）

1. **Python Playwright 用系统 Python312**：`C:\Users\31310\AppData\Local\Programs\Python\Python312\python.exe`
2. **必须 `unset NODE_OPTIONS`**：本机 NODE_OPTIONS=`--use-system-ca`，会让 Playwright driver 启动直接崩
   （报 `Connection closed while reading from the driver`）
3. 扩展加载模式：
   ```python
   p.chromium.launch_persistent_context(
       user_data_dir=<临时目录>, headless=False,
       args=[f"--disable-extensions-except={EXT}", f"--load-extension={EXT}"])
   ```
   每次测试前 `shutil.rmtree` 掉 profile 目录保证干净环境。
4. （更正）console 抓不抓得到取决于 world：**isolated world 的 log 抓不到**；MAIN world
   （数据层注入方案用）的 log **可以**被 `page.on("console")` 抓到。稳妥做法仍是 DOM 断言。
5. 扩展目录里**不能有任何 `_` 开头的文件/目录**（Chrome 保留名，整个扩展拒绝加载）。
6. 截图用 `page.screenshot(path=...)` 并**亲眼 Read 看图**；数值断言（
   `getBoundingClientRect()`）配合使用，只看数字不看图会漏掉排版/裂图问题。
7. 诊断脚本别忘了带 `--load-extension` 参数——忘带时页面是原生 B 站，得出完全错误的结论。

## B站首页关键 DOM（兜底/参考用）

- 推荐流容器：`.container.is-version8`（CSS grid，4 列 × ~271px）。
- 卡片：`.bili-video-card`，标题 `.bili-video-card__info--tit`、UP主 `--author`、
  日期 `--date`、封面 `.bili-video-card__image--wrap img`、时长 `.bili-video-card__stats__duration`、
  推荐语标签 `.bili-video-card__info--icon-text`（绑 rcmdReason）。
- B 站会给封面 URL 追加裁剪后缀（实测会插成 `xxx@.avif?query` 或追加
  `@672w_378h_1c_!web-home-common-cover`）。本地封面服务（8787）按 `[@!]` 切分路径剥掉它。

## 开发热重载

- `dev-server.js`：Node HTTP 服务 (127.0.0.1:8765) 监听 mtime，`/status` 返回 `reload|ok`。
- `background.js` 每 2s fetch 它，返回 `reload` 就 `chrome.runtime.reload()`（onInstalled 再刷新 B 站标签页）。
- 改完代码保存即自动生效；仅首次装扩展需手动在 `chrome://extensions/` 点一次 Reload。
- 端口被占：PowerShell `Stop-Process -Id <pid> -Force`（bash 的 taskkill //F 可能杀不掉）。
