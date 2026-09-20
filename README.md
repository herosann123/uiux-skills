# UIUX 技能组 · 一个总控 + 五个专业 UI 技能

让 agent 独立完成有质感的 UI 工作全链路:**骨架 → 质感 → 动效 → 组件状态 → 实测验收**。
总控按任务缺口路由,小修小补直接做,不强制跑全流程。

| 技能 | 负责层 | 亮点 |
|------|--------|------|
| **uiux-director** | 总控路由 | 缺口路由表 · 三纪律(按缺口/小任务直做/轻量交接) |
| **ui-layout-skill** | 骨架 | 刻意低约束:5 条底线 + 决策问题 + 可替换起点配方 |
| **ui-visual-skill** | 质感 | 4 套气质色板 + 5 套字体配对 + 廉价感反模式 |
| **ui-interaction-skill** | 动效 | 24 种模式 + 输出/评审/验收协议 + 移动端铁律 |
| **ui-component-skill** | 组件 | 8 态状态矩阵 + 表单 UX 铁律 + 页面四态流 |
| **ui-verify-skill** | 验收 | 视口矩阵实测 + 可复制检测脚本 + 已知陷阱 + 四态结论 |

---

## 安装

### ZCode / Claude Code
把**六个技能文件夹**放进项目 `.claude/skills/`(或用户级技能目录):

```text
你的项目/.claude/skills/
├── uiux-director/            (SKILL.md)
├── ui-layout-skill/          (SKILL.md + PATTERNS.md)
├── ui-visual-skill/          (SKILL.md + PATTERNS.md + INDEX.md + demo.html)
├── ui-interaction-skill/     (SKILL.md + PATTERNS.md + INDEX.md + evals/ + demos/ + examples/)
├── ui-component-skill/       (SKILL.md + PATTERNS.md)
└── ui-verify-skill/          (SKILL.md + PATTERNS.md)
```

> 拷**文件夹本身**,不要只拷 SKILL.md,也不要多套一层壳。装完重载技能列表。

### Codex CLI / IDE
同样结构放 `.agents/skills/` 下。

### 不装 agent,只想看效果
- `ui-visual-skill/demo.html` — 同页「廉价感 ↔ 高级感」一键切换
- `ui-interaction-skill/demos/*.html` — 24 种交互模式可玩演示
- `ui-interaction-skill/examples/` — 综合成品(邀请函 / 家庭参观页,单文件可直接发朋友)

---

## 使用示例

```text
$uiux-director 从零做一个咖啡店官网            → 总控排环节,逐技能执行
$uiux-director 这个页面不够高级                → 路由到 ui-visual-skill
$uiux-director 帮我验收这个页面                → 路由到 ui-verify-skill,四态报告
/ui-interaction-skill hero                     → 直接查动效映射表
我的表单组件老出问题                            → 触发 ui-component-skill 状态矩阵
```

也可以完全不提技能——总控的触发词覆盖「做页面 / 不够高级 / 加动效 / 帮我看看 / 手机上不对劲」。

---

## 设计原则

1. **按缺口路由**:只补缺的环节;改个文案直接做,不启动全套
2. **低约束哲学**:layout 刻意只守 5 条底线,配方全部标注"起点不是答案";视觉反模板味,交互守性能预算
3. **证据先行**:verify 的每项结论都要可复现证据,四态结论,不放行未验证项
4. **实战沉淀**:移动端铁律、隐藏标签页陷阱、动态元素渐入坑、em/百分比数字对齐——全部来自真实踩坑

## 版本

uiux-director 1.0.0 · ui-layout-skill 1.0.0 · ui-visual-skill 1.0.0 ·
ui-interaction-skill 2.1.0 · ui-component-skill 1.0.0 · ui-verify-skill 1.0.0 · MIT
