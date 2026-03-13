# 抖店浏览器自动化 - 踩坑录 & 最佳实践

> 实战总结，避免重复踩坑。每条都是真实踩过的。

---

## 🔴 必读：JS evaluate 的限制

抖店 browser automation 的 `evaluate` 只接受**单个表达式**，有严格限制：

```javascript
// ❌ 不能用语句级关键字
const x = 1              // 报错
let x = 1                // 报错
var x = 1                // 报错
statement1; statement2   // 分号报错

// ✅ 正确写法：单表达式 + || 链式
document.querySelector('.btn')?.click() || 'ok'
document.body.innerText.match(/共 \d+ 件商品/) || 'not found'

// ✅ 多步骤：用括号和逗号运算符
(document.querySelector('.search-input').value = '123', document.querySelector('.search-input').dispatchEvent(new Event('input', {bubbles:true})))
```

**记住**：末尾加 `|| 'ok'` 防止返回 undefined 被当错误。

---

## 🔴 Ref 失效问题

**所有 ref（e1、e114 等）在 evaluate 调用后立即失效！**

```
snapshot → 得到 ref=e76 (搜索框)
evaluate (任意JS操作)
→ e76 现在已失效！再用会报 "Element not found"
```

**解决方案**：
```javascript
// ❌ 弃用 ref 点击（容易失效）
browser action=act request={kind:click, ref:e114}

// ✅ 用 JS 直接操作 DOM
browser action=act request={kind:evaluate, fn:"document.querySelector('button.批量下架')?.click() || 'ok'"}

// ✅ 搜索框操作（不依赖 ref）
browser action=act request={kind:evaluate, fn:"(document.querySelector('input[placeholder*=\"商品名称\"]').value='123') || 'set'"}
```

---

## 🔴 虚拟滚动：商品列表只渲染可见行

商品列表页使用虚拟滚动（Virtual Scrolling），**DOM 中只有当前视窗内的行**。

**错误做法**：直接 `querySelectorAll('.product-row')` 以为能获取所有商品

**正确做法**：滚动 + 分段采集

```javascript
// 步骤1：先不滚，读第一屏（约10条）
document.querySelectorAll('[class*="row"] [class*="id"]').map(el => el.innerText)

// 步骤2：滚动到中部（大约一半的位置）
document.querySelector('.virtual-list, [class*=VirtualList], .ecom-table-body').scrollTop = 500

// 步骤3：再读一次（会有新的行）
// 重复直到覆盖所有商品

// 实际验证可用的滚动 JS
document.querySelector('.ecom-table-body') && document.querySelector('.ecom-table-body').scrollTo(0, 400)
```

**省力方法**：用搜索框！输入逗号分隔的商品ID，精确过滤。

---

## 商品 ID 提取（最可靠方式）

```javascript
// 提取当前视窗内所有商品行的ID
[...document.querySelectorAll('[class*=row]')].map(row => {
  const idEl = row.querySelector('[class*=id], a[href*="goods"]')
  return idEl ? (idEl.innerText.match(/\d{15,}/) || [''])[0] : ''
}).filter(Boolean)

// 更简单：直接匹配15位以上数字（抖店商品ID格式）
document.body.innerText.match(/\d{18,}/g)

// 获取商品总数
document.body.innerText.match(/共 ?\d+ ?件商品/)
```

---

## 批量下架 - 完整流程

**步骤一：搜索目标商品**

```javascript
// 清空并填入逗号分隔的商品ID
var input = document.querySelector('input[placeholder*="商品名称"]')
input.value = '1234567890123456789,9876543210987654321,1111111111111111111'
input.dispatchEvent(new Event('input', {bubbles:true}))
input.dispatchEvent(new Event('change', {bubbles:true}))
```

等300ms后：

```javascript
// 点击查询按钮（避免用 ref）
[...document.querySelectorAll('button')].find(b => b.innerText.trim() === '查询')?.click()
```

**步骤二：全选**

```javascript
// 点击全选复选框（表头的第一个 checkbox）
document.querySelector('thead input[type=checkbox], .ecom-table-thead input[type=checkbox]')?.click()
// 或
document.querySelector('.ecom-checkbox-input')?.click()
```

**步骤三：点批量下架**

```javascript
[...document.querySelectorAll('button')].find(b => b.innerText.includes('批量下架'))?.click()
```

**步骤四：确认弹窗**

```javascript
// ⚠️ 弹窗里有多个按钮，要精确匹配"仍要下架"
[...document.querySelectorAll('button')].find(b => b.textContent.trim() === '仍要下架')?.click() || 'clicked'
```

---

## Profile 选择

**永远用 `profile=openclaw`**，不要用 `profile=chrome`。

`openclaw` profile 已保存抖店登录态，`chrome` 没有。

---

## targetId 管理

每次 `browser action=open` 都会返回一个 `targetId`。

**必须把这个 targetId 传给后续所有操作**，否则会操作错误的 tab：

```
browser action=open targetUrl="..." profile=openclaw
→ 返回 targetId: "AA64AB5526E49316E55C84435082C339"

之后所有操作都要带上这个 targetId：
browser action=act targetId="AA64AB5526E49316E55C84435082C339" request={...}
browser action=screenshot targetId="AA64AB5526E49316E55C84435082C339"
```

---

## 输入文字的正确方式

**React 组件监听的是合成事件，普通 value 赋值无效**：

```javascript
// ❌ 这样 React 组件不会感知到变化
document.querySelector('input').value = 'xxx'

// ✅ 必须同时触发 React 合成事件
var el = document.querySelector('input[placeholder*="商品名称"]')
var nativeInputValueSetter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set
nativeInputValueSetter.call(el, 'xxx')
el.dispatchEvent(new Event('input', { bubbles: true }))

// 或者用 browser act type（更稳定）
browser action=act request={kind:type, ref:eXXX, text:"xxx"}
// 注意：type 操作前要先 click 聚焦！
```

---

## 页面加载等待

加载抖店页面需要等待：

```
browser action=open → 立刻截图 = 空白页
等1-2秒后 → 再截图 = 内容加载完成
```

**实践**：`open` 后立刻 `screenshot`，如果空白，再 `screenshot` 一次。通常第二张就有内容了。

---

## 图片上传（目前无法自动化）

抖店商品图片上传来自 React 的 `<input type="file">`，原生 `setInputFiles()` 被 React 拦截，无效。

**已知限制**：无法通过 browser automation 上传图片，需要手动操作或使用 1688「铺货」功能（但铺货功能来自 Chrome 扩展「代发助手王」的 Shadow DOM，同样无法自动化）。

**当前对策**：上传图片步骤手动完成，其余步骤自动化。

---

## 常用 CSS 选择器速查

| 用途 | 选择器 |
|------|--------|
| 搜索框 | `input[placeholder*="商品名称"]` |
| 查询按钮 | `button` find by text `查询` |
| 批量下架 | `button` find by text `批量下架` |
| 确认下架 | `button` find by text `仍要下架` |
| 全选框（表头） | `thead input[type=checkbox]` |
| 分页下一页 | `.ecom-g-pagination-next, [class*=pagination] [class*=next]` |
| 商品总数 | 文本匹配 `/共 ?\d+ ?件商品/` |
| 虚拟滚动容器 | `.ecom-table-body` |
