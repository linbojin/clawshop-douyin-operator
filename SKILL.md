---
name: clawshop-douyin-operator
description: 抖音小店（抖店）后台运营自动化。用于：(1) 浏览器访问抖店后台读取商品/订单/数据 (2) 商品批量下架/上架管理 (3) 电商罗盘数据读取 (4) 1688选品与利润计算 (5) 每日巡检报告生成。当用户提到抖店运营、商品管理、上下架操作、日常巡检、选品、订单查看时触发。
---
## ⚠️ 浏览器操作铁律
**所有浏览器操作必须用 `profile=openclaw`，严禁用 `profile=chrome`！**
- `profile=openclaw` = 本地 OpenClaw 浏览器，随时可用 ✅
- `profile=chrome` = 需要用户 attach tab，cron 里完全不可用 ❌


# 抖店运营 Skill

通过浏览器自动化（`profile=openclaw`）访问抖店后台。

> ⚠️ **踩坑警告**：浏览器自动化有很多坑，操作前必读 [browser-automation.md](references/browser-automation.md)

---

## 核心 URL 速查

基础域: `https://fxg.jinritemai.com`

| 功能 | 路径 |
|------|------|
| 商品管理（在售） | `/ffa/g/list?sov_draft_status=0&sov_goodsType=0` |
| 商品管理（审核驳回） | `/ffa/g/list?sov_draft_status=3` |
| 商品创建 | `/ffa/g/create` |
| 订单管理 | `/ffa/morder/order/list` |
| 发货中心 | `/ffa/morder/logistics/ewaybill-delivery` |
| 经营概览（罗盘） | `/ffa/mcompass/overview` |
| 商机中心 ⭐ | `/ffa/bu/NewBusinessCenter` |
| 体验分 | `/ffa/eco/experience-score` |
| 售后工作台 | `/ffa/maftersale/aftersale/list` |

---

## 每日巡检流程

1. **打开订单管理**，截图确认今日订单数
2. **打开商品管理**，JS 读取商品总数：
   ```javascript
   document.body.innerText.match(/共 ?\d+ ?件商品/)
   ```
3. **检查审核驳回**：打开 `?sov_draft_status=3` 页面
4. **读取飞书多维表**：更新在售商品追踪
5. **生成日报**，发飞书 DM 给店主

日报格式：
```
📊 每日巡检日报
🗓️ YYYY-MM-DD

在售商品：X个 | 今日订单：X单 | 审核驳回：X个
[选品候选库状态]
[需要店主确认的事项]
```

---

## 批量下架流程

> **完整踩坑指南** → [browser-automation.md](references/browser-automation.md)

```
1. 打开商品管理页
2. 搜索框输入逗号分隔的商品ID → 点查询
3. 全选 → 批量下架
4. 确认弹窗：JS 找"仍要下架"按钮点击
```

关键 JS（弹窗确认，ref 会过期！）：
```javascript
[...document.querySelectorAll('button')].find(b => b.textContent.trim() === '仍要下架')?.click() || 'clicked'
```

---

## 商机中心（选品核心）

路径：`/ffa/bu/NewBusinessCenter`

> 商机中心选品逻辑已由 `clawshop-inspiration` Skill 统一管理（数据流/审美流等7种选品流）。

---

## 飞书 Bitable 商品库

> 商品候选库 token 统一通过 `clawshop-data-manager` Skill 管理：
> ```
> gateway config.get path=skills.entries.clawshop-data-manager.env
> → 取 CLAWSHOP_APP_TOKEN 和 CLAWSHOP_TABLE_ID
> ```
> 字段结构见 `clawshop-data-manager/SKILL.md`。

---

## 全量商品读取（全量79件验证可行，2026-03-03）

### ✅ 唯一可靠方法：分页翻页 + 每页滚动采集

> 2026-03-03 实战验证：79件商品，4页全量拿到，精确。
> 虚拟滚动每页只渲染~7条，必须边滚边采集，每页约采集20条；点击页码翻页而非依赖URL。

**步骤：**

1. 打开在售商品页（重置所有筛选）：
   ```
   https://fxg.jinritemai.com/ffa/g/list?status=2
   ```
   等待页面出现"售卖中"（8秒）。

2. 用 `window._all = {}` 初始化全局缓存，存 `ID → 上架日期`。

3. **第1页采集**：滚动+extract
   ```javascript
   // act evaluate
   () => {
     window._all = {};
     const extract = () => {
       document.querySelectorAll('table tbody tr').forEach(r => {
         const t = r.innerText;
         const id = t.match(/ID:(\d{15,})/);
         const dt = t.match(/202\d\/\d{2}\/\d{2}/);
         if (id && dt) window._all[id[1]] = dt[0];
       });
     };
     // 滚动采集
     return new Promise(resolve => {
       const scrollEl = document.scrollingElement;
       scrollEl.scrollTop = 0;
       let i = 0;
       const step = () => {
         extract();
         scrollEl.scrollTop += 500;
         i++;
         if (i < 15) setTimeout(step, 200);
         else resolve(Object.keys(window._all).length);
       };
       step();
     });
   }
   ```

4. **翻页（第2、3、4页）**：点击页码 li，再重复滚动采集
   ```javascript
   // act evaluate（传入页码 N）
   async () => {
     const pageBtn = [...document.querySelectorAll('li')]
       .find(li => li.innerText.trim() === '2' && li.className.includes('pagination-item'));
     if (!pageBtn) return 'no page btn';
     pageBtn.click();
     await new Promise(r => setTimeout(r, 2000));  // 等页面切换
     const scrollEl = document.scrollingElement;
     scrollEl.scrollTop = 0;
     const before = Object.keys(window._all).length;
     for (let i = 0; i < 15; i++) {
       scrollEl.scrollTop += 500;
       await new Promise(r => setTimeout(r, 150));
       document.querySelectorAll('table tbody tr').forEach(r => {
         const t = r.innerText;
         const id = t.match(/ID:(\d{15,})/);
         const dt = t.match(/202\d\/\d{2}\/\d{2}/);
         if (id && dt) window._all[id[1]] = dt[0];
       });
     }
     return { before, after: Object.keys(window._all).length };
   }
   ```
   对每一页（2、3、4…）各调用一次，改 `'2'` 为对应页码即可。

5. **结果导出**：
   ```javascript
   () => Object.entries(window._all).sort((a,b) => a[1] < b[1] ? 1 : -1)
   ```

6. 用 Python 对比 Bitable 候选库，批量更新上架日期/下架状态。

**注意事项：**
- 页码 li 必须用 `li.className.includes('pagination-item')` 过滤，否则会误匹配
- 每次翻页后等 2 秒再开始滚动，否则新页数据还没渲染
- 总条数用 `document.body.innerText.match(/共 ?(\d+)/)` 确认，计算需要几页
- 上架日期正则：`202\d\/\d{2}\/\d{2}`，从 row 的 innerText 直接提取

### ❌ 不要用导出方式

- `导出查询商品` → 点击后生成任务，需在历史报表下载
- 历史报表 (`/ffa/fxg-bill/history-report`)：文件只保留 **24小时**，容易过期
- 下载链接需要 session cookie，curl 直接下会 403，需 browser fetch
- 整条路径太长、太脆，容易在任一环节断链
- **结论：费时费力，放弃这条路**

### Bitable 商品库字段

> 字段结构以 `feishu_bitable_list_fields(app_token, table_id)` 实时返回为准。
> token 通过 `gateway config.get path=skills.entries.clawshop-data-manager.env` 获取。

### 价格解析 Python 片段

```python
def parse_price(price_str):
    """'￥4.88 ~ ￥9.16' → 4.88（取最低）"""
    nums = [float(x.replace('￥','').strip()) for x in price_str.split('~') if x.replace('￥','').strip().replace('.','').isdigit() or '.' in x.replace('￥','').strip()]
    return nums[0] if nums else 0.0

def parse_date_ms(time_str):
    """'2026/02/25 22:35:44' → 毫秒时间戳"""
    from datetime import datetime
    dt = datetime.strptime(time_str.strip(), "%Y/%m/%d %H:%M:%S")
    return int(dt.timestamp() * 1000)
```

---

## ⚠️ 铁律

1. **定价操作必须店主确认**，不能自己改价格
2. **不能刷单、不违规操作**
3. **虚拟滚动**：商品列表每页只渲染可见行，必须先切100条/页再滚动JS采集
4. **图片上传无法自动化**，需手动操作
5. **全量同步标准流程**：切100条/页 → 滚动JS采集 → Python解析 → Bitable批量写入（先清空旧记录）

---

## 参考文档

- [browser-automation.md](references/browser-automation.md) — ⭐ **浏览器踩坑录**（必读）
- [strategy.md](references/strategy.md) — 运营策略手册
