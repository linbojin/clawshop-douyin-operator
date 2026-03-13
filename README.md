# clawshop-douyin-operator 🛒

**抖音小店（抖店）后台运营自动化** — [OpenClaw](https://github.com/openclaw/openclaw) Agent Skill

---

## 功能

- 🔍 **商品管理**：全量读取在售商品（虚拟滚动分页采集）
- 📦 **批量下架/上架**：搜索 + 全选 + 确认弹窗全流程自动化
- 📊 **每日巡检**：订单数、审核驳回、候选库状态汇总日报
- 🧭 **商机中心**：对接 `clawshop-inspiration` 的 7 种选品流
- 💾 **Bitable 同步**：商品状态写入飞书多维表格（via `clawshop-data-manager`）

---

## 文件结构

```
clawshop-douyin-operator/
├── SKILL.md                        # Skill 主文件（OpenClaw 加载入口）
└── references/
    ├── browser-automation.md       # 浏览器自动化踩坑录 & 最佳实践
    └── strategy.md                 # 抖店运营策略手册
```

---

## 使用前提

- 已安装 [OpenClaw](https://github.com/openclaw/openclaw)
- `profile=openclaw` 浏览器已登录抖店后台（`fxg.jinritemai.com`）
- 配合 [`clawshop-data-manager`](https://github.com/linbojin/clawshop-data-manager) 使用飞书商品库

---

## 安装

```bash
# 克隆到 OpenClaw workspace skills 目录
git clone https://github.com/linbojin/clawshop-douyin-operator \
  ~/.openclaw/workspace-clawshop/skills/clawshop-douyin-operator
```

---

## 相关 Skill

| Skill | 用途 |
|-------|------|
| [clawshop-data-manager](https://github.com/linbojin/clawshop-data-manager) | 飞书商品候选库管理 |
| [clawshop-inspiration](https://github.com/linbojin/clawshop-inspiration) | 全渠道选品商机发现 |
| [1688-shopkeeper](https://github.com/next-1688/1688-shopkeeper) | 1688搜品/铺货 |

---

## License

MIT
