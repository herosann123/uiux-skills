# 快速索引 · UI 视觉质感（字体 + 配色）

> 与 `ui-interaction-skill` 配套:那个管动效,本 skill 管质感。
> 主文档:`SKILL.md` · 完整配方:`PATTERNS.md` · 演示:`demo.html`(Before/After 切换)

---

## 气质 → 配方 速查

| 气质关键词 | 色板 | 字体配对 | 一句话 |
|-----------|------|---------|--------|
| 温暖 / 人文 / 邀请 | A 编辑暖夜 | Fraunces + Noto Serif SC | 奶油纸底 + 赤陶 + 金,细衬线大标题 |
| 克制 / 画廊 / SaaS | B 极简画廊 | Inter 单字族 | 近无彩,朱砂 ≤2 处,标签宽排 |
| AI / 开发者 / 夜间 | C 深空科技 | Space Grotesk + Mono | 近黑蓝 + 薄荷,表面明度阶梯 |
| 传承 / 酒店 / 财经 | D 老钱墨绿 | Playfair + Noto Serif | 米白 + 墨绿 + 金色细线 |

气质含糊 → 选 B;想混两个 → 先砍一个。

## 五个最快的「去廉价」开关

1. 大标题字重 400 → **300**,英文加 `letter-spacing: -0.03em`
2. 小标签加 **`letter-spacing: 0.28em` + uppercase**(10–11px)
3. 硬编码灰 → **同一 ink 的透明度阶梯**(72% / 52%)
4. 纯黑纯白 → **带色温的近似色**(`#1a1510` / `#f7f3ea`)
5. accent 砍到**每屏 ≤ 2 处**

## 廉价感红线(详见 SKILL.md §6)

默认蓝 · 彩虹渐变+大圆角+重阴影三件套 · 中文斜体 · #000/#fff · 一屏 3+ accent · 5 字族混用 · 长正文居中

## 文件清单

```
ui-visual-skill/
├── SKILL.md          # 主入口(决策表 + 纪律 + 反模式)
├── PATTERNS.md       # 4 色板 + 5 字体配对完整代码
├── INDEX.md          # 本文件
└── demo.html         # 同内容 Before/After 一键切换演示
```
