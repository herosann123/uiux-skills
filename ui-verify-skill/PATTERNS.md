# UI 实测验收 · 可复制检测脚本

> 在浏览器 JS 环境执行（如 agent 的 `evaluate`）。全部只读或可复原,不改页面状态。
> 脚本是起点,按页面实际交互增删。

---

## P1 溢出扫描（定位具体元素）

```js
(() => {
  const vw = document.documentElement.clientWidth;
  const docOverflow = document.documentElement.scrollWidth - document.documentElement.clientWidth;
  const bad = [];
  document.querySelectorAll('*').forEach(el => {
    const r = el.getBoundingClientRect();
    if (r.width > 0 && (r.right > vw + 1 || r.left < -1)) {
      bad.push({
        el: el.tagName + (typeof el.className === 'string' ? '.' + el.className.split(' ')[0] : ''),
        left: Math.round(r.left), right: Math.round(r.right),
        position: getComputedStyle(el).position
      });
    }
  });
  return { vw, docOverflow, suspects: bad.slice(0, 12) };
})()
```
判读:`docOverflow > 0` 时 `suspects` 里通常有元凶;注意 `position:absolute/fixed` 的装饰层(裁掉即可)与真实内容溢出的区别。

## P2 可交互元素枚举（走查清单的底稿）

```js
(() => {
  const els = document.querySelectorAll('button, a, input, select, textarea, [onclick], [role="button"], [tabindex]');
  return [...els].map(el => ({
    tag: el.tagName,
    name: el.getAttribute('aria-label') || el.textContent.trim().slice(0, 20) || el.placeholder || '(unnamed)',
    visible: el.getBoundingClientRect().width > 0,
    touchTargetOk: (function () {
      const r = el.getBoundingClientRect();
      return r.width >= 44 && r.height >= 44;
    })(),
    hasFocusVisible: !!el.matches(':focus-visible') // 需 Tab 聚焦后单独查
  }));
})()
```
走查:对每个可见项执行真实点击/输入,记录"动作 → 实际结果"。

## P3 资源与报错

```js
// 页面加载前注入(或用 evaluate 在 load 后立即读):
(() => ({
  brokenImages: [...document.images].filter(i => !i.complete || i.naturalWidth === 0).map(i => i.src.slice(0, 60)),
  fontLoaded: [...document.fonts].filter(f => f.status === 'loaded').map(f => f.family),
  jsErrors: window.__errs || '未挂收集器(需 load 前注入)'
}))()
```

## P4 交互分支验证（触摸路径合成事件）

```js
// 桌面浏览器验证"触摸分支代码路径":临时替换 capture + 指明 pointerType
const fire = (el, type, x, y) => el.dispatchEvent(new PointerEvent(type, {
  pointerId: 7, pointerType: 'touch', isPrimary: true,
  bubbles: true, cancelable: true, clientX: x, clientY: y
}));
// 元素原型补丁(仅测试环境): 合成事件无有效 pointerId
Element.prototype.setPointerCapture = Element.prototype.setPointerCapture || function(){};
// 典型序列: pointerdown → pointermove(≥阈值) → pointerup;另测 pointercancel 复位
```
注意:这只验证**代码路径**,不等于真机手感;真机项标 ⚠️。

## P5 强制触摸环境（matchMedia 桩,验证媒体查询分支）

```js
// 在 <style> 之前注入:
window.matchMedia = (q => (query) => {
  if (/pointer:\s*fine|hover:\s*hover/.test(query)) return { matches: false, media: query };
  if (/pointer:\s*coarse|hover:\s*none/.test(query)) return { matches: true, media: query };
  return window.matchMedia.orig(query);
})(window.matchMedia);
```
只证明分支逻辑正确;CSS `@media` 本身不受 JS 桩影响,仍标注局限。

## P6 视口遍历循环（agent 驱动）

```
for 视口 in [390×844, 320×568, 768×1024, 1024×768, 1280×800]:
    setViewport(视口)
    等 ≥500ms(布局/断点稳定)
    P1 溢出扫描 → 记录
    截图(首屏 + 核心区) → 记录
    横滑/翻转类交互复测 → 记录
```

## P7 键盘可达性走查

```
1. Tab 从地址栏出发逐个聚焦:焦点环是否可见(:focus-visible 样式)
2. 图标按钮:读 aria-label(无 = 未通过)
3. 弹层:打开后 Esc 关闭、遮罩点击关闭、关闭后焦点归还触发元素
```

---

## 验收报告模板

```markdown
# 验收报告 · <页面名>
- 日期/环境/工具:
- 被测版本:(URL 或文件 + 版本参数)
- 本轮改动范围:

| # | 验收项 | 视口 | 状态(✅❌⚠️➖) | 证据(动作→结果/数值/截图路径) |
|---|--------|------|---------------|-------------------------------|

## 新发现问题
## 遗留风险与未验证项
## 结论:整体 [放行 / 不放行 + 原因]
```
