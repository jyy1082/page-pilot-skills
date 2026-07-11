# page-pilot-skills

**中文** · [English](./README.md)

**版本 0.1.0** · 完整版本历史见 [CHANGELOG.md](./CHANGELOG.md)

把 [page-pilot-recorder](https://github.com/jyy1082/page-pilot-recorder) 录制出来的步骤数组，转换成一个可复用、带参数的"技能"：一句简短描述、一份命名好的参数列表，以及把具体值替换成 `{{参数名}}` 占位符之后的原始步骤——这样同一段录制内容，以后换个值就能重新跑一遍，而不是被锁死在录制时打的那个具体值上。

这是一个更大计划里的第一步：先做录制和打标签（这个仓库负责的部分）；AI 驱动的检索、自动填参数放在这一层之上，以后再做。**这里面完全不涉及任何 AI**——就是一个人在停止录制之后马上会用到的确认界面，加一个小巧的本地存储接口。哪些值该变成参数、叫什么名字、要不要保存，全部还是人自己决定和确认。

## Demo

用 [page-pilot-recorder](https://github.com/jyy1082/page-pilot-recorder) 录制点什么，不直接运行或者复制步骤，而是弹出归档面板：

```js
import { PagePilotRecorder } from 'page-pilot-recorder'
import { showArchivePanel } from 'page-pilot-skills'

const recorder = new PagePilotRecorder()
recorder.start()
// ...正常操作页面...
const steps = recorder.stop()

const skill = await showArchivePanel(steps)
// 会弹出一个面板：任务描述、检测到的候选参数（带建议名字，勾选/不勾选）、
// 步骤列表（每一条都能删掉）、以及一个"高风险"复选框。
// 选"保存为技能"会自动保存（这个函数自己就会存，不需要额外调用）并返回保存好的记录；
// 选"仅本次使用"会返回 null，什么都不保存。
```

## 什么会被识别成候选参数

每个 `type` 步骤的 `text`、每个 `select` 步骤的 `value`、每个 `check` 步骤的 `checked` 状态——每一个都会尝试按下面这个顺序给出一个人类能看懂的建议名字：指向这个字段的 `<label for="...">`、包裹这个字段的 `<label>`、`aria-label`、`placeholder`，最后是 `name` 属性。如果这些都找不到，会给一个"参数N"这种占位名字，用户必须自己改个名字才能保存成参数。

- `select` 的值默认**勾选**（换个下拉选项重新跑一遍，本来就是这类操作最常见的用途）
- `check`（复选框/单选框）默认**不勾选**——通常是流程里固定不变的一部分，不太值得重新做成参数
- 超过 200 字符的值默认**不勾选**——更可能是自由文本（备注、说明），这类内容更适合每次固定不变，而不是每次都要换的值

密码框完全不会出现在这里——`page-pilot-recorder` 本身就拒绝录制密码框的内容，压根没有东西可以被检测到。

## 存下来的是什么，故意不存的又是什么

保存下来的技能记录大概长这样：

```json
{
  "id": "skill_1720000000000_ab12cd",
  "domain": "admin.example.com",
  "description": "新增员工",
  "steps": [
    { "type": "click", "target": "#add-btn" },
    { "type": "type", "target": "#lastName", "text": "{{姓氏}}" },
    { "type": "select", "target": "#department", "value": "{{部门}}" }
  ],
  "parameters": [{ "name": "姓氏" }, { "name": "部门" }],
  "fragile": false,
  "highRisk": false,
  "createdAt": "2026-...",
  "updatedAt": "2026-..."
}
```

- **示例值永远不会被存下来**——只存参数的**名字**。哪怕传给它的草稿对象里顺带带了一个示例值，`saveSkill()` 在真正写入存储之前，也会把名字以外的所有东西都剥掉。这是故意的：光是因为录制的时候用过一次，姓名、id 这些真实值就不应该一直躺在 `localStorage` 里。
- **技能是按域名区分的**（默认用 `location.hostname`）——在一个网站上存的技能，切到另一个网站的时候不会出现。
- **`fragile`**：如果有步骤的选择器退到了结构路径兜底方案（参考 `page-pilot-recorder` 的选择器生成策略），就会标记这个——提醒你用之前review一下，但不会因此不让你保存。
- **`highRisk`**：如果某个点击的目标文字或者选择器命中了常见的危险操作关键词（删除、提交、支付、转账等——中英文都覆盖），面板里这个复选框会默认预先勾选上——用户随时可以手动改成勾选或者不勾选。这只是一个启发式的提醒，不是绝对保证；这个仓库本身不会基于这个标记强制执行什么（这是以后"谁来跑这个技能"那一层该管的事，比如跑高风险技能之前强制要求确认）。

## API

| 方法 | 说明 |
|---|---|
| `detectParameters(steps)` | 扫描步骤数组，返回带建议名字的候选参数 |
| `hasFragileSteps(steps)` | 是否有步骤的选择器是结构路径兜底方案 |
| `isHighRisk(steps)` | 是否有点击的目标/选项看起来像危险操作 |
| `buildSkillDraft(description, steps, acceptedParams)` | 构建一个替换好占位符的技能草稿对象 |
| `saveSkill(draft, domain?)` | 保存一份草稿（或者按 id 更新已有的）；只保留参数名字，其余全部剥掉 |
| `listSkills(domain?)` | 列出某个域名下保存的技能，按最近更新时间排序 |
| `getSkill(id, domain?)` | 按 id 获取单个技能 |
| `deleteSkill(id, domain?)` | 按 id 删除技能 |
| `showArchivePanel(steps, options?)` | 完整的确认界面；自己负责保存，返回保存好的记录，或者"仅本次使用"时返回 `null` |

只要 `domain` 是可选参数的地方，默认值都是 `location.hostname`。

## 这个项目目前故意不做的事

- **没有 AI。** 这里面任何地方都不会调用模型。检测到的参数名字建议，是基于 DOM 检查得出的，不是语言理解的结果。
- **没有检索/匹配。** 没有"根据新指令找到对应技能"这种能力——这是下一层要做的，建立在这个仓库存下来的数据基础上。
- **运行时不会自动填参数。** 保存好的技能里，步骤依然带着 `{{参数名}}` 占位符；把真实值换回去、再执行，同样是下一层的工作。
- **不支持跨设备同步。** 存储用的是普通的 `localStorage`，只在保存时所在的那个浏览器里有效。

## 测试

```bash
npm install
npm test
```

跑的是真实浏览器测试（Playwright + Chromium，通过 `@sparticuz/chromium` 拿到——具体原因见 [page-pilot-recorder 的 README](https://github.com/jyy1082/page-pilot-recorder#testing)），覆盖参数检测的命名优先级、存储的完整读写流程、按域名隔离、以及完整的归档面板 UI 流程（改参数名字、删除一条噪音步骤、高风险复选框，以及"保存"和"仅本次使用"两条路径）。

## 协议

MIT
