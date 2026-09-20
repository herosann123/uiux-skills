# 快速索引 · UI 交互模式

> 17 种交互模式的一行速查。
> 主文档：`SKILL.md` · 完整代码：`PATTERNS.md`

---

## 基础 10 种（demo: `10-ui-interactions.html`）

| ID | 名称 | 一句话 | 适用场景 |
|----|------|--------|---------|
| ① | 视差滚动 Parallax | 多层不同速度的 translateY | Hero 背景、作品集 |
| ② | 滚动触发 Reveal | 元素进入视口 → 0→1 动画 | 段落淡入、卡片入场 |
| ③ | 卡片堆叠 Swipe | 拖拽 + 阈值飞出/弹性回弹 | Tinder 式匹配卡 |
| ④ | 磁吸按钮 Magnetic | 指针偏移 → 按钮 translate | 主 CTA（桌面端） |
| ⑤ | 文字揭示 Text Reveal | 字符串拆词 → stagger 入场 | 标题、卖点、模态框 |
| ⑥ | SVG 路径绘制 | stroke-dashoffset 100% → 0% | Logo、Loading、签名 |
| ⑦ | 弹性回弹 Spring | F = -kx - cv 物理动画 | 下拉刷新、Tab 指示器 |
| ⑧ | 数字翻牌 Counter | 0~9 垂直堆叠 + translateY | 数据统计、评分、倒计时 |
| ⑨ | 3D 倾斜 Tilt | perspective + rotateX/Y 跟光标 | 特性卡片、电商卡 |
| ⑩ | 玻璃拟态 Glass | backdrop-filter: blur + 半透明 | Modal、Toggle、状态卡 |

## 进阶 7 种（demo: `advanced-ui-v2.html`）

| ID | 名称 | 一句话 | 适用场景 |
|----|------|--------|---------|
| ⑪ | 极光渐变 Aurora | 多色大圆 + mix-blend-mode: screen | Hero 高级背景 |
| ⑫ | 粒子网络 Particles | Canvas + 鼠标排斥 + 临近连线 | AI 产品落地页 |
| ⑬ | 水波纹涟漪 Ripple | click 位置 → radial scale + fade | 按钮点击反馈 |
| ⑭ | 粘性滚动堆叠 Sticky | position:sticky 容器 + scroll 动画 | 流程叙事、Onboarding |
| ⑮ | FLIP 共享元素 | First-Last-Invert-Play 列表→详情 | 作品集卡片→案例 |
| ⑯ | View Transitions | `document.startViewTransition()` | 主题切换、SPA 路由 |
| ⑰ | AI 流式打字机 | 字符 setState + 可变 delay + 光标 | AI Chat、生成预览 |

## 前沿 7 种（demo: `advanced-ui-v3.html`）

| ID | 名称 | 一句话 | 适用场景 |
|----|------|--------|---------|
| ⑱ | 自定义光标 Custom Cursor | 双层 + 阻尼惯性 + hover 放大 | Linear / Vercel / Stripe |
| ⑲ | 水平滚动叙事 H-Scroll | 垂直滚动 → 水平 track 平移 | Apple 产品页、品牌叙事 |
| ⑳ | 噪点纹理 Noise | Canvas 随机像素 + steps() 抖动 | 暗色背景、质感提升 |
| ㉑ | RGB 分离 Glitch | 3 层文字 + screen + 关键帧偏移 | 故障感标题、Tech 风 |
| ㉒ | 滚动驱动数字 Scroll# | ScrollTrigger.progress → lerp 数字 | 数据叙事、产品参数 |
| ㉓ | 命令栏 Cmd+K | 快捷键唤起 + 模糊搜索 + 键盘导航 | 工具型产品、Power User |
| ㉔ | Blob Morph 液态形变 | SVG `<path>` `d` 属性插值 | Stripe 风、空状态、Loading |
| ㉕ | Folding Drawer 千层酥 | 同位多卡 + 阶梯偏移 + 顶卡抽拉 | 色卡、设计系统、配色面板 |

---

## 模块 → 模式 速查

| 模块 | 推荐 |
|------|------|
| Hero 主视觉 | ① 视差 + ⑤ 揭示 |
| Hero 高级背景 | ⑪ 极光 + ⑫ 粒子 |
| Hero 顶级背景 | ⑲ 水平叙事 + ⑳ 噪点 |
| 主 CTA 按钮 | ④ 磁吸（桌面）/ ⑦ 弹性 |
| 数字徽章 | ⑧ 数字翻牌 / ㉒ 滚动驱动 |
| Loading 占位 | ⑥ SVG / ⑦ 弹性 / ㉔ Blob |
| 卡片 → 详情 | ⑮ FLIP / ⑯ View Transitions |
| 色卡 / 配色面板 | ㉕ Folding Drawer |
| 有限集合顺序浏览（≤8 项） | ㉕ Folding Drawer |
| 主题切换 / SPA | ⑯ View Transitions |
| AI 对话气泡 | ⑰ 流式打字机 |
| 高端落地页品牌感 | ⑱ 自定义光标 + ⑳ 噪点 + ㉑ Glitch |
| 垂直滚动叙事 | ⑭ 粘性滚动堆叠 / ⑲ 水平叙事 |
| Power User 工具 | ⑫ 粒子 + ⑬ 涟漪 + ㉓ Cmd+K |
| 数据展示 | ㉒ 滚动驱动数字 + ⑧ 翻牌 |

---

## 工具栈推荐

- **Web**: GSAP + ScrollTrigger + Draggable（CDN）
- **React**: Framer Motion + react-spring
- **Vue**: @vueuse/motion + GSAP
- **小程序**: WXS + wx.createAnimation
- **iOS**: UIKit Dynamics + Core Animation

---

## 性能铁律

- 只动画 `transform` + `opacity`（GPU 加速）
- 移动端同时运行动效 ≤ 2 个
- blur ≤ 24px（中端机 ≤ 12px）
- 必须监听 `prefers-reduced-motion` 提供静态 fallback

---

## 移动端 / 平板适配铁律（触屏必读）

- **拖拽类交互（③⑦⑮）必须给拖拽面加 `touch-action: none`**，否则触摸拖动被浏览器当作页面滚动，pointer 事件被 pointercancel 掐断
- 所有拖拽都要监听 **`pointercancel`** 做复位（来电 / 系统手势会随时中断拖拽）
- 纯鼠标交互（④磁吸 ⑨倾斜 ⑱光标）用 `matchMedia('(hover: hover) and (pointer: fine)')` 门控；**不要用屏幕宽度判断** —— iPad Pro 横屏 >1366px 仍是触摸设备
- 触摸替代方案：磁吸 → 按下吸附 + 松手 elastic 回弹；3D 倾斜 → 按住拖动倾斜（pointermove 同时覆盖鼠标与手指）；粒子排斥 → 手指 pointermove 跟随
- 统一用 **Pointer Events** 而非 mouse/touch 双轨；按钮加 `touch-action: manipulation` 消除 300ms 双击缩放延迟
- ⑲ 水平叙事在触摸设备改为**原生 `overflow-x` + `scroll-snap` 滑动**，pin+scrub 仅桌面
- ㉓ Cmd+K 列表项必须**可点选**（移动端没有 Enter）；输入框 `font-size ≥ 16px` 防 iOS 聚焦缩放
- 持续动画（⑫粒子 ⑳噪点）用 **IntersectionObserver 离屏自动暂停**（省电流畅）
- **大面积 `blur` 层上不要叠 `mix-blend-mode`**（移动端掉帧大户）；移动端去 blend 用普通透明度叠加，深底上视觉几乎一致
- **FLIP / 共享元素过渡只用 `transform` 反演**（translate+scale），禁止过渡 `left/top/width/height`——后者每帧重排
- 滚动触发类渐入默认做成**可重入**：滚出视口下方即重置，回看重新播放（一次性动画让页面"看过就死"）
- 内部滚动容器：`-webkit-overflow-scrolling: touch` + `overscroll-behavior: contain`（防滚动穿透）
- 视口高度用 `100dvh`（fallback `100vh`）；旋转 / resize 后**重算缓存的 getBoundingClientRect**
- 定宽卡片改 `width: min(320px, 100%)`；按钮组 `flex-wrap: wrap`；短横屏用 `@media (max-height: 700px)` 压缩 demo 高度

---

## 输出 / 评审 / 减法（v2.1 新增，详见 SKILL.md §11）

- 模式推荐一律**四层输出**：必须满足 / 推荐方案 / 可选实验 / 禁止或避免
- 「再加动效」先过**减法决策表**：信息价值 · 重复说明 · 任意强调 · 过量效果 · 原生替代（先删后加）
- 评审走**盲评包**：只给截图 + 中性元数据；无截图 = 不做视觉评价、不编分数
- 验收用**四态结论**：通过 / 未通过 / 未验证 / 不适用；实现者自测 ≠ 证据
- 对抗性测试场景：`evals/scenarios.md`

---

## 文件清单

```
ui-interaction-skill/
├── SKILL.md                          # 主入口（agent 读这个）
├── PATTERNS.md                       # 24 种模式完整代码模板
├── INDEX.md                          # 本文件（速查）
├── evals/scenarios.md                # 对抗性测试场景（v2.1）
├── ui-visual-skill/                  # 姊妹 skill：字体排版与配色质感
│   ├── SKILL.md                      #   气质→配方决策表 + 纪律 + 反模式
│   ├── PATTERNS.md                   #   4 色板 + 5 字体配对完整代码
│   ├── INDEX.md                      #   速查
│   └── demo.html                     #   Before/After 切换演示
├── party-invitation.html             # 综合应用示例（动效 + 质感配方 A）
├── 10-ui-interactions.html           # 基础 10 种 demo
├── advanced-ui-v2.html               # 进阶 7 种 demo
└── advanced-ui-v3.html               # 前沿 7 种 demo
```