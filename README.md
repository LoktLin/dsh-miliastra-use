# dsh-miliastra-use

> 原神·千星奇域（Miliastra Wonderland）**`dsh-miliastra` 插件的使用经验与工具链实操** —— 一个 DSH 技能（skill）。

**它管什么**：什么时候调哪个工具 · 怎么定位关卡 / 活文件 / 日志 · `deploy` 三步 · 备份与还原的安全约定 · 截图取证 · 试玩探针 · 素材与生成器 · 模拟器预测试 · **真实踩坑全集**。

## 它不管什么（各归其位）

| 你要做的事 | 去哪个技能 |
|---|---|
| 写玩法 Lua 本体（铁律 / API 边界 / 代码风格 / 门禁） | `miliastra-code` |
| 界面视觉与版面（栅格 / 字号 / 配色 / 层序 / 可点四态） | `miliastra-ui` |
| 需求边界确认（奇葩需求 / bug 夹带需求 / 边界清单） | `miliastra-bb` |
| 改插件代码本身（Host / Client / 挂载 / 契约兼容） | `dsh-plugin-win10` |

## 安装

技能就是一棵文件树的复制 —— DSH 实时 watch 技能目录，放进去即生效：

```powershell
git clone <本仓库> "$env:USERPROFILE\.dsh\skills\dsh-miliastra-use"
# 或者：把本目录整个拷进 ~/.dsh/skills/dsh-miliastra-use
```

装好后 DSH 会在相关话题自动加载它（触发词来自 `SKILL.md` 的 `description`）。

## 内容

```
SKILL.md                       ← 入口
  0 一句话心智模型              1 开工前定位（每次会话/换图都要重做）
  2 工具地图（什么时候用哪个）   3 部署与备份安全（最容易误解，务必读）
  4 取证与判据（只报数字，不下判决）
  5 ★★ 真实踩坑全集            5.1 字号与几何 · 5.2 层级与交互 · 5.3 代码语义
                                5.4 工具与部署 · 5.5 外部库/框架（不要搬）
  6 工程纪律（插件工具链版）     7 环境降级
references/plugin-toolbox.md   ← 工具详解：每个 op 的回执看什么、坑在哪
```

## 三条底色纪律

1. **工具只报数字，不下判决** —— "能不能过"是作者的判断。
2. **证据分两轴**：`evidence_source`（`documented` / `provided_unverified` / `observed` / `inferred` / `unknown`）× `device_status`（`passed` / `failed` / `pending` / `not_required`）；**没跑真机就写 `pending`**。
3. **门禁红了不许回退** —— 只许「修到过」或「如实标 `pending` 交付」；不许回退旧实现 / 删功能 / 拿空壳顶。

## 与插件的关系

本技能讲 [`dsh-miliastra`](https://github.com/LoktLin/dsh-miliastra)（GPL-3.0-only）**怎么用**：插件本体、安装、面板与工具 schema 都在插件仓库。
工具清单**以插件当前版本为准** —— 这里给的是**用法与经验**，不复述 schema。

## License

[GPL-3.0-only](LICENSE)（与 `dsh-miliastra` 插件一致）。
