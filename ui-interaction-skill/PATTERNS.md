# UI 交互模式技术手册 · PATTERNS.md

> 24 个模式的原理、代码模板、调参指南、避坑。
> 配套主文档：`./SKILL.md` · 核心 demo：`./10-ui-interactions.html`（①-⑩）· 进阶 demo：`./advanced-ui-v2.html`（⑪-⑰）· 前沿 demo：`./advanced-ui-v3.html`（⑱-㉔）

---

## ① 视差滚动 Parallax Scroll

### 原理
**滚动距离 × 速度系数 = 每层 translateY**。每层系数不同就形成深度差。

### 适用模块
- Hero 主视觉背景层
- 作品集 / 营销活动页背景
- 长图文叙事的章节分隔
- 产品详情页大图背景

### 不适用
- ❌ 长文阅读页（影响阅读）
- ❌ 数据密集工具（干扰效率）

### 核心代码（vanilla）

```javascript
const layers = document.querySelectorAll('[data-speed]');

window.addEventListener('scroll', () => {
  const y = window.scrollY;
  layers.forEach(layer => {
    const speed = parseFloat(layer.dataset.speed);
    layer.style.transform = `translateY(${y * speed}px)`;
  });
}, { passive: true });
```

### GSAP ScrollTrigger 版（推荐）

```javascript
gsap.registerPlugin(ScrollTrigger);

// 每个 layer 独立绑定更灵活
layers.forEach(layer => {
  const speed = parseFloat(layer.dataset.speed);
  gsap.to(layer, {
    y: () => window.innerHeight * speed,
    ease: 'none',
    scrollTrigger: {
      start: 0,
      end: 'max',
      scrub: true
    }
  });
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 速度系数 | 0.1 ~ 0.5 | 0 = 不动，1 = 跟滚动同步 |
| 层数 | ≤ 4 | 超过 4 层开始混乱 |
| 滚动源 | window | 不要用内部 scroll（性能差） |

### 避坑

- 系数差要拉开（如 0.1 / 0.4 / 0.8），相邻接近看不出效果
- 用 `transform: translate3d` 强制 GPU 加速
- 移动端考虑关闭（用 `matchMedia`）

```javascript
ScrollTrigger.matchMedia({
  '(min-width: 768px)': () => { /* 启用 */ },
  '(max-width: 767px)': () => { /* 关闭 */ }
});
```

---

## ② 滚动触发 Scroll-Triggered Reveal

### 原理
**滚动进度 0→1 = 元素动画进度 0→1**。用 IntersectionObserver 或 GSAP ScrollTrigger 的 `scrub` 实现。

### 适用模块
- 内容长文（段落淡入）
- 作品集（作品卡淡入）
- 营销页（卖点依次出现）
- 数据仪表盘（卡片入场）

### 不适用
- ❌ 已经主动态的组件（如 feed 流）
- ❌ 单一独立元素（用 CSS 动画够了）

### 核心代码（vanilla IntersectionObserver）

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('revealed');
      observer.unobserve(entry.target); // 一次性
    }
  });
}, { threshold: 0.2 });

document.querySelectorAll('[data-reveal]').forEach(el => observer.observe(el));
```

### GSAP ScrollTrigger 版（更平滑）

```javascript
gsap.fromTo('.card',
  { scale: 0.8, opacity: 0, y: 60 },
  {
    scale: 1, opacity: 1, y: 0,
    scrollTrigger: {
      trigger: '.card',
      start: 'top 80%',
      end: 'bottom 20%',
      scrub: 0.5,        // 关键：跟手
      toggleActions: 'play none none reverse'
    }
  }
);
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `start` | `top 80%` | 元素顶端进入视口 80% 时触发 |
| `scrub` | `0.5` ~ `true` | 数字 = 跟随延迟秒数；true = 实时 |
| `threshold` | 0.2 | IntersectionObserver 触发比例 |

### 避坑

- 内部 scroll 容器必须显式指定 `scroller`：

```javascript
ScrollTrigger.create({
  trigger: '.inner-scroll',
  scroller: '.inner-scroll',  // ← 关键
  start: 'top top',
  end: 'bottom bottom',
  scrub: true
});
```

- 一次性入场用 `toggleActions: 'play none none none'`，别用 scrub

---

## ③ 卡片堆叠 Card Stack Swipe

### 原理
**跟手阶段**实时绑定 `transform = translate(dx, dy) rotate(dx*0.08)`；
**松手阶段**判定 `|dx| > 阈值` → 飞出 or 弹性回弹。

### 适用模块
- Tinder 式匹配卡
- Feed 流顶部特色卡
- 产品对比卡（一组看一张）
- Onboarding 引导步骤
- 卡片收集 / 评估类应用

### 不适用
- ❌ 传统列表（用 swipe-to-dismiss）
- ❌ 表单卡（不要让用户滑掉）
- ❌ 移动端竖向滑动（桌面端专属体验）

### 核心代码（vanilla pointer events）

```javascript
const card = document.querySelector('.card');
let dx = 0, dy = 0, startX = 0, startY = 0, dragging = false;
const THRESHOLD = 100;

card.addEventListener('pointerdown', (e) => {
  dragging = true;
  startX = e.clientX; startY = e.clientY;
  card.setPointerCapture(e.pointerId);
});

card.addEventListener('pointermove', (e) => {
  if (!dragging) return;
  dx = e.clientX - startX;
  dy = e.clientY - startY;
  card.style.transform = `translate(${dx}px, ${dy}px) rotate(${dx * 0.08}deg)`;
});

card.addEventListener('pointerup', () => {
  if (!dragging) return;
  dragging = false;
  if (Math.abs(dx) > THRESHOLD) {
    const dir = dx > 0 ? 1 : -1;
    gsap.to(card, {
      x: dir * 600, y: dy, rotation: dir * 30, opacity: 0,
      duration: 0.45, ease: 'power3.in',
      onComplete: recycleCard
    });
  } else {
    gsap.to(card, {
      x: 0, y: 0, rotation: 0,
      duration: 0.7, ease: 'elastic.out(1, 0.5)'
    });
  }
});
```

### GSAP Draggable 版（更简洁）

```javascript
Draggable.create(card, {
  type: 'x,y',
  onDrag() {
    this.target.style.transform = `translate(${this.x}px, ${this.y}px) rotate(${this.x * 0.08}deg)`;
  },
  onRelease() {
    if (Math.abs(this.x) > THRESHOLD) {
      gsap.to(this.target, { x: this.x > 0 ? 600 : -600, opacity: 0, duration: 0.45, onComplete: recycle });
    } else {
      gsap.to(this.target, { x: 0, y: 0, rotation: 0, duration: 0.7, ease: 'elastic.out(1, 0.5)' });
    }
  }
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 飞出阈值 | 80 ~ 120 px | 太低容易误触发，太高难操作 |
| 旋转系数 | `dx * 0.05` ~ `0.1` | 越大越夸张 |
| 回弹曲线 | `elastic.out(1, 0.5)` | `s` 越小越有弹性 |
| 飞出距离 | 屏幕宽度 + 100px | 确保完全出屏 |

### 避坑

- 用 `pointer events` 而非 `touch + mouse` 单独处理（PointerEvent 统一两者）
- 跟手时用 `setPointerCapture` 防止鼠标移出丢失
- 循环卡片用 `positions = positions.map(p => (p+1) % N)` 旋转索引，不要操作 DOM

---

## ④ 磁吸按钮 Magnetic Button

### 原理
**指针偏移 × 强度系数 = 按钮 translate**。强度 0.2~0.4 最舒服。

### 适用模块
- 主 CTA 按钮（落地页）
- 关键操作按钮（购买、订阅）
- 桌面端导航主要按钮
- 作品集 / 个人页主入口

### 不适用
- ❌ 移动端（触摸坐标不稳）
- ❌ 表单次要按钮（Submit、Cancel 等）
- ❌ 工具栏按钮群（互相干扰）

### 核心代码

```javascript
const btn = document.querySelector('.magnetic-btn');

btn.addEventListener('mousemove', (e) => {
  const r = btn.getBoundingClientRect();
  const cx = r.left + r.width / 2;
  const cy = r.top + r.height / 2;
  const dx = (e.clientX - cx) * 0.3;   // ← 强度系数
  const dy = (e.clientY - cy) * 0.3;
  btn.style.transform = `translate(${dx}px, ${dy}px)`;
});

btn.addEventListener('mouseleave', () => {
  btn.style.transform = 'translate(0, 0)';
});
```

### 进阶：整块区域磁吸

```javascript
const area = document.querySelector('.magnetic-area');
const btn = area.querySelector('.magnetic-btn');

area.addEventListener('mousemove', (e) => {
  const r = btn.getBoundingClientRect();
  const cx = r.left + r.width / 2;
  const cy = r.top + r.height / 2;
  const dx = (e.clientX - cx) * 0.3;
  const dy = (e.clientY - cy) * 0.3;
  btn.style.transform = `translate(${dx}px, ${dy}px)`;
});

area.addEventListener('mouseleave', () => {
  btn.style.transform = 'translate(0, 0)';
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 强度系数 | 0.2 ~ 0.4 | 0.1 = 轻微；0.5 = 明显；1 = 完全跟手 |
| 监听范围 | 按钮自身 vs 周围 80px | 周围范围更宽容 |
| 回位动画 | `transition: transform 0.4s ease` | 鼠标离开时不要硬跳 |

### 避坑

- 移动端必须禁用（用 `matchMedia('(hover: hover)')` 检测）

```javascript
if (window.matchMedia('(hover: hover)').matches) {
  // 启用磁吸
}
```

- 不要叠加 hover 变色，否则 transform 会冲突
- 文本不要超出按钮边界（按钮是 `display: inline-block`）

---

## ⑤ 文字揭示 Text Reveal

### 原理
**字符串拆词 → 每个词装进 `overflow:hidden` 的 mask → 内部 span 从下方滑入**。
配合 GSAP stagger 控制节奏。

### 适用模块
- Hero 标题
- 营销页卖点
- Onboarding 步骤说明
- 段落引言
- 模态框标题
- Toast 通知

### 不适用
- ❌ 长段落（不要逐字动画）
- ❌ 关键 CTA 文本（延迟用户操作）
- ❌ 表单标签（影响填写节奏）

### 核心代码（vanilla + span 包裹）

```javascript
function splitText(el) {
  const html = el.innerHTML;
  const parts = html.split(/(\s+|<[^>]+>)/);
  let out = '';
  parts.forEach(p => {
    if (p.match(/^\s+$/) || p === '' || p.startsWith('<')) {
      out += p;
    } else {
      p.split(/(\s+)/).forEach(w => {
        if (w.match(/^\s+$/) || w === '') out += w;
        else out += `<span class="word"><span>${w}</span></span>`;
      });
    }
  });
  el.innerHTML = out;
}

document.querySelectorAll('[data-reveal]').forEach(splitText);
```

```css
.word { display: inline-block; overflow: hidden; vertical-align: top; }
.word > span { display: inline-block; transform: translateY(110%); }
```

```javascript
gsap.to('.word > span', {
  yPercent: 0,
  duration: 0.9,
  ease: 'expo.out',
  stagger: 0.045
});
```

### GSAP SplitText 版（需俱乐部会员）

```javascript
const split = new SplitText('.heading', { type: 'words,chars' });
gsap.from(split.words, {
  yPercent: 100,
  duration: 0.9,
  ease: 'expo.out',
  stagger: 0.04
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| stagger | 30 ~ 60 ms | 紧凑=30；从容=60 |
| duration | 0.7 ~ 1.2 s | 短标题 0.7，长标题 1.2 |
| 缓动 | `expo.out` | 推荐，结尾干脆 |
| 单位 | 词 vs 字符 | 词更易读，字符更精致 |

### 避坑

- 保留 `<em>` `<strong>` 等标签不被破坏（用正则保留标签）
- 内联元素 `display: inline-block` 才能 transform
- 容器不能 `white-space: nowrap`，否则只显示一行

---

## ⑥ SVG 路径绘制 Path Drawing

### 原理
**`stroke-dasharray = 路径总长`，`stroke-dashoffset` 从总长动到 0** = 视觉上画出。

### 适用模块
- 启动 Logo / 品牌标识
- Loading 状态（品牌元素）
- 签名 / 笔迹效果
- 数据可视化描边动画
- 图标首次出现
- 数字徽章边框

### 不适用
- ❌ 复杂闭合路径（视觉混乱）
- ❌ 超过 3 个同时绘制（信息过载）
- ❌ 必须立刻可点击的关键按钮

### 核心代码

```javascript
const path = document.querySelector('.path');
const length = path.getTotalLength();

// 初始化：路径全部偏移出去
path.style.strokeDasharray = length;
path.style.strokeDashoffset = length;

// 动画：偏移归 0 = 画出
gsap.to(path, {
  strokeDashoffset: 0,
  duration: 1.8,
  ease: 'power2.inOut'
});
```

### 多路径顺序绘制

```javascript
paths.forEach((p, i) => {
  const L = p.getTotalLength();
  p.style.strokeDasharray = L;
  p.style.strokeDashoffset = L;
  gsap.to(p, {
    strokeDashoffset: 0,
    duration: 1.8,
    delay: i * 0.2,           // ← 顺序延迟
    ease: 'power2.inOut'
  });
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| duration | 1.5 ~ 2.5 s | 太短看不清，太长焦躁 |
| 缓动 | `power2.inOut` | 起笔收笔都自然 |
| stroke-width | 1.5 ~ 3 px | 太粗显笨重 |
| 路径长度 | ≤ 1000 为佳 | 浏览器对超长路径渲染吃力 |

### 避坑

- 必须 `fill: none`，否则会填充内部
- 路径动画结束后要 `strokeDashoffset = 0`，否则虚线错位
- 描边动画**不能**用 `transition`（SVG 属性不受 CSS transition 影响），必须用 GSAP 或 requestAnimationFrame
- 用 `pathLength="1"` 属性可把路径长度标准化为 1，简化计算

```html
<path pathLength="1" stroke-dasharray="1" stroke-dashoffset="1" />
```

---

## ⑦ 弹性回弹 Spring Physics

### 原理
**胡克定律 F = -kx - cv**，每帧更新 `a → v → x`。`k` 越大越硬，`c` 越大越快收敛。

### 适用模块
- 拖拽元素松手回弹
- 加载占位（小球/指示器）
- 按钮按下回弹
- 下拉刷新
- 触底反弹
- Tab 指示器跟随

### 不适用
- ❌ 全局导航过渡（太慢）
- ❌ 表单提交反馈（太突兀）
- ❌ 文字入场（用 expo.out 更合适）

### 核心代码（vanilla rAF）

```javascript
let x = 0, v = 0, target = 0;
const k = 0.08, d = 0.78;  // stiffness, damping

function tick() {
  const a = -k * (x - target) - d * v;
  v += a;
  x += v;
  ball.style.transform = `translate(${x}px, 0)`;
  requestAnimationFrame(tick);
}
tick();
```

### GSAP 版（最简单）

```javascript
gsap.to(ball, {
  x: 0,
  duration: 1,
  ease: 'elastic.out(1, 0.5)'  // 第一个参数是振幅，第二个是周期
});
```

### Framer Motion 版（React 首选）

```javascript
const { x } = useSpring({
  x: 0,
  config: { tension: 200, friction: 20 }  // k, d 的等价物
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| k (stiffness) | 0.05 ~ 0.15 | 越大越快到达 |
| d (damping) | 0.7 ~ 0.85 | 越大越早静止 |
| GSAP elastic.out 参数 | `(1, 0.3)` 弹性强 / `(1, 0.7)` 收敛快 | 第一参 = 振幅，第二参 = 周期 |
| Framer tension | 150 ~ 300 | 越大越弹 |
| Framer friction | 15 ~ 25 | 越大越快停 |

### 调参秘方

| 感觉 | k | d |
|------|---|---|
| 软糖 | 0.05 | 0.6 |
| 标准 | 0.08 | 0.78 |
| 紧实 | 0.12 | 0.85 |
| 金属 | 0.15 | 0.92 |

### 避坑

- 不要用 `cubic-bezier` 模拟弹性（达不到真正的过冲）
- 跟手期间禁用弹簧（会乱跳），松手才启用
- 收敛阈值：`|v| < 0.01 && |x - target| < 0.01` 时停止 rAF

---

## ⑧ 数字翻牌 Counter Flip

### 原理
**0~9 垂直堆叠在高度 = 1.2em 的容器内，显示 n = `translateY(-n * 10%)`**。
配合 ease.expo 出场 = 顺滑翻牌。

### 适用模块
- 数据统计（访问量、销售额、用户数）
- 计时器 / 倒计时
- 评分 / 数字徽章
- 排行榜
- 数字评分（电商产品）

### 不适用
- ❌ 持续变化的数字（如股价）— 翻牌会一直抖
- ❌ 不止 1 个数字同时翻（视觉过载）
- ❌ 字号 < 14px（看不清翻动）

### 核心代码

```javascript
// 1. 准备每位数字的垂直条
function makeDigit(container) {
  for (let i = 0; i < 10; i++) {
    const span = document.createElement('span');
    span.textContent = i;
    container.appendChild(span);
  }
}

// 2. 切换数字
function setDigit(strip, n) {
  gsap.to(strip, {
    yPercent: -n * 10,
    duration: 0.6,
    ease: 'expo.out'
  });
}

// 3. 总动效
const obj = { value: 0 };
gsap.to(obj, {
  value: 12345.67,
  duration: 2.4,
  ease: 'expo.out',
  onUpdate: () => {
    const str = obj.value.toFixed(2).padStart(8, '0');
    [...str].forEach((ch, i) => setDigit(strips[i], parseInt(ch)));
  }
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| duration | 1.5 ~ 2.5 s | 太短数字没看清，太长焦躁 |
| 单帧 duration | 0.5 ~ 0.8 s | 每位数字翻动的速度 |
| 缓动 | `expo.out` | 推荐，先快后慢停 |
| 字数 | ≤ 10 | 超过会变长条 |

### 避坑

- 数字之间用 `,` `.` 分隔符时，要单独占位（不参与翻转）
- 总位数要固定（`padStart`），否则位数变化会跳动
- 用 `clearProps: 'transform'` 清理前一个 tween，避免叠加

---

## ⑨ 3D 倾斜 3D Tilt Follow

### 原理
**父元素 `perspective` 建立 3D 空间；子元素 `rotateX/Y` 与「指针相对中心」偏移成正比**。
配合径向高光跟随 = 真实感拉满。

### 适用模块
- 特性卡片（hover 时倾斜）
- 产品详情卡（hover 展示立体感）
- 电商商品卡（提升点击欲望）
- 个人作品集卡片
- 数据可视化卡片

### 不适用
- ❌ 文字密集卡片（晕动症 + 阅读困难）
- ❌ 移动端（用普通 hover 即可）
- ❌ 列表项（叠加混乱）
- ❌ 关键 CTA（用户焦点被分散）

### 核心代码

```javascript
const card = document.querySelector('.tilt-card');

card.addEventListener('mousemove', (e) => {
  const r = card.getBoundingClientRect();
  const cx = r.left + r.width / 2;
  const cy = r.top + r.height / 2;
  const dx = (e.clientX - cx) / (r.width / 2);   // 归一化 -1~1
  const dy = (e.clientY - cy) / (r.height / 2);

  card.style.transform = `
    perspective(1200px)
    rotateY(${dx * 12}deg)
    rotateX(${-dy * 12}deg)
    scale3d(1.02, 1.02, 1.02)
  `;
});

card.addEventListener('mouseleave', () => {
  card.style.transform = 'perspective(1200px) rotateY(0) rotateX(0) scale3d(1, 1, 1)';
});
```

### 进阶：径向高光

```css
.tilt-card { position: relative; }
.tilt-shine {
  position: absolute;
  inset: 0;
  border-radius: inherit;
  pointer-events: none;
  background: radial-gradient(
    circle at var(--mx, 50%) var(--my, 50%),
    rgba(255,255,255,0.5),
    transparent 40%
  );
  opacity: 0;
  transition: opacity 0.3s ease;
}
.tilt-card:hover .tilt-shine { opacity: 1; }
```

```javascript
shine.style.setProperty('--mx', ((e.clientX - r.left) / r.width * 100) + '%');
shine.style.setProperty('--my', ((e.clientY - r.top) / r.height * 100) + '%');
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| perspective | 800 ~ 1500 px | 越大越平，越小越夸张 |
| 最大旋转角度 | ±8° ~ ±14° | 超过 15° 用户会头晕 |
| scale | 1.02 ~ 1.05 | 微缩放增强悬浮感 |
| 强度系数 | 0.8 ~ 1.0 | 鼠标到边缘才满角度 |

### 避坑

- 移动端用 `matchMedia('(hover: hover)')` 禁用
- 内部图片要用 `transform: translateZ(0)` 否则不动
- `transform-style: preserve-3d` 保证 3D 效果在子元素延伸
- 必须监听 `mouseleave`，否则鼠标移走卡片不变回

---

## ⑩ 玻璃拟态 Glassmorphism

### 原理
**`backdrop-filter: blur()` 模糊后面内容 + 半透明背景 + 1px 内描边**。
blur 是状态属性，**无法直接动画**，只能 toggle class。

### 适用模块
- 模态弹窗 / 抽屉
- 设置面板开关 / 主题切换
- 状态卡（on/off 视觉切换）
- 浮层 / 通知中心
- 播放器控件
- 头像 / 卡片顶部覆盖

### 不适用
- ❌ 纯白背景（看不见效果）
- ❌ 内容密集区域（影响可读性）
- ❌ 低端机 / 微信小程序（性能差）
- ❌ 同一屏 > 2 个 glass 卡（性能崩溃）

### 核心代码

```css
.glass {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 16px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.2);
}
```

### Toggle 切换

```css
.glass.on {
  backdrop-filter: blur(20px) saturate(180%);
  background: rgba(255, 255, 255, 0.2);
}
.glass.off {
  backdrop-filter: blur(0);
  background: rgba(255, 255, 255, 0.05);
}
```

```javascript
document.querySelector('.toggle').addEventListener('click', (e) => {
  e.currentTarget.classList.toggle('on');
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| blur | 12 ~ 24 px | 桌面 20，移动 ≤ 12 |
| saturate | 150% ~ 200% | 提升模糊后颜色饱和度，避免灰蒙 |
| 背景透明度 | 10% ~ 25% | 太透看不清边，太实没玻璃感 |
| 边框透明度 | 25% ~ 40% | 弱化玻璃边缘，提供视觉锚点 |

### 避坑

- 必须加 `-webkit-backdrop-filter`（Safari 兼容性）
- 移动端模糊**必须降级**：

```javascript
const isMobile = window.innerWidth < 768;
document.querySelector('.glass').style.backdropFilter = 
  `blur(${isMobile ? 8 : 20}px) saturate(180%)`;
```

- 玻璃上的文字颜色必须有足够对比度（WCAG AA ≥ 4.5:1）
- `backdrop-filter` 是重绘属性，**不要动画它**（只能 toggle）

---

## ⑪ 极光渐变 Aurora Gradient

### 原理
**多个大尺寸圆形（blob）绝对定位 → 大半径 `blur` → `mix-blend-mode: screen` 叠加**。
每个 blob 用不同动画曲线 + 不同 `animation-delay` 错开，整体感觉像极光缓慢呼吸。
鼠标移动时可以再加一层 `transform: translate()` 微调位置，强化「跟随感」。

### 适用模块
- Hero 主视觉背景层
- 品牌落地页全屏背景
- 产品页第一屏
- 营销活动页 banner
- 玻璃拟态卡的底层（增强对比）

### 不适用
- ❌ 文字密集的内容区（背景喧宾夺主）
- ❌ 列表页 / 后台工具（性能浪费）
- ❌ 与视差滚动叠加（GPU 占用爆炸）
- ❌ 低端移动设备（blur 是重绘重灾区）

### 核心代码（纯 CSS）

```css
.aurora {
  position: absolute;
  inset: 0;
  overflow: hidden;
  background: #0a0a0a;        /* 深色底 */
}

.aurora-blob {
  position: absolute;
  width: 60vw; height: 60vw;
  border-radius: 50%;
  filter: blur(80px) saturate(140%);
  mix-blend-mode: screen;     /* 关键：色彩叠加 */
  animation: float 22s ease-in-out infinite;
}

.blob-1 { background: #ff6b6b; top: -15%; left: -10%; }
.blob-2 { background: #4ecdc4; top: 30%; right: -15%; animation-delay: -7s; }
.blob-3 { background: #ffe66d; bottom: -10%; left: 20%; animation-delay: -14s; }
.blob-4 { background: #a78bfa; top: 10%; left: 40%; animation-delay: -3s; }

@keyframes float {
  0%, 100% { transform: translate(0, 0) scale(1); }
  33%      { transform: translate(40px, -30px) scale(1.1); }
  66%      { transform: translate(-30px, 40px) scale(0.95); }
}
```

### 鼠标跟随增强版

```javascript
const aurora = document.querySelector('.aurora');
aurora.addEventListener('mousemove', (e) => {
  const rect = aurora.getBoundingClientRect();
  const x = (e.clientX - rect.left) / rect.width - 0.5;
  const y = (e.clientY - rect.top) / rect.height - 0.5;
  document.querySelectorAll('.aurora-blob').forEach((blob, i) => {
    blob.style.transform = `translate(${x * (i+1) * 20}px, ${y * (i+1) * 20}px)`;
  });
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| blob 数量 | 3~5 | 太多糊成一片；太少没层次 |
| blur 半径 | 60~120 px | 越大越梦幻，越小越立体 |
| saturate | 120%~160% | 提升模糊后颜色饱和度，避免灰蒙 |
| 动画时长 | 15~30 s | 越慢越"高级"，越快越焦躁 |
| animation-delay | 负数 | 让首屏不全部从 0 开始 |
| 配色 | 同色系或邻近色 | 撞色会显得 low |

### 避坑

- **必须深色底**，浅色底 screen 混合看不见
- 移动端**必须降级**：
  ```css
  @media (max-width: 768px) {
    .aurora { display: none; }
    .container { background: #0a0a0a; }
  }
  ```
- `prefers-reduced-motion` 时直接 `animation: none`
- 不要把 blob 数量加到 6 个以上，移动端会卡

---

## ⑫ 粒子网络 Particle Network

### 原理
**Canvas 2D 绘制 N 个粒子**，每个粒子有位置 + 速度矢量，每帧更新：
1. 粒子按速度漂移
2. 鼠标靠近时，粒子被推开（排斥力）
3. 距离 < 阈值的粒子之间画连线，形成「网络」

### 适用模块
- AI 产品落地页背景
- 科技 / 数据类产品 Hero
- 加载页面装饰
- 仪表盘空状态
- 个人主页背景

### 不适用
- ❌ 内容密集区（喧宾夺主）
- ❌ 长时间停留的页面（耗电 + 视觉疲劳）
- ❌ 移动端首屏（用纯色或静态图替代）
- ❌ 与极光 / 玻璃拟态叠加（性能崩溃）

### 核心代码

```javascript
class Particle {
  constructor(w, h) {
    this.x = Math.random() * w;
    this.y = Math.random() * h;
    this.vx = (Math.random() - 0.5) * 0.5;
    this.vy = (Math.random() - 0.5) * 0.5;
    this.r = Math.random() * 2 + 1;
  }
  update(w, h, mouse) {
    this.x += this.vx;
    this.y += this.vy;
    // 边界反弹
    if (this.x < 0 || this.x > w) this.vx *= -1;
    if (this.y < 0 || this.y > h) this.vy *= -1;
    // 鼠标排斥
    const dx = this.x - mouse.x;
    const dy = this.y - mouse.y;
    const d = Math.hypot(dx, dy);
    if (d < 120) {
      const force = (120 - d) / 120;
      this.x += (dx / d) * force * 3;
      this.y += (dy / d) * force * 3;
    }
  }
  draw(ctx) {
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.r, 0, Math.PI * 2);
    ctx.fillStyle = 'rgba(184, 89, 64, 0.7)';  // 与品牌色一致
    ctx.fill();
  }
}

const canvas = document.getElementById('p');
const ctx = canvas.getContext('2d');
const mouse = { x: -9999, y: -9999 };
let particles = [];

function resize() {
  canvas.width = canvas.offsetWidth;
  canvas.height = canvas.offsetHeight;
  particles = Array.from({ length: 80 }, () => new Particle(canvas.width, canvas.height));
}

canvas.addEventListener('mousemove', (e) => {
  const r = canvas.getBoundingClientRect();
  mouse.x = e.clientX - r.left;
  mouse.y = e.clientY - r.top;
});

canvas.addEventListener('mouseleave', () => {
  mouse.x = mouse.y = -9999;
});

function tick() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  particles.forEach(p => {
    p.update(canvas.width, canvas.height, mouse);
    p.draw(ctx);
  });
  // 画连线
  for (let i = 0; i < particles.length; i++) {
    for (let j = i + 1; j < particles.length; j++) {
      const d = Math.hypot(particles[i].x - particles[j].x, particles[i].y - particles[j].y);
      if (d < 100) {
        ctx.strokeStyle = `rgba(184, 89, 64, ${(100 - d) / 100 * 0.3})`;
        ctx.lineWidth = 0.5;
        ctx.beginPath();
        ctx.moveTo(particles[i].x, particles[i].y);
        ctx.lineTo(particles[j].x, particles[j].y);
        ctx.stroke();
      }
    }
  }
  requestAnimationFrame(tick);
}

resize();
tick();
window.addEventListener('resize', resize);
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 粒子数量 | 60~100（桌面）/ 20~40（移动） | O(n²) 连线，太多掉帧 |
| 排斥半径 | 100~150 px | 太小没存在感，太大画面乱 |
| 连线距离 | 80~120 px | 与排斥半径匹配 |
| 速度 | 0.3~0.8 | 越快越躁动 |
| 粒子颜色 | rgba 带 0.5~0.8 alpha | 不能纯实色，会糊 |

### 避坑

- **`requestAnimationFrame` 不能停**，停了粒子会"凝固"
- 连线循环是 O(n²/2)，100 个粒子 = 4950 次比较，移动端必卡 → 必须降到 40 以下
- 用 `Math.hypot` 而非 `Math.sqrt(dx*dx + dy*dy)`（更快）
- 监听 `mouseleave` 重置 mouse 坐标，否则鼠标离开后粒子还会被排斥
- 必须监听 `resize` 重建粒子（否则窗口变化后粒子飞到外面）
- 不要在 iframe 里跑（性能极差）

---

## ⑬ 水波纹涟漪 Ripple Effect

### 原理
**点击时在按钮内动态插入一个 `position: absolute` 的小圆**（定位 = 鼠标点击位置 - 按钮左上角），CSS 用 `transform: scale(0) → scale(N)` + `opacity: 1 → 0` 同时过渡。
过渡结束后 `remove()` 元素，避免 DOM 堆积。

### 适用模块
- 主 CTA 按钮（点击反馈）
- 表单提交按钮
- 列表项 / 卡片点击
- 移动端 tap 反馈
- 任何需要触觉反馈的可点击元素

### 不适用
- ❌ 图标按钮（小按钮涟漪不明显）
- ❌ 链接 `<a>`（会被 `:active` 自带反馈替代）
- ❌ 频繁点击的元素（DOM 抖动）
- ❌ 与 transform 动效叠加的按钮（坐标系冲突）

### 核心代码

```html
<button class="btn">Click me</button>
```

```css
.btn {
  position: relative;
  overflow: hidden;       /* 关键：涟漪不能溢出 */
  isolation: isolate;     /* 建立层叠上下文 */
}

.ripple {
  position: absolute;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.4);
  transform: scale(0);
  animation: ripple 0.6s ease-out forwards;
  pointer-events: none;
}

@keyframes ripple {
  to {
    transform: scale(4);
    opacity: 0;
  }
}
```

```javascript
document.querySelectorAll('.btn').forEach(btn => {
  btn.addEventListener('click', (e) => {
    const rect = btn.getBoundingClientRect();
    const ripple = document.createElement('span');
    ripple.className = 'ripple';
    const size = Math.max(rect.width, rect.height);
    ripple.style.width = ripple.style.height = size + 'px';
    ripple.style.left = (e.clientX - rect.left - size / 2) + 'px';
    ripple.style.top  = (e.clientY - rect.top  - size / 2) + 'px';
    btn.appendChild(ripple);
    setTimeout(() => ripple.remove(), 700);
  });
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| duration | 500~700 ms | 太短看不见，太长拖沓 |
| 涟漪色 | `rgba(255,255,255,0.4)` 浅底 / `rgba(0,0,0,0.2)` 深底 | 与按钮主色对比 |
| scale | 3~5 | 必须大到能覆盖整个按钮 |
| ease | `ease-out` | 起手快，收尾自然 |

### 避坑

- **必须** `position: relative` + `overflow: hidden`，否则涟漪溢出
- **必须** `isolation: isolate` 或 `z-index` 隔离，否则被父级 transform 影响
- 涟漪**必须从点击位置发出**，不能从按钮中心（廉价感）
- 移动端要兼容触摸：用 `pointerdown` 而非 `click`（延迟低 100ms）
- 频繁点击时用**节流**，或用 CSS `animation` 复用同名元素
- 涟漪元素必须有 `pointer-events: none`，否则拦截后续点击

---

## ⑭ 粘性滚动堆叠 Sticky Scroll Stack

### 原理
**容器高度 = N × 视口高度**（让滚动条够长），内部 stage 用 `position: sticky; top: 0; height: 100vh` 把舞台"钉"住。
多个面板叠在 stage 内，scroll 进度 → 面板透明度/缩放/位移。
GSAP ScrollTrigger 的 `pin: true` 是最简洁实现。

### 适用模块
- 作品集项目分镜展示
- 产品流程介绍（设计 → 开发 → 上线）
- Onboarding 引导（多步骤）
- 品牌故事长图文
- 案例研究的章节切换

### 不适用
- ❌ 内容密集的长文阅读页
- ❌ 移动端（sticky 在 iOS Safari 有 bug）
- ❌ 超过 5 屏的堆叠（滚动疲劳）
- ❌ 列表 / 表格类内容
- ❌ 与视差背景叠加（容易乱套）

### 核心代码（CSS + GSAP）

```html
<div class="container">       <!-- 总高度 = 3 × 100vh -->
  <div class="stage">          <!-- sticky 钉住 -->
    <div class="panel panel-0">Step 1</div>
    <div class="panel panel-1">Step 2</div>
    <div class="panel panel-2">Step 3</div>
  </div>
</div>
```

```css
.container { height: 300vh; }
.stage {
  position: sticky;
  top: 0;
  height: 100vh;
  overflow: hidden;
}
.panel {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transform: scale(0.85);
}
```

```javascript
gsap.timeline({
  scrollTrigger: {
    trigger: '.container',
    start: 'top top',
    end: 'bottom bottom',
    scrub: 1,           // 平滑跟随
    pin: '.stage'       // 自动 pin（不用手动 sticky）
  }
})
.to('.panel-0', { opacity: 1, scale: 1, duration: 1 }, 0)
.to('.panel-0', { opacity: 0, scale: 0.85, duration: 1 }, 1)
.to('.panel-1', { opacity: 1, scale: 1, duration: 1 }, 1)
.to('.panel-1', { opacity: 0, scale: 0.85, duration: 1 }, 2)
.to('.panel-2', { opacity: 1, scale: 1, duration: 1 }, 2);
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 面板数 | 3~5 | 太多滚动疲劳 |
| scrub | 0.5~1 | 数字 = 跟随秒数；越大越"慢电影" |
| scale 起始 | 0.85~0.95 | 营造"放大入场"感 |
| 单屏时长 | 0.8~1.2 (timeline 单位) | 太短看不出，太长拖沓 |
| 进入动效 | opacity + scale | 不要加位移，会和 sticky 冲突 |

### 避坑

- 移动端**必须降级**为普通竖排：
  ```javascript
  ScrollTrigger.matchMedia({
    '(min-width: 768px)': () => { /* 启用 timeline */ },
    '(max-width: 767px)': () => { /* 改成普通 stack */ }
  });
  ```
- 容器**必须有高度**（3~5 × 100vh），否则没滚动空间
- 进度指示器（dot）必须随 panel 同步切换
- `scrub` 不要用 `true`（太硬），用数字（秒）平滑
- iOS Safari 上 `position: sticky` 嵌套过深会失效 → 容器不要超过 5 层

---

## ⑮ FLIP 共享元素 FLIP Shared Element

### 原理
**F.L.I.P. 四步法**（由 Paul Lewis 提出）：

1. **First** — 记录源元素的 `getBoundingClientRect()`
2. **Last** — 让目标元素到达最终位置，立即再读取一次 rect
3. **Invert** — 计算两次 rect 的 `deltaX/Y` `scaleX/Y`，给目标元素应用反向 transform，让它**看起来还在 First 位置**
4. **Play** — 用 `transition` 把 transform 归零，配合 `requestAnimationFrame` 触发渲染

> 这是 Google Photos、Airbnb 等做"列表→详情无缝变形"的核心技术。

### 适用模块
- 作品集（缩略图 → 全屏案例）
- 电商（商品卡 → 商品详情）
- 相册（缩略图 → 大图）
- 新闻列表（卡片 → 文章）
- 音乐 App（歌曲卡 → 播放器）

### 不适用
- ❌ 列表项结构差异大（共享元素无法定位）
- ❌ 频繁切换的列表（性能 + 视觉疲劳）
- ❌ 没有稳定 key/id 的元素（无法做动画）

### 核心代码

```javascript
function open(card) {
  const detail = document.getElementById('detail');
  const inner = document.getElementById('detailInner');

  // 1. First — 记录源元素位置
  const firstRect = card.getBoundingClientRect();

  // 2. Last — 把目标元素先放到源位置（视觉上覆盖）
  inner.style.position = 'fixed';
  inner.style.left = firstRect.left + 'px';
  inner.style.top = firstRect.top + 'px';
  inner.style.width = firstRect.width + 'px';
  inner.style.height = firstRect.height + 'px';
  inner.style.transition = 'none';

  // 显示详情容器
  detail.classList.add('open');

  // 3. Invert — 计算 delta，下一帧再读 Last
  requestAnimationFrame(() => {
    // 读取最终位置（详情已展开）
    const lastRect = inner.getBoundingClientRect();
    const dx = firstRect.left - lastRect.left;
    const dy = firstRect.top - lastRect.top;
    const sx = firstRect.width / lastRect.width;
    const sy = firstRect.height / lastRect.height;

    // 应用反向 transform
    inner.style.transform = `translate(${dx}px, ${dy}px) scale(${sx}, ${sy})`;
    inner.style.transformOrigin = 'top left';

    // 4. Play — 用 transition 归零
    requestAnimationFrame(() => {
      inner.style.transition = 'transform 0.5s cubic-bezier(0.4, 0, 0.2, 1)';
      inner.style.transform = 'none';
    });
  });
}

function close() {
  // 同样的 FLIP 反向：详情 → 源位置
  // ...
}
```

### React / Vue 版（共享元素库）

```javascript
// React：react-flip-toolkit
import { Flipper, Flipped } from 'react-flip-toolkit';
<Flipper flipKey={selectedId}>
  {items.map(item => (
    <Flipped key={item.id} flipId={item.id}>
      <Card data={item} onClick={() => setSelected(item.id)} />
    </Flipped>
  ))}
</Flipper>

// Vue：vue-flip-toolkit 类似 API
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| duration | 350~500 ms | 太短看不出"变形"，太长像 PPT |
| ease | `cubic-bezier(0.4, 0, 0.2, 1)` | 标准 material ease；不要用弹性 |
| transform-origin | `top left` 或源元素中心 | 决定缩放基准点 |
| 动画属性 | transform（必须） | 不要动画 width/height（layout 重排） |

### 避坑

- **必须**用 `requestAnimationFrame` 双层嵌套（先取消 transition → 再读 rect → 再开 transition）
- 列表项**必须**有稳定的 `flipId` / `key`，否则动画错位
- 源元素和目标元素的**内容必须能识别为同一对象**（图片、标题），否则变形像 bug
- 关闭时也要做 FLIP（反向），不能直接 fade
- 用 `position: fixed` 而非 absolute，避免父级 transform 影响
- transform 必须从 `none` 归零，不能从 `translate(0,0)`（性能差）

---

## ⑯ View Transitions API

### 原理
**浏览器原生 API**：把"DOM 更新"包在 `document.startViewTransition(cb)` 里，浏览器自动对所有变化元素做 cross-fade + scale。
可以用 `::view-transition-*` CSS 伪元素精细控制每个元素的过渡方式。
**Chrome 111+ / Edge 支持；Safari 18+ / Firefox 仍在推进。**

### 适用模块
- 主题切换（Light / Dark）
- SPA 路由切换
- Tab 切换
- 模态框打开 / 关闭
- 列表过滤（show/hide）
- 任何"切换可见性"的场景

### 不适用
- ❌ 老浏览器（必须降级）
- ❌ 复杂动画（API 默认 cross-fade，自定义成本高）
- ❌ 需要兼容 iOS 14- 的项目

### 核心代码

```javascript
// 检测支持
const supportsVT = typeof document.startViewTransition === 'function';

// 主题切换
toggle.addEventListener('click', () => {
  if (!supportsVT) {
    document.body.classList.toggle('dark');
    return;  // 老浏览器直接切换，无过渡
  }
  document.startViewTransition(() => {
    document.body.classList.toggle('dark');
  });
});
```

### 自定义某个元素的过渡

```css
/* 主题切换时，logo 元素特殊过渡（旋转 + 缩放） */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 0.3s;
}

::view-transition-old(logo),
::view-transition-new(logo) {
  animation-duration: 0.4s;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}
```

```javascript
// 给元素起 view-transition-name
document.querySelector('.logo').style.viewTransitionName = 'logo';
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 默认 duration | 250~300 ms | 浏览器默认约 250ms |
| 复杂场景 | 400~500 ms | 自定义元素可单独设 |
| ease | 默认 `ease` | 可改 `cubic-bezier(0.4, 0, 0.2, 1)` 更顺滑 |

### 避坑

- **必须**包成 feature detection：
  ```javascript
  if (document.startViewTransition) { ... }
  ```
- 主题切换的 CSS 变量必须在 `:root` 和 `:root.dark` 都定义
- `view-transition-name` 必须**全局唯一**，否则冲突
- 频繁点击时连续 transition 会重叠 → 防抖
- 不要在 transition 中改 DOM（会破坏快照）

---

## ⑰ AI 流式打字机 AI Typewriter

### 原理
**逐字符 setState（不是 innerHTML 整段写入）**：
1. 用 `setTimeout`/`requestAnimationFrame` 递归调用
2. 每个字符延迟**随机化**（20~80ms），让节奏像真人
3. **标点处自动慢一拍**（中文：，。？！；英文：, . ! ? ;）
4. 末尾加**闪烁光标**，告诉用户"还没完"

### 适用模块
- AI Chat（ChatGPT / Claude / Gemini 风格）
- AI 生成的内容预览（图片描述、代码生成）
- AI 配音 / 字幕
- Loading 占位（"AI 正在思考..."）
- 引导页文案逐句登场

### 不适用
- ❌ 长段落（> 200 字）— 用户会不耐烦，要提供"跳过"
- ❌ 关键操作反馈（用户要点按钮）
- ❌ 表单提示（影响填写节奏）

### 核心代码

```javascript
function typeWriter(text, el, speed = 'normal') {
  let i = 0;
  const ranges = {
    fast:   [10, 30],
    normal: [20, 80],
    slow:   [80, 200]
  };
  const [min, max] = ranges[speed];

  function tick() {
    if (i >= text.length) return;
    el.textContent += text[i++];

    // 可变延迟
    let delay = min + Math.random() * (max - min);

    // 标点处慢一拍
    const prev = text[i - 1];
    if ('，。？！；, . ! ? ;'.includes(prev)) {
      delay = max * 2;
    }
    // 换行后稍微停顿（更像换气）
    if (prev === '\n') delay = max * 1.5;

    setTimeout(tick, delay);
  }
  tick();
}

// 使用
typeWriter('你好世界！这是一段流式输出。', document.getElementById('out'));
```

### 闪烁光标

```css
.cursor {
  display: inline-block;
  width: 2px;
  height: 1em;
  background: currentColor;
  margin-left: 2px;
  vertical-align: text-bottom;
  animation: blink 0.8s steps(2) infinite;
}

@keyframes blink {
  50% { opacity: 0; }
}
```

```javascript
// 流式结束时隐藏光标
function typeWriter(text, el) {
  // ... 同上
  if (i >= text.length) {
    document.querySelector('.cursor')?.classList.add('done');
    return;
  }
  // ...
}
```

### React 版（最常用）

```jsx
function AIResponse({ text }) {
  const [displayed, setDisplayed] = useState('');

  useEffect(() => {
    setDisplayed('');  // 重置
    let i = 0;
    let timer;
    function tick() {
      if (i >= text.length) return;
      setDisplayed(text.slice(0, i + 1));
      i++;
      const ch = text[i - 1];
      let delay = 20 + Math.random() * 60;
      if ('，。？！；, . ! ? ;'.includes(ch)) delay = 160;
      timer = setTimeout(tick, delay);
    }
    tick();
    return () => clearTimeout(timer);
  }, [text]);

  return <span>{displayed}<span className="cursor" /></span>;
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 字符延迟 | 20~80 ms | 太慢焦躁，太快不像 AI |
| 标点延迟 | 字符 max × 2~3 | 让"停顿"自然 |
| 速度档位 | 3 档（快/中/慢） | 用户可控是核心 |
| 光标闪烁 | 0.8~1s, steps(2) | 不要 ease，要 steps（硬切换） |

### 避坑

- **必须**保留换行符：把 `white-space: pre-wrap` 加到容器
- 完成后**必须隐藏光标**（否则一直闪）
- 中文按字符切，英文按字符切；emoji 占 2 个字符宽度
- 用户中途点击"停止"时立即 `clearTimeout` + 显示完整
- 长文本必须提供"跳过"按钮
- 不要用 `innerHTML += char`（会重新解析，性能差 + 闪烁）
- SSE / WebSocket 流式接收时，**逐 token 触发** typeWriter，而不是等全部到位再开始
- 中文文本注意不要把整段当一个 string 写入 → 必须按字符迭代

---

## 附录 A：组合配方完整代码

### Premium Hero

```javascript
// 视差背景
gsap.utils.toArray('[data-speed]').forEach(layer => {
  gsap.to(layer, {
    y: () => window.innerHeight * parseFloat(layer.dataset.speed),
    ease: 'none',
    scrollTrigger: { scrub: true }
  });
});

// 标题文字揭示
const split = new SplitText('.hero-title', { type: 'words' });
gsap.from(split.words, {
  yPercent: 100,
  duration: 0.9,
  ease: 'expo.out',
  stagger: 0.05,
  delay: 0.3
});

// CTA 磁吸
cta.addEventListener('mousemove', (e) => {
  const r = cta.getBoundingClientRect();
  cta.style.transform = `translate(${(e.clientX - r.left - r.width/2) * 0.3}px, 
                                   ${(e.clientY - r.top - r.height/2) * 0.3}px)`;
});
```

### Tinder Discover

```javascript
// 卡片堆叠
Draggable.create('.card-top', {
  type: 'x,y',
  onDrag() {
    this.target.style.transform = `translate(${this.x}px, ${this.y}px) rotate(${this.x * 0.08}deg)`;
  },
  onRelease() {
    if (Math.abs(this.x) > 100) {
      gsap.to(this.target, { x: this.x > 0 ? 600 : -600, opacity: 0, duration: 0.4, onComplete: recycle });
    } else {
      gsap.to(this.target, { x: 0, y: 0, rotation: 0, duration: 0.7, ease: 'elastic.out(1, 0.5)' });
    }
  }
});

// 底部操作按钮 - 弹性反馈
actionBtn.addEventListener('click', () => {
  gsap.fromTo(actionBtn, { scale: 0.85 }, { scale: 1, duration: 0.5, ease: 'elastic.out(1, 0.5)' });
});
```

### Dashboard Premium

```javascript
// 数字翻牌
const obj = { value: 0 };
gsap.to(obj, {
  value: 1234567,
  duration: 2,
  ease: 'expo.out',
  onUpdate: () => setCounter(obj.value)
});

// 玻璃拟态开关
switchEl.addEventListener('click', () => {
  switchEl.classList.toggle('on');
  panelEl.classList.toggle('glass-on');
});
```

### Aurora Hero（⑪ + ⑤ + ⑬）

```html
<!-- 4 个 blob + 玻璃卡 -->
<div class="hero">
  <div class="aurora"><div class="blob b1"></div><div class="blob b2"></div>
                     <div class="blob b3"></div><div class="blob b4"></div></div>
  <h1 class="title">你的标题</h1>
  <button class="cta">立即开始</button>
</div>
```

```css
.aurora { position: absolute; inset: 0; overflow: hidden; }
.blob {
  position: absolute; width: 60vw; height: 60vw; border-radius: 50%;
  filter: blur(80px) saturate(140%); mix-blend-mode: screen;
  animation: float 22s ease-in-out infinite;
}
.cta { position: relative; overflow: hidden; }
```

```javascript
// 标题文字揭示
gsap.from('.title', { y: 60, opacity: 0, duration: 1, ease: 'expo.out', delay: 0.3 });

// CTA 涟漪
document.querySelectorAll('.cta').forEach(btn => {
  btn.addEventListener('click', (e) => {
    const rect = btn.getBoundingClientRect();
    const ripple = document.createElement('span');
    ripple.className = 'ripple';
    const size = Math.max(rect.width, rect.height);
    ripple.style.cssText = `width:${size}px;height:${size}px;left:${e.clientX-rect.left-size/2}px;top:${e.clientY-rect.top-size/2}px`;
    btn.appendChild(ripple);
    setTimeout(() => ripple.remove(), 700);
  });
});
```

### AI Product Demo（⑫ + ⑰ + ⑯）

```javascript
// 1. 粒子背景（⑫）
const canvas = document.getElementById('particles');
const ctx = canvas.getContext('2d');
const particles = Array.from({ length: 80 }, () => ({
  x: Math.random() * canvas.width, y: Math.random() * canvas.height,
  vx: (Math.random() - 0.5) * 0.5, vy: (Math.random() - 0.5) * 0.5
}));
// ... tick() 持续画粒子 + 连线

// 2. 流式打字机（⑰）— 配合 SSE
const eventSource = new EventSource('/api/chat');
const bubble = document.getElementById('bubble');
let displayed = '';
eventSource.onmessage = (e) => {
  const token = e.data;
  displayed += token;
  bubble.textContent = displayed;
};

// 3. 主题切换用 View Transitions（⑯）
document.querySelector('.theme-toggle').addEventListener('click', () => {
  if (document.startViewTransition) {
    document.startViewTransition(() => document.body.classList.toggle('dark'));
  } else {
    document.body.classList.toggle('dark');
  }
});
```

### Portfolio Showcase（⑪ + ⑭ + ⑮）

```javascript
// 极光背景（⑪）+ 粘性堆叠（⑭）+ FLIP（⑮）组合
// 详见各模式代码，本质是 3 个独立 init 串行加载
initAurora();   // 极光
initSticky();   // 粘性堆叠（作品章节）
initFLIP();     // 卡片 → 详情
```

---

## 附录 B：prefers-reduced-motion 通用模板

```javascript
const motionQuery = window.matchMedia('(prefers-reduced-motion: reduce)');

function initMotion() {
  if (motionQuery.matches) {
    // 用户偏好减少动效：所有元素直接显示 final state
    gsap.set('.animatable', { clearProps: 'all' });
    document.querySelectorAll('.animatable').forEach(el => {
      el.style.opacity = 1;
      el.style.transform = 'none';
    });
    return; // 跳过所有动效初始化
  }

  // 正常初始化动效
  initParallax();
  initScrollTrigger();
  initTextReveal();
  // ...
}

// 监听变化（用户在系统设置里改也会响应）
motionQuery.addEventListener('change', initMotion);
initMotion();
```

---

## 附录 C：常用检测工具

```javascript
// 检测 hover 能力（区分桌面/移动）
const supportsHover = window.matchMedia('(hover: hover)').matches;

// 检测触摸设备
const isTouch = 'ontouchstart' in window;

// 检测 Safari（backdrop-filter 兼容性）
const isSafari = /^((?!chrome|android).)*safari/i.test(navigator.userAgent);

// 检测性能等级
const isLowPerf = navigator.hardwareConcurrency < 4 || navigator.deviceMemory < 4;

// 综合判断：是否启用复杂动效
const enableAdvancedMotion = supportsHover && !isLowPerf;
```

---

## ⑱ 自定义光标 Custom Cursor

### 原理
**隐藏原生光标 + 渲染双层（点立即跟随 + 环阻尼惯性）**。每帧用 rAF + lerp 让外环追赶内点位置；悬停可交互元素时给外环加 class 放大/换色。

### 适用模块
- Linear / Vercel / Stripe 风的高端产品页
- 作品集 / 个人品牌页
- 任何希望「交互入口本身就是视觉语言」的页面

### 不适用
- ❌ 移动端（`@media (hover: none)` 直接隐藏）
- ❌ 数据密集工具（分散注意力）

### 核心代码

```javascript
const dot = document.createElement('div'); dot.className = 'cursor-dot';
const ring = document.createElement('div'); ring.className = 'cursor-ring';
document.body.appendChild(dot);
document.body.appendChild(ring);

let mx = window.innerWidth/2, my = window.innerHeight/2;
let rx = mx, ry = my;

document.addEventListener('mousemove', (e) => {
  mx = e.clientX; my = e.clientY;
  dot.style.transform = `translate(${mx}px, ${my}px) translate(-50%,-50%)`;
}, { passive: true });

(function tick() {
  rx += (mx - rx) * 0.18;     // 阻尼 0.18 视觉最舒服
  ry += (my - ry) * 0.18;
  ring.style.transform = `translate(${rx}px, ${ry}px) translate(-50%,-50%)`;
  requestAnimationFrame(tick);
})();

document.querySelectorAll('.hover-target, a, button').forEach(el => {
  el.addEventListener('mouseenter', () => ring.classList.add('hover'));
  el.addEventListener('mouseleave', () => ring.classList.remove('hover'));
});
```

```css
body { cursor: none; }
@media (max-width: 900px) { body { cursor: auto; } }
.cursor-dot, .cursor-ring {
  position: fixed; pointer-events: none; z-index: 9999;
  transform: translate(-50%, -50%);
}
.cursor-dot { width: 6px; height: 6px; background: var(--accent); border-radius: 50%; }
.cursor-ring {
  width: 36px; height: 36px; border: 1.5px solid var(--ink);
  border-radius: 50%;
  transition: width .25s ease, height .25s ease, border-color .25s ease;
}
.cursor-ring.hover {
  width: 60px; height: 60px;
  border-color: var(--accent);
  background: rgba(184, 89, 64, 0.08);
}
```

### 调参指南
- **lerp 0.18** 是甜蜜点：0.1 太粘，0.4 太快失去惯性
- **环初始 36px / hover 60px**：放大比例 ≈ 1.7x
- 加 `.cursor-trail`（小点跟随，opacity 0.4）增加层次但不要超过 3 个

### 无障碍 fallback
```javascript
const supportsHover = window.matchMedia('(hover: hover)').matches;
if (!supportsHover) {
  dot.remove(); ring.remove();
  document.body.style.cursor = 'auto';
  return;
}
```

---

## ⑲ 水平滚动叙事 Horizontal Scroll

### 原理
**垂直滚动 → 水平平移**。GSAP ScrollTrigger `pin` 容器，`scrub` 让 `track` 的 `x` 跟 `scrollY` 同步。

### 适用模块
- Apple 风产品页 / 发布会式滚动
- 品牌叙事页（4~6 屏横向作品集）
- Onboarding 引导（横屏分步骤）

### 不适用
- ❌ 移动端（移动端要改成竖排卡片堆叠）
- ❌ 短页面（横屏 1~2 屏无意义）

### 核心代码（GSAP）

```javascript
const wrapper = document.querySelector('.hscroll-wrapper');
const track = document.querySelector('.hscroll-track');

gsap.to(track, {
  x: () => -(track.scrollWidth - wrapper.clientWidth),
  ease: 'none',
  scrollTrigger: {
    trigger: wrapper,
    start: 'top top',
    end: () => `+=${track.scrollWidth - wrapper.clientWidth + 100}`,
    pin: true,                  // 钉住 wrapper
    scrub: 1,                   // 1 秒平滑
    invalidateOnRefresh: true,  // resize 时重算
    onUpdate: (self) => {
      const idx = Math.min(3, Math.floor(self.progress * 4));
      document.querySelectorAll('.hscroll-pip').forEach((p, i) =>
        p.classList.toggle('active', i === idx)
      );
    }
  }
});
```

```html
<div class="hscroll-wrapper" style="height: 600px; overflow: hidden;">
  <div class="hscroll-track" style="display: flex; width: 400%;">
    <div class="hscroll-panel">…</div>
    <div class="hscroll-panel">…</div>
    <div class="hscroll-panel">…</div>
    <div class="hscroll-panel">…</div>
  </div>
</div>
```

### 调参指南
- **scrub: 1** 比 `true` 更可控（可调秒数）
- **end** 比 `track.scrollWidth - wrapper.clientWidth` 多 +100~200 是缓冲
- **`pinSpacing: true`**（默认）让容器外留出滚动距离

### 无障碍 fallback
```javascript
const isMobile = window.innerWidth < 900;
if (isMobile) {
  track.style.flexDirection = 'column';
  track.style.width = '100%';
}
```

---

## ⑳ 噪点纹理 Noise Overlay

### 原理
**Canvas 每帧生成随机灰度像素**，叠在背景上 + `mix-blend-mode: overlay/soft-light` + 短间隔抖动 = 「胶片 / 印刷品」质感。

### 适用模块
- 暗色 Hero 背景
- 高端品牌落地页
- 设计工具 / 创意产品的"贵气"暗示

### 不适用
- ❌ 浅色背景（噪点会被吃掉）
- ❌ 性能敏感场景（必须 0.5x 分辨率）

### 核心代码

```javascript
const canvas = document.getElementById('noiseCanvas');
const ctx = canvas.getContext('2d');

function setupNoise() {
  const rect = canvas.parentElement.getBoundingClientRect();
  canvas.width = Math.floor(rect.width * 0.5);   // 关键：0.5x 分辨率
  canvas.height = Math.floor(rect.height * 0.5);
}

function renderNoise() {
  const w = canvas.width, h = canvas.height;
  const img = ctx.createImageData(w, h);
  const d = img.data;
  for (let i = 0; i < d.length; i += 4) {
    const v = (Math.random() * 255) | 0;
    d[i] = d[i+1] = d[i+2] = v;
    d[i+3] = 255;
  }
  ctx.putImageData(img, 0, 0);
  requestAnimationFrame(renderNoise);
}

setupNoise();
renderNoise();
window.addEventListener('resize', setupNoise);
```

```css
.noise-canvas {
  position: absolute; inset: 0;
  width: 100%; height: 100%;
  opacity: 0.25;
  mix-blend-mode: overlay;
}
```

### 性能要点
- **必须 0.5x 分辨率**（否则帧率爆炸）
- **提供开关按钮**（低性能设备用户主动关闭）
- 静态方案：`SVG feTurbulence`（不抖但便宜）

### 无障碍 fallback
```javascript
if (motionQuery.matches) {
  cancelAnimationFrame(raf);
  canvas.style.opacity = '0.1';
}
```

---

## ㉑ RGB 分离 Glitch

### 原理
**同一文字 3 层（R/G/B），`mix-blend-mode: screen` 叠加；仅在关键帧的 90%~98% 区间微抖位置**，创造「信号失稳」感。

### 适用模块
- Linear / Vercel 落地页的 H1
- 故障感 / Cyber 风品牌
- Tech 产品发布会标题

### 不适用
- ❌ 商务 / 金融 / 医疗（视觉冲突）
- ❌ 长时间停留的内容（用户视觉疲劳）

### 核心代码

```html
<h2 class="glitch-title">
  <span class="layer red" aria-hidden="true">FRACTURED.</span>
  <span class="layer blue" aria-hidden="true">FRACTURED.</span>
  <span class="base">FRACTURED.</span>
</h2>
```

```css
.glitch-title { position: relative; display: inline-block; }
.glitch-title .layer {
  position: absolute; top: 0; left: 0;
  width: 100%; height: 100%;
  mix-blend-mode: screen;
  display: block;
}
.glitch-title .layer.red {
  color: #ff3030;
  animation: glitchRed 3s infinite linear alternate-reverse;
}
.glitch-title .layer.blue {
  color: #3030ff;
  animation: glitchBlue 2.7s infinite linear alternate-reverse;
}
@keyframes glitchRed {
  0%, 90%, 100% { transform: translate(0, 0); }
  92% { transform: translate(-3px, 1px); }
  94% { transform: translate(2px, -1px); }
  96% { transform: translate(-2px, 2px); }
  98% { transform: translate(3px, 0); }
}
@keyframes glitchBlue {
  0%, 88%, 100% { transform: translate(0, 0); }
  90% { transform: translate(3px, -1px); }
  93% { transform: translate(-2px, 1px); }
  96% { transform: translate(2px, -2px); }
}
```

### 调参指南
- **偏移 2~4px** 是上限；超过 6px 会像真正的 bug
- **抖动区间 90%~98%**（不是全程）—— 关键是「偶发」而不是「持续」
- 3 层动画时长略微错开（3s / 2.7s / 3.3s）避免规律感

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .glitch-title .layer { display: none; }
  .glitch-title .base { color: inherit; }
}
```

---

## ㉒ 滚动驱动数字 Scroll-Driven Numbers

### 原理
**ScrollTrigger.progress (0~1) → lerp 到目标数字 (0~N)**。数字随用户滚动「滚出来」。

### 适用模块
- Apple 产品参数页
- 财报页 / 数据长图 / 年度回顾
- 性能指标展示

### 不适用
- ❌ 单屏静态页面（没滚动可监听）
- ❌ 多组数字密集更新（用户认知跟不上）

### 核心代码

```javascript
const numEl = document.getElementById('scrollNumber');
let current = 0;

ScrollTrigger.create({
  trigger: track,                // 比 sticky 容器更高的轨道
  start: 'top top',
  end: 'bottom bottom',
  onUpdate: (self) => {
    const target = Math.round(self.progress * 240);
    current += (target - current) * 0.15;   // lerp 0.15 平滑
    numEl.textContent = Math.round(current);
  }
});
```

```css
.scroll-number {
  font-variant-numeric: tabular-nums;   /* 关键：数字等宽 */
  font-family: var(--serif);
}
```

### 调参指南
- **lerp 0.15** 让数字追手但有惯性
- **`font-variant-numeric: tabular-nums`** 必须开，否则数字宽度变化导致布局抖动
- 单位（fps / % / MB）用斜体衬线字体放在数字旁边 —— 制造 Apple 风

### 无障碍 fallback
```javascript
if (motionQuery.matches) {
  numEl.textContent = '240';  // 直接显示最终值
  return;
}
```

---

## ㉓ 命令栏 Command Palette (Cmd+K)

### 原理
**全局快捷键 Cmd/Ctrl+K 唤起浮层 + 模糊匹配 + 键盘导航**。按 ↑↓ 移动高亮，Enter 执行，Esc 关闭。

### 适用模块
- IDE / Dev Tool / 设计工具
- Notion / Linear / GitHub / Vercel / Raycast
- 任何"功能多但入口不能塞满导航栏"的产品

### 不适用
- ❌ 5 个功能的简单网站（命令栏反而是负担）
- ❌ 移动端（移动端直接用全屏搜索页）

### 核心代码

```javascript
const overlay = document.getElementById('cmdkOverlay');
const input = document.getElementById('cmdkInput');
const list = document.getElementById('cmdkList');

const commands = [
  { group: 'Navigation', icon: '◐', title: 'Go to Dashboard', desc: '...', kbd: 'G D' },
  { group: 'Actions',     icon: '✎', title: 'New Document',    desc: '...', kbd: 'N' },
];

// 模糊匹配：query 字符在 title 中按顺序出现即可
function fuzzyMatch(text, q) {
  const t = text.toLowerCase(), query = q.toLowerCase();
  let qi = 0;
  for (let i = 0; i < t.length && qi < query.length; i++) {
    if (t[i] === query[qi]) qi++;
  }
  return qi === query.length;
}

// 关键词高亮
function highlight(text, q) {
  if (!q) return text;
  const re = new RegExp('(' + q.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + ')', 'gi');
  return text.replace(re, '<mark>$1</mark>');
}

// 全局快捷键
document.addEventListener('keydown', (e) => {
  if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
    e.preventDefault();
    overlay.classList.toggle('open');
  }
  if (e.key === 'Escape') overlay.classList.remove('open');
});
```

```css
.cmdk-overlay {
  position: fixed; inset: 0;
  background: rgba(0,0,0,0.4);
  backdrop-filter: blur(8px);
  display: none; align-items: flex-start; justify-content: center;
  padding-top: 12vh;
}
.cmdk-overlay.open { display: flex; }
.cmdk-panel {
  width: 100%; max-width: 640px;
  background: var(--bg); border-radius: 12px;
  box-shadow: 0 24px 64px rgba(0,0,0,0.2);
  transform: scale(0.95) translateY(-10px); opacity: 0;
  transition: all 0.18s cubic-bezier(0.4, 0, 0.2, 1);
}
.cmdk-overlay.open .cmdk-panel { transform: scale(1) translateY(0); opacity: 1; }
```

### 必备功能清单
- [x] **Cmd/Ctrl+K** 唤起
- [x] **Esc** 关闭
- [x] **↑↓** 上下导航
- [x] **Enter** 执行选中
- [x] **关键词高亮**（`<mark>` 包裹匹配字符）
- [x] **分组标签**
- [x] **空状态文案**

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .cmdk-panel { transition: none; transform: none; opacity: 1; }
}
```

---

## ㉔ Blob Morph 液态形变

### 原理
**SVG `<path>` 的 `d` 属性在多组「同点数」路径之间用 GSAP 平滑插值**，配合渐变填充产生「液态金属」质感。Stripe / Linear 落地页招牌元素。

### 适用模块
- Stripe 风 Hero 空状态插画
- Loading 占位（让等待本身变成视觉享受）
- 品牌 icon / mascot 呼吸动画
- 空状态插画

### 不适用
- ❌ 复杂的角色 / 精细图案（点数太多 GSAP 算不动）
- ❌ 需要文字可识别的场景（形变期间无法阅读）

### 核心代码

```javascript
// 关键：所有 path 必须有相同的「锚点数」M/C 命令数量
const paths = [
  'M100,20 C140,20 180,60 180,100 C180,140 140,180 100,180 C60,180 20,140 20,100 C20,60 60,20 100,20 Z',  // circle
  'M100,20 C140,20 140,60 180,80 C200,100 180,140 140,140 C140,180 100,200 80,160 C60,200 20,180 40,140 C0,140 0,100 20,80 C0,60 40,20 80,40 C80,20 100,20 100,20 Z',  // flower
  'M100,20 L120,80 L180,80 L130,120 L150,180 L100,140 L50,180 L70,120 L20,80 L80,80 Z',  // star
  // ...
];

const shape = document.getElementById('blobShape');

function morphTo(idx) {
  gsap.to(shape, {
    attr: { d: paths[idx] },
    duration: 1.4,
    ease: 'elastic.out(1, 0.55)'  // 弹性回归
  });
}

document.querySelectorAll('.blob-key').forEach((k, i) =>
  k.addEventListener('click', () => morphTo(i))
);
```

```html
<svg class="blob-svg" viewBox="0 0 200 200">
  <defs>
    <linearGradient id="blobGradient" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" stop-color="#b85940"/>
      <stop offset="50%" stop-color="#c9a368"/>
      <stop offset="100%" stop-color="#2d4a3e"/>
    </linearGradient>
  </defs>
  <path class="blob-shape" id="blobShape"
        d="M100,20 C140,20 180,60 180,100 ..." fill="url(#blobGradient)"/>
</svg>
```

### 调参指南
- **点数必须一致**：每个 path 用相同数量的 `M/C/L` 命令，否则 morph 会跳变
- **弹性 `elastic.out(1, 0.55)`**：amplitude=1, period=0.55 —— 视觉最有"果冻感"
- **duration 1.2~1.6s**：太快失去"液态"感，太慢显得拖沓
- **3~6 个形状**是合理上限；太多用户记不住差异

### 制作 path 工具
- Figma / Illustrator 画好形状 → 导出 SVG path → **手动补齐点数**
- 或者用 `flubber-js` 库自动处理点数对齐

### 无障碍 fallback
```javascript
if (motionQuery.matches) {
  shape.setAttribute('d', paths[0]);   // 静态第一个
  // 不启动 setInterval 自动循环
}
```

---

## ㉕ Folding Drawer 千层酥抽拉

### 原理
**多张卡同位定位 → 每张按 `data-stack-pos` 累加 translate+scale+opacity → 顶卡激活时向左前方"抽拉"滑出**。与 ③ 卡片堆叠（拖拽阈值飞出）相反：㉕ 是**键盘 / 滚轮 / 触摸驱动**的「轮播式抽拉」，每张卡都保留可见边缘（千层酥视觉）。

### 适用模块
- 颜色方案 / 色卡展示（用户实测场景）
- 设计系统组件库切换
- 作品集分类入口（每张=一个项目）
- 报告 / 仪表盘的视角切换
- 任何"有限集合 + 顺序浏览"的内容（≤ 8 项最佳）

### 不适用
- ❌ 长列表 / Feed（用普通滚动）
- ❌ 内容需要详细阅读（千层酥的尺寸不利于阅读）
- ❌ 需要拖拽飞出的 Tinder 式匹配（用 ③）

### 核心代码

```html
<div class="stage">
  <div class="deck" id="deck">
    <div class="card" data-color="1">
      <div class="face"><!-- 内容 --></div>
      <div class="bookmark">
        <span class="bookmark-dot"></span>
        <div class="bookmark-text">
          <div class="bookmark-num">01</div>
          <div class="bookmark-title">Coral Sunset</div>
        </div>
      </div>
    </div>
    <!-- 更多卡片 -->
  </div>
</div>
```

```css
/* —— 卡片层叠（只动 transform / opacity） —— */
.card {
  position: absolute; top: 0; left: 0;
  width: 380px; height: 440px;
  transition:
    transform 0.75s cubic-bezier(0.34, 1.22, 0.64, 1),
    opacity 0.55s ease,
    box-shadow 0.5s ease;
  will-change: transform, opacity;
}

/* —— 激活态：向左前方"抽拉"出来 —— */
.card.active {
  transform: translate(-14px, -8px) scale(1.02);
  z-index: 100;
  box-shadow:
    0 4px 10px rgba(0,0,0,0.14),
    0 24px 50px rgba(0,0,0,0.32),
    0 60px 120px rgba(0,0,0,0.38);
}

/* —— 千层酥偏移阶梯 —— */
.card[data-stack-pos="1"] { transform: translate(20px, 14px) scale(0.94); opacity: 0.85; }
.card[data-stack-pos="2"] { transform: translate(40px, 28px) scale(0.88); opacity: 0.68; }
.card[data-stack-pos="3"] { transform: translate(60px, 42px) scale(0.82); opacity: 0.50; }
.card[data-stack-pos="4"] { transform: translate(80px, 56px) scale(0.76); opacity: 0.32; }
.card[data-stack-pos="5"] { transform: translate(100px, 70px) scale(0.70); opacity: 0.18; }

/* —— 书签：从卡片右边缘"伸出来"（作为 .card 子元素，跟着 transform 走） —— */
.bookmark {
  position: absolute;
  right: -130px; top: 28px;
  width: 130px; height: 52px;
  cursor: pointer;
  touch-action: manipulation;
}
.card.active .bookmark { opacity: 0.32; filter: saturate(0.4); }   /* 用户偏好：激活=已消费=暗 */
```

```javascript
const cards = Array.from(document.querySelectorAll('.card'));
const order = cards.slice();   // [0] = 当前激活

function applyStack() {
  order.forEach((card, i) => {
    if (i === 0) { card.classList.add('active'); card.removeAttribute('data-stack-pos'); }
    else         { card.classList.remove('active'); card.dataset.stackPos = i; }
  });
}

function next() { order.push(order.shift()); applyStack(); }
function prev() { order.unshift(order.pop()); applyStack(); }
function goTo(card) {
  const idx = order.indexOf(card); if (idx <= 0) return;
  order.splice(idx, 1); order.unshift(card); applyStack();
}

// 滚轮（锁定 200ms 防止疯狂触发）
let wheelLock = false;
window.addEventListener('wheel', (e) => {
  e.preventDefault();
  if (wheelLock || Math.abs(e.deltaY) < 4) return;
  wheelLock = true; setTimeout(() => wheelLock = false, 200);
  e.deltaY > 0 ? next() : prev();
}, { passive: false });

// 触摸（40px 阈值）
let ty = null;
window.addEventListener('touchstart', (e) => { ty = e.touches[0].clientY; }, { passive: true });
window.addEventListener('touchend', (e) => {
  if (ty === null) return;
  const dy = ty - e.changedTouches[0].clientY;
  if (Math.abs(dy) > 40) dy > 0 ? next() : prev();
  ty = null;
}, { passive: true });

// 键盘
window.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowDown' || e.key === 'ArrowRight') { e.preventDefault(); next(); }
  if (e.key === 'ArrowUp'   || e.key === 'ArrowLeft')  { e.preventDefault(); prev(); }
});

// 书签点击直接跳转
cards.forEach(c => c.querySelector('.bookmark')?.addEventListener('click', (e) => {
  e.stopPropagation(); goTo(c);
}));

applyStack();
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 卡片数量 | ≤ 8 | 超过 8 张后面的层完全不可见 |
| 偏移步长 | 20px / 14px | x/y 偏移量；差值 ≥ 6 否则看不出层次 |
| 缩放 | 0.94 → 0.70 | 5 级递减；≥ 6 级最后一级 ≤ 0.66 失去边缘感 |
| 透明度 | 1 → 0.18 | 末位 0.18 是「可感知但已不抢戏」的甜区 |
| 抽出动画 | `cubic-bezier(0.34, 1.22, 0.64, 1)` | 弹性过冲 ~20%，比 plain ease 更有「拉出」感 |
| duration | 0.6~0.8s | 太短像闪烁，太长显得拖沓 |
| wheel 锁 | 200ms | 防意外连续滚；不要太长否则跟手感差 |
| touch 阈值 | 40px | 太小误触，太大不易触发 |

### 必备功能清单
- [x] **滚轮切换** + 200ms 防抖
- [x] **键盘 ↑↓←→** 切换
- [x] **触摸滑动** ≥40px 切换
- [x] **书签点击**直接跳转（不是装饰）
- [x] **active 视觉态**：transform + scale + 强 box-shadow + 书签高亮（亮的）
- [x] **非 active 视觉态**：偏移+缩放+降透明度 + 书签变暗（≤0.3）
- [x] **书签语义对照**：正在展示的卡 = 亮书签；折叠在堆里的卡 = 暗书签
- [x] **千层酥边缘保留**：最后一级 opacity ≥ 0.18，scale ≥ 0.70
- [x] **counter** 显示当前位置
- [x] **响应式**：5 档断点（1024/720/540/400 + 横屏）
- [x] **`prefers-reduced-motion`** 关闭 transition

### 反模式（必须避免）
- ❌ **旋转/折叠 3D 效果** → 用户实测反馈：「千层酥只要抽拉感，不要旋转」；旋转显得廉价且难读
- ❌ **动 `width/height`** → 每帧 layout 重排，掉帧；只动 `transform` + `opacity`
- ❌ **书签做成独立列表** → 失去「从卡延伸出来」的物理感；书签必须是 `.card` 的子元素
- ❌ **隐藏后续卡（opacity 0）** → 千层酥的核心是「边缘可见」；末位也保留 0.18 透明度
- ❌ **层级数 > 5** → 偏移累计会超出容器，视觉上也读不出层次
- ❌ **键盘响应把页面滚动了** → 一定要 `e.preventDefault()`，否则页面跟着滚
- ❌ **wheel 不 preventDefault** → 桌面端页面会滚动
- ❌ **末位卡缩放 < 0.66** → 边缘太窄看不见

### 移动端铁律
- `touch-action: pan-y`：保留页面竖向手势，但允许 JS 监听 touchstart/end
- `.bookmark` 加 `-webkit-tap-highlight-color: transparent` + `touch-action: manipulation`
- **card + bookmark 总宽必须 ≤ 视口宽度**（在 540/400px 断点收窄）
- 横屏（`max-height: 500px`）压缩卡高 + 字号
- 在 540px 以下隐藏 `meta` / `hint` 等装饰元素

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .card, .bookmark { transition: none !important; }
  .card.active { transform: none; }   /* 顶卡不"抽拉"出来，但仍可见 */
}
```

```javascript
// 键盘 alt/ctrl 修饰键不响应（避免与屏幕阅读器冲突）
// 已经过 viewport 检查：基本所有屏幕阅读器都支持 data-stack-pos / data-color attribute
```

### 与其他模式的关系
- **vs ③ 卡片堆叠**：③ 是「拖拽阈值飞出 / 弹性回弹」（Tinder 风）；㉕ 是「键盘/滚轮/触摸驱动轮播」（色卡 / 配色面板风）。两者完全不重叠
- **vs ⑮ FLIP 共享元素**：⑯ 是「卡片 → 详情页」的过渡；㉕ 是「同页内集合浏览」
- **可叠加 ⑤ 文字揭示**（千层酥顶卡的标题文字逐字入场）或 ⑧ 数字翻牌（counter 数字变化时翻牌）

---

## ㉖ 原生手感滚动 Native-Feel Scrolling

### 原理
**优先用浏览器原生滚动（transform 交给合成线程），再用 `scroll-snap` 做吸附、`overscroll-behavior` 做边界、CSS `scroll-behavior` / Lenis 式 lerp 做平滑**。「顺滑」的本质 = 惯性（momentum）+ 吸附（snap）+ 不掉帧（合成线程滚动）。

> 铁律：**永远不要用 JS 每帧改 `scrollTop`/`translateY` 去劫持原生滚动**（`wheel`/`touchmove` 里 preventDefault 再手写位移 = 掉帧 + 惯性手感全失）。要平滑就用 `scroll-snap` + `scrollend` 事件；要「丝滑匀速」整页滚动才用 Lenis。

### 适用模块
- 手机端任何横向卡片轨道（首页金刚区、精选推荐、Stories）
- 全屏引导页（Onboarding，每屏一页）
- 图片浏览器（一图一屏）
- 需要平滑滚动的长落地页（Lenis）

### 不适用
- ❌ 竖向信息流（用普通原生滚动即可，snap 反而烦人）
- ❌ 桌面端鼠标滚轮 + 强制 snap（滚半格卡住，体验差；桌面用 JS 分页按钮）

### 核心代码（横向 snap 轨道 —— 90% 的移动端场景）

```html
<div class="track">
  <div class="card">…</div>
  <div class="card">…</div>
  <div class="card">…</div>
</div>
```

```css
.track {
  display: flex;
  gap: 12px;
  overflow-x: auto;
  scroll-snap-type: x mandatory;   /*  proximity = 松手自由，mandatory = 必吸附 */
  scroll-padding: 0 20px;          /* 吸附时两侧留白对齐 */
  overscroll-behavior-x: contain;  /* 滚到头不传染给页面 */
  -webkit-overflow-scrolling: touch; /* 老 iOS 惯性 */
  scrollbar-width: none;           /* FF 隐藏滚动条 */
}
.track::-webkit-scrollbar { display: none; }
.card {
  flex: 0 0 min(78vw, 320px);      /* 78vw：下一张露一角，暗示可滑 */
  scroll-snap-align: start;
  scroll-margin: 0 6px;
}
```

```javascript
// 分页点 / 进度指示：监听 scroll，用 rAF 节流 + lerp 平滑
const track = document.querySelector('.track');
let ticking = false;
track.addEventListener('scroll', () => {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    const idx = Math.round(track.scrollLeft / (track.scrollWidth - track.clientWidth) * (N - 1));
    dots.forEach((d, i) => d.classList.toggle('on', i === idx));
    ticking = false;
  });
}, { passive: true });   // passive:true 是性能铁律，别 preventDefault
```

### 进阶：Lenis 式平滑滚动（整页丝滑，可选）

```javascript
// 只适合「整页叙事落地页」；信息流禁用（用户要的是快速定位，不是丝滑）
let target = 0, current = 0;
window.addEventListener('wheel', (e) => {
  if (reducedMotion) return;
  e.preventDefault();
  target = clamp(target + e.deltaY, 0, maxScroll);
}, { passive: false });

function raf() {
  current += (target - current) * 0.09;   // lerp 0.08~0.12
  window.scrollTo(0, current);
  requestAnimationFrame(raf);
}
requestAnimationFrame(raf);
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| snap 类型 | `x mandatory` | 引导页/Stories 用 mandatory；商品轨道用 proximity 更宽容 |
| 卡宽 | `min(78vw, 320px)` | 露出下一张 22%，是最强的「可滑动」暗示 |
| lerp | 0.08~0.12 | Lenis 默认 0.1；越小越「漂」，越大越「跟」 |
| `overscroll-behavior` | `contain` | 内滚容器必加，防滚动穿透 + 禁掉 Chrome 亮发光下拉 |
| 吸附动画 | `scroll-snap-stop: always` | 高速滑动时强制一张一张停（Stories 翻页），普通轨道用默认 |

### 避坑
- ❌ **JS 劫持 touchmove 手写滚动** → 惯性没了、掉帧、和系统手势打架；原生滚动 + snap 即可
- ❌ **scroll 监听里直接改 DOM 不节流** → 一帧 60 次回调，必卡；rAF 节流
- ❌ **嵌套轨道不加 `overscroll-behavior: contain`** → 横轨滚完自动滚页面，用户「被甩走」
- ❌ **mandatory snap 卡片间距不均** → 吸附位置错乱；统一用 gap + scroll-padding
- ❌ **100vh 容器内 snap** → 移动端地址栏伸缩导致 vh 抖动，用 `100dvh`
- ❌ 桌面鼠标滚轮 + mandatory → 滚轮一次跨多张但被 snap 拽回来，桌面端配按钮或改 `proximity`

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .track { scroll-snap-type: none; scroll-behavior: auto; }
  /* Lenis 平滑滚动直接不启动（见上面 JS 的 reducedMotion 判断） */
}
```
- 键盘可达：卡片加 `tabindex="0"`，方向键聚焦后原生滚动即可翻页

---

## ㉗ 滑动返回 Interactive Swipe-Back

### 原理
**左边缘 20px 热区监听 pointerdown → 新页面 `translateX` 跟手（0→100%）→ 松手按位移/速度二选一：回弹 or 完成返回**。精髓在「跟手 + 可中断」：用户按住页面能拖到一半再推回去，这是原生 App 和 Web 之间最大的手感分水岭。

### 适用模块
- 手机端 H5 / WebApp 的页面栈（详情页、二级页返回）
- 全屏图片查看器返回
- 小程序自定义返回手势（配合 wx 页面栈）

### 不适用
- ❌ 桌面端（无触摸，直接给返回按钮/浏览器后退）
- ❌ 页面内有横向滚动轨道（手势冲突；需热区判定 + 首次 move 方向锁）

### 核心代码

```javascript
const EDGE = 24;                 // 边缘热区宽度 px
const page = document.querySelector('.page');
let startX = 0, startY = 0, tracking = false, locked = null;

document.addEventListener('pointerdown', (e) => {
  if (e.clientX > EDGE) return;          // 只在左边缘热区生效
  tracking = true; locked = null;
  startX = e.clientX; startY = e.clientY;
  page.style.transition = 'none';        // 跟手阶段关掉 transition
});

document.addEventListener('pointermove', (e) => {
  if (!tracking) return;
  const dx = e.clientX - startX, dy = e.clientY - startY;
  if (!locked) {                          // 首次 move 判定意图：横 or 竖
    if (Math.abs(dx) < 8 && Math.abs(dy) < 8) return;
    locked = Math.abs(dx) > Math.abs(dy) ? 'h' : 'v';
    if (locked === 'v') { tracking = false; return; }   // 竖向让给页面滚动
  }
  const progress = clamp(dx / innerWidth, 0, 1);
  page.style.transform = `translateX(${progress * 100}%)`;
  // 背后露出上一层：跟着 progress 变暗/缩放
  underlay.style.transform = `translateX(${(progress - 1) * 30}%) scale(${0.92 + progress * 0.08})`;
});

document.addEventListener('pointerup', (e) => {
  if (!tracking || locked !== 'h') return;
  tracking = false;
  const dx = e.clientX - startX;
  const vx = dx / (performance.now() - t0);           // px/ms 速度
  const commit = dx > innerWidth * 0.4 || vx > 0.5;    // 位移 40% 或快速轻扫
  page.style.transition = 'transform 0.32s cubic-bezier(0.32, 0.72, 0, 1)';
  if (commit) {
    page.style.transform = 'translateX(100%)';
    page.addEventListener('transitionend', () => history.back(), { once: true });
  } else {
    page.style.transform = 'translateX(0)';
  }
});
document.addEventListener('pointercancel', () => {    // 来电/系统手势中断
  tracking = false;
  page.style.transition = 'transform 0.32s cubic-bezier(0.32, 0.72, 0, 1)';
  page.style.transform = 'translateX(0)';
});
```

```css
.page {
  position: fixed; inset: 0;
  background: #fff;
  box-shadow: -4px 0 24px rgba(0,0,0,0.18);   /* 页面右侧投影暗示层级 */
  touch-action: pan-y;    /* 竖向交给原生滚动，横向归手势 */
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 边缘热区 | 20~28px | iOS 22px；太宽误触竖滑 |
| 提交阈值 | 位移 >40% 宽 **或** 速度 >0.5 px/ms | 满足其一即提交；快速轻扫很短距离也能返回 |
| 回弹/完成曲线 | `cubic-bezier(0.32, 0.72, 0, 1)` | iOS 标准「快进缓出」，0.3~0.35s |
| 底层视差 | `translateX((p-1)*30%)` | 上层页拖出时底层从 -30% 跟进，制造景深 |
| 拖拽阻尼 | progress = dx / width 直接 1:1 | 不要加阻尼——返回手势要完全跟手 |

### 避坑
- ❌ **跟手阶段开着 CSS transition** → 页面永远慢半拍，必须先 `transition: none`
- ❌ **不锁方向** → 页面里有竖向滚动时，斜着滑就误触发返回；首次 move 判定 h/v 后锁定
- ❌ **只按位移不按速度判定** → 用户快速轻扫 60px 没反应，觉得「失灵」
- ❌ **忘记 `pointercancel`** → 来电/通知中心下拉中断后页面卡在半路
- ❌ **commit 后直接 history.back()** → 过渡动画被页面卸载吃掉；等 `transitionend` 再退
- ❌ 全屏热区（`pointerdown` 无边缘判断）→ 每次点按都开始拖页面，输入框没法用

### 无障碍 fallback
- 手势是**增强不是唯一入口**：左上角必须保留可见的返回按钮
- `prefers-reduced-motion: reduce` 时直接 `history.back()` 无动画

---

## ㉘ Tab 滑动切换 Swipeable Tabs（跟手指示器）

### 原理
**内容区 = 原生横向 snap 滚动容器（㉖），指示器 = 一个绝对定位的滑块，`transform: translateX(scrollLeft / (w_content - w_view) * (w_track - w_thumb))` 用 scrollLeft 驱动**。内容靠原生滚动天然跟手 + 惯性，指示器用 JS 同步 —— 不用任何 transform 库就能做出「iOS 分段控制」的手感。

### 适用模块
- 手机端顶部 Tab（首页/发现/关注）
- 分段控制（Segmented Control）
- 全屏 Pager（Onboarding、 Stories）
- 桌面 + 移动统一的 Tab 页

### 不适用
- ❌ Tab 数 > 6（横滑发现成本太高；改纵向抽屉导航）
- ❌ Tab 内容含复杂表单（误滑切页丢输入状态）

### 核心代码

```html
<nav class="tabs" role="tablist">
  <div class="ink-track"><div class="ink" id="ink"></div></div>
  <button role="tab" class="tab on">推荐</button>
  <button role="tab" class="tab">关注</button>
  <button role="tab" class="tab">视频</button>
</nav>
<div class="pages" id="pages">
  <section class="page">…</section>
  <section class="page">…</section>
  <section class="page">…</section>
</div>
```

```css
.pages {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  overscroll-behavior-x: contain;
  scrollbar-width: none;
}
.pages::-webkit-scrollbar { display: none; }
.page { flex: 0 0 100%; scroll-snap-align: center; }

.tabs { position: relative; display: flex; }
.ink {
  position: absolute; bottom: 0; left: 0;
  height: 3px; border-radius: 3px;
  background: currentColor;
  width: var(--tab-w);
  will-change: transform;
  transition: none;          /* 跟手阶段由 JS transform 驱动，不需要 transition */
}
```

```javascript
const pages = document.getElementById('pages');
const ink = document.getElementById('ink');
const tabs = [...document.querySelectorAll('.tab')];
let ticking = false;

pages.addEventListener('scroll', () => {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    const max = pages.scrollWidth - pages.clientWidth;
    const p = max ? pages.scrollLeft / max : 0;          // 0~1 全局进度
    // 指示器 x = p * (指示器轨道宽 - 指示器宽)
    ink.style.transform = `translateX(${p * inkTrackFree}px)`;
    ticking = false;
  });
}, { passive: true });

// 点击 Tab → 平滑滚动到对应页（与手势共用同一条轨道）
tabs.forEach((t, i) => t.addEventListener('click', () => {
  pages.scrollTo({ left: i * pages.clientWidth, behavior: 'smooth' });
}));

// scrollend 更新选中态（Chrome 114+；不支持则用 setTimeout 兜底）
pages.addEventListener('scrollend', () => {
  const i = Math.round(pages.scrollLeft / pages.clientWidth);
  tabs.forEach((t, j) => t.classList.toggle('on', i === j));
  tabs[i].focus({ preventScroll: true });
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 指示器动画 | 跟手无 transition；点 Tab 时靠 `scroll-behavior: smooth` | 指示器天然平滑，别再加 CSS transition 造成二次平滑 |
| 指示器宽度 | Tab 平均宽或与文字等宽 | 与文字同宽更精致（需要 JS 量文字宽） |
| scrollend 兜底 | 不支持时 `setTimeout(120ms)` 节流改选中态 | Safari 2024 年才支持 scrollend |
| 页数上限 | ≤ 6 | 超过改纵向导航 |

### 避坑
- ❌ **用 JS transform 手写 Pager** → 丢惯性、丢原生滚动性能；snap 容器白送的
- ❌ **指示器位置用 `left` 动画** → 每帧重排；必须 `transform: translateX`
- ❌ **scroll 回调里改 Tab 选中态** → 滑动过程中高亮乱跳；选中态等 `scrollend`
- ❌ **页面内竖向滚动 + 容器 `touch-action: none`** → 内容没法上下滚；容器不设 none，靠方向判定
- ❌ **点击 Tab 用 `pages.scrollLeft = x` 赋值** → 瞬移没有平滑；用 `scrollTo({behavior:'smooth'})`
- ❌ **Tab 宽度不定还用等分指示器** → 指示器和文字对不齐，廉价感来源第一名

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .pages { scroll-behavior: auto; scroll-snap-type: none; }
}
```
- `role="tablist" / role="tab" / aria-selected` 完整语义；方向键切 Tab（`keydown` → scrollTo）

---

## ㉙ 页面推入/推出过渡 Push / Pop Transition

### 原理
**进入页从 `translateX(100%)` 滑入，退出页同步 `translateX(-25%)` 视差后退 + 变暗** —— 双层同动 + iOS 曲线 = 「原生导航」的体感。返回（pop）则镜像：进入页滑回右侧，上一页从 -25% 归位。关键是**两页同时动**，单页滑入的「廉价滑出」一眼假。

### 适用模块
- 手机 WebApp 路由切换（列表 → 详情）
- 任何「层级关系」的页面切换（下钻）
- 替代 `View Transitions` 在移动端路由的默认 fade

### 不适用
- ❌ 平级切换（Tab 间用 ㉘；模态用 ㉚）
- ❌ 长内容页过渡时整页 blur（性能崩）

### 核心代码

```css
:root { --ios-ease: cubic-bezier(0.32, 0.72, 0, 1); }

.stack { position: relative; }
.page {
  position: fixed; inset: 0;
  background: var(--bg);
  transition: transform 0.38s var(--ios-ease), filter 0.38s ease, opacity 0.38s ease;
}
.page.below {                 /* 被压在下层的页面 */
  transform: translateX(-25%) scale(0.98);
  filter: brightness(0.6);
  pointer-events: none;
  z-index: 1;
}
.page.top {                    /* 当前页 */
  transform: translateX(0);
  z-index: 2;
}
.page.entering {               /* push 进来的新页 */
  z-index: 3;
  animation: pushIn 0.38s var(--ios-ease) both;
}
.page.exiting {                /* 即将登场（被推出视口） */
  animation: pushOut 0.38s var(--ios-ease) both;
}
@keyframes pushIn  { from { transform: translateX(100%); } to { transform: translateX(0); } }
@keyframes pushOut { from { transform: translateX(0);   } to { transform: translateX(-25%) scale(0.98); filter: brightness(0.6); } }
```

```javascript
function push(newPageEl) {
  const current = document.querySelector('.page.top');
  current.classList.replace('top', 'exiting');
  newPageEl.classList.add('entering');
  newPageEl.addEventListener('animationend', () => {
    current.classList.replace('exiting', 'below');
    newPageEl.classList.replace('entering', 'top');
  }, { once: true });
}
function pop() {   // 镜像
  const top = document.querySelector('.page.top');
  const below = document.querySelector('.page.below');
  below.classList.add('exiting');                  // 反向动画
  top.style.transform = 'translateX(100%)';
  top.addEventListener('transitionend', () => {
    top.remove();
    below.classList.replace('exiting', 'top');     // below 的过渡回 0
  }, { once: true });
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| duration | 0.35~0.45s | iOS ≈ 0.35s；超过 0.5s 拖沓，低于 0.25s 看不清层级 |
| 曲线 | `cubic-bezier(0.32, 0.72, 0, 1)` | iOS 导航曲线；终点前几乎匀速、无过冲（导航过渡不要 spring 过冲） |
| 退出页位移 | -22%~-28% + 变暗 | 位移小于 15% 感知不到景深，大于 35% 抢戏 |
| 层级数 | ≤ 3 | 每层都是 fixed DOM；超过 3 层内存/滚动位置管理复杂 |
| 新页入场 | 大标题文字可加 ⑤ stagger 揭示 | 页面停在 0 后再开始，别同步 |

### 避坑
- ❌ **只有新页动、旧页静止** → 一眼 Web 味；双层同动是灵魂
- ❌ **用 spring/elastic 做导航** → 过冲在层级切换里像 bug（参考 SKILL §7.2）
- ❌ **过渡期间旧页还能点** → `pointer-events: none` 必须加
- ❌ **返回后滚动位置丢失** → 页面用 `display:none` 而非 `remove()`；或记录 `scrollY` 恢复
- ❌ **`100vh` 页面 + `position: fixed`** → 地址栏伸缩抖动；用 `100dvh`
- ❌ **pop 方向做反（新页从左进）** → 下钻右进、返回右出，方向反了用户直接迷路

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .page { transition: none !important; animation: none !important; }
  .page.below { display: none; }   /* 直接切，无层级视差 */
}
```

### 与其他模式的关系
- **vs ⑯ View Transitions**：⑮/⑯ 是通用共享元素/路由过渡；㉙ 是**移动端下钻导航专用**的双层 push/pop，语义更强
- **vs ⑮ FLIP**：⑮ 强在「共享元素连续性」（卡片→详情）；㉙ 强在「层级方向感」。可组合：push 过渡中卡片本身用 FLIP
- **必备搭档**：㉗ 滑动返回（手势可中断版的 pop）

---

## ㉚ 底部抽屉 Bottom Sheet（snap 点 + 拖拽收起）

### 原理
**抽屉有 2~3 个 snap 点（collapsed / half / full），拖 handle 跟手移动，松手按「最近的 snap 点 + 速度方向」二选一归位；backdrop 透明度与抽屉位置线性联动**。速度判定是灵魂：快速上滑直接飞到全屏，不用拖完全程。

### 适用模块
- 地图 App 的 POI 详情卡（经典）
- 分享面板 / 操作菜单（ActionSheet 升级版）
- 筛选器、评论列表
- 音乐播放器迷你条 → 全屏

### 不适用
- ❌ 需要严格模态的表单（用户可能误收起；用居中 Modal）
- ❌ 桌面端（用右侧 Drawer 或 Modal）

### 核心代码

```html
<div class="sheet-backdrop" id="backdrop"></div>
<div class="sheet" id="sheet">
  <div class="handle" id="handle"><div class="grip"></div></div>
  <div class="sheet-body">…内容…</div>
</div>
```

```css
.sheet {
  position: fixed; left: 0; right: 0; bottom: 0;
  height: 75dvh;
  border-radius: 20px 20px 0 0;
  background: #fff;
  box-shadow: 0 -8px 40px rgba(0,0,0,0.18);
  transform: translateY(100%);            /* 初始隐藏在屏幕下 */
  transition: transform 0.42s cubic-bezier(0.32, 0.72, 0, 1);
  touch-action: none;                      /* 拖拽面禁掉原生滚动 */
  overscroll-behavior: contain;
}
.sheet.snap { transition: transform 0.42s cubic-bezier(0.32, 0.72, 0, 1); }
.sheet.dragging { transition: none; }      /* 跟手阶段无动画 */
.sheet-body { overflow-y: auto; touch-action: pan-y; }  /* 内容区恢复竖向滚动 */
```

```javascript
const sheet = document.getElementById('sheet');
const SNAP_POINTS = [                        // px，从抽屉顶部算
  innerHeight - 64,                          // collapsed：只露 handle
  innerHeight * 0.45,                        // half
  0                                          // full
];
let startY = 0, startTop = 0, dragging = false, lastY = 0, lastT = 0, vy = 0;

function currentTop() { return sheet.getBoundingClientRect().top; }

handle.addEventListener('pointerdown', (e) => {
  dragging = true; vy = 0;
  startY = lastY = e.clientY; lastT = performance.now();
  startTop = currentTop();
  sheet.classList.add('dragging');
  sheet.setPointerCapture(e.pointerId);
});
handle.addEventListener('pointermove', (e) => {
  if (!dragging) return;
  const dy = e.clientY - startY;
  const now = performance.now();
  vy = (e.clientY - lastY) / (now - lastT || 16);   // px/ms
  lastY = e.clientY; lastT = now;
  let top = startTop + dy;
  top = clamp(top, 0, SNAP_POINTS[0]);              // 不许拖出边界
  sheet.style.transform = `translateY(${top - innerHeight}px)`;
  backdrop.style.opacity = String(1 - top / SNAP_POINTS[0]);  // backdrop 联动
});
handle.addEventListener('pointerup', () => {
  dragging = false;
  sheet.classList.remove('dragging');
  const top = currentTop();
  let target;
  if (Math.abs(vy) > 0.5) {                          // 快速甩：方向说了算
    target = vy < 0 ? nearestAbove(top) : nearestBelow(top);
  } else {
    target = SNAP_POINTS.reduce((a, b) =>           // 慢速：就近吸附
      Math.abs(b - top) < Math.abs(a - top) ? b : a);
  }
  if (target >= SNAP_POINTS[0]) closeSheet();       // 拖到底 = 关
  else snapTo(target);
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| snap 点数 | 2~3 | collapsed / half / full；4 个点用户记不住 |
| 甩动阈值 | \|vy\| > 0.5 px/ms | 快速轻甩直达目标；慢拖就近吸附 |
| snap 曲线 | `cubic-bezier(0.32, 0.72, 0, 1)` 0.4s | 同 ㉙ 导航曲线，整 App 手感统一 |
| handle 热区 | 高度 ≥ 44px | 可见 grip 条 36×5px，但热区给足 |
| backdrop 联动 | opacity = 1 - progress | 抽屉拖低时背景渐透，暗示「可以关了」 |

### 避坑
- ❌ **handle 用 click 展开 / 无跟手拖拽** → 手感直接掉到 H5 水平；跟手是底线
- ❌ **整个 sheet 设 `touch-action: none`** → 内容区没法竖向滚动；只有 handle 区禁掉
- ❌ **内容区滚到顶后继续上拖没接管** → 顶部 edge-overscroll 应转为收起抽屉（监听 body scrollTop===0 时的 touchmove 向下）
- ❌ **只按位置吸附不看速度** → 快速上滑 100px 停在原地，用户觉得「没反应」
- ❌ **backdrop 固定不透明** → 抽屉拖低时背景还是全黑，没有「正在关闭」的反馈
- ❌ **忘 `pointercancel`** → 系统手势中断后抽屉悬在半空

### 无障碍 fallback
- handle 同时是 `<button>`（键盘 Esc 关闭 + focus trap）；`role="dialog" aria-modal="true"`
- `prefers-reduced-motion`：snap 改 `transition: none`，直接跳位

---

## ㉛ 列表项滑动操作 Swipe Actions（滑动显露按钮）

### 原理
**行内容层 `translateX(dx)` 跟手、操作按钮层垫在下面（右侧露出删除/置顶）**。松手：超过阈值 → 按钮展开态吸附；否则回弹关闭。已有展开行时，触摸其他行先收起它（互斥）。

### 适用模块
- 消息列表（删除/置顶/已读）
- 待办清单（完成/删除）
- 购物车（移入收藏/删除）

### 不适用
- ❌ 操作 > 3 个的行（塞不下；长按出菜单 ㉜）
- ❌ 桌面端列表（hover 直接给按钮，别让鼠标用户做滑动）
- ❌ 行内有横向元素的（图片轮播）—— 手势打架

### 核心代码

```html
<li class="row">
  <div class="actions">
    <button class="act pin">置顶</button>
    <button class="act del">删除</button>
  </div>
  <div class="row-content">…</div>   <!-- 滑动层 -->
</li>
```

```css
.row { position: relative; overflow: hidden; }
.actions {
  position: absolute; inset: 0;
  display: flex; justify-content: flex-end;
}
.act { width: 76px; }
.row-content {
  position: relative;
  background: #fff;
  touch-action: pan-y;               /* 竖向滚动不受影响 */
  transition: transform 0.3s cubic-bezier(0.32, 0.72, 0, 1);
}
.row-content.dragging { transition: none; }
```

```javascript
const OPEN_W = -152;                  // 两按钮总宽（负 = 向左）
let row = null, startX = 0, startTX = 0, locked = null;

list.addEventListener('pointerdown', (e) => {
  row = e.target.closest('.row-content');
  if (!row) return;
  startX = e.clientX; startTX = getTX(row); locked = null;
  // 互斥：按到别的行时，收起已展开的行
  document.querySelectorAll('.row-content.open').forEach(r => {
    if (r !== row) closeRow(r);
  });
});

list.addEventListener('pointermove', (e) => {
  if (!row) return;
  const dx = e.clientX - startX;
  if (!locked) {
    if (Math.abs(dx) < 8) return;
    if (Math.abs(e.clientY - startY) > Math.abs(dx)) { row = null; return; } // 竖向让行
    locked = 'h';
    row.classList.add('dragging');
  }
  let tx = startTX + dx;
  if (tx > 0) tx = tx / 4;            // 右侧无按钮：加阻尼，可感知边界
  tx = clamp(tx, OPEN_W * 1.15, 0);   // 左侧最多多拖 15% 弹性余量
  row.style.transform = `translateX(${tx}px)`;
});

list.addEventListener('pointerup', (e) => {
  if (!row || locked !== 'h') { row = null; return; }
  const tx = getTX(row);
  row.classList.remove('dragging');
  if (tx < OPEN_W * 0.5) openRow(row);   // 过半展开
  else closeRow(row);
  row = null;
});
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 按钮宽 | 68~80px/个 | 单指拇指热区；2 个为宜，最多 3 |
| 展开阈值 | 露出 >50% 或轻甩 | 同 ㉗，速度 + 位移双判定更顺 |
| 右侧橡皮筋 | dx/4 阻尼 | 告诉用户「这边没有操作」，比硬卡死高级 |
| 动画 | 0.3s iOS 曲线 | 展开时按钮依次淡入（stagger 30ms）更精致 |

### 避坑
- ❌ **全宽一步到位（点按出按钮）** → 失去「物理感」；必须跟手
- ❌ **不互斥（多行同时展开）** → 界面乱；新触摸先收其他行
- ❌ **滑动中列表还能竖滚** → 方向判定 + 锁定（同 ㉗）
- ❌ **展开态滚出屏幕** → 该行滚出视口时自动收起（IntersectionObserver）
- ❌ **删除没有撤销** → 误滑删除无法挽回；Toast + 撤销 5s
- ❌ **行内容层 `touch-action: none`** → 列表没法竖向滚；用 pan-y + 方向锁

### 无障碍 fallback
- 每行**必须有可见的替代操作入口**（长按菜单或行尾 ⋯ 按钮）；手势是加速器不是唯一路径
- `aria-hidden` 垫底按钮在收起时不可聚焦；展开时才进 tab 顺序

---

## ㉜ 触感反馈 + 按压缩放 Haptics & Press Feedback

### 原理
**视觉压缩（scale 0.96~0.98）+ 触觉脉冲（`navigator.vibrate`）+ 听觉静音默认** 三通道里的前两个。Android Chrome 支持 Web Vibration API；iOS Safari 不支持（WKWebView / 原生壳里可用 HapticEngine）。降级链：原生触感 → 视觉压缩 → 纯色变。

### 适用模块
- 所有移动端主操作按钮（CTA、发送、支付）
- 滑动翻页卡到 snap 点的瞬间（㉖⑳配合）
- ㉛ 删除确认、㉚ snap 吸附瞬间
- 长按呼出菜单（long-press 500ms + scale 1.02 放大 + 触感）

### 不适用
- ❌ 每次列表滚动都震（骚扰）
- ❌ 桌面端 vibrate 无意义（只保留 scale）
- ❌ 失败/错误时震（负面触感留给系统通知）

### 核心代码

```css
/* —— 按压缩放：active 态，零 JS —— */
.press {
  transition: transform 0.18s cubic-bezier(0.34, 1.56, 0.64, 1);  /* 回弹带一点过冲 */
  touch-action: manipulation;   /* 消除 300ms 双击缩放延迟 */
  -webkit-tap-highlight-color: transparent;  /* 去掉 Android 灰色高亮块 */
}
.press:active { transform: scale(0.96); }
.press:active .press-icon { transform: scale(0.88); }  /* 图标比容器缩更多 = 呼吸层次 */
```

```javascript
/* —— 触感模式表：不同语义用不同波形，别全用同一种 —— */
const HAPTIC = {
  tap:      8,               // 轻点确认：主按钮按下
  success:  [16, 40, 24],    // 成功：双脉冲（保存成功/发送完成）
  warn:     [10, 60, 10, 60, 10],  // 警告：三连震（删除/不可用）
  snap:     4,               // snap 吸附瞬间：极轻（翻页/指示器到站）
};
function haptic(type = 'tap') {
  if (!navigator.vibrate) return;         // iOS Safari：静默降级
  navigator.vibrate(HAPTIC[type]);
}
```

```javascript
/* —— 长按菜单（500ms + 松手前放大） —— */
let lpTimer = null, pressed = null;
list.addEventListener('pointerdown', (e) => {
  const card = e.target.closest('.longpress');
  if (!card) return;
  pressed = card;
  lpTimer = setTimeout(() => {
    haptic('warn');
    card.style.transform = 'scale(1.03)';     // 轻微放大 = 已激活
    card.classList.add('menu-open');           // 弹出上下文菜单
  }, 500);
});
['pointerup', 'pointercancel', 'pointerleave'].forEach(ev =>
  list.addEventListener(ev, () => clearTimeout(lpTimer)));
```

### 触感时机 × 波形速查

| 时机 | 波形 | 时长 |
|------|------|------|
| 主按钮按下 | tap（单脉冲 8ms） | 即时 |
| snap 到位 / 翻页停住 | snap（4ms 极轻） | 吸附瞬间 |
| 操作成功 | success 双脉冲 | 完成 callback 里 |
| 删除 / 不可用 | warn 三连震 | 拦截时 |
| 长按激活 | warn | 满 500ms 时 |

### 避坑
- ❌ **高频震动**（滚动每行震一次）→ 蜂鸣器既视感 + 耗电；只在「状态改变」的瞬间震
- ❌ **scale 压缩 < 0.94** → 像坏掉；0.96~0.97 是甜区
- ❌ **只做 scale 不改颜色** → 有色盲/强光下感知不到；scale + 色变双通道
- ❌ **`vibrate(100)` 长震做按钮反馈** → 像来电；交互触感一律 ≤ 20ms
- ❌ **长按没给视觉反馈** → 用户不知道按住有没有效果；500ms 时放大 + 触感同时给
- ❌ 忘 `-webkit-tap-highlight-color: transparent` → Android 灰块 + 压缩动画双层叠加很脏

### 无障碍 fallback
- 触感 + scale 都是增强；状态改变必须有文字/颜色变化（三冗余原则）
- `prefers-reduced-motion: reduce` 时去掉 scale 动画只保留即时色变

---

## ㉝ 橡皮筋阻尼 Rubber-Band Overscroll

### 原理
**内容滚到顶（`scrollTop ≤ 0`）还继续下拉时，位移按阻尼公式 `pull = 150 * (1 - 1/(dx/150 + 1))` 映射** —— 手指拖 300px 内容只移 ~100px，且越拉越费劲；松手 spring 回弹。这是 iOS 手感的标志性细节，Web 端要自己实现（原生只在 body 上有 rubber band，内部容器没有）。

> 与 ㊲ 下拉刷新的区别：㉝ 只做「拉扯 + 回弹」的视觉阻尼；㊲ 是完整刷新流程（spinner + 触发 + 成功态）。做刷新就把 ㊲ 里的阻尼公式换成 ㉝ 这条。

### 适用模块
- 内部滚动容器（聊天窗、评论区）滚到顶继续下拉的回弹
- 自绘 canvas 列表（无原生滚动可用）
- 配合 ㊱ 拖拽排序（列表顶部/底部拖动时让出空间）

### 不适用
- ❌ body 本身（原生 iOS 自带 rubber band，别重复实现）
- ❌ 加了 `overscroll-behavior: contain` 后还想让页面跟着弹

### 核心代码

```javascript
// 在内部滚动容器上实现橡皮筋（仅顶部示例；底部对称）
const scroller = document.querySelector('.scroller');
const inner = scroller.querySelector('.scroller-inner');   // 需要一层 inner 做位移
let rubber = null;   // { startY, active }

const RUBBER_C = 150;                       // 阻尼常数：越小越「紧」

function rubberDamp(dx) {                   // 1 - 1/(x/c + 1)：位移越大增益越小
  return RUBBER_C * (1 - 1 / (Math.abs(dx) / RUBBER_C + 1));
}

scroller.addEventListener('pointerdown', (e) => {
  if (scroller.scrollTop <= 0) rubber = { startY: e.clientY, active: false };
});
scroller.addEventListener('pointermove', (e) => {
  if (!rubber) return;
  const dy = e.clientY - rubber.startY;
  if (dy <= 0) { rubber = null; return; }              // 向上滑交还原生滚动
  if (!rubber.active) {
    if (scroller.scrollTop > 0) return;               // 已被原生滚走，放手
    rubber.active = true;
    inner.style.transition = 'none';
  }
  e.preventDefault();                                  // 到顶了才拦截
  inner.style.transform = 'translateY(' + rubberDamp(dy) + 'px)';
});
function releaseRubber() {
  if (!rubber?.active) { rubber = null; return; }
  rubber = null;
  inner.style.transition = 'transform .5s cubic-bezier(0.175, 0.885, 0.32, 1.2)';  // 带一点回弹过冲
  inner.style.transform = 'translateY(0)';
}
scroller.addEventListener('pointerup', releaseRubber);
scroller.addEventListener('pointercancel', releaseRubber);
```

```css
.scroller { overflow-y: auto; overscroll-behavior-y: contain; touch-action: pan-y; }
.scroller-inner { will-change: transform; }   /* 只有它动，不引起重排 */
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 阻尼常数 C | 120~180 | iOS ≈ 150；越小回弹感越「紧实」，越大越「松」 |
| 回弹曲线 | `cubic-bezier(0.175, 0.885, 0.32, 1.2)` | 尾端轻微过冲 = 果冻感 |
| 回弹时长 | 0.45~0.6s | 太快像弹片，太慢像掉进棉花 |

### 避坑
- ❌ **1:1 跟手下拉** → 没有「拉不动」的重量感；阻尼公式才是灵魂
- ❌ **没到顶就 preventDefault** → 正常滚动全被劫持；必须 `scrollTop <= 0` 才接管
- ❌ **move 里查 scrollTop 但用了缓存值** → 边滚边拉状态错乱；每次实时读
- ❌ **回弹用 spring 循环手写** → 纯回弹用 CSS 曲线足够，rAF 循环浪费电
- ❌ **双层 padding 拉伸替代 transform** → 重排；只 transform inner

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .scroller-inner { transition: none !important; }
}
```
- 手感细节非信息承载，reduced-motion 下直接无动画回位即可

---

## ㉞ 捏合缩放 Pinch-to-Zoom（双指手势）

### 原理
**Pointer Events 记录两个指针：`scale = 当前两指距离 / 起始距离 × baseScale`，`transform-origin` 锁定在双指起点**。松手时 scale < 1 弹回 1（不许缩没），scale 超过 1.8x 吸附到全屏预览。pivot 计算是精髓 —— 缩放永远围绕手指起点，而不是元素中心。

### 适用模块
- 图片查看器（IM 聊天图片、相册）
- 地图 / 白板画布
- 卡片详情（pinch 放大进入全屏阅读）

### 不适用
- ❌ 普通文本内容页（系统双击缩放已够用）
- ❌ 列表页（pinch 会误伤滚动；只在全屏 viewer 里开）

### 核心代码

```javascript
const img = document.getElementById('zoomImg');
const pointers = new Map();          // pointerId → {x, y}
let pinch = null;                   // { dist0, scale0, cx, cy }
let scale = 1, tx = 0, ty = 0;

function dist(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

stage.addEventListener('pointerdown', (e) => {
  pointers.set(e.pointerId, { x: e.clientX, y: e.clientY });
  if (pointers.size === 2) {
    const [a, b] = pointers.values();
    // pivot：双指起点的中点 —— transform-origin 的锚
    const rect = img.getBoundingClientRect();
    const cx = (a.x + b.x) / 2 - rect.left, cy = (a.y + b.y) / 2 - rect.top;
    pinch = { dist0: dist(a, b) || 1, scale0: scale,
              ox: (cx / rect.width * 100).toFixed(1) + '% ' + (cy / rect.height * 100).toFixed(1) + '%' };
    img.style.transition = 'none';
    img.style.transformOrigin = pinch.ox;
    stage.setPointerCapture(e.pointerId);
  }
});
stage.addEventListener('pointermove', (e) => {
  if (!pointers.has(e.pointerId)) return;
  pointers.set(e.pointerId, { x: e.clientX, y: e.clientY });
  if (pointers.size !== 2 || !pinch) return;
  const [a, b] = pointers.values();
  scale = clamp(pinch.scale0 * dist(a, b) / pinch.dist0, 0.4, 3);
  img.style.transform = `scale(${scale})`;
});
function endPinch(e) {
  pointers.delete(e.pointerId);
  if (pointers.size < 2 && pinch) {
    pinch = null;
    img.style.transition = 'transform .35s cubic-bezier(0.32, 0.72, 0, 1)';
    if (scale < 1) { scale = 1; img.style.transform = 'scale(1)'; haptic('snap'); }      // 缩没 → 弹回
    else if (scale > 1.8) { scale = 2.4; img.style.transform = 'scale(2.4)'; haptic('snap'); }  // 过全屏阈值 → 吸附
  }
}
stage.addEventListener('pointerup', endPinch);
stage.addEventListener('pointercancel', endPinch);

// 双击 toggle 0.25s 内两次 tap
let lastTap = 0;
stage.addEventListener('pointerdown', (e) => {
  const now = performance.now();
  if (pointers.size === 1 && now - lastTap < 250) {
    const target = scale > 1.1 ? 1 : 2;             // 放大态双击回 1
    scale = target;
    img.style.transition = 'transform .3s cubic-bezier(0.32, 0.72, 0, 1)';
    img.style.transform = `scale(${target})`;
    haptic('snap');
  }
  lastTap = now;
});
```

```css
.zoom-stage { touch-action: none; }   /* 双指区禁掉原生缩放/滚动，全权接管 */
.zoom-img { will-change: transform; transform-origin: center; }
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| scale 边界 | 0.4 ~ 3 | 拖拽期放宽到 0.4，给「过度缩放」的物理感；松手才吸附 |
| 回弹阈值 | scale < 1 | 缩到 1 以下松手必弹回 |
| 放大吸附 | scale > 1.8 → 2.4 | 「放大到全屏」的语义点，用吸附而非自由停 |
| 双击窗口 | 250ms | 比系统 dblclick 快，避免触发浏览器双击缩放 |
| stage | `touch-action: none` | 必须显式声明，否则浏览器抢双指 |

### 避坑
- ❌ **transform-origin 固定 center** → 缩放围绕中心转，手指位置「漂移」，一眼假
- ❌ **pointerup 只清一个指针就退出 pinch** → `pointers.size < 2` 才算结束
- ❌ **拖拽期就 clamp 死 1~2.5** → 失去「过度缩放」手感；边界放宽、松手吸附
- ❌ **不 setPointerCapture** → 手指出 stage 后指针丢失，缩放卡死
- ❌ **图片层单独监听且没阻止冒泡** → 与单指拖动 pan 打架

### 无障碍 fallback
- 必须提供「+/−」缩放按钮（键盘/读屏可达）；pinch 是增强
- 双击 / 按钮路径在 reduced-motion 下即时跳变无动画

---

## ㉟ 坍缩头图 Collapsing Header（Stretch + Shrink 双态）

### 原理
**下拉时（scrollTop < 0 不可滚），头图按 `scale = 1 + overscroll/300` 放大 + origin bottom 锚定（stretch 拉伸感）；上滚时，头部按滚动进度 `translateY(scrollTop * 0.5) + 收敛高度`（视差 + 收缩）**。全程只读 scroll 事件 + transform，不劫持滚动 —— 这是最容易被做成掉帧灾难又最容易做对的模式。

### 适用模块
- 个人主页（头像 + 大横幅，Twitter/小红书风）
- 商品详情页（头图 + 吸顶导航）
- 音乐 App 正在播放页

### 不适用
- ❌ 头图与正文信息强关联的阅读页（视差会误导阅读位置）
- ❌ 无大图的纯工具页（没有 stretch 的必要）

### 核心代码

```html
<header class="hero">
  <div class="hero-img" id="heroImg"></div>
  <div class="hero-bar" id="heroBar"><b>个人主页</b></div>
</header>
<main class="feed">…</main>
```

```css
.hero { position: relative; height: 220px; }
.hero-img {
  position: absolute; inset: 0;
  transform-origin: 50% 100%;        /* 关键：底边锚定，下拉时往上长 */
  background: linear-gradient(135deg, #4c6ef5, #845ef7);
  will-change: transform;
}
.hero-bar {
  position: absolute; left: 0; right: 0; bottom: 0;
  padding: 10px 16px;
  background: rgba(0,0,0,.35); backdrop-filter: blur(10px);
  color: #fff; font-size: 16px;
  will-change: transform, opacity;
}
```

```javascript
const heroImg = document.getElementById('heroImg');
const heroBar = document.getElementById('heroBar');
const HERO_H = 220;
let ticking = false;

window.addEventListener('scroll', () => {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    const y = window.scrollY;                 // 往下滚 y > 0；iOS 顶部下拉 y < 0
    if (y <= 0) {
      // stretch：下拉放大（阻尼后），bar 淡出
      heroImg.style.transform = `scale(${1 + Math.min(-y, 300) / 300 * 0.35})`;
      heroBar.style.opacity = String(clamp(1 + y / 80, 0, 1));
    } else {
      // shrink：上滚时头图视差（半速）+ bar 沿头图底边上移直到吸顶
      const p = clamp(y / HERO_H, 0, 1);      // 0~1 收敛进度
      heroImg.style.transform = `translateY(${y * 0.5}px) scale(${1 + (1 - p) * 0})`;
      heroBar.style.transform = `translateY(${y * 0.5}px)`;   // 与头图同半速 → 视差
      heroBar.style.opacity = '1';
      // 吸顶后切换成 sticky 导航（透明度/高度随 p 收敛）
      heroBar.style.height = `${56 - p * 8}px`;
    }
    ticking = false;
  });
}, { passive: true });
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| stretch 上限 | +0.35 scale | 下拉 300px 封顶放大 1.35x；再多没意义 |
| 视差速度 | 0.5×（头图半速） | 头图滚得比正文慢 → 深度感 |
| bar 收敛 | 高度 -8px + 字号 -1px | 保守；shrink 剧烈会闪 |
| 吸顶点 | `p = 1`（滚过 HERO_H） | 换成固定导航，bar 退出 |

### 避坑
- ❌ **scroll 回调里同步读布局又写布局** → 强制同步布局抖动；只写 transform + rAF 节流
- ❌ **stretch 改 height/paddingTop** → 重排；只 `scale` + `origin: bottom`
- ❌ **下拉放大 origin 在 center** → 头图上下都长，底部露缝；必须底边锚定
- ❌ **bar 视差速度 ≠ 头图视差速度** → bar 和图分离漂移，穿帮
- ❌ **把 scroll 设 `{passive: false}`** → 本模式只读不拦，passive true 保性能

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .hero-img, .hero-bar { transform: none !important; }
}
```
- stretch/视差纯装饰，直接关；吸顶导航功能保留（无动画跳变）

---

## ㊱ 长按拖拽排序 Drag-to-Reorder

### 原理
**长按 300ms 触发「浮起」（scale 1.05 + 阴影 + 触感）→ 该行 absolute 跟手 → 其余行按「目标 index 与当前 index 的差」用 transform 平滑让位 → pointerup 落位结算**。精髓是让位动画用「按 index 差 translateY(rowHeight)」，拖动经过即换 index，行像水一样分开。

### 适用模块
- 歌单排序、收藏夹整理
- 看板列内卡片排序
- 表单字段排序

### 不适用
- ❌ 长列表（> 30 项，拖到目标位置太久；改数字输入序号）
- ❌ 桌面端（直接给上/下移动按钮，鼠标长按反人类）

### 核心代码

```javascript
const list = document.getElementById('reorderList');
const rows = Array.from(list.children);
const ROW_H = 64;                       // 行高（含 margin）
let drag = null;                         // { row, idx, ghost, startY }

list.addEventListener('pointerdown', (e) => {
  const row = e.target.closest('.rrow');
  if (!row) return;
  drag = { row, idx: rows.indexOf(row), startY: e.clientY, moved: false, timer: null };
  drag.timer = setTimeout(() => liftRow(e.clientY), 300);   // 长按 300ms 才浮起
});

function liftRow(y) {
  if (!drag) return;
  drag.moved = true;
  drag.row.setPointerCapture?.(lastPointerId);
  drag.row.classList.add('lifted');      // scale(1.05) + 大阴影 + 半透明 + haptic
  haptic('warn');
  // 浮起后该行 absolute 化，同时留一个 ghost 占位（或 width:100% placeholder）
  const rect = drag.row.getBoundingClientRect();
  drag.startTop = rect.top; drag.offsetY = y - rect.top;
  drag.row.style.width = rect.width + 'px';
  drag.row.style.position = 'absolute';
  drag.row.style.top = (drag.startTop - list.getBoundingClientRect().top) + 'px';
  drag.row.style.zIndex = 10;
  rows.forEach(r => { if (r !== drag.row) r.style.transition = 'transform .25s var(--ios-ease)'; });
}

list.addEventListener('pointermove', (e) => {
  if (!drag) return;
  if (!drag.moved) {
    // 未浮起：移动超过 10px = 想滚动，取消长按
    if (Math.abs(e.clientY - drag.startY) > 10) { clearTimeout(drag.timer); drag = null; }
    return;
  }
  e.preventDefault();
  const dy = e.clientY - drag.startY;
  drag.row.style.transform = `translateY(${dy}px) scale(1.05)`;
  // 换 index：拖过行高一半即换
  const shift = Math.round((drag.row.getBoundingClientRect().top + 32 - drag.startTop) / ROW_H);
  const newIdx = clamp(drag.idx + shift, 0, rows.length - 1);
  if (newIdx !== drag.idx) {
    moveVisual(drag.idx, newIdx);
    drag.idx = newIdx;
    haptic('snap');                      // 每次换位轻震一次 —— 「咔哒」到位感
  }
});

function moveVisual(from, to) {
  // 让位：from→to 之间的行按方向 translateY(±ROW_H)
  rows.forEach((r, i) => {
    if (r === drag.row) return;
    let pos = i;
    if (from < to && i > from && i <= to) pos = i - 1;
    else if (from > to && i < from && i >= to) pos = i + 1;
    r.dataset.pos = pos;
    r.style.transform = pos !== rows.indexOf(r) ? `translateY(${(pos - rows.indexOf(r)) * ROW_H}px)` : '';
  });
}

list.addEventListener('pointerup', () => {
  if (!drag) return;
  clearTimeout(drag.timer);
  if (!drag.moved) { drag = null; return; }
  const { row, idx } = drag;
  // 结算：DOM 真实重排 + 复位样式
  rows.splice(rows.indexOf(row), 1);
  rows.splice(idx, 0, row);
  row.style.transition = 'transform .25s var(--ios-ease), top .25s var(--ios-ease)';
  row.style.transform = '';
  requestAnimationFrame(() => {
    // 依次复位所有行样式，再 appendChild 按新顺序
    rows.forEach(r => { r.style.transform = ''; r.style.transition = ''; });
    rows.forEach(r => list.appendChild(r));
    setTimeout(() => {
      row.style.position = ''; row.style.top = ''; row.style.width = ''; row.style.zIndex = '';
      row.classList.remove('lifted');
    }, 260);
  });
  haptic('success');
  drag = null;
});
list.addEventListener('pointercancel', () => {
  if (drag?.moved) { /* 同 pointerup 的结算 */ }
  clearTimeout(drag?.timer); drag = null;
});
```

```css
.rrow { background: var(--panel-2); border-radius: 12px; height: 64px; margin-bottom: 0; }
.rrow.lifted {
  background: #2a2e3f;
  box-shadow: 0 12px 32px rgba(0,0,0,.5);
  opacity: .92;
  touch-action: none;
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 长按时长 | 280~350ms | 短于 250 会和滚动打架，长于 400 等得心焦 |
| 浮起幅度 | scale 1.04~1.06 | 太小不像「拿起来」，太大遮挡邻行 |
| 让位动画 | 0.22~0.3s iOS 曲线 | 拖动经过时水流感 |
| 换位阈值 | 拖过行高一半 | 换位瞬间 snap 触感 = 「咔哒」 |
| 滚动接驳 | 行到视口边 80px 时启动 8px/frame 自动滚 | 长列表必需 |

### 避坑
- ❌ **点按立即浮起** → 和滚动/点击无法区分；必须长按门槛
- ❌ **让位用 insertBefore 实时改 DOM** → 每次换位整表重排；让位只动 transform，松手才结算 DOM
- ❌ **行高不固定** → ROW_H 计算 ω 错位；行高不一致时先量 `getBoundingClientRect`
- ❌ **换位无触感** → 静默换位感知弱；snap 轻震 + 让位动画双反馈
- ❌ **没处理 pointercancel** → 来电中断后行悬在半空

### 无障碍 fallback
- 必须提供「上移/下移」按钮或序号编辑（读屏 + 键盘可达）；拖拽是增强
- reduced-motion：浮起/让位动画关，即时重排

---

## ㊲ 完整下拉刷新 Pull-to-Refresh（跟手流程组件）

### 原理
**scrollTop = 0 且继续下拉 → spinner 随阻尼位移出现并按比例旋转 → 超过阈值（80px）箭头/文案切换「松手刷新」→ 松手：吸到 loading 位自转 0.8s → 完成态打勾 → 自动收起**。阻尼公式同 ㉝；与 ㉝ 的分工：㉝ 管「拉扯手感」，㊲ 管「完整状态机」。

### 适用模块
- 任何移动端信息流 / 列表页顶部
- 小程序自定义下拉（配合 wx.enablePullDownRefresh 关掉系统版）

### 不适用
- ❌ 非首屏列表（内部嵌套容器别放刷新；刷新语义属于「整页」）
- ❌ 桌面端（用刷新按钮；鼠标没有下拉心智）

### 核心代码

```html
<div class="ptr-zone" id="ptrZone">
  <div class="ptr-indicator" id="ptrIndicator">
    <div class="ptr-spinner" id="ptrSpinner"></div>
    <span class="ptr-label" id="ptrLabel">下拉刷新</span>
  </div>
  <div class="ptr-content" id="ptrContent">…列表内容…</div>
</div>
```

```css
.ptr-zone { position: relative; overflow-y: auto; overscroll-behavior-y: contain; }
.ptr-indicator {
  position: absolute; top: 0; left: 0; right: 0;
  height: 72px;
  display: flex; align-items: center; justify-content: center; gap: 8px;
  transform: translateY(-100%);
}
.ptr-spinner {
  width: 22px; height: 22px; border-radius: 50%;
  border: 2.5px solid rgba(255,255,255,.15);
  border-top-color: var(--accent);
}
.ptr-spinner.done { border-color: var(--ok); border-top-color: var(--ok); }  /* 完成态整体变 ✓ 色圆 */
```

```javascript
const TRIGGER = 80, LOADING_HOLD = 60;   // 触发阈值 / loading 吸附位
let ptr = null, state = 'idle';          // idle | pulling | loading | done

function damp(d) { return 130 * (1 - 1 / (d / 130 + 1)); }   // 同 ㉝

ptrZone.addEventListener('pointerdown', (e) => {
  if (ptrZone.scrollTop <= 0 && state === 'idle') ptr = { y0: e.clientY };
});
ptrZone.addEventListener('pointermove', (e) => {
  if (!ptr) return;
  const d = e.clientY - ptr.y0;
  if (d <= 0 || ptrZone.scrollTop > 0) { ptr = null; resetIdle(); return; }
  state = 'pulling';
  e.preventDefault();
  const pull = damp(d);
  ptrContent.style.transition = 'none';
  ptrContent.style.transform = `translateY(${pull}px)`;
  ptrIndicator.style.transform = `translateY(${-100 + clamp(pull, 0, 100)}%)`;
  ptrSpinner.style.transform = `rotate(${pull * 3}deg)`;      // 跟手旋转 3°/px
  ptrLabel.textContent = pull > TRIGGER ? '松开刷新' : '下拉刷新';
});
ptrZone.addEventListener('pointerup', () => {
  if (!ptr) return;
  const pull = getPull();
  ptr = null;
  if (pull > TRIGGER) startLoading();
  else snapBack();
});

function snapBack() {
  ptrContent.style.transition = 'transform .45s cubic-bezier(0.32, 0.72, 0, 1)';
  ptrContent.style.transform = 'translateY(0)';
  ptrIndicator.style.transition = 'transform .45s cubic-bezier(0.32, 0.72, 0, 1)';
  ptrIndicator.style.transform = 'translateY(-100%)';
  setTimeout(resetIdle, 460);
}
function startLoading() {
  state = 'loading';
  haptic('tap');
  ptrContent.style.transition = 'transform .35s cubic-bezier(0.32, 0.72, 0, 1)';
  ptrContent.style.transform = `translateY(${LOADING_HOLD}px)`;
  ptrIndicator.style.transition = 'transform .35s cubic-bezier(0.32, 0.72, 0, 1)';
  ptrIndicator.style.transform = 'translateY(${-100 + 60}px)';  // 吸到 loading 位
  ptrSpinner.style.animation = 'spin 0.8s linear infinite';
  ptrLabel.textContent = '刷新中…';
  // 实际请求：fetch(...).finally(finishRefresh)
  setTimeout(finishRefresh, 900);
}
function finishRefresh() {
  state = 'done';
  ptrSpinner.style.animation = 'none';
  ptrSpinner.classList.add('done');
  ptrLabel.textContent = '已更新';
  haptic('success');
  setTimeout(snapBack, 500);      // 打勾停留 500ms 再收起
}
function resetIdle() { state = 'idle'; ptrSpinner.classList.remove('done'); }
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 触发阈值 | 80~96px（阻尼前） | iOS ≈ 80；阻尼后实际位移 ≈ 55px |
| spinner 跟手转速 | 2~3°/px | 转太慢「没在转」，太快晕 |
| loading 吸附位 | 56~64px | spinner 完整露出 + 一点呼吸空间 |
| 完成态停留 | 400~600ms | 打勾要给看清的时间 |
| 阻尼常数 | 130（同 ㉝） | 拉扯感和系统一致 |

### 避坑
- ❌ **1:1 跟手无阻尼** → 一步拉到底像 Android 4.x；阻尼是高级感来源
- ❌ **触发后 spinner 从 0 开始转** → 拉扯阶段的跟手旋转就是预告，松手时转速要延续
- ❌ **完成态一闪而过** → 用户不确定刷没刷成功；打勾停留 500ms + success 触感
- ❌ **在内部滚动容器也放刷新** → 用户在评论区下拉误触整页刷新；刷新只属于页面级
- ❌ **loading 中还允许再拉** → 状态机必须有 idle/pulling/loading/done 四态互斥

### 无障碍 fallback
- 必须提供可见刷新按钮（读屏/键盘可触达）；PTR 是加速器
- reduced-motion：跟手旋转关闭，触发后直接显示 loading 态

---

## ㊳ 惯性甩动 Momentum Fling（摩擦衰减滑轨）

### 原理
**拖拽释放时记录末速 vx（px/ms），rAF 循环里 `x += v * 16; v *= 0.94` 摩擦衰减** —— 甩出去的元素自己「滑行减速」，撞到边界反弹 0.4 倍速度。用于没有原生滚动的自绘选择器（滚轮、时间选择、值滑轨）。

### 适用模块
- 滚轮式选择器（时间 / 日期 / 城市三级联动）
- 值选择滑轨（音量、字重、色温）
- 自绘 canvas 时间轴

### 不适用
- ❌ 可用 ㉖ scroll-snap 解决的场景（原生惯性永远更顺）
- ❌ 桌面端（鼠标拖拽的心智是「精准」不是「甩」）

### 核心代码

```javascript
const rail = document.getElementById('rail');
const knob = document.getElementById('knob');
let x = 0, min = 0, max = 300;      // 值域（px）
let drag = null, fling = null;

function render() { knob.style.transform = `translateX(${x}px)`; }

rail.addEventListener('pointerdown', (e) => {
  drag = { x0: e.clientX, sx: x, lastX: e.clientX, lastT: performance.now(), v: 0 };
  cancelAnimationFrame(fling); fling = null;      // 接住正在滑行的轨道
  rail.setPointerCapture(e.pointerId);
});
rail.addEventListener('pointermove', (e) => {
  if (!drag) return;
  const now = performance.now();
  drag.v = (e.clientX - drag.lastX) / Math.max(now - drag.lastT, 1);  // px/ms
  drag.lastX = e.clientX; drag.lastT = now;
  x = clamp(drag.sx + e.clientX - drag.x0, min, max);
  render();
});
rail.addEventListener('pointerup', () => {
  if (!drag) return;
  const v = drag.v; drag = null;
  if (Math.abs(v) < 0.15) return;                  // 慢速放手 = 不甩
  let vel = v * 16;                                // 换算 px/frame
  (function glide() {
    x += vel;
    if (x < min) { x = min; vel = -vel * 0.4; }    // 边界反弹 0.4 倍
    else if (x > max) { x = max; vel = -vel * 0.4; }
    else vel *= 0.94;                              // 摩擦衰减
    render();
    if (Math.abs(vel) > 0.2) fling = requestAnimationFrame(glide);
  })();
});
rail.addEventListener('pointercancel', () => { drag = null; cancelAnimationFrame(fling); });
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 摩擦系数 | 0.92~0.96 / frame | 0.94 ≈ 滑 12 帧收敛；越小越「涩」，越大越「溜冰」 |
| 反弹恢复 | 0.35~0.5 倍速度 | 太大蹦跶，太小像撞墙 |
| 起甩阈值 | \|v\| ≥ 0.15 px/ms | 慢速放手不甩，避免误飞 |
| 吸附（可选） | 滑行中最近刻度 `x = round(x/44)*44` | 滚轮选择器必配；松手前 0.2s 开始吸 |

### 避坑
- ❌ **释放速度取整段平均值** → 被起始慢速稀释；只取最后 3~5 个 move 的末速
- ❌ **无边界反弹** → 甩到底「啪」急停；0.4 倍反弹是「有东西接住你」的安全感
- ❌ **滑行中新拖拽没取消 rAF** → 两套位移叠加飞出
- ❌ **摩擦放在 16ms 假设外** → 用 rAF 每帧 `vel *= 0.94` 而非 setInterval；掉帧设备会滑得更久（可接受）
- ❌ **滑轨忘了 touch-action: none** → 浏览器把拖动当滚动没收事件

### 无障碍 fallback
- 滑轨必须有 `<input type="range">` 或 ± 按钮等价物（键盘方向键可调）
- reduced-motion：释放后直接缓动到最近刻度，无滑行

---

## ㊴ 堆叠卡片页 Stacked Cards（iOS 多任务风）

### 原理
**多层全屏卡片按 `scale(1 - n*0.05) translateY(n*4px)` 退后，前层拖动时后层按进度「提前跟进」** —— 拖 30%，第二层已滑到 60% 的位置（跟进系数 2x）：整个牌堆像一叠纸被捻开。松手速度判定：飞出 = 下一层成为新前层（spring 到位），回弹 = 全体归位。iOS 天气 / 多任务手势的 Web 复刻。

### 适用模块
- 沉浸式卡片浏览（天气、故事卡、多任务）
- 表单向导的「回退堆栈」可视化
- 匹配类应用（Tinder 是 ③ 飞出；㊴ 是「捻开一叠」）

### 不适用
- ❌ 卡片内容需要精读（后面的层缩到 0.9x 就难读）
- ❌ 层数 > 4（第 5 层 scale 0.75 不可读，纯 GPU 浪费）

### 核心代码

```javascript
const layers = Array.from(document.querySelectorAll('.stack-layer'));
const FOLLOW = 2, SCALE_STEP = 0.05, SHIFT_STEP = -12;
let stk = null;   // { startX, base: layers.map(→当前transform) }

function layoutStack() {
  layers.forEach((el, i) => {
    el.style.zIndex = 100 - i;
    if (i === 0) { el.style.transform = 'translateX(0)'; el.style.opacity = 1; }
    else {
      el.style.transform =
        `translateX(${i * SHIFT_STEP}px) translateY(${i * 4}px) scale(${1 - i * SCALE_STEP})`;
      el.style.opacity = i <= 3 ? 1 : 0;
    }
  });
}
layoutStack();

layers[0].addEventListener('pointerdown', (e) => {
  stk = { startX: e.clientX, w: innerWidth };
  layers.forEach(el => el.style.transition = 'none');
});
window.addEventListener('pointermove', (e) => {
  if (!stk) return;
  const p = clamp((e.clientX - stk.startX) / stk.w, 0, 1);   // 只往右捻
  layers.forEach((el, i) => {
    const my = Math.max(0, 1 - i * FOLLOW);                   // i=1 提前 2 倍进度
    el.style.transform =
      `translateX(${p * my * 100 + (i > 0 ? 0 : 0) * i}%) translateY(${i * 4}px) scale(${1 - i * SCALE_STEP + p * my * SCALE_STEP})`;
  });
  // 精确版（上面的简式会漂移，正式实现用 base + delta）：
});
window.addEventListener('pointerup', (e) => {
  if (!stk) return;
  const dx = e.clientX - stk.startX;
  const vx = dx / 16;   // 此处应记录末速，同 ㉗
  const commit = dx > stk.w * 0.35 || vx > 30;
  layers.forEach(el => el.style.transition = 'transform .4s cubic-bezier(0.32, 0.72, 0, 1)');
  if (commit) {
    layers[0].style.transform = 'translateX(110%)';
    layers[0].style.opacity = '0';
    // 全体往前补一层（第二层 spring 到前层位，后面的层跟进到新位置）
    setTimeout(() => {
      layers.push(layers.shift());
      layoutStack();
      layers[0].addEventListener('pointerdown', arguments.callee); // 注意：正式实现重新绑事件
    }, 400);
  } else {
    layoutStack();   // 全体弹回
  }
  stk = null;
});
```

> demo 参见 `mobile-ui-pro.html` 的「堆叠卡片」段 —— 正式实现以它为准，比上面的示意完整（base+delta 消除漂移）。

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| scale 递减 | 0.05/层 | 后层更小 = 深度；0.1/层 太陡 |
| 跟进系数 | 2x | 前层拖 50% 后层已就位 —— 「整叠在动」的关键 |
| 捻开方向 | 只往一个方向（右） | 双向捻的认知负担高 |
| 飞出位移 | 110% 宽 | 留一点边缘残影比完全消失更有层次 |

### 避坑
- ❌ **每层独立拖拽事件** → 只有前层可拖；后层拖动会乱
- ❌ **跟进系数 = 1** → 后层纹丝不动，看起来是「前层飞出」不是「捻开一叠」
- ❌ **换层后没重排 z-index / 事件** → 第二层永远点不到
- ❌ **超过 4 层** → 后面的层缩到不可读，纯浪费 GPU

### 无障碍 fallback
- 每层内容提供「查看详情」按钮（点按也能进入下一层）
- reduced-motion：换层用 fade 即时切换

---

## ㊵ 液态填充 Liquid Swipe（SVG 贝塞尔按钮）

### 原理
**SVG `<path>` 两段贝塞尔画一个「水滴前沿」，path 宽度 = 进度 %，控制点随进度轻微前凸（液态表面张力）**。点击/进度推进时前沿以 elastic 曲线涌到 100%，完成后再涌回或切换状态。比普通 fill 高级在于「前沿永远鼓着」—— 像推水。

### 适用模块
- 高端 CTA / 订阅按钮的 hover/按压填充
- 播放器进度条 / 音量条
- 「液态」感的主题切换开关

### 不适用
- ❌ 频繁触发的普通按钮（动画 0.6s+ 会拖累操作节奏）
- ❌ 精确进度需求（液态前沿不适合读精确值；用普通进度条）

### 核心代码

```html
<button class="liquid" id="liquidBtn">
  <svg class="liquid-svg" viewBox="0 0 200 48" preserveAspectRatio="none" aria-hidden="true">
    <path class="liquid-fill" id="liquidFill" d="M0,48 L0,48 C0,16 0,16 0,0 L200,0 C200,16 200,32 200,48 Z"/>
  </svg>
  <span class="liquid-label">订阅</span>
</button>
```

```javascript
// 进度 p ∈ [0,1] → path：前沿在 x = p*200，凸起 bulge 随速度变化
function liquidPath(p, bulge) {
  const x = p * 200;
  // 前沿是两段三次贝塞尔：从底部到顶部，控制点向前凸 bulge
  return `M0,48 L${x - bulge},48
          C${x + bulge * 0.6},40 ${x + bulge},24 ${x + bulge * 0.4},0
          L200,0 L200,48 Z`;
}
const fill = document.getElementById('liquidFill');
let lp = 0, lt = null;   // 当前进度 / 目标进度

function setLiquid(target) { lt = target; tick(); }
function tick() {
  cancelAnimationFrame(fill._raf);
  (function step() {
    const d = lt - lp;
    if (Math.abs(d) < 0.001) { lp = lt; }
    else {
      lp += d * 0.12;                          // 液体阻尼追赶
      fill._raf = requestAnimationFrame(step);
    }
    const bulge = 14 + Math.abs(d) * 60;        // 移动越快，前沿越凸（表面张力）
    fill.setAttribute('d', liquidPath(clamp(lp, -0.2, 1.2), bulge));
  })();
}
liquidBtn.addEventListener('click', () => setLiquid(lt > 0.5 ? 0 : 1));
setLiquid(0);
```

```css
.liquid {
  position: relative; overflow: hidden;
  width: 200px; height: 48px; border: none; border-radius: 24px;
  background: rgba(110,168,254,.15);
  color: var(--accent); font-weight: 700; cursor: pointer;
  touch-action: manipulation;
}
.liquid-svg { position: absolute; inset: 0; width: 100%; height: 100%; }
.liquid-fill { fill: var(--accent); }
.liquid-label { position: relative; z-index: 1; transition: color .3s; }
/* 液体过半时文字反白 —— 用 mix-blend-mode 一步到位 */
.liquid-label { mix-blend-mode: difference; color: #fff; }
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 追赶系数 | 0.1~0.14 / frame | 液体「黏稠感」；0.3 就成了普通动画 |
| 基础凸起 | 12~16px | 静止时前沿也保持鼓形（水滴感） |
| 速度凸起 | +|d|×60 | 越快越凸，停下了回落 —— 表面张力的物理感 |
| 文字反白 | `mix-blend-mode: difference` | 免 JS 判断进度阈值 |
| 点击场景 | 全程 0.6~0.9s | 涌过去 + 稳定，太快不像液体 |

### 避坑
- ❌ **前沿是直线（普通 width fill）** → 那是进度条不是液态；凸起是灵魂
- ❌ **凸起恒定不随速度** → 死板；速度响应才有「活」感
- ❌ **preserveAspectRatio="none" 忘了设** → 按钮尺寸一变 path 就变形
- ❌ **用在每屏多个按钮上** → 同屏 > 2 个液态源 = 廉价感；点睛用
- ❌ **path 每帧重建导致 GC 压力** → 字符串模板足够快，但别加 SVG filter（blur 会崩）

### 无障碍 fallback
```css
@media (prefers-reduced-motion: reduce) {
  .liquid-svg { display: none; }
  .liquid.on { background: var(--accent); color: #0b1220; }   /* 纯色态切换 */
}
```
- 状态（已订阅/未订阅）不能只靠液态表达 —— 文字必须同步变化（三冗余原则）

---

## ㊶ 双击点赞爱心爆裂 Double-Tap Like

### 原理
**250ms 内两次 pointerdown（单指）判定为双击 → 在第二次点击坐标 spawn 爱心 span：scale 0→1.3→1 弹出 + 350ms 停留 + 向上漂浮淡出**。连击计数：每次 burst 距上次 < 800ms 则 count+1 并重置消隐 timer；计数数字同步 bump（配合 ㊼）。爱心从**点击位置**长出来（不是中心）—— 这是「手感正确」的关键。

### 适用模块
- Feed 图片/视频（Instagram / 小红书式点赞）
- 评论区长按点赞（长按 + 爱心更沉浸；按住即 burst）
- 收藏、mark、applause 等情绪表达

### 不适用
- ❌ 桌面端双击（双击=选词的心智；桌面用 hover 心形按钮）
- ❌ 内容本身可双击缩放的区域（㉞ pinch viewer 里别混用双击语义，先想清楚优先级）

### 核心代码

```javascript
const media = document.getElementById('feedMedia');
let lastTap = 0, likeCount = 1244, burstTimer = null;

media.addEventListener('pointerdown', (e) => {
  if (e.pointerType === 'mouse' && e.button !== 0) return;
  const now = performance.now();
  if (now - lastTap < 250) { burst(e.clientX, e.clientY); lastTap = 0; }
  else lastTap = now;
});

function burst(x, y) {
  haptic('tap');
  likeCount++;
  likeEl.textContent = fmt(likeCount);
  bumpBadge(likeEl);                                   // ㊼ 数字弹跳

  const rect = media.getBoundingClientRect();
  const heart = document.createElement('div');
  heart.className = 'like-burst';
  heart.textContent = '❤️';
  heart.style.left = (x - rect.left) + 'px';
  heart.style.top = (y - rect.top) + 'px';
  media.appendChild(heart);
  setTimeout(() => heart.remove(), 1000);
  clearTimeout(burstTimer);                            // 连击窗口
  burstTimer = setTimeout(() => setLiked(true), 800);
}
```

```css
.like-burst {
  position: absolute; z-index: 5;
  font-size: 64px; pointer-events: none;
  transform: translate(-50%, -50%);
  animation: likePop .95s cubic-bezier(0.32, 0.72, 0, 1) forwards;
}
@keyframes likePop {
  0%   { transform: translate(-50%, -50%) scale(0); opacity: 0; }
  22%  { transform: translate(-50%, -50%) scale(1.3); opacity: 1; }   /* 过冲 1.3 */
  38%  { transform: translate(-50%, -50%) scale(1); }
  100% { transform: translate(-50%, -120px) scale(0.9) rotate(-8deg); opacity: 0; }
}
.media { position: relative; touch-action: manipulation; }  /* 消除双击缩放延迟 */
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 双击窗口 | 250ms | 与浏览器/系统双击一致；超时窗口里的第二次 tap 别误判 |
| 过冲 | scale 1.3 | 1.2~1.35 甜区；无过冲「没弹性」，1.5 像玩具 |
| 漂浮 | 向上 120px + rotate ±8° | 随机 ±15° 更自然（JS 里随机一下 rotate） |
| 连击窗口 | 800ms | 窗口内继续 +1；窗外重开一击 |

### 避坑
- ❌ **爱心从卡片中心冒出** → 违反「从点击位置长出来」的物理直觉
- ❌ **双击没禁用浏览器缩放** → 加 `touch-action: manipulation`，否则双击触发系统放大
- ❌ **连击计数用 click 事件** → click 有 300ms 延迟嫌疑，且无法拿到按下坐标；用 pointerdown
- ❌ **爱心可点（没 pointer-events: none）** → 挡住下层内容
- ❌ **每击都弹 toast** → burst 本身就是反馈；toast 是重复说明（违反减法表）

### 无障碍 fallback
- 必须有可见的心形按钮（`aria-pressed` 表达已赞）；双击是加速器
- reduced-motion：爱心直接出现 200ms 消失，无过冲无漂浮

---

## ㊷ 横滑关闭通知/浮层 Swipe-to-Dismiss

### 原理
**整卡跟手 translateX + 微旋转（dx × 0.05°）+ 不透明度随位移衰减 → 阈值飞出（translateX ±130% + opacity 0）→ 卡片高度坍缩归零（height→0 动画）让下方内容流上来**。与 ㉛ 的分工：㉛ 是行**内容**滑动露出按钮（行还在）；㊷ 是**整卡离场**（通知消失）。高度坍缩是列表场景的灵魂细节 —— 卡飞走后洞要合上。

### 适用模块
- 应用内通知横幅（聊天/系统通知）
- 临时浮层卡（新功能提示、公告）
- 消息中心列表项快速清除

### 不适用
- ❌ 需要精确操作的卡（误滑代价高；用 ㉛ 显露确认按钮）
- ❌ 桌面端通知（鼠标点 × 关闭；横滑是触摸心智）

### 核心代码

```javascript
function makeDismissible(card) {
  let d = null;
  card.addEventListener('pointerdown', (e) => {
    d = { x0: e.clientX, y0: e.clientY, lastX: e.clientX, lastT: performance.now(), v: 0, lock: null };
    card.style.transition = 'none';
  });
  card.addEventListener('pointermove', (e) => {
    if (!d) return;
    const dx = e.clientX - d.x0, dy = e.clientY - d.y0;
    if (!d.lock) {
      if (Math.abs(dx) < 8 && Math.abs(dy) < 8) return;
      d.lock = Math.abs(dx) > Math.abs(dy) ? 'h' : 'v';
      if (d.lock === 'v') { resetCard(); return; }     // 竖向交给页面滚动
    }
    const now = performance.now();
    d.v = (e.clientX - d.lastX) / Math.max(now - d.lastT, 1);
    d.lastX = e.clientX; d.lastT = now;
    card.style.transform =
      'translateX(' + dx + 'px) rotate(' + dx * 0.05 + 'deg)';   // 微旋转 = 「拎起来」
    card.style.opacity = String(1 - Math.min(Math.abs(dx) / 260, 0.55));
  });
  card.addEventListener('pointerup', () => {
    if (!d) return;
    const dx = getTX(card), v = d.v; d = null;
    if (Math.abs(dx) > card.offsetWidth * 0.4 || Math.abs(v) > 0.5) flyOut(card, dx);
    else resetCard();
  });
  card.addEventListener('pointercancel', () => { d = null; resetCard(); });
}
function resetCard() {
  card.style.transition = 'transform .3s cubic-bezier(0.32,0.72,0,1), opacity .3s';
  card.style.transform = ''; card.style.opacity = '';
}
function flyOut(card, dx) {
  haptic('success');
  card.style.transition = 'transform .32s cubic-bezier(0.32,0.72,0,1), opacity .32s';
  card.style.transform = 'translateX(' + (dx > 0 ? 1 : -1) * 130 + '%) rotate(' + dx * 0.05 + 'deg)';
  card.style.opacity = '0';
  card.addEventListener('transitionend', () => {
    // 高度坍缩：height + margin + padding 归零（这里是唯一允许动 height 的地方——离场时刻）
    card.style.height = card.offsetHeight + 'px';
    requestAnimationFrame(() => {
      card.style.transition = 'height .28s var(--ios-ease), margin .28s var(--ios-ease)';
      card.style.height = '0'; card.style.margin = '0'; card.style.padding = '0';
      setTimeout(() => card.remove(), 300);
    });
  }, { once: true });
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 微旋转 | dx × 0.05° | 超过 0.1° 卡片像要翻倒；旋转让「拎起」感真实 |
| 不透明度下限 | ≥ 0.45 | 完全消失会让用户怀疑「是不是删错了」 |
| 提交阈值 | 40% 宽 or vx > 0.5px/ms | 与 ㉗㉛ 统一 |
| 高度坍缩 | 0.28s iOS 曲线 | 比飞出略快 —— 洞合上后视线才移走 |

### 避坑
- ❌ **飞出后洞留在那** → 列表出现空洞；高度坍缩必须做
- ❌ **没有微旋转** → 平移像「滑块」不像「甩卡片」
- ❌ **不透明度降到 0** → 用户对「删了什么」失去感知；留 0.45
- ❌ **误滑无补救** → 删除类通知配 toast 撤销（同 ㉛ 原则）
- ❌ **卡内有按钮还触发横滑** → 按钮区 `e.stopPropagation()`，或滑动从非按钮区起手才允许

### 无障碍 fallback
- 每卡保留 × 关闭按钮（键盘/读屏可达）
- reduced-motion：飞出/坍缩改即时 remove

---

## ㊸ 3D 滚轮选择器 Wheel Picker

### 原理
**item 距滚动中心的偏移 n → `rotateX(n × 22°) translateZ(28px)` + opacity 衰减 = 圆柱面错觉**；容器 `perspective: 900px`。滚动本体用**原生 overflow-y + scroll-snap**（㉖ 原则），scroll 事件里重算每个 item 的 3D transform，scrollend 落定选中值。原生惯性 + 3D 曲面 = iOS 时间选择器。

### 适用模块
- 时间 / 日期 / 时长选择
- 城市三级联动
- 字号、倍速、单位选择

### 不适用
- ❌ 选项 > 30 个（3D 轮不适合海量；改搜索列表）
- ❌ 需要多选（轮式是单选心智）

### 核心代码

```html
<div class="wheel" id="wheel">
  <div class="wheel-item" style="height:44px">00</div>  <!-- 头尾各补 2 个空白 item 做缓冲 -->
  <div class="wheel-item">…</div>
</div>
```

```css
.wheel {
  height: 220px;                        /* 5 × item 44px */
  overflow-y: auto;
  scroll-snap-type: y mandatory;
  overscroll-behavior: contain;
  perspective: 900px;                    /* 3D 曲面的根 */
  scrollbar-width: none;
  touch-action: pan-y;
  -webkit-mask-image: linear-gradient(transparent, #000 22%, #000 78%, transparent);  /* 上下羽化 */
}
.wheel::-webkit-scrollbar { display: none; }
.wheel-item {
  height: 44px; scroll-snap-align: center;
  display: flex; align-items: center; justify-content: center;
  font-size: 20px; color: var(--ink-dim);
  will-change: transform;
}
.wheel-item.sel { color: var(--ink); font-weight: 700; }
```

```javascript
const ITEM_H = 44;
let wTick = false;
wheel.addEventListener('scroll', () => {
  if (wTick) return;
  wTick = true;
  requestAnimationFrame(() => {
    const cy = wheel.scrollTop + wheel.clientHeight / 2;      // 滚动中心线
    wheel.querySelectorAll('.wheel-item').forEach(it => {
      const n = ((it.offsetTop + ITEM_H / 2) - cy) / ITEM_H;   // 距中心的 item 数（带小数）
      const ang = clamp(n, -3, 3) * -22;                       // 圆柱面：越远越「躺倒」
      it.style.transform = 'rotateX(' + ang + 'deg) translateZ(' + (Math.abs(n) < 1 ? 28 : 10) + 'px)';
      it.style.opacity = String(clamp(1 - Math.abs(n) / 3, 0.25, 1));
    });
    wTick = false;
  });
}, { passive: true });

wheel.addEventListener('scrollend', () => {
  const idx = Math.round(wheel.scrollTop / ITEM_H);
  items.forEach((it, i) => it.classList.toggle('sel', i === idx));
  haptic('snap');                                     // 落定一震 = 「咔哒入槽」
  output.textContent = items[idx].textContent;
});
wheel.scrollTop = 2 * ITEM_H;                          // 初始停在缓冲后的第 0 项
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 可见行数 | 5（奇数） | 中间行 + 上下各 2；3 行曲面感不足，7 行浪费高度 |
| 每档角度 | 20~26° | iOS ≈ 22°；太大像转筒，太小没曲面 |
| translateZ | 中心 28px / 外围 10px | 选中项「凸出来」—— 深度反馈 |
| 羽化 mask | 22%/78% | 上下渐隐让边缘 item 「绕到背面」的感觉 |
| 角度钳制 | \|n\| ≤ 3 | 超出视口的 item 不用算，省性能 |

### 避坑
- ❌ **JS 手写滚轮惯性** → 原生滚动 + snap 的惯性最顺；自绘是低级错误（㊳ 只用于非滚动场景）
- ❌ **perspective 加在 item 上** → 每个 item 各自为中心，没有公共圆柱面；perspective 必须在容器
- ❌ **每个 item 都 will-change 常驻** → 内存压力；只给视口附近 item 开
- ❌ **scroll 回调里同步读 offsetTop 又写 transform** → 布局抖动；rAF 节流 + 读写分离
- ❌ **头尾不补空白缓冲** → 第一项/最后一项永远到不了中心线

### 无障碍 fallback
- wheel 只是皮肤：底层必须存在 `<select>` 或 stepper 按钮（键盘可操作），`aria-hidden` 装饰轮
- reduced-motion：3D 变换关闭，保留平面 snap 列表

---

## ㊹ 主题圆形揭示 Circular Reveal（View Transitions 扩展）

### 原理
**点击切换按钮时记录按钮坐标 → `document.startViewTransition()` 包住主题 class 翻转 → CSS 把 `::view-transition-new(root)` 的 clip-path 从 `circle(0 at x y)` 扩到 `circle(150% at x y)`** —— 新主题像墨水从按钮扩散开（iOS 17 时钟风）。不支持 VT 的浏览器降级普通 fade。

### 适用模块
- 深浅色模式切换（最经典）
- 主题皮肤切换 / 强调色切换
- 「编辑模式」全屏进入

### 不适用
- ❌ 高频切换（每次 VT 截屏全页合成，0.5s+ 内别连点）
- ❌ 低端机默认开启（VT 合成开销大；按设备降级 fade）

### 核心代码

```javascript
const themeBtn = document.getElementById('themeBtn');
themeBtn.addEventListener('click', () => {
  const { left, top, width, height } = themeBtn.getBoundingClientRect();
  const x = left + width / 2, y = top + height / 2;
  const r = Math.hypot(Math.max(x, innerWidth - x), Math.max(y, innerHeight - y));

  const flip = () => document.documentElement.classList.toggle('light');

  if (!document.startViewTransition || reduced) { flip(); return; }   // 降级

  const vt = document.startViewTransition(flip);
  vt.ready.then(() => {
    document.documentElement.animate(
      { clipPath: ['circle(0px at ' + x + 'px ' + y + 'px)',
                   'circle(' + r + 'px at ' + x + 'px ' + y + 'px)'] },
      { duration: 480, easing: 'cubic-bezier(0.32, 0.72, 0, 1)',
        pseudoElement: '::view-transition-new(root)' }   // 只对新快照做揭示
    );
  });
});
```

```css
/* 关闭 VT 默认的交叉淡入，为圆形揭示让路 */
::view-transition-old(root),
::view-transition-new(root) { animation: none; mix-blend-mode: normal; }
::view-transition-old(root) { z-index: 1; }        /* 旧主题垫底当画布 */
::view-transition-new(root) { z-index: 2; }        /* 新主题从按钮位置扩散 */

/* 按钮本体也做一个小细节：图标 sun/moon 旋转交替 */
.theme-ico { transition: transform .4s cubic-bezier(0.34,1.56,0.64,1); }
html.light .theme-ico { transform: rotate(180deg) scale(1.15); }
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 半径 | `hypot(最远角)` | 一次到位盖满全屏，不用二次扩展 |
| duration | 450~550ms | 太快看不出扩散方向，太慢拖沓 |
| 曲线 | iOS 曲线 `cubic-bezier(0.32,0.72,0,1)` | 扩散是导航级动作，不要 spring |
| 揭示起点 | 点击的按钮中心 | 用户按哪从哪扩散 —— 因果感 |

### 避坑
- ❌ **clip-path 起点用屏幕中心** → 失去「从我这按下的地方长出来」的因果感
- ❌ **不给 VT 的伪元素清 animation** → 默认交叉淡入和 clip-path 打架，呈现两次过渡
- ❌ **每次切换都 150% 半径 clip-path 常驻** → 揭示动画结束 VT 自动清理；别自己把 clip-path 写进全局样式
- ❌ **没做能力检测** → Firefox 旧版无 VT；`document.startViewTransition` 判断是唯一开关

### 无障碍 fallback
- 降级链已内置：无 VT / reduced-motion → 直接翻转 class，即时切换
- 按钮必须有 `aria-pressed` 或 `aria-label="切换深色模式"` 语义

---

## ㊺ 按住说话 Hold-to-Talk（波形 + 上滑取消）

### 原理
**按住按钮 → 波形条组随机高度抖动 + 计时 → 手指上滑超 80px 出现「松开取消」态（按钮变红 + 波形冻结）→ 松手判定：y 位移 < 80px 发送（气泡插入 + ㊽ 庆祝可选），≥ 80px 取消（气泡飞散）**。`pointercancel` 一律按取消处理 —— 来电中断绝不发半句话。

### 适用模块
- IM 语音消息（微信式）
- 语音评论 / 语音搜索
- 对讲功能

### 不适用
- ❌ 网页正文区（无语音心智；用输入框）
- ❌ 页面有全局语音助手冲突时

### 核心代码

```javascript
const talkBtn = document.getElementById('talkBtn');      // 大圆形 mic 按钮
const bars = Array.from(document.querySelectorAll('.talk-bar'));   // 波形条 5~9 根
let ht = null, timer = null, barTimer = null;

talkBtn.addEventListener('pointerdown', (e) => {
  ht = { y0: e.clientY, cancel: false, t0: performance.now() };
  talkBtn.setPointerCapture(e.pointerId);
  talkBtn.classList.add('holding');                       // scale 1.06 + 背景亮起
  haptic('tap');
  barTimer = setInterval(() => bars.forEach(b => {       // 波形随机抖动
    b.style.height = (8 + Math.random() * 26) + 'px';
  }), 110);
  timer = setInterval(() => {                             // 计时
    timeEl.textContent = fmtT(performance.now() - ht.t0);
  }, 100);
});

talkBtn.addEventListener('pointermove', (e) => {
  if (!ht) return;
  const up = ht.y0 - e.clientY;
  ht.cancel = up > 80;                                    // 上滑 80px → 取消态
  talkBtn.classList.toggle('canceling', ht.cancel);
  stateEl.textContent = ht.cancel ? '松开取消' : '松开发送 · 上滑取消';
});

talkBtn.addEventListener('pointerup', () => {
  if (!ht) return;
  const cancel = ht.cancel; ht = null;
  stopTalk();
  if (cancel) { haptic('warn'); showBubbleCanceled(); }   // 气泡虚化消散
  else { haptic('success'); showVoiceBubble(); }          // 插入语音气泡（时长带进 UI）
});
talkBtn.addEventListener('pointercancel', () => {        // 系统中断 = 静默取消
  ht = null; stopTalk();
});

function stopTalk() {
  clearInterval(timer); clearInterval(barTimer);
  talkBtn.classList.remove('holding', 'canceling');
  bars.forEach(b => b.style.height = '8px');
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 取消阈值 | 上滑 80px | 与拇指划出按钮区域的自然距离匹配 |
| 波形频率 | 100~130ms 换一次高度 | 太快乱，太慢呆 |
| 波形条 | 5~9 根 | 奇数对称好看 |
| 最短时长 | 500ms | 低于则提示「太短」不发送（防误触） |
| 计时显示 | 每 100ms 刷新 | 数字要 `tabular-nums` |

### 避坑
- ❌ **pointercancel 当发送处理** → 来电/手势中断把半句话发出去，事故级 bug
- ❌ **波形用 JS 每帧随机** → 神经质；100ms 节拍 + CSS transition 才有「呼吸感」
- ❌ **取消无触感区分** → 发送 success 双脉冲 / 取消 warn 三连震，波形也要区分（冻结 vs 抖动）
- ❌ **上滑取消后松手仍发送** → ht.cancel 判定要在 move 里持续更新，别只在起始记一次
- ❌ **按钮长按没防滚动** → `touch-action: none`（语音按钮内不存在滚动需求）

### 无障碍 fallback
- 必须提供普通「点按录音 / 再点停止」模式（无长按心智的用户与读屏用户）
- reduced-motion：波形静态 + 计时照常

---

## ㊻ 骨架屏 Shimmer（同尺寸防抖动）

### 原理
**灰底条 + 高光斜带从左到右扫过（`linear-gradient` + `background-position` 动画）**。核心纪律：**骨架布局与真实内容逐像素同尺寸**（同宽高、同圆角、同间距），数据到达后骨架淡出、内容从 `translateY(8px)`+opacity 0 淡入 —— 两者重叠的瞬间没有布局跳动。

### 适用模块
- Feed 卡片 / 详情页首屏占位
- 图片位占位（图片 onload 再替换）
- 下拉刷新后的刷新区

### 不适用
- ❌ 秒级以内的接口（骨架闪一下反而干扰；直接转圈或无 loading）
- ❌ 内容尺寸不可预知的瀑布流（骨架猜不准 = 抖动更大；用模糊低质图占位）

### 核心代码

```css
.sk { position: relative; overflow: hidden; background: #232838; border-radius: 8px; }
.sk::after {
  content: '';
  position: absolute; inset: 0;
  transform: translateX(-100%);           /* 用 transform 扫光，别动 background-position（要重绘） */
  background: linear-gradient(90deg, transparent, rgba(255,255,255,.07) 40%, rgba(255,255,255,.12) 50%, rgba(255,255,255,.07) 60%, transparent);
  animation: shimmer 1.4s infinite;
}
@keyframes shimmer { to { transform: translateX(100%); } }

/* 与真实内容同尺寸对齐：骨架直接复用内容类的尺寸 */
.skeleton-card { /* 与 .real-card 相同的 padding/gap/图片高 */ }

/* 完成过渡：骨架 → 内容，两帧重叠替换 */
.loaded .sk { opacity: 0; transform: scale(0.98); transition: opacity .25s; pointer-events: none; }
.loaded .real { opacity: 1; transform: translateY(0); }
.real { position: absolute; inset: 0; opacity: 0; transform: translateY(8px); transition: opacity .3s, transform .3s var(--ios-ease); }
```

```html
<div class="card-wrap" id="card">            <!-- 固定尺寸的锚容器 -->
  <div class="sk skeleton-card" id="sk">…骨架条…</div>
  <div class="real" id="real">…真实内容…</div>
</div>
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 扫光周期 | 1.2~1.6s | 快于 1s 像报警灯，慢于 2s 像卡死 |
| 高光带 | ~50% 宽，峰值 alpha 0.1~0.14 | 骨架要安静；高光是「呼吸」不是「霓虹」 |
| 完成过渡 | 骨架 0.25s 淡出 + 内容 0.3s 淡入上移 | 谁先谁后无所谓，**尺寸必须零抖动** |
| 最长骨架时长 | 3s | 超时换错误态 + 重试按钮；骨架不能无限闪 |

### 避坑
- ❌ **骨架尺寸猜的（图片高度随便给）** → 内容到达瞬间布局跳动 —— 骨架屏存在的全部意义就是消抖
- ❌ **扫光动 background-position** → 每帧重绘；transform 扫光是合成层
- ❌ **骨架闪现 <300ms** → 数据太快时骨架成了噪点；300ms 内到达直接上内容
- ❌ **骨架与内容排版不一致（行数/圆角不同）** → 用户看到两次「跳版」
- ❌ **多卡片各自节奏** → 同屏骨架共享一个 shimmer 相位（`animation-delay` 递增 0~200ms 即可，别完全随机）

### 无障碍 fallback
- `aria-busy="true"` 标在骨架容器上，读屏可感知加载中
- reduced-motion：扫光停止，骨架静态灰块

---

## ㊼ 角标弹跳 Badge Bump（数字变化动画族）

### 原理
**数字变化瞬间：scale 1→1.35→1（`cubic-bezier(0.34,1.56,0.64,1)` 回弹）+ 颜色脉冲；增量用「+N」小字从原值上飞出淡出**。retrigger 技巧：`el.classList.remove('bump'); void el.offsetWidth; el.classList.add('bump')` 强制重启动画。99+ 截断、去重合并（连击时只 bump 不叠加数字）。

### 适用模块
- 点赞 / 消息 / 购物车数量变化（高频）
- Tab 未读数、导航角标
- 直播观看人数上涨

### 不适用
- ❌ 实时流数字（每秒都在变的高频行情 —— 弹跳变成抽搐，直接数字滚动）
- ❌ 关键金额/时间（动画延迟感知，金融数字直接换）

### 核心代码

```css
.badge {
  display: inline-flex; align-items: center; justify-content: center;
  min-width: 22px; height: 22px; padding: 0 6px;
  border-radius: 99px;
  background: var(--danger); color: #fff;
  font-size: 12px; font-weight: 700;
  font-variant-numeric: tabular-nums;
}
.badge.bump { animation: badgeBump .45s cubic-bezier(0.34, 1.56, 0.64, 1); }
@keyframes badgeBump {
  0% { transform: scale(1); }
  40% { transform: scale(1.35); background: #ff7d8e; }   /* 颜色同步脉冲 */
  100% { transform: scale(1); }
}
.badge .fly {                                   /* +N 飞出 */
  position: absolute; left: 50%;
  font-size: 12px; font-weight: 800; color: var(--danger);
  animation: flyUp .7s ease-out forwards; pointer-events: none;
}
@keyframes flyUp {
  from { transform: translate(-50%, 0); opacity: 1; }
  to   { transform: translate(-50%, -22px); opacity: 0; }
}
```

```javascript
function bumpBadge(el, delta) {
  el.classList.remove('bump');
  void el.offsetWidth;                       // 强制 reflow 重启动画 —— retrigger 唯一正解
  el.classList.add('bump');
  if (delta > 0) {
    const fly = document.createElement('span');
    fly.className = 'fly';
    fly.textContent = '+' + delta;
    el.appendChild(fly);
    setTimeout(() => fly.remove(), 700);
  }
  haptic('tap');
}
function setBadgeCount(el, n) {
  el.textContent = n > 99 ? '99+' : String(n);   // 截断
  bumpBadge(el, ...);
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 弹跳幅度 | 1.3~1.4 | 低于 1.2 感知不到，高于 1.5 干扰相邻元素 |
| duration | 0.4~0.5s | 回弹动画的过冲需要时间展开 |
| 颜色脉冲 | 同色系 +10% 亮度 | 颜色+形状双通道（三冗余原则的轻量版） |
| +N 飞出 | 向上 22px / 0.7s | 只在 delta ≥ 2 时展示；+1 每次飞太闹 |
| 触感 | tap 单脉冲 | 与视觉 bump 同帧发出 |

### 避坑
- ❌ **连击时数字一格一格加** → 合并：动画 450ms 内的增量等动画结束一次性 bump
- ❌ **动画没重启（class 已在）** → `void offsetWidth` retrigger；或者用 `el.getAnimations().forEach(a => a.cancel())`
- ❌ **角标宽度不定导致布局抖** → `min-width` + `tabular-nums`
- ❌ **每个 tab 都 bump** → 只 bump 用户当前不看的位置（别打扰当前焦点区）
- ❌ **reduced-motion 下还飞 +N** → 静态数字直接换

### 无障碍 fallback
- `aria-live="polite"` 放在数字容器上，读屏播报变化（「3 条新消息」）
- reduced-motion：无 scale 无飞字，只有颜色一闪（颜色也不能是唯一通道，需配合 aria-live 文本）

---

## ㊽ 庆祝粒子 Confetti / Emoji Rain（成功情绪反馈）

### 原理
**emoji 先预渲染成离屏 canvas 位图（每个精灵只画一次），每帧只做 `drawImage` 贴图**。粒子从触发点向上抛（vx ±3、vy -8~-13），每帧 `vy += 0.35` 重力、旋转；落出屏底销毁，全部消亡即停 rAF。同屏同时只允许一场（新庆祝替换旧的）。

> **为什么必须预渲染**：彩色 emoji 是 canvas 上最贵的绘制操作之一。130 个粒子 × 每帧 `ctx.font` + `fillText`（emoji）× 60fps ≈ 每秒 7800 次字形光栅化 —— 实测必掉帧。预渲染后每帧只剩 130 次位图 blit，手机上满帧跑。

### 适用模块
- 支付成功 / 下单完成 / 发布成功
- 达成目标（打卡满 7 天、通关）
- 运营活动抢到券

### 不适用
- ❌ 常规成功（保存设置之类 —— 一次 toast 足够，撒花就是噪音）
- ❌ 深色阅读沉浸页（打破氛围）
- ❌ 同一动作可无限重复触发的入口（每次点都撒花=通货膨胀）

### 核心代码

```javascript
const fx = document.getElementById('fx');      // 全屏 fixed canvas，pointer-events: none
const ctx = fx.getContext('2d');
const FX_DPR = Math.min(devicePixelRatio || 1, 2);   // DPR 钳制：手机 DPR 3 全屏画布 = 3x 像素
let parts = [], fxRaf = null;

function resizeFx() {
  fx.width = innerWidth * FX_DPR;
  fx.height = innerHeight * FX_DPR;
  fx.style.width = innerWidth + 'px';
  fx.style.height = innerHeight + 'px';
  ctx.setTransform(FX_DPR, 0, 0, FX_DPR, 0, 0);
}

// 关键：emoji × 3 档尺寸 → 离屏 canvas 精灵（只画一次）
let sprites;
function buildSprites(chars) {                  // chars: ['🎉','✨','🎊']
  sprites = [];
  chars.forEach(ch => [16, 24, 30].forEach(size => {
    const c = document.createElement('canvas');
    const s = Math.ceil(size * FX_DPR);
    c.width = s; c.height = s;
    const cc = c.getContext('2d');
    cc.font = s + 'px serif';
    cc.textBaseline = 'middle'; cc.textAlign = 'center';
    cc.fillText(ch, s / 2, s / 2);
    sprites.push({ img: c, size });
  }));
}
buildSprites(['🎉', '✨', '🎊']);

function confetti(x, y) {
  cancelAnimationFrame(fxRaf);
  resizeFx();
  const count = innerWidth < 480 ? 80 : 130;    // 小屏减粒子
  parts = Array.from({ length: count }, () => {
    const sp = sprites[Math.random() * sprites.length | 0];
    return {
      x, y,
      vx: (Math.random() - 0.5) * 7,
      vy: -(8 + Math.random() * 6),
      rot: Math.random() * Math.PI * 2,
      vr: (Math.random() - 0.5) * 0.3,
      img: sp.img, size: sp.size
    };
  });
  haptic('success');
  (function loop() {
    ctx.setTransform(FX_DPR, 0, 0, FX_DPR, 0, 0);
    ctx.clearRect(0, 0, innerWidth, innerHeight);
    parts.forEach(p => {
      p.vy += 0.35; p.x += p.vx; p.y += p.vy;
      p.rot += p.vr; p.vx *= 0.99;
      ctx.setTransform(FX_DPR, 0, 0, FX_DPR, p.x, p.y);   // setTransform 替代 save/restore
      ctx.rotate(p.rot);
      ctx.drawImage(p.img, -p.size / 2, -p.size / 2, p.size, p.size);
    });
    parts = parts.filter(p => p.y < innerHeight + 40);
    if (parts.length) fxRaf = requestAnimationFrame(loop);
    else { ctx.setTransform(FX_DPR, 0, 0, FX_DPR, 0, 0); ctx.clearRect(0, 0, innerWidth, innerHeight); }
  })();
}

function celebrateAt(el) {
  const r = el.getBoundingClientRect();
  confetti(r.left + r.width / 2, r.top);
}
```

```css
#fx { position: fixed; inset: 0; pointer-events: none; z-index: 999; }
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 粒子数 | 130（小屏 80） | < 60 稀疏寒酸；> 200 即使预渲染低端机也吃紧 |
| 重力 | 0.35 / frame² | 0.2 飘太久，0.5 直坠 |
| 初速 | vy -8~-13 | 抛物线顶点略高于触发点即可 |
| 精灵尺寸 | 16 / 24 / 30 三档 | 大小随机是层次感来源；位图按 FX_DPR 放大烘焙 |
| emoji 集 | 3~5 种 | 单一 emoji 显得模板化，太多像贴纸墙 |
| DPR | 钳制 ≤ 2 | DPR 3 的手机全屏画布 ≈ 300 万像素/帧合成 |
| 出场率 | 一个用户流只 1 处 | 稀缺才有情绪价值 |

### 避坑
- ❌ **每帧每粒子 `ctx.font` + emoji `fillText`** → 每秒近 8000 次彩色字形光栅化，必掉帧；**预渲染离屏位图是唯一正解**（实测从掉帧到满帧的差距就在这一步）
- ❌ **canvas 不钳制 DPR** → DPR 3 的手机上全屏画布是普通屏的 9 倍像素；`Math.min(dpr, 2)` 肉眼无损
- ❌ **每粒子 save/restore** → 130 次/帧变换栈操作；`setTransform` 一步到位
- ❌ **canvas 挡住点击** → `pointer-events: none` 忘了加 = 全页面点不了
- ❌ **粒子用 DOM 元素** → 130 个 span 动画秒掉帧；canvas 是唯一解
- ❌ **rAF 不清理** → 全部落幕后循环还在跑，发热耗电
- ❌ **从屏幕中心喷** → 从触发按钮/触发点喷，因果感（同 ㊹ 圆形揭示）
- ❌ **每次 burst 重建精灵** → 精灵只需建一次；burst 只随机引用

### 无障碍 fallback
- 庆祝是纯情绪装饰：核心信息（「支付成功」）必须已有文字/toast，粒子只锦上添花
- reduced-motion：直接不喷；触感 success 保留

---

# 第 7 档 · 前沿 CSS 原生动效（49–56）

> 2023–2025 年浏览器陆续落地的 CSS 原生动画能力，趋势是**把过去必须 JS 手写的动效交还 CSS**：滚动驱动、`display:none` 过渡、纯 CSS 弹簧曲线、`height:auto` 过渡、锚点定位、跨文档转场。Unicode 圈号止于 ㊿，本档起用阿拉伯数字编号。
> **使用总则：渐进增强**——增强声明包 `@supports`，基线体验不依赖这些特性；所有模式写法自带零成本降级。浏览器支持度数据以本手册编写时点为准（2026-10），发布前用 caniuse 复核。

## 49 滚动驱动动画 Scroll-Driven Animations

### 原理
`animation-timeline` 把动画进度绑定到**滚动位置**而不是时间：`view()` = 元素在视口中的可见进度，`scroll()` = 滚动容器进度。滚动即播放、松手即停，合成器驱动不占主线程，比 IntersectionObserver + rAF 更顺滑且零 JS。② 滚动触发、① 视差、㉒ 滚动驱动数字的现代纯 CSS 实现。

### 适用模块
- 滚动 reveal（替代 ② 的 IO 版本）
- 顶部阅读进度条
- 视差背景层
- 滚动驱动的数字 / 进度环

### 不适用
- ❌ 需要触发后自行播完的入场动画（时间驱动更合适）
- ❌ 拖拽类跟手交互（仍用 JS）

### 核心代码

```css
/* 1) 滚动 reveal：元素进入视口 0%→60% 区间完成入场 */
@keyframes rise-in {
  from { opacity: 0; transform: translateY(48px); }
  to   { opacity: 1; transform: none; }
}
@supports (animation-timeline: view()) {
  .rise {
    animation: rise-in linear both;        /* both 必须：区间外保持端点态 */
    animation-timeline: view();
    animation-range: entry 0% entry 60%;
  }
}

/* 2) 顶部阅读进度条（页面级） */
@keyframes grow-x { from { transform: scaleX(0); } to { transform: scaleX(1); } }
@supports (animation-timeline: scroll()) {
  .progress {
    position: fixed; inset: 0 0 auto 0; height: 3px; transform-origin: 0 50%;
    animation: grow-x linear both;
    animation-timeline: scroll(root block);
  }
}

/* 3) 视差：同一滚动反向驱动背景层 */
@keyframes drift-up { from { transform: translateY(8vh); } to { transform: translateY(-8vh); } }
.parallax-bg {
  animation: drift-up linear both;
  animation-timeline: view();
  animation-range: cover;
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| animation-range | entry 0% entry 60% | 入场在进入视口六成处完成；太晚用户已划过 |
| fill-mode | both | 缺了区间外会闪回无动画态 |
| timeline | view() / scroll(root block) | 页面级用 root；容器内用 nearest |
| 视差幅度 | ±8~12vh | 超过会露出背景边缘 |
| easing | linear | scrub 场景 ease 会作用在进度映射上，通常不要 |

### 避坑
- ❌ **把初始态写在元素上**（如 `.rise { opacity: 0 }`）→ 不支持的浏览器元素**永久隐身**；初始态只写进 `@keyframes` 的 from，且整个 animation 声明包 `@supports`
- ❌ **动画里改 layout 属性**（top / width）→ 丢掉合成器加速；只用 transform / opacity
- ❌ **reduced-motion 不处理** → 整段包 `@media (prefers-reduced-motion: no-preference)`
- ❌ **给 scrub 动画配 spring 曲线** → 滚动反向时会看到曲线回放，观感错乱

### 支持与降级
Chrome/Edge 115+；Safari 与 Firefox 以 caniuse 实查为准。写法本身零成本降级：不支持时元素静态可见（无入场动画但内容完整）。

## 50 @starting-style 原生进出场

### 原理
CSS 过渡的死角：元素从 `display:none` 显示或刚插入 DOM 时没有「上一帧」，transition 不触发。`@starting-style` 声明「第一帧」样式，配合 `transition-behavior: allow-discrete` 让 `display` / `overlay` 这类离散属性也参与过渡 —— 弹层 / dialog / toast 的进出场完全不需要 JS 切 class。

### 适用模块
- dialog / popover 打开与关闭（离场也平滑，这是 JS 方案最难写的部分）
- toast 插入动画
- tooltip 淡入

### 不适用
- ❌ 需要跟手拖拽的浮层（仍 JS）
- ❌ 高频复用同节点反复重触发（重新触发语义依赖重新渲染）

### 核心代码

```css
.sheet {
  /* 关闭态 = 离场过渡的目标态 */
  opacity: 0; transform: translateY(24px) scale(.97);
  transition: opacity .32s cubic-bezier(.32,.72,0,1),
              transform .32s cubic-bezier(.32,.72,0,1),
              display .32s allow-discrete,     /* display 跳变期间保持渲染 */
              overlay .32s allow-discrete;     /* top layer 提升也过桥 */
}
.sheet:popover-open, dialog.sheet:open {
  opacity: 1; transform: none;
}
@starting-style {
  /* 第一帧：进场从这里过渡到打开态 */
  .sheet:popover-open, dialog.sheet:open { opacity: 0; transform: translateY(24px) scale(.97); }
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| duration | 280~360ms | 弹层标准时长 |
| 曲线 | cubic-bezier(.32,.72,0,1) | iOS sheet 手感 |
| 位移 | 16~28px translateY | 太大像「掉下来」 |

### 避坑
- ❌ **只写 @starting-style 不写 allow-discrete** → 进场有过渡、**离场瞬间消失**（display 直接跳变），最常见的翻车点
- ❌ **transition 只列 opacity/transform** → display 跳变会让整段过渡作废；**四条都要列**
- ❌ **关闭态基样式漏写** → 浏览器不知道往哪过渡；关闭态值 = @starting-style 值
- ❌ **和 Web Animations 抢同一属性** → 后到者赢，排查半小时的那种 bug

### 支持与降级
Chrome 117+ / Safari 17.5+ / Firefox 129+。老浏览器进场退化为瞬时出现（功能完整）；追求一致体验用 @supports 分支保留旧 JS class 方案。

## 51 Popover API 零 JS 弹层

### 原理
`popover` 属性把元素提升到 **top layer**（永远在最上层、自带 backdrop、Esc / 外点自动关闭、焦点管理），一个 `popovertarget` 按钮即可开关 —— 过去手写 modal 的 30 行 JS（遮罩、Esc、外点、z-index、焦点回收）全部免费。

### 适用模块
- 下拉菜单 / 操作菜单
- tooltip 型说明卡
- 非阻塞通知（`popover="manual"`）
- **现代弹层三件套**：51 popover（载体）+ 50 @starting-style（进出场）+ 54 锚点（定位）

### 不适用
- ❌ 需要阻塞交互的确认框（用 `<dialog>` + `showModal()`，popover 不阻塞）
- ❌ 需要复杂拖拽的浮层

### 核心代码

```html
<button popovertarget="menu">打开菜单</button>
<div id="menu" popover class="sheet">
  <button>复制链接</button><button>置顶</button><button>删除</button>
</div>
```

```css
.menu { border: none; border-radius: 14px; padding: 6px; box-shadow: 0 12px 40px rgb(0 0 0 / .25); }
.menu::backdrop { background: rgb(0 0 0 / .1); }
```

```javascript
// 需要感知开关时机（埋点 / 联动动画）：
menu.addEventListener('toggle', e => e.newState === 'open' && onOpen());
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| popover 值 | auto（默认） | auto = 外点/Esc 关且同级互斥；manual = 常驻（toast 用） |
| backdrop | transparent ~ rgb(0 0 0/.15) | 菜单类透明，模态感用 dialog |

### 避坑
- ❌ **auto popover 里再开 auto popover** → 父层被自动关掉；嵌套用 manual 或改 dialog
- ❌ **自己写 display 控制显隐** → popover 属性自带显隐逻辑，再写会打架；显隐动画只交给 50
- ❌ **往 ::backdrop 上放交互** → backdrop 点击 = light dismiss，点了就关

### 支持与降级
Chrome 114+ / Safari 17+ / Firefox 125+，已是 Baseline。老浏览器 feature-detect 后回退 dialog/class 方案。

## 52 linear() 弹簧缓动

### 原理
`linear()` 缓动函数接受任意个「进度 位置%」采样点，可以把整条弹簧位移曲线（含 1~2 次过冲震荡）编码进一条 CSS 声明 —— ⑦ 的 JS 物理弹簧在**非跟手场景**（hover、展开、指示器落位）下的纯 CSS 替代。

### 适用模块
- Tab 指示器 / segmented control 落位
- hover 卡片抬起回弹
- toggle 拨杆

### 不适用
- ❌ 拖拽松手回弹（需要真实末速，仍用 ⑦ rAF）
- ❌ duration > 800ms 的动画（弹簧尾巴被拉长变粘滞）

### 核心代码

```css
/* 采样点由生成器导出（stiffness≈100 / damping≈12）：
   https://linear-easing-generator.netlify.app  */
:root {
  --spring: linear(
    0, 0.0036 1.1%, 0.0185 2.2%, 0.049 3.4%, 0.096 4.7%, 0.171 6.1%,
    0.269 7.5%, 0.386 8.9%, 0.583 10.9%, 0.857 13.8%, 1.004 15.7%,
    1.125 17.5%, 1.197 19.3%, 1.226 20.9%, 1.220 22.3%, 1.187 23.5%,
    1.134 24.6%, 1.006 26.4%, 0.923 27.3%, 0.855 28.1%, 0.796 28.9%,
    0.685 30.2%, 0.609 31.4%, 0.553 32.6%, 0.517 33.7%, 0.497 34.9%,
    0.494 36.1%, 0.504 37.3%, 0.524 38.6%, 0.713 45.2%, 0.767 48.3%,
    0.806 51.3%, 0.829 54.4%, 0.839 57.4%, 0.836 60.5%, 0.832 66.2%,
    0.850 72.0%, 0.876 77.8%, 0.902 83.8%, 0.924 90.0%, 0.940 96.3%, 0.956 100%
  );
}
.tab-indicator { transition: transform .55s var(--spring); }
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 过冲峰值 | 1.15~1.25 | 超过 1.3 像果冻，低于 1.1 感知不到 |
| 震荡次数 | 1~2 次 | 弹簧高级感在「一次过冲一次回落」 |
| duration | 450~650ms | 曲线自带节奏，时长要给足 |
| 生成器 | linear-easing-generator | 调 stiffness/damping 导出，别手拍点 |

### 避坑
- ❌ **手写采样点** → 容易出折角；用生成器从弹簧参数导出
- ❌ **跟手场景用它** → 松手回弹需要衔接真实末速，只有 ⑦ 的 rAF 物理做得到
- ❌ **var() 失效无兜底** → 不支持的浏览器整条声明作废回落默认 ease；在意就 @supports 双写 `cubic-bezier(.34,1.56,.64,1)`

### 支持与降级
Chrome 113+ / Safari 17.2+ / Firefox 112+。默认降级（回落 ease）通常可接受。

## 53 height:auto 过渡（grid 0fr→1fr / interpolate-size）

### 原理
手风琴千百年的痛：`height: auto` 不能过渡。两条路：**A. `grid-template-rows: 0fr→1fr`**（全兼容，默认首选）；**B. `interpolate-size: allow-keywords`** 直接让 auto 参与插值（新，代码直白）。

### 适用模块
- 手风琴 FAQ
- 展开 / 收起面板
- 通知卡离场后高度坍缩合洞（㊷ 的纯 CSS 版）

### 不适用
- ❌ 中途跟手的拖拽展开（Bottom Sheet ㉚ 仍 JS）

### 核心代码

```css
/* A. 全兼容：grid 0fr→1fr（首选） */
.acc {
  display: grid; grid-template-rows: 0fr;
  transition: grid-template-rows .38s cubic-bezier(.32,.72,0,1);
}
.acc.open { grid-template-rows: 1fr; }
.acc > .body { overflow: hidden; min-height: 0; }   /* min-height:0 是灵魂，漏了 0fr 不生效 */

/* B. 新特性：interpolate-size（Chrome 129+） */
@supports (interpolate-size: allow-keywords) {
  :root { interpolate-size: allow-keywords; }      /* 必须写 :root，元素上写无效 */
  .acc-b { height: 0; transition: height .38s cubic-bezier(.32,.72,0,1); }
  .acc-b.open { height: auto; }
  /* calc-size(auto, size + 24px) 可在 auto 基础上加余量 */
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 曲线 | cubic-bezier(.32,.72,0,1) | iOS 展开手感 |
| duration | 320~420ms | 手风琴标准 |
| overflow | hidden / clip | 插值期间内容必须裁剪 |

### 避坑
- ❌ **grid 法忘写 `min-height: 0`** → grid item 默认 `min-height: auto`，0fr 压不下去；这是「不生效」的第一嫌疑
- ❌ **interpolate-size 写在元素上** → 只在 `:root` 声明才生效
- ❌ **内容里有未定尺寸图片** → 插值中图片载入会跳变；预留 `aspect-ratio`

### 支持与降级
grid 法：全浏览器。interpolate-size：Chrome 129+。默认用 A，B 仅作增强。

## 54 锚点定位 Anchor Positioning

### 原理
`anchor-name` + `position-anchor` + `anchor()` / `position-area`：浮层声明自己锚定谁，浏览器负责定位、间距、**视口贴边自动翻折** —— tooltip / 下拉的 getBoundingClientRect + 翻转逻辑全部退役。

### 适用模块
- tooltip / hint
- 下拉菜单跟随触发按钮
- 表单错误提示锚定字段

### 不适用
- ❌ 跟手移动的浮层
- ❌ 拖拽浮出卡片

### 核心代码

```css
.save-btn { anchor-name: --save; }

/* 降级基线先行：不支持的浏览器按普通 absolute 走（或 JS 定位） */
.tip { position: absolute; bottom: 130%; left: 50%; translate: -50% 0; }

@supports (anchor-name: --a) {
  .tip {
    position-anchor: --save;
    position-area: top;                            /* 默认在按钮上方 */
    position-try-fallbacks: flip-block, flip-inline;  /* 贴边自动翻折 */
    margin-bottom: 8px;
  }
  /* 等宽跟随触发元素：width: anchor-size(--save width); */
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| position-area | top / bottom span-right | 常用方位 |
| fallbacks | flip-block, flip-inline | 视口边缘翻折组合 |
| 间距 | margin 8px | 用 margin 撑，别用 translate 硬编码 |

### 避坑
- ❌ **只写增强不写基线** → 不支持的浏览器浮层飞到页面左上角；**先写可用默认态，再 @supports 增强**
- ❌ **anchor-name 重名** → 同页多个锚点必须唯一命名
- ❌ **锚定元素被隐藏** → 锚点消失浮层回落 static；隐藏按钮时同步隐藏浮层

### 支持与降级
Chrome/Edge 125+（其余在途）。需要全浏览器统一行为时用 floating-ui；否则基线 absolute + @supports 增强。

## 55 跨文档 View Transitions（MPA）

### 原理
⑯ 只覆盖 SPA；`@view-transition { navigation: auto; }` 让**普通多页站**在跳页时也走浏览器原生转场：浏览器截旧页快照 → 拦截导航 → 新页首绘后交叉过渡，共享元素靠两页**同名** `view-transition-name` 无缝飞行。

### 适用模块
- 多页官网 / 博客（列表页 ↔ 文章页，封面图飞过去）
- 无框架 SSR 站点

### 不适用
- ❌ SPA（用 ⑯）
- ❌ 依赖 POST 数据的跳转（跨文档 VT 只走同源导航）

### 核心代码

```css
/* 两页都加载的公共 CSS */
@view-transition { navigation: auto; }
@media (prefers-reduced-motion: reduce) {
  @view-transition { navigation: none; }
}

/* 共享元素：列表页与详情页的同一张图同名 */
.hero-img { view-transition-name: hero-img; }
::view-transition-old(hero-img), ::view-transition-new(hero-img) { animation-duration: .45s; }
```

```html
<!-- 新页：让关键内容先渲染、再放行转场截图，避免快照拍到空白 -->
<link rel="expect" href="#hero-img" blocking="render">
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 共享元素 | ≤ 2 个/页 | 每个元素一次截图，多了转场拖沓 |
| duration | 300~450ms | 默认 250ms 偏快 |
| rel=expect | hero 元素 | 保证快照时有内容 |

### 避坑
- ❌ **两页 CSS 不同步** → name 只在一页存在就没有共享飞行，退化为整体淡入淡出（本身可接受）
- ❌ **忘记 rel="expect"** → 新页没渲染出 hero 就截图，飞入一张空白
- ❌ **把 bfcache 返回当 bug** → 前进转场、后退瞬时恢复是预期行为
- ❌ **拿 VT 挡路由逻辑** → VT 不改变导航语义，别当拦截器用

### 支持与降级
Chrome/Edge 126+；Firefox/Safari 以 caniuse 实查。不支持 = 普通跳页，零成本降级。

## 56 滚动标记 ::scroll-marker（实验性）

### 原理
CSS Scroll Markers：横向 snap 轨道的每个 item 生成一个 `::scroll-marker` 伪元素，自动汇入容器的 `::scroll-marker-group`，浏览器自动高亮当前 snap 项 —— 轮播圆点的 JS 同步逻辑归零。

### 适用模块
- 横滑卡片轨道指示点
- 全屏 onboarding 分页点

### 不适用
- ❌ 需要点击跳转的点（当前 spec 点位不可交互；等 stable 再上生产）

### 核心代码

```css
.track { overflow-x: auto; scroll-snap-type: x mandatory; }
.track::scroll-marker-group {
  display: flex; gap: 8px; justify-content: center;
  position: fixed; bottom: 16px; inset-inline: 0;
}
.track .card::scroll-marker {
  content: ""; width: 8px; height: 8px; border-radius: 50%;
  background: rgb(255 255 255 / .3);
  transition: background .3s, scale .3s;
}
@supports selector(::scroll-marker) {
  /* 选中态选择器以当前 spec 为准（下例按 :target-current 写），
     上线前对照 MDN 复核 —— 本模式整体标记为实验性 */
  .track .card:target-current::scroll-marker { background: var(--accent); scale: 1.25; }
}
```

### 调参指南

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 点径 | 6~8px | 8px 以上像儿童 UI |
| 高亮 | 变色 + scale 1.25 | 双通道提示 |

### 避坑
- ❌ **当稳定特性上生产** → 仅最新 Chromium 有；生产用 IO 点位方案兜底（㉘ 同款）
- ❌ **marker-group 忘了定位** → 默认渲染在轨道流末尾

### 支持与降级
Chrome 133+。`@supports selector(::scroll-marker)` 探测；降级 = 现有 JS 指示点方案。

## 附：渲染层前沿速览（选型时知道有这些，暂不进正式模式）

| 技术 | 一句话 | 何时用 | 代价 |
|------|--------|--------|------|
| **SDF 着色器转场** | WebGL fragment shader 用 signed distance field 混合两张纹理，dissolve / blob 转场 | 高端官网 hero 切换、作品集封面 | 需 shader 功底；纹理尺寸规整；fallback fade |
| **液态玻璃折射** | backdrop-filter blur + SVG feDisplacementMap 位移贴图，玻璃边缘折射（Apple Liquid Glass 风） | iOS 26 风组件、dock、工具条 | feDisplacementMap 每帧重采样贵，控制在小面积 |
| **可变字体动效** | font-variation-settings 的 wght/wdth 轴随滚动或 hover 插值，字重「呼吸」 | 品牌大标题、编辑部风落地页 | 变量字体体积；只有数值轴可插值 |
| **WebGPU 粒子** | compute shader + 存储缓冲跑百万级粒子，repulsion / curl noise 全在 GPU | 压轴级 hero、生成艺术背景 | WebGPU 覆盖率不足，需 WebGL fallback 双写 |

> 决策口径：这些属于「预算充足 + 明确要压轴效果」的选项；日常界面升级优先 49–56 的 CSS 原生方案。

---

## 附：移动端手感三定律（㉖-㉜ 总纲）

1. **跟手优先**：任何拖拽，松手前 `transition: none` 1:1 跟随；松手后才上曲线
2. **速度 + 位移双判定**：快速轻扫要能触发（速度 > 0.5px/ms），慢拖按就近吸附/过半阈值
3. **原生滚动是白送的**：能 `scroll-snap` 就别 JS 手写 transform pager —— 惯性、边缘回弹、滚动条可见性全免费

高级补充律（㉝-㊵）：

4. **阻尼统一公式**：所有「拉扯 / stretch」手感都用对数阻尼 `c * (1 - 1/(dx/c + 1))`，c ≈ 150 —— 橡皮筋（㉝）、下拉刷新（㊲）、Pinch 边界共用一套物理
5. **让位只动 transform，松手才结算 DOM**：拖拽排序（㊱）、堆叠卡片（㊴）拖动过程中的让位/换层全用 transform 预览，pointerup 一次性真实重排
6. **末速取最后 3~5 个 move**：fling / swipe 判定速度不要用整段平均 —— 起手慢会稀释释放速度，只取 release 前的末速
7. **增强非唯一路径**：pinch / 拖拽排序 / 下拉刷新都必须有按钮或键盘等价物；手势是加速器，不是入口
8. **反馈从触发点长出来**：点赞爱心、庆祝粒子、主题揭示全部从用户按下的坐标发出 —— 因果感是高级反馈和廉价特效的分界线（㊶ ㊹ ㊽ 共用）
9. **成功要克制**：常规成功给 toast 就够；庆祝粒子（㊽）只用在「用户流里唯一的情绪高点」，每屏每流 ≤ 1 处 —— 稀缺才有情绪价值

---

## 附：错误速查（搬不上台面的踩坑）

| 症状 | 原因 | 修法 |
|------|------|------|
| 顶卡"抽拉"出来卡顿 | transition 时间太长（> 1s） | 缩到 0.6~0.8s |
| 卡片变换时边缘锯齿 | 没开 `will-change: transform` | 加在 .card 上 |
| 滚轮触发页面滚动 | wheel 没 `preventDefault` | 监听加 `{ passive: false }` 并 `e.preventDefault()` |
| 触摸时跟手卡顿 | 没设 `touch-action: pan-y` | 加在 .card 上 |
| 末位卡完全看不见 | opacity 设到 0 或 scale 设太小 | 末位保留 opacity 0.18 / scale 0.70 |
| 移动端书签被裁切 | card+bookmark 总宽 > 视口 | 540px / 400px 断点收窄 card / bookmark |
| active 卡被后面的卡盖住 | z-index 设错 | .card.active 用 `z-index: 100`，其他递减 |
| 书签 hover 时整体晃动 | 书签作为 .card 子元素但加了 transform | hover 时只动 `.bookmark` 自身，不要动 .card |
| 拖拽跟手"慢半拍" | 跟手阶段没关 transition | pointerdown 时 `transition: none`，松手再恢复 |
| 快速轻扫没反应 | 判定只看位移不看速度 | 加速度判定：`vx > 0.5 px/ms` 直接提交 |
| 页面拖到一半卡死 | 没监听 `pointercancel` | 加 pointercancel 复位（来电/系统手势会中断） |
| 内部横轨滚完页面跟着滚 | 缺 `overscroll-behavior: contain` | 加在内滚容器上 |
| 竖滑列表误触发滑动返回/滑操作 | 没做方向锁 | 首次 pointermove 判定 h/v，锁定后不再改判 |
| snap 轨道吸不上/位置漂移 | mandatory + 间距不统一 | 统一 gap + `scroll-padding`，或降 proximity |
| iOS 上 vibrate 无效 | Safari 不支持 Vibration API | 静默降级到 scale+色变；原生壳里用 HapticEngine |
| 橡皮筋一下拉到底 | 没用阻尼公式，1:1 跟手 | 换 `c*(1-1/(dx/c+1))` 对数阻尼 |
| Pinch 时图片乱漂 | transform-origin 固定 center | origin 锁双指起点中点 |
| Pinch 中一指抬起就退出 | 只判 pointerup 次数 | `pointers.size < 2` 才结束 pinch |
| 拖拽排序换位时整表闪烁 | 换位用 insertBefore 实时改 DOM | 让位用 transform，松手才 appendChild 结算 |
| fling 甩太远/太近 | 速度用整段平均 | 只取最后 3~5 个 move 的末速 |
| 堆叠卡片「不像一叠」 | 后层不跟进 | 前层进度 ×2 的跟进系数，后层提前就位 |
| 液态按钮前沿是直线 | 只改 width/clip-path | SVG path 贝塞尔前沿，凸起随速度变化 |
| stretch 头图底部露缝 | transform-origin 在 center | origin 改 50% 100%（底边锚定） |
| 双击点赞触发浏览器缩放 | 没 `touch-action: manipulation` | 加上；双击判定用 pointerdown 不是 click |
| 双击爱心从中心冒出 | burst 坐标写死 | 用 pointerdown 的 clientX/Y 定位 |
| 横滑通知后列表留洞 | 卡飞走后没做高度坍缩 | transitionend 里 height→0 再 remove |
| 3D 轮没有圆柱面 | perspective 加在 item 上 | perspective 必须在滚动容器上 |
| 圆形揭示出现两次过渡 | VT 默认交叉淡入没清 | `::view-transition-*` animation: none |
| 按住说话来电中断仍发送 | pointercancel 当成功处理 | pointercancel 一律静默取消 |
| 骨架屏内容到达时跳动 | 骨架尺寸是猜的 | 骨架与真实内容逐像素同尺寸 |
| 角标动画第二次不播 | class 已在没法重启 | `void el.offsetWidth` retrigger |
| confetti 页面点不了 | canvas 忘了 pointer-events:none | 加上，且粒子全灭后清 rAF |
| confetti 掉帧 | 每帧每粒子 ctx.font + emoji fillText | emoji 预渲染离屏位图，每帧只 drawImage |
| confetti 高分屏更卡 | canvas 没钳制 DPR | `Math.min(dpr, 2)`，DPR3 全屏画布像素 ×9 |