# UI 组件与状态 · 配方代码

> Token 占位（`--ink`/`--accent` 等）按 ui-visual-skill 色板替换；
> 动效曲线引用 `--ease: cubic-bezier(0.22,1,0.36,1)` 与 `--spring: cubic-bezier(0.34,1.56,0.64,1)`。

---

## A. 按钮状态矩阵（完整 8 态）

```css
.btn {
  /* 默认 */
  background: var(--surface-2); color: var(--ink);
  border: 1px solid var(--line); border-radius: 4px;
  padding: 12px 22px; min-height: 44px;          /* 触控目标底线 */
  font: 500 13px/1 var(--sans); letter-spacing: 0.04em;
  cursor: pointer; position: relative; overflow: hidden;
  transition: background .2s ease, color .2s ease, transform .25s var(--spring);
}
.btn:hover   { background: var(--ink); color: var(--bg); }       /* 仅悬停设备有意义 */
.btn:active  { transform: scale(0.96); }                          /* 按压即时反馈 */
.btn:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; } /* 键盘焦点环 */
.btn:disabled { opacity: 0.45; cursor: not-allowed; transform: none; }
.btn.loading { color: transparent !important; pointer-events: none; }
.btn.loading::after {
  content: ''; position: absolute; inset: 0; margin: auto;
  width: 16px; height: 16px; border-radius: 50%;
  border: 2px solid var(--line); border-top-color: var(--accent);
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
/* 错误/成功通常由容器或 aria-live 区表达,按钮自身只承担 loading 与禁用 */
@media (prefers-reduced-motion: reduce) { .btn { transition: none; } .btn:active { transform: none; } }
```

```js
// 提交防重三行
form.addEventListener('submit', async (e) => {
  e.preventDefault();
  if (btn.classList.contains('loading')) return;   // 防重
  btn.classList.add('loading');
  try { await save(); /* 成功态 */ } catch { /* 错误态 */ }
  finally { btn.classList.remove('loading'); }
});
```

## B. 输入框校验态（时机：blur 后即时,提交全量,输入中纠错恢复）

```css
.field input {
  width: 100%; min-height: 44px; padding: 12px 14px;
  font-size: 16px;                                  /* 防 iOS 聚焦缩放 */
  border: 1px solid var(--line); border-radius: 8px;
  background: var(--bg); color: var(--ink); outline: none;
  transition: border-color .2s ease;
}
.field input:focus { border-color: var(--accent); }
.field.error input { border-color: var(--danger); }
.field .msg { display: none; font-size: 12px; margin-top: 6px; }
.field.error .msg { display: flex; gap: 5px; align-items: center; color: var(--danger); }
/* 错误 = 颜色+文字+图标 三重提示 */
```

```js
const rules = {
  email: v => /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(v) || '邮箱格式不对,应含 @ 和域名',
};
input.addEventListener('blur', () => validate(input));            // 离开即查
form.addEventListener('submit', e => {                            // 提交全量
  e.preventDefault();
  const ok = [...form.querySelectorAll('input')].map(validate).every(Boolean);
  if (ok) submit();
});
input.addEventListener('input', () => {                           // 改回合法即清错
  if (field.classList.contains('error')) validate(input);
});
function validate(input) {
  const r = rules[input.name]?.(input.value.trim());
  const field = input.closest('.field');
  field.classList.toggle('error', r !== true);
  field.querySelector('.msg').textContent = r === true ? '' : r;
  input.setAttribute('aria-invalid', r !== true);
  return r === true;
}
```

## C. 页面四态（loading/empty/error/success）

```html
<div class="view" data-state="loading">   <!-- JS 切 data-state -->
  <div class="state loading">…骨架屏…</div>
  <div class="state empty">…</div>
  <div class="state error">…</div>
  <div class="state content">…真实内容…</div>
</div>
```
```css
.view [data-state] > .state { display: none; }
.view[data-state="loading"] .state.loading { display: block; }
.view[data-state="empty"]  .state.empty  { display: block; }
.view[data-state="error"]  .state.error  { display: block; }
.view[data-state="ready"]  .state.content{ display: block; }
```

```html
<!-- 骨架屏:结构预览,防布局跳动(shimmer 1.2s 循环) -->
<div class="skeleton" style="height:180px"></div>
<style>
.skeleton {
  border-radius: 8px;
  background: linear-gradient(90deg, var(--surface) 25%, var(--surface-2) 50%, var(--surface) 75%);
  background-size: 200% 100%;
  animation: shimmer 1.2s infinite;
}
@keyframes shimmer { to { background-position: -200% 0; } }
@media (prefers-reduced-motion: reduce) { .skeleton { animation: none; } }
</style>
```

```html
<!-- 空状态:解释 + 出口 -->
<div class="state empty">
  <span class="emoji">🗂️</span>
  <h3>还没有收藏</h3>
  <p>看到喜欢的内容时,点星星就能存到这里。</p>
  <button class="btn">去逛逛 →</button>   <!-- 出口必须有 -->
</div>
<!-- 错误状态:解释 + 重试 -->
<div class="state error">
  <span class="emoji">⚠️</span>
  <h3>加载失败了</h3>
  <p>网络似乎不太稳,内容没有加载出来。</p>
  <button class="btn primary" onclick="retry()">重试</button>
</div>
```

## D. Toast（操作反馈）

```css
.toast {
  position: fixed; left: 50%; bottom: calc(24px + env(safe-area-inset-bottom));
  transform: translate(-50%, 8px); opacity: 0; z-index: 500;
  background: var(--ink); color: var(--bg);
  border-radius: 8px; padding: 11px 18px; font-size: 13px;
  transition: opacity .25s var(--ease), transform .25s var(--ease);
  pointer-events: none; max-width: min(90vw, 420px);
}
.toast.show { opacity: 1; transform: translate(-50%, 0); }
```
```js
function toast(msg, ms = 2400) {
  let t = document.querySelector('.toast');
  if (!t) { t = document.createElement('div'); t.className = 'toast'; document.body.appendChild(t); }
  t.textContent = msg;
  requestAnimationFrame(() => t.classList.add('show'));
  clearTimeout(t._h);
  t._h = setTimeout(() => t.classList.remove('show'), ms);
}
```

## E. 组件状态走查清单（交给 ui-verify-skill 执行）

- [ ] 每个交互组件的 8 态逐个过(存在哪些/表现是否符合 §1)
- [ ] 提交连点不重复发送
- [ ] 校验错误:文字具体、aria 关联、输入合法即清除
- [ ] 空状态有出口、错误状态有重试
- [ ] Toast 不遮挡关键操作、自动消失
