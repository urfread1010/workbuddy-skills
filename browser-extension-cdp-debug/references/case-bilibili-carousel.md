# 案例：B站信息流轮播图替换（改B站代码到崩溃 vs 自写半小时搞定）

## 背景

项目是一个 Chrome MV3 插件，替换 B站首页信息流（feed）的内容。轮播图（carousel）位于首屏第一排，右侧有4张视频卡片。

需求：
1. 显示自己的封面图（来自本地服务 `127.0.0.1:8787`）
2. 轮播图高度和第二排视频卡片的封面底线对齐
3. 支持自动播放（5秒切换）、左右按钮、圆点导航、触摸滑动
4. 轮播数量必须和后台数据一致（不能有空白图）

## 方案一：改B站原生轮播（失败）

### 尝试1：只换 `<img src>`

**假设**：B站轮播用的是 `<img src="...">`，我直接改 src 就能换图。

**实际**：B站用的是 `<picture>` + `<source>`：
```html
<picture class="bili-video-card__cover">
  <source srcset="//i2.hdslb.com/xxx.avif" type="image/avif">
  <source srcset="//i2.hdslb.com/xxx.webp" type="image/webp">
  <img src="//i2.hdslb.com/xxx">
</picture>
```

HTML规范：`<picture>` 内只要有匹配的 `<source>`，浏览器就忽略 `<img src>`。换 img 没用，浏览器还是去取B站的 avif/webp → 裂图。

**教训1**：先 `curl https://www.bilibili.com/ | grep '<picture'` 取样，30秒就能看到真相。不要猜。

### 尝试2：改 `<source srcset>`

**假设**：把 `<source>` 的 srcset 换成我们的 data URL。

**实际**：B站的 AdBlock 警告组件 `.extension-tips-v2` 用 `padding-top: 357px` 撑出轮播高度。我们加了一行 `el.style.display = 'none'` 想把它隐藏 → 直接把357px padding 吃掉了，轮播高度坍缩。

**教训2**：改B站一个DOM元素可能影响完全不相关的布局。B站内部的CSS是"黑盒"，你不知道哪条规则依赖哪个DOM。

### 尝试3：修复高度坍缩

**尝试**：
- 给 `.carousel-area` 加 `overflow: hidden` → 治标不治本
- 给 shim-card 改 `padding-top` → 影响其他卡片
- 给 `.recommended-swipe-body` 加固定高度 → 宽度变了高度不跟着变

**教训3**：B站的CSS层层嵌套、互相依赖，改一处动全身。你以为在"修高度"，其实在和整个布局系统博弈。

### 尝试4：用 `visibility:hidden` 替代 `display:none`

**假设**：`visibility:hidden` 保留空间，`display:none` 不保留。

**实际**：轮播组件不响应我们的修改——Vue 的 reactivity 把我们的修改覆盖了。我们改的是 DOM，但 Vue 下次渲染时从 store 重建 DOM，我们的修改全丢了。

**教训4**：和 Vue 打持久战是打不过的。Vue 每次渲染都会从 store 重建 DOM，你改 DOM = 白改。你需要改 store → 但改 store 又触发其他 computed → 连锁反应。

### 尝试5：用 MutationObserver 持续监听

**假设**：用 MutationObserver 检测到 Vue 重建 DOM 后，再换一次图片。

**实际**：MutationObserver 每次触发都重新执行替换 → 触发Vue重新渲染 → 再次触发 Observer → 无限循环。而且即使我们的图片换了，`<source>` 还在，浏览器还是优先取 source → 图片继续裂。

**教训5**：MutationObserver 不是银弹。Observer 改 DOM 可能触发其他 Observer → 循环、抖动、性能问题。

### 总结方案一的代价

| 尝试 | 耗时 | 结果 |
|---|---|---|
| 换 img src | 2h | 图片裂（source压制） |
| 换 source srcset | 1h | 高度坍缩 |
| 修高度坍缩 | 3h | 引入新bug |
| visibility替代display | 1h | Vue覆盖修改 |
| MutationObserver | 2h | 无限循环 |
| **总计** | **9h+** | **全部失败** |

## 方案二：自写 BFCarousel（成功）

### 决策

**借鉴B站 UI，不碰B站组件。**

- 借鉴视觉：16:9 比例、圆角、圆点导航、左右按钮
- 借鉴交互：自动播放5秒、hover暂停、圆点点击切换、触摸滑动
- 实现：vanilla JS，零依赖，独立作用域（不碰 B站 DOM 结构）

### 实现

`carousel/carousel.js` + `carousel/carousel.css`，~150行代码。

```
container (inject.js 创建)
  └── <a href="https://www.bilibili.com/">
        └── .bfc-carousel
              ├── .bfc-track (display: flex, width: N * 100%)
              │     ├── .bfc-slide (width: 100%, 含 <img>)
              │     ├── .bfc-slide
              │     └── .bfc-slide
              ├── button.prev
              ├── button.next
              └── .bfc-dots
                    ├── span.bfc-dot.active
                    ├── span.bfc-dot
                    └── span.bfc-dot
```

inject.js 调用方式：
```javascript
window.BFCarousel.create(link, {
  images: ourCarouselCovers,  // data URL 数组
  height: targetH,            // 动态计算，和第二排卡片对齐
  interval: 5000
});
```

### 踩过的坑（第二轮，但这次是自己代码，修得快）

| 问题 | 根因 | 修复 |
|---|---|---|
| 点击按钮页面刷新 | click事件冒泡到父 `<a>` 导致导航 | `e.preventDefault(); e.stopPropagation()` |
| 切换图片只动了一张图 | track 宽度 = 1张图宽（CSS width:100% 在 flex 容器 auto 时退化） | JS 显式设置 `track.style.width = N * w + 'px'` |
| translateX 百分比算错 | `translateX(-100%)` 是相对于 track 自身宽度 | 改为 `-(currentIndex / N) * 100`% |

每个坑修起来不超过10分钟——因为代码是自己的，逻辑清晰。

## 核心对比

| | 改B站组件 | 自写组件 |
|---|---|---|
| 控制力 | 和 Vue/PNA/AdBlock 博弈 | 完全可控 |
| 调试 | 猜框架的隐藏逻辑 | 看什么就是什么 |
| 修复成本 | 改一处引入 3 个新 bug | 改一处动一处 |
| 耗时 | 9h+ 全失败 | 0.5h 搞定 |

## 可复用结论

1. **第三方SPA站点的组件尽量自写**。借鉴 UI/UX，不借鉴实现。
2. **先取样再动手**。`curl` 看一眼真实 DOM 就能避免90%的猜测。
3. **最小验证**：新建独立 HTML 验证组件逻辑，确认后再集成。
4. **避免 MutationObserver 改第三方 DOM**：会和站点的框架打架。
5. **自写组件的宽度计算**：flex 容器里 `width: 100%` 在容器 auto 时退化，必须显式设置具体像素值。
6. **事件冒泡**：content script 里 `<button>` 放在 `<a>` 里时，必须 stopPropagation + preventDefault。
