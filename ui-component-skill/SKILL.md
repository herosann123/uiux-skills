---
name: ui-component-skill
description: 组件状态完备性与页面状态流：先列状态矩阵再写样式（默认/悬停/焦点/激活/禁用/加载/错误），表单校验时机与错误呈现，页面级 loading/empty/error/success 四态设计。用于做组件、表单、反馈机制；不含视觉定调（走 ui-visual-skill）与动效模式（走 ui-interaction-skill）。
type: design-pattern
version: 1.0.0
user-invocable: true
argument-hint: "[组件名|状态流] [--states]"
license: MIT
trigger:
  - 做按钮/输入框/卡片/弹窗/Toast 等组件
  - 做表单与校验反馈
  - 页面需要 loading/空状态/错误态
  - 组件"只设计了正常样子"
when_not_to_use:
  - 纯布局结构（走 ui-layout-skill）
  - 动效模式选择（走 ui-interaction-skill）
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# UI 组件与状态 Skill · 状态完备性

> 核心方法：**先列状态矩阵,再写第一行样式**。一个组件的质感不在于正常态多好看,
> 而在于每个状态都有设计、状态之间有过渡、错误态会说话。

---

## 1. 状态矩阵（做任何交互组件的第一步)

| 状态 | 何时存在 | 必须表达 |
|------|---------|---------|
| 默认 | 静止可交互 | 可点击的视觉暗示 |
| 悬停 hover | 指针悬停（**仅 hover:hover 设备**） | 可点击确认 |
| 焦点 focus-visible | 键盘 Tab 到达 | 清晰焦点环（不许 `outline: none` 裸奔） |
| 激活 active | 按下瞬间 | 即时按压反馈（scale/加深） |
| 禁用 disabled | 不可用 | 降透明度 + `cursor: not-allowed` + 为什么禁用（title/文案） |
| 加载 loading | 异步进行中 | 转圈/进度 + 禁止重复提交 |
| 错误 error | 校验失败/请求失败 | 颜色 + 文字说明 + 图标（三重提示,不只变红） |
| 成功 success | 完成 | 确认反馈 + 下一步 |

**写组件前先回答：这 8 个状态里哪些存在？漏掉的状态就是未来的 bug。**

## 2. 表单 UX 铁律

1. **校验时机**：单字段 `blur` 之后即时校验；提交时全量校验；**输入中不清错误**（除非改回合法）
2. **错误呈现**：字段边框变色 + 字段下方具体文字（"邮箱格式不对,应含 @" 而非 "输入错误"）+ `aria-describedby` 关联
3. **提交防重**：请求进行中按钮进入 loading 态并禁用
4. **输入字号 ≥16px**：防 iOS 聚焦自动放大
5. **永不惩罚**：不清空用户已输入内容；错误信息对事不对人

## 3. 页面状态流（每个异步视图都要有四态）

| 状态 | 设计要求 | 反模式 |
|------|---------|--------|
| loading | 骨架屏（结构预览）> spinner；布局占位防跳动 | 白屏转圈超过 1s |
| empty 空状态 | 解释为什么空 + **给下一步动作**（按钮/引导） | 一行"暂无数据"就完了 |
| error 错误 | 说明发生了什么 + **重试按钮** + 程序化兜底 | 白屏或纯红字 |
| success 成功 | 明确反馈 + 下一步入口 | 静默成功,用户不知道成没成 |

**顺序**：设计视图时按 `loading → 内容 OR empty OR error` 画全状态图,再动手。

## 4. 反模式

- ❌ 只设计正常态的组件（hover 都没有更别说 error）
- ❌ `outline: none` 不给替代焦点样式 → 键盘用户迷路
- ❌ 提交按钮不做防重 → 重复下单/重复发送
- ❌ 错误只把边框变红,不给文字 → 色盲用户与健忘用户双双出局
- ❌ 空状态不给出口 → 用户困死在页面上
- ❌ 禁用按钮不解释原因
- ❌ 弹层不能 Esc/遮罩关闭

## 5. 输出契约

沿用 `../ui-interaction-skill/SKILL.md` §11.1 四层分级：
必须满足 = 状态矩阵完备 + 焦点可见 + 错误三重提示 + 提交防重；
推荐 = PATTERNS.md 对应配方按当前 tokens 落地；可选 = 微动效（走 interaction 模式）；禁止 = §4。

## 6. 相关资源

- 状态矩阵与四态的完整代码：`./PATTERNS.md`
- 按压/涟漪/弹性等反馈动效：`../ui-interaction-skill`（④⑦⑬）
- 组件配色/字体落地：`../ui-visual-skill`
- 组件验收：`../ui-verify-skill`（交互走查 + 无障碍自检）
