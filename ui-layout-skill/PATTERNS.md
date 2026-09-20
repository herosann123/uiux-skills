# UI 布局骨架 · 配方与 Token

> 所有配方是**起点，不是答案**。换、拆、混都允许；唯一不许的是违反 SKILL.md §1 的五条底线。

---

## A. 间距 scale token（三选一,全站统一）

```css
/* normal 标准(默认建议)——8px 基 */
:root {
  --sp-1: 8px;  --sp-2: 16px; --sp-3: 24px;
  --sp-4: 32px; --sp-5: 48px; --sp-6: 64px; --sp-7: 96px;
}
/* tight 紧凑——4px 基,数据密集/工具型 */
:root { --sp-1: 4px; --sp-2: 8px; --sp-3: 12px; --sp-4: 16px; --sp-5: 24px; --sp-6: 32px; --sp-7: 48px; }
/* loose 宽松——12px 基,品牌/画廊 */
:root { --sp-1: 12px; --sp-2: 24px; --sp-3: 36px; --sp-4: 48px; --sp-5: 72px; --sp-6: 96px; --sp-7: 144px; }
```

## B. 容器与断点（内容驱动）

```css
/* 容器:max-width + 百分比,永远不用固定 px 全宽 */
.container { width: 100%; max-width: 1120px; margin-inline: auto; padding-inline: clamp(20px, 4vw, 48px); }

/* 内容自适应:能不写媒体查询就不写 */
.card-grid { display: grid; gap: var(--sp-3); grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); }
.fluid-type { font-size: clamp(32px, 5vw, 56px); }

/* 媒体查询只做"形态翻转":多栏↔单栏、导航切换。
   断点值从内容挤压临界点试出来,下面是常见起点(可改) */
@media (min-width: 720px)  { /* 双栏起点 */ }
@media (min-width: 1024px) { /* 侧栏/三栏起点 */ }
```

## C. 页面骨架起点（六种,HTML 注释级）

### C1 营销落地页（单列叙事 + 全宽 Hero + 交替节奏）

```html
<header class="hero">…首屏主张 + 单一 CTA…</header>
<main>
  <section class="feature"><!-- 图左文右 --></section>
  <section class="feature alt"><!-- 文左图右(交替制造节奏,非必须) --></section>
  <section class="proof"><!-- 数据/证言,网格 --></section>
  <section class="cta"><!-- 收口:重复主张 + CTA --></section>
</main>
<footer>…</footer>
```
不用：内容是并列比较(定价表/功能矩阵)时,中段改卡片网格。

### C2 内容/阅读页

```html
<div class="article-layout">   <!-- ≥1024: 1fr 65ch 1fr / 小屏单列 -->
  <aside class="toc">…目录(桌面显示)…</aside>
  <article style="max-width: 68ch; margin-inline:auto;">…正文…</article>
  <aside class="meta">…作者/相关…</aside>
</div>
```

### C3 表单/流程页

```html
<main class="flow" style="max-width:480px; margin-inline:auto; padding: var(--sp-5) var(--sp-3);">
  <h1>…</h1>
  <form>…字段 ≤ 一屏;长表单分步,每步一个任务…</form>
</main>
```
不用：一个页面管理几十个字段的后台表单——那改用分区面板(参考 C4 的分组思想)。

### C4 仪表盘

```html
<div class="shell">          <!-- display:grid; grid: "side main" -->
  <aside class="side">…导航…</aside>
  <main>
    <header class="topbar">…</header>
    <div class="widgets">    <!-- grid: repeat(auto-fit, minmax(240px,1fr)) -->
      <section class="widget">…</section>…
    </div>
  </main>
</div>
```
移动端：侧栏收成抽屉/底部 Tab,widgets 退化单列。

### C5 展示/画廊

```html
<!-- 错落网格:交代给 grid auto-flow dense;或横向叙事(见 ui-interaction-skill ⑨) -->
<div class="masonry" style="columns: 3 260px; column-gap: var(--sp-3);">
  <figure>…</figure>…
</div>
```

### C6 移动 App 式

```html
<div class="app">
  <main class="screen">…当前屏…</main>
  <nav class="tabbar">…3~5 个 Tab,图标+文字…</nav>  <!-- position: fixed; bottom: 0; safe-area -->
</div>
```

## D. 视觉层级（≤4 层的落地方式）

```css
:root {
  --fs-1: clamp(40px, 8vw, 72px);   /* H1 主张 */
  --fs-2: clamp(24px, 4vw, 36px);   /* H2 分区 */
  --fs-3: 18px;                      /* H3/卡片题 */
  --fs-4: 15px;                      /* 正文 */
}
/* 正文以下的小字(标签/注释)算 --fs-4 的字号变体,不算新一层 */
```

## E. 溢出自检（交给 ui-verify-skill 执行）

```js
// 任意视口下:
document.documentElement.scrollWidth - document.documentElement.clientWidth  // 应为 0
// 元素级扫描:遍历 *,找 getBoundingClientRect().right > innerWidth + 1 的元素
```
