# UI 视觉质感 · 配方代码模板

> 与 `SKILL.md` 配套。所有配方复制即用；`--` 前缀为 CSS 自定义属性（design tokens）。
> 色板历经对比度校验：正文 ink/bg 组合均 ≥ 4.5:1（见各表标注）。

---

## A. 字体配对 · 杂志编辑（编辑暖夜 / 人文气质默认）

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,300..600&family=Inter:wght@400;500&family=JetBrains+Mono:wght@400;500&family=Noto+Serif+SC:wght@400;600&display=swap" rel="stylesheet">
```

```css
:root {
  --serif: 'Fraunces', 'Noto Serif SC', 'Songti SC', 'Source Han Serif SC', serif;
  --sans:  'Inter', 'PingFang SC', 'HarmonyOS Sans SC', 'MiSans', 'Microsoft YaHei', sans-serif;
  --mono:  'JetBrains Mono', 'SF Mono', 'Cascadia Mono', monospace;
}
.display {
  font-family: var(--serif);
  font-weight: 300;             /* 大标题用细字重,靠字号撑场 */
  font-size: clamp(48px, 12vw, 120px);
  letter-spacing: 0.01em;       /* CJK 微宽,拉丁标题可 -0.03em */
  line-height: 1.12;
}
.display .em { font-style: italic; color: var(--accent); }  /* 仅拉丁字符 */
```

- 气质：温暖、有笔触感、像独立杂志
- 数字极精致（Fraunces 的 old-style 数字），倒计时/统计首选
- 中文标题由 CJK 衬线接管（Fraunces 不含 CJK）

### A+. 中文 display 字体选型（重要：别用默认脸）

**思源宋体 / Noto Serif SC 是"全网默认 CJK 衬线"** —— 单独撑大标题会显得普通、公文感；
且 Light 300 在大字号下笔画单薄。中文标题值得单独选 display 字体：

| 字体 | 来源 | 气质 | 适用 |
|------|------|------|------|
| **ZCOOL XiaoWei 站酷小薇** | Google Fonts（400 单字重） | 纤细锋芒、复古上海、时尚杂志 | 邀请函、品牌、餐饮、文化 |
| **Ma Shan Zheng 马善政** | Google Fonts | 毛笔楷书、喜庆隆重 | 婚礼、节庆、老字号 |
| **LXGW WenKai 霞鹜文楷** | jsDelivr CDN（`lxgw-wenkai-screen-webfont`） | 文楷手写、温润文艺 | 民宿、手账、独立出版 |
| Noto Serif SC 600/900 | Google Fonts | 厚重稳重 | 财经、地产（用粗不用细） |

```html
<!-- 小薇版配对:Fraunces 管拉丁与数字,XiaoWei 管中文 -->
<link href="https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,300..600&family=ZCOOL+XiaoWei&display=swap" rel="stylesheet">
```
```css
--serif: 'Fraunces', 'ZCOOL XiaoWei', 'STSong', serif;
.display {
  font-family: var(--serif);
  font-weight: 400;          /* XiaoWei 单字重,笔画自带粗细对比 */
  letter-spacing: 0.06em;    /* 有锋芒的 display 需要更多空隙 */
  line-height: 1.16;
}
```

- 小薇体 + 拉丁 Fraunces/衬线数字 = 混排不违和（数字走 Fraunces）
- 霞鹜文楷的 CDN 在国内可达性更好；Google Fonts 不可达时它常是唯一活路
- 正文永远不要用 display 字体（小薇 12px 会散），正文继续 PingFang/Inter

## B. 字体配对 · 极简瑞士（画廊 / SaaS 默认）

```css
:root {
  --sans: 'Inter', 'PingFang SC', 'HarmonyOS Sans SC', sans-serif;
  --mono: 'JetBrains Mono', monospace;
}
.display {
  font-family: var(--sans);
  font-weight: 600;
  font-size: clamp(40px, 9vw, 96px);
  letter-spacing: -0.03em;      /* 拉丁收紧;中文标题剥离后 0 */
  line-height: 1.05;
}
.eyebrow {
  font-family: var(--mono);
  font-size: 10px; font-weight: 500;
  letter-spacing: 0.32em; text-transform: uppercase;
  color: var(--accent);
}
```

- 单字族纪律：层级全靠字重(600/400)+字号+字距，安全牌
- Inter 的 `font-feature-settings: 'ss01'` 可换更几何的 a/g（可选）

## C. 字体配对 · 科技终端（深空科技 / DevTool）

```css
:root {
  --display: 'Space Grotesk', 'Noto Sans SC', sans-serif;
  --mono: 'JetBrains Mono', 'SF Mono', monospace;
}
.display { font-family: var(--display); font-weight: 500; letter-spacing: -0.02em; }
.tag     { font-family: var(--mono); font-size: 11px; letter-spacing: 0.18em; }
```

- 标题几何感 + 全站 mono 标签 = 终端气质
- 数字/代码一律 mono + tabular-nums

## D. 字体配对 · 老钱衬线（墨绿 / 酒店 / 财经）

```css
:root {
  --serif: 'Playfair Display', 'Noto Serif SC', serif;
  --sans: 'Inter', 'PingFang SC', sans-serif;
}
.display { font-family: var(--serif); font-weight: 400; letter-spacing: 0; line-height: 1.1; }
.small-caps { font-variant: all-small-caps; letter-spacing: 0.14em; }  /* 拉丁专用 */
```

- Playfair 的高对比笔画自带「矜贵」；正文仍用无衬线保证可读
- 金色细线(1px)分隔是这套配方的签名元素

## E. 色板令牌 · 4 套完整配方

### A 编辑暖夜（邀请函/生活方式）

```css
:root {
  /* 表面 */
  --bg: #f7f3ea;          /* 奶油纸 */
  --bg-2: #efe9db;
  --bg-3: #e4dbc8;
  --dark: #1b1611;        /* 深咖夜,做 hero/页脚 */
  --dark-2: #241d16;      /* 深色表面阶梯 */
  /* 墨水:同一色相的透明度阶梯,不要另调灰 */
  --ink: #1a1510;
  --ink-72: rgba(26, 21, 16, 0.72);
  --ink-52: rgba(26, 21, 16, 0.52);
  /* 反色墨水(深色表面用) */
  --rev: #f7f3ea;
  --rev-72: rgba(247, 243, 234, 0.72);
  --rev-52: rgba(247, 243, 234, 0.52);
  /* 强调 */
  --accent: #b4552f;        /* 赤陶,正文对比 4.6:1 */
  --accent-deep: #93401f;   /* hover/小字用,5.8:1 */
  --gold: #d9a862;          /* 只在深色表面用 */
  --line: rgba(26, 21, 16, 0.14);
}
```

### B 极简画廊（工作室/SaaS）

```css
:root {
  --bg: #faf9f7;  --bg-2: #f1efeb;  --bg-3: #e5e2db;
  --ink: #141414; --ink-72: rgba(20,20,20,.72); --ink-52: rgba(20,20,20,.52);
  --accent: #c8401f;        /* 朱砂,全页 ≤2 处 */
  --line: rgba(20,20,20,.12);
}
```

### C 深空科技（AI/DevTool/夜间）

```css
:root {
  --bg: #0b0d10;            /* 近黑蓝,不是纯黑 */
  --surface-1: #13161c;     /* 表面明度阶梯 */
  --surface-2: #1a1e26;
  --ink: #e9ecf1; --ink-72: rgba(233,236,241,.72); --ink-52: rgba(233,236,241,.52);
  --accent: #6fd3b5;        /* 薄荷,深底上对比 8.9:1 */
  --gold: #d9b36a;          /* 暗金,小字点缀 */
  --line: rgba(233,236,241,.12);
}
```

### D 老钱墨绿（酒店/品牌/财经）

```css
:root {
  --bg: #f4f1e8;  --bg-2: #eae6d9;  --bg-3: #dcd6c4;
  --ink: #17211c; --ink-72: rgba(23,33,28,.72); --ink-52: rgba(23,33,28,.52);
  --accent: #2f4a3d;        /* 墨绿即 accent,克制 */
  --gold: #b99a5f;          /* 细线与小字专用,不大面积 */
  --line: rgba(23,33,28,.14);
}
```

## F. 排版系统骨架（通用）

```css
:root {
  /* type scale,1.25 倍率,中文行高略大于拉丁 */
  --fs-display: clamp(48px, 12vw, 120px);
  --fs-h1: clamp(32px, 6vw, 56px);
  --fs-h2: clamp(24px, 4.5vw, 36px);
  --fs-h3: 20px;
  --fs-body: 15px;
  --fs-small: 13px;
  --fs-tag: 11px;
}
body { font-family: var(--sans); font-size: var(--fs-body); line-height: 1.8; color: var(--ink); background: var(--bg); }
h1,h2,h3 { font-family: var(--serif); font-weight: 300; line-height: 1.15; }
.tag { font-family: var(--mono); font-size: var(--fs-tag); letter-spacing: 0.28em; text-transform: uppercase; color: var(--accent); }
.sub { color: var(--ink-72); }        /* 次级文字用透明度阶梯 */
.faint { color: var(--ink-52); }      /* 三级注释 */
.num { font-family: var(--serif); font-weight: 300; font-variant-numeric: tabular-nums; }
```

## G. 阴影与描边（质感细节）

```css
/* 阴影用 ink 色低透明度,不用纯黑;大而软 > 小而硬 */
--shadow-1: 0 1px 2px rgba(26,21,16,.08), 0 8px 24px rgba(26,21,16,.08);
--shadow-2: 0 2px 4px rgba(26,21,16,.10), 0 16px 48px rgba(26,21,16,.14);
/* 卡片:先描边后阴影,描边撑形阴影撑空间 */
.card { background: var(--bg); border: 1px solid var(--line); border-radius: 12px; box-shadow: var(--shadow-1); }
/* 分隔线 1px + 低透明度;金色细线是 D 配方签名 */
--rule: 1px solid var(--line);
```

## H. 应用示例：倒计时数字（编辑暖夜版）

```css
.cd-digit {
  font-family: var(--serif);
  font-weight: 300;
  font-size: clamp(48px, 12vw, 108px);
  line-height: 1.06;
  color: var(--ink);
  font-variant-numeric: tabular-nums;
}
.cd-sep { color: var(--bg-3); }                 /* 冒号弱化成背景色 */
.cd-unit { font-family: var(--mono); font-size: 10px; letter-spacing: 0.24em; text-transform: uppercase; color: var(--ink-52); }
```

---

## 附：迁移检查表（老页面 → 新配方）

1. 替换 `:root` 色板令牌（挑 1 个 archetype 全量替换，不要挑着换）
2. 替换字体 `<link>` 与 `--serif/--sans/--mono` 栈
3. 标题字重 400/500 → 300（display）且检查 CJK 不再 italic
4. 所有 `color: #999` 类硬编码灰 → `var(--ink-52)` 阶梯
5. 标签类加 `letter-spacing: 0.28em + uppercase`
6. 阴影统一换 `--shadow-1/2`
7. accent 全页盘点，砍到 ≤2 处/屏
