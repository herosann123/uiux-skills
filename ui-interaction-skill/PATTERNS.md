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