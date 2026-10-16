---
name: library-multitable-workspace
description: 用 WorkBuddy「资料库」搭建多表数据驱动工作台 / 看板 HTML 并部署时必读。当页面需要同时关联多张 database（如待办 + 打卡 + 记账 + 看板），要通过 lint_database_sdk_usage.py 与 lint_schema.py 两道校验，或出现 DSDK002 / DSDK003 / DSDK006 / DSDK008 / DSDK009 / DSDK011 / DSDK012 / R10 报错时使用。覆盖 databaseId 字面量的合规写法、合并 schema 校验法、选择器锚点要求、草稿缓存三件套。
display_name: "资料库多表看板"
display_name_en: "Multi-table Workspace"
description_zh: "用 WorkBuddy 资料库搭多表联动的工作台／看板页面并部署时必读，覆盖 DSDK 系列与 R10 报错的实测解法。"
description_en: "Essential when building and deploying multi-table dashboards or workspaces on the WorkBuddy Library. Covers tested fixes for the DSDK and R10 errors."
visibility: "public"
agent_created: true
version: 1.0.0
author: "kyle"
---

# 资料库多表数据页搭建要点

**适用**：一次任务里创建 ≥2 张 database，并生成一个 HTML 页面同时读写这些表（典型：个人工作台 / 运营看板 / 台账）。

**不适用**：只有一张表的场景——直接看内置 `library` skill 的单表流程即可，本 skill 只补多表场景特有的坑。

## 依赖声明

本 skill 依赖 WorkBuddy 内置的 **`library`（资料库）** skill，其校验脚本位于该 skill 目录下：

- `lint_database_sdk_usage.py` —— 校验前端 SDK 调用是否合规（DSDK 系列规则）
- `lint_schema.py` —— 校验表结构定义（R1–R10 规则）
- `create_database.py` / `import_html.py` / `publish_page.py` —— 建表、导入页面、发布
- `skills/library/page/data-page-flow.md` —— 单表页面的完整流程文档

本 skill 不重复这些内容，只补多表场景的增量部分。

---

## 一、DSDK002 是最大陷阱：每个调用点都要 databaseId 字面量

`lint_database_sdk_usage.py` 的 `_has_database_id()` 只接受两种形态：

- `{ databaseId: 'db_xxxx' }` —— 字面量
- `{ databaseId: DATABASE_ID }` —— 且 HTML 中存在 `var DATABASE_ID = '...'`

**以下全部 FAIL（exit 2，报 DSDK002）**：

```js
// ❌ 通用封装：传参进去的 databaseId 不是字面量
function loadAll(dbId) { return db.query({ databaseId: dbId, pageSize: 200 }); }
function add(dbid, props) { return db.addRecord({ databaseId: dbid, properties: props }); }

// ❌ 数组/映射里取值
db.getSchema({ databaseId: JOBS[i][1] });
```

**正确做法**：每张表写一组独立函数，字面量直接内联。多表页面就是 3 组 `loadX / addX / updX / delX`。

```js
function loadTodos(cur, acc, guard) {
  acc = acc || []; guard = guard || 0;
  if (guard > 100) { state.todos = acc; return Promise.resolve(acc); }
  return db.query({ databaseId: 'db_TODO0000000000', pageSize: 200, startCursor: cur })
    .then(function (res) {
      acc = acc.concat(res.results || []);
      var next = res.nextCursor;
      if (res.hasMore && next && next !== cur && (res.results || []).length) {
        return loadTodos(next, acc, guard + 1);
      }
      state.todos = acc;
      return acc;
    });
}
```

> 递归分页的 `guard` 上限是必要的：`hasMore` 与 `nextCursor` 在某些边界下会互相矛盾，没有护栏会无限递归。

批量导入时若用通用 helper 会被 DSDK002 拦住 → 另写一个 `rawAdd(kind, props)`，里面用 `if/else` 分支，**每个分支各写一次带字面量的 `db.addRecord(...)`**（3 个分支 = 3 个合规调用点）。这样批量循环里只刷新一次，不会每条记录 reload 一次。

## 二、多表页面怎么过 DSDK003 / DSDK006 / DSDK009

这三条规则把 `properties['字段名']`、`row['字段名']`、query 的 `sorts/filter` 字段名拿去和**单一** schema 的 `properties` 比对。多表页面传任一单表 schema 都会误报。

**做法**：把各表字段名取并集，生成一个合并 schema 再跑 lint。

```bash
python3 - <<'PY'
import json
fields = {}
for f in ['schema_a.json', 'schema_b.json', 'schema_c.json']:
    for k, v in json.load(open(f))['properties'].items():
        fields[k] = v
json.dump({"title": "merged", "properties": fields},
          open('merged_schema.json', 'w'), ensure_ascii=False)
PY
python3 "$L/lint_database_sdk_usage.py" --schema-file merged_schema.json --html index.html
```

副产品：合并 schema 会强制各表字段名不冲突到无法区分，反而能暴露命名问题。

## 三、lint_schema.py 的 R10：选择器类名必须在标记里真实存在

`--html` 模式下 R10 会用 `html.parser` 扫真实的 `class` / `id` / `[attr]`，**不会看 JS 字符串里的模板**。若 `display_selector` 写成 `.todo-title`，HTML 里必须有带这个 class 的元素。

**做法**：在页面尾部放一个隐藏模板块，把三类行模板的 class 都列出来，渲染时同时复用这些 class：

```html
<div id="tpl">
  <div class="todo-row"><span class="todo-title"></span><span class="todo-priority"></span>
  <span class="todo-status"></span><span class="todo-due"></span><span class="todo-note"></span></div>
  <div class="habit-row"><span class="habit-name"></span><span class="habit-date"></span>
  <span class="habit-done"></span><span class="habit-note"></span></div>
  <div class="fin-row"><span class="fin-title"></span><span class="fin-type"></span>
  <span class="fin-amount"></span><span class="fin-category"></span><span class="fin-date"></span>
  <span class="fin-note"></span></div>
</div>
```
```css
#tpl { display: none }
```

并在真实渲染的 HTML 片段里带上同名 class（`row-title todo-title`、`chip chip-date todo-due` 等），既满足 R10，也让 class 有实际语义。

## 四、DSDK008：含 addRecord 必须出现三件套

`localStorage.getItem` + `localStorage.setItem` + `localStorage.removeItem`（或 `clear`），缺一即 FAIL，且用**去除注释后的源码**匹配（注释里写没用）。

做法：给每个表单挂 300ms 去抖的草稿缓存——

- `input` / `change` 时 `setItem` 全表单字段
- `renderSelectOptions()` 之后 `getItem` 回填
- 提交成功后 `removeItem`

**跳过** `type=password`、`data-no-cache`、以及名称含 密码 / 身份证 / secret / token / key 的字段。

含 `uploadImage` 时另需 IndexedDB 缓存（DSDK008 第二段）——若不涉及图片字段就**别写** `db.uploadImage`，免得被要求补齐 IndexedDB 代码。

## 五、DSDK011 / DSDK012：绑定标注

元素文本直接来自或派生自 database（含统计数字）时，加成对属性：

```html
<p data-sp-bindable="database" data-sp-database-id="db_XXX0000000000">共 128 单</p>
```

- 静态标记里写成**字面量**（lint 只扫静态标记；JS 里用 `setAttribute('data-sp-database-id', DB_X)` 的两参数形式不会被静态扫到，也不会误报）。
- **别**在 JS 里拼 `'data-sp-database-id="' + DB_X + '"'` 这类字符串。
- 派生统计（KPI、进度环百分比）也要标，取主要来源表 id。

## 六、调用顺序（多表版）

1. 各表 canonical schema（options id 由 `Math.random().toString(36).slice(2,10) + Date.now().toString(36)` 生成，`options_map` 二层 key = option id，因 `options_value_key="value"`）
2. `lint_schema.py --schema <每张表>`（无 `--html`）先过 R1–R9
3. **等用户显式确认 schema** 再建表（沉默 ≠ 确认）
4. 逐张 `create_database.py`，记下返回的 `database_id` 与最终 option id
5. 写 HTML
6. `lint_schema.py --schema <各表> --html` → 过 R10
7. 合并 schema → `lint_database_sdk_usage.py` → 过 DSDK 系列
8. 图片外链自检（grep 三连，无输出 = 合格）
9. `import_html.py page.html --file-name "语义名.html" --databases '[{"id":"..."},{"id":"..."}]'`
10. 解析 `KS_IMPORT_OK` 里的 `url`，用 `present_files` 打开；**不**自动发布（`publish_page.py` 只认用户明确说「发布 / 部署」）

## 七、界面上的硬要求

- 首屏「今天要处理」置顶：逾期标红 + 一键处理；昨天没做完的顺延
- 渲染函数**互不调用**；统一 `refreshAll()` 入口（避免日历 / 列表互调导致栈溢出）
- 预置 3–5 条示例数据含 1 条逾期，并给「清空示例数据」按钮（示例记录靠名称白名单识别）
- 首屏有「导出 JSON」「导入恢复」；清空数据二次确认
- **支出红、收入绿**，货币 ¥；图标用内联 SVG，不用 emoji；零外链
