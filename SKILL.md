---
name: dsh-miliastra-use
description: 原神·千星奇域（Miliastra）dsh-miliastra 插件的**使用经验与工具链实操**——什么时候调哪个工具、怎么定位关卡/活文件/日志、deploy 三步、备份还原安全约定、截图取证、试玩探针、素材与生成器、模拟器预测试、**真实踩坑全集**（字号几何/层级压盖/漏local/OnStart/全局赋值/图片SetImage/EnableUpdate/素材rotationZ/清理热区/颜色构造…）。**只要涉及千星奇域插件、miliastra 工具、levelScript 部署、活文件备份、.gil 读取、截图取证、试玩/日志排障、素材/像素画/粒子生成、模拟器验证、控件/层级/字号/踩坑，就用这个技能**——即使对方只说"帮我部署一下""截图看看""日志里没东西""控件建不出来""字不见了""按钮点不到"。边界：写玩法 Lua 本体/界面视觉走 miliastra-code / miliastra-ui；需求拆解走 miliastra-bb；纯 DSH 插件开发（改插件代码本身）走 dsh-plugin-win10。
degradation: 工具不可用时退回工作区手工命令（docs/工作区常用操作.md）；无截图/模拟器只做可计算部分（坐标表/回执字段核对），画面判断交给人截图；无真机试玩时如实标 pending 并给验证建议。
---

# dsh-miliastra 插件使用经验

> 这份技能回答一个问题：**「我现在该调哪个工具、怎么调、调完怎么确认」** + **「这个坑我踩过没有」**。
> 不讲玩法怎么写（那是 `miliastra-code`）、不讲界面怎么好看（那是 `miliastra-ui`）。
> 定位：**工具链实操手册** + **踩过的坑全集** + **判据口径**。
>
> 本手册对应 **dsh-miliastra 0.7.2**（Host 改动必须重启 `dsh web`；`miliastra_echo` 回的 `version` = **运行中 Host** 的版本）。

## 0. 一句话心智模型

```
活文件(.lua) ──deploy──▶ 沙箱 external_lua_file\ ──编辑器存盘──▶ .gil 内嵌快照 ──试玩──▶ 游戏跑的那份
     ▲                          ▲                                    ▲
  工作区/插件写的           人手点存盘（AI 不能）                人点试玩（AI 不能）
```

**deploy ≠ 进游戏**：游戏跑的是**地图存盘时嵌进 `.gil` 的快照**，不是活文件本身。
⇒ 固定三步：**`deploy` → 编辑器存盘 → 试玩**；试玩前用 `miliastra_map op=script` 看 `match:true` + `isCurrent:true`。

## 1. 开工前定位（每次会话/换图都要重做）

```text
miliastra_health {brief:true}     ← 当前关卡 ID / 活文件目录 / 日志目录 / 编辑器在不在跑（<1KB）
miliastra_code {op:"inspect"}     ← 活文件 SHA + 部署指纹（有没有被写回旧版）
miliastra_map {op:"script"}       ← 挂载名 + embedded vs live（match 才代表游戏跑的是这版）
```

- **路径随账号与换图变化，禁止写死**：`<账号ID>` / `<关卡ID>` 都会变；换图 = 换活文件目录。
- **`editorHint` 只是间接证据**：插件**没有**「编辑器正在编辑哪张图」的直接通道（窗口标题不含关卡名 / `.gil` 不锁 / 无自动保存心跳）。
  ⇒ **改码/部署前问作者一句「现在在哪张图」**（一次只问一件事）。
- **多脚本工程**：`deploy` **必须显式 `file=` + `level=`**（默认目标是"当前关卡"，换图/导入模板会改掉它 ⇒ 曾把 A 图脚本写进 B 图）；看回执 `dest` 再往下走。

## 2. 工具地图（什么时候用哪个）

| 你想做什么 | 调什么 | 关键回执/判据 |
|---|---|---|
| **部署 lua 进沙箱** | `miliastra_code {op:"deploy", source, file, level}` | `dest` 含关卡号 ✓；自动备份 + SHA 校验；失败 `rolledBack:true` = **安全网，不是事故** |
| **还原某份备份** | `miliastra_code {op:"restore", backup?, file?}` | 备份名能推断目标就以它为准；与 `file` 不符 ⇒ **拒绝写盘**（`RESTORE_TARGET_MISMATCH`） |
| **看地图里有什么** | `miliastra_map`（`summary` / `regions` / `clientui` / `anatomy` / `nodes`） | `regions` = 顶层区地图；`clientui` = 控件谱系；`anatomy` = 节点图能力画像 + 挂载主 + 信号清单 |
| **判画面对不对** | `miliastra_shot {op:"capture" / "burst"}` | 日志只能回答"代码跑了没"，画面要看截图；短局用 `burst awaitPlaytest:true` |
| **读运行时日志** | `miliastra_log {op:"errors" / "runs" / "metrics"}` | **怀疑报错先跑 `op=errors`**（按形态捞，不看标签——真机报错行可能没有 `[...]` 前缀） |
| **问游戏一句** | `miliastra_probe`（deploy 模板 → 人试玩 → collect → **记得 restore**） | 部署会临时覆盖活文件；**必须提醒人去点试玩** |
| **存/查素材** | `miliastra_asset {op:"add"/"list"/"catalog"/"sound-search"}` | 按内容寻址，同图只存一份；**绝不自动删**（删要 `confirm:true`） |
| **生成像素画/渐变字/粒子** | `miliastra_gen {op:"pixel-art"/"text-gradient"/"vfx-lua"/"struct-json"}` | 交接值（容器/模板索引）**从 `.gil` 自动读或问人，不许编** |
| **游戏外先跑一遍** | `miliastra_sim {op:"bind"/"verify"/"frames"/"cases"}` | deploy 前的预测试；**不等于真机通过**（官方素材/联机不覆盖） |
| **节点图为什么不动** | `miliastra_kb {op:"qa" / "node"}` | 离线蒸馏 QA + 558 节点词典；`op=list/doc/search` 是在线第三方（会发 query） |
| **查 UI 契约** | `miliastra_code {op:"lint-ui"}` / `node tools/check-ui-contract.mjs` | 字号档位 / `h ≥ 字号×1.4` / 8 的倍数 / 画哪=点哪 |
| **一次到位部署** | `miliastra_code {op:"deploy", level, file, source, gates:true, sync:["<镜像绝对路径>"]}` | `gates` = **工作区那两道门**（没过**不写盘**；找不到工具就报 `GATES_NO_TOOLS`）；`sync` = 二进制同步镜像 + SHA 复验；回执 `shas` = **四方对照**（源码/活文件/镜像[]/地图里嵌的）；`checklist[]` = 存盘→重开一局→看 `match:true`（**引用，别复述**） |
| **生成物落盘（不进上下文）** | `miliastra_gen {op:"vfx-lua", …, saveTo:"<绝对路径>"}` | 给了 `saveTo` ⇒ 只回摘要（实测 `vfx-lua` **64 314 B → 4 351 B**）；**不给时行为一字节不变** |
| **回执要精简骨架** | `… {receipt:"min"}`（`code` / `sim` / `log`） | `summaryOnly` = 去体积、留字段；`receipt:"min"` = **换骨架、留结论**（实测 `log errors` 2151 → 768 B）；非名单 op 原样返回 |
| **探针改写（不动真源）** | `miliastra_sim {op:"bind", source, boot:{cur:3, mode:"build"}}` | 改的是**内存副本**（真源一个字节不动）；`probe.patched` 逐条列改了什么。⚠️ `complete:true` 会**如实回 unsupported**（不猜玩法数据） |
| **量一张图的配色** | `miliastra_asset {op:"measure", source:"<图片绝对路径>", cols:64}` | 主色**众数** + 连通块（灰/饱和）；**坐标单位是采样格**，不是原图像素 |
| **错误形态解释** | `miliastra_log {op:"errors", explain:true}` | 默认**不带**那 8 条固定解释（只给 `formsCount`）⇒ 要才传 |
| **判据出处** | `miliastra_health {brief:true}` | `memoryDoc` 指向 `案子/<地图>/AGENTS.md` 的「记忆」段（**引用，别复述**） |
| **点名一棵控件子树** | `miliastra_map {op:"clientui", root:<控件id>, summaryOnly:true}` | **配 `summaryOnly` 一起用**：slim **4 215 B** vs 全量 **195 777 B**（差 46 倍） |

## 3. 部署与备份安全（这条最容易误解，务必读）

**核心：`rolledBack` / `safetyBackup` 是"安全网"，不是"出事故了"。**

- **写盘四步**：备份（失败即中止）→ 原子写 → SHA-256 校验 → 无 BOM 检查。
- **校验比不上 ⇒ 自动还原到覆盖前那一版** —— 那笔写入没生效，**版本一点没丢**。回执 `rolledBack:true` 说的是这件事。
- **备份体系** = 活文件唯一安全网：两份备份（固定名 `<原名>.bak` + 时间戳历史）+ `op=restore` 不传 `backup` 就用 `.bak`（不用挑版本）。
- **只读不删**：素材/截图/备份**绝不自动删**；删要 `confirm:true`，连字节删再加 `deleteFile:true`。
- **BOM 是硬要求**：活文件与备份都是 UTF-8 无 BOM（原神打印 `Read text file with BOM header may cause Lua error`）；写 `.lua` 用 `edit`/`write`/node，**不用 PowerShell `Set-Content`**（会加 BOM + 管道压行）。

## 4. 取证与判据（只报数字，不下判决）

- **证据双轴**：`evidence_source`（documented / provided_unverified / observed / inferred / unknown）+ `device_status`（passed / failed / pending / not_required）。
- **工具只报数字**：回执给 `count` / `bytes` / `sha` / `pattern matched`，**不给"通过/不通过"**；判断留给创作者（给 2~3 个选项 + 各自代价）。
- **`.gia` 不是实时的**：局在跑时磁盘上没有这个文件（一局结束后才落盘）⇒ 判"在不在试玩"用 `miliastra_playtest op=status`（`output_log.txt` 信号，延迟 0.07~0.18 秒）。
- **"日志里没有" ≠ "没跑"**：`.gia` 落盘会随机失败 ⇒ 判画面用 `miliastra_shot`。
- **截图纪律**：`op=burst` 连拍每张 ~2.6~3.5 秒，短局（<20 秒）别等 8 秒再连拍 ⇒ 用 `startAfterSec` 小值 + `untilGone:true`。

## 5. ★★ 真实踩坑全集（每条都是真机白跑换来的，见出处）

> **这张表是这份技能的核心价值**：没踩过的 AI 看不见这些坑，会以为"代码没问题"。

### 5.1 字号与几何（视觉"静默失败"类）

| 坑 | 症状 | 判据/修法 | 出处 |
|---|---|---|---|
| **文本框高度容不下字号** | **整条一个像素都不画**、不裁切、不报错、`visible` 仍 true、模拟器不模拟 | `h ≥ 字号 × 1.4`（CJK 行高）；字号档位只 64/52/28/22 ⇒ 64→h≥90、52→h≥73、28→h≥44、22→h≥38 | `R12`/`UI3`/工作区实测（真机白跑两次） |
| **图片控件不显式 SetImage** | 真机满屏 `?` 占位符（模拟器以前会补方块，现在也不补） | `c:SetImage(Enum.ImageSource.StaticReference, <号>)`；`imageColor` 只改颜色不改"有没有图" | `V1`/工作区实测 |
| **素材 `rotationZ` 写不进去** | 角度改不动 | 只能请创作者在编辑器转，**AI 别承诺** | `miliastra-ui` |
| **颜色构造 `FromRGB` 没 alpha** | 想半透明忘传 `a` 得到实色 | `FromRGB`/`FromRGBA` 都是 0–255；要透明度必须 `FromRGBA`；`a=nil`=255 不透明 | API 口径 |

### 5.2 层级与交互

| 坑 | 症状 | 判据/修法 | 出处 |
|---|---|---|---|
| **层级=创建顺序**（后建在上） | 弹窗盖住按钮、角图盖住格子字 | 按钮/点击热区**永远最后建**；要压最上的**只建在最上层脚本**；会互压的**必须同一脚本** | `UI5`/层级三坑（"下一关按钮呢"） |
| **`SetAsLastSibling` 只管同一父级** | 跨父级提不动；**根节点恒返回 false** | 判据用 `ret ~= false`；遍历必须按创建顺序（⛔ 不能用 `pairs`） | 工作区实测 |
| **跨脚本清理误伤热区** | `view` 写"隐藏所有非自己建的"→ 点击彻底失效 | 清理必须**白名单**（显式放行 `光标检测区域` 与黑板控件名） | `miliastra-ui` 血泪 |
| **光标事件硬前置** | 点击无反应 | 祖先容器必须 `showCursor=true` 才派发；弹窗出现那帧置 true、关掉置 false | `L2-06` |
| **模拟器层级方向相反** | 模拟器新控件在最底层、真机后建的在上 | **模拟器的层级不能推真机** | 工作区实测 |

### 5.3 代码语义（Lua 侧高发）

| 坑 | 症状 | 判据/修法 | 出处 |
|---|---|---|---|
| **漏 `local` 写进黑板/全局** | 多脚本互相踩、调试看不出 | 只有登记过的跨脚本字段才允许写全局；`node tools/scan-lua-globals.mjs` 自查 | `R2` |
| **Lua 读未绑定名字 = nil 不报错** | `failDone` 置真 → **整块界面冻结**（工作区最高发 bug，一天栽三次） | `local` 必须早于使用（`R3`）；`OnUpdate` 全体包 `pcall` + 聚合故障块（`R1`） | 工作区实测 |
| **运行期给全局赋值真机不生效** | 标志位/缓存/累加器失效，刷爆日志（实测 13983 行 / 2.8MB） | 一律用 `local`（upvalue）；自查 `scan-lua-globals.mjs` | 工作区实测 |
| **`OnStart` 不是每局都跑** | 跨局残留上一局状态 | 需 `freshStart()` + 新局检测 | `L2-03` |
| **黑板有两个写手** | 状态互写打架 | 一个字段只允许一个写入者 | `L2-04` |
| **`EnableTick` 不存在** | 静默失效 | 实为 `EnableUpdate(enabled)` | API 核对（官方技能写错） |

### 5.4 工具与部署

| 坑 | 症状 | 判据/修法 | 出处 |
|---|---|---|---|
| **deploy ≠ 进游戏** | 改了没效果 | 固定三步 deploy → 编辑器存盘 → 试玩；`match:false` 不要让人试玩 | `deploy≠进游戏` |
| **deploy/restore 不传 file 会挑错** | 按 `.gil` 挂载名挑目标，曾把 `表现 view.lua` 覆盖成 3936 B | **必须显式 `file=` + `level=`**；备份名能推断目标就以它为准，不符**拒绝写盘** | 工作区事故（P0 已修） |
| **`rolledBack:true` 是安全网** | 误以为"出事故了" | 那笔写入没生效、版本没丢；看 `safetyBackup` | 工作区实测 |
| **`.gia` 不是实时** | "日志里没有"误判"没跑" | 一局结束后才落盘；用 `miliastra_playtest op=status` 判在不在跑 | 工作区实测 |
| **试玩探针用完必须还原** | 临时覆盖活文件 | `miliastra_probe` 后务必 `miliastra_code {op:"restore"}` | 工具纪律 |
| **构建产物不许直接改** | 改了会被 build 覆盖 | 判据：存在 `tools/build-*.mjs` + 片段目录 ⇒ 你手上是产物；改片段后重建 | `R9` |
| **PowerShell `Set-Content` 写 .lua/.md** | 加 BOM + 管道压行 ⇒ Lua `--` 吞掉后面一切 | 一律用 `edit`/`write`/node `fs` | 工作区实测两次 |
| **`errors` 两档结论相反**（已修） | 全量说"这份日志没意义"、瘦身却给 48 条带行号的报错 ⇒ **去追不属于本局的旧行号** | `errorsMeaningless:true` ⇒ `errors` 为 `null` 是**设计**：**既别当"零报错"、也别当"没跑"**；结论看 `count` / `kindCounts` | 本轮实测（作者拍板 A） |
| **拿两次"最新日志"做比较** | 换局 / 试玩后两次调用各自解析到**不同的 `.gia`** ⇒ 对比结论无效 | **先 `op=sessions` 发现、再显式钉 `file`**（回归/对比一律这么做） | 本轮实测 |
| **把 `measure` 的坐标当原图像素** | 块的位置/大小对不上原图 | 坐标单位是**采样格**（`cols`×`rows`）；要像素精度就调大 `cols`（≤512） | 本轮实测 |
| **自己复述"存盘→试玩→对账"** | 每轮手写一遍（实测 ~60 次） | 直接**引用** `deploy` 回执的 `checklist[]` | 本轮实测 |
| **部署四连手搓** | 门禁 → 部署 → 镜像同步 → 比 SHA 跑四次 | 一次 `deploy {gates:true, sync:[…]}` 拿回 `shas` 四方对照 | 本轮实测 |

### 5.5 外部库/框架（不要搬）

| 坑 | 症状 | 判据/修法 | 出处 |
|---|---|---|---|
| **SUIT / flex / 9-slice / z-index 不能搬** | 平台缺管道（无绘制原语/几何 getter 读不到/层级=创建顺序） | **只取概念、别引 API**；用本平台可算数字重写 | `miliastra-ui §1.6` |
| **LUIS 许可陷阱** | MIT 但附加"禁止用于 AI 训练" | 搬任何外部代码前先看 LICENSE | `miliastra-ui` |

## 6. 工程纪律（插件工具链版）

- **发现问题先问人**（一次一件事 + 证据 + 判断 + 2~3 选项 + 代价）；工程/技术（部署/重构/跑门禁）**直接做**，玩法本体（规则/数值/文案）**先问**。
- **门禁是证据不是否决权**：红了 ⇒ 修到过 / 如实标 `pending` 交付；**不许回退、删功能、拿空壳顶**。
- **发版四步同一提交**：版本号四处一起改 → `gen-readme-tools --write` → `build-sim-play --check` + `npm test` 全绿 → 提交/tag/`main` 快进/`npm publish`（GitHub Release 页人工建）。
- **schema 体积：软线 40 KB / 硬线 50 KB**（2026-10-07 作者拍板；先例链 26.3 → 32 → 34 → 50 KB）。过软线只警告，过硬线才红。
  ★ **纪律：能力（op / 参数）一律进 schema；只有解释性长文才下沉到 skill / 文档。**
  不写进 schema 的参数 = AI **看不到 = 等于没有** —— 真实案例：`clientui` 的 `root`（点名任意控件回子树）**实现早就写好了**，
  却因为当年顶穿 34 KB 只能留在代码里 ⇒ **AI 整整一个版本调不出来**（0.7.2 才补回 schema）。

**发布前自检（本技能自带）**：`node scripts/scan-release.mjs` —— 扫四类：**敏感信息**（令牌 / 私钥 / 带凭据 URL / 本机绝对路径 / 游戏账号目录 / 内网 IP / 手机号 / 邮箱）· **技能外信息**（工作区约定目录、跨技能相对路径）· **断链**（markdown 链接与反引号路径在本仓解析不到）· **日期水印**。
只报事实与位置、**不下判决**，密钥脱敏显示；配 `--selftest`（正反例）· `--json` · `--allow <子串>` · 行内 `scan-release:allow` 豁免。

## 7. 环境降级

- **无 DSH/无插件**：退回工作区手工命令（`docs/工作区常用操作.md`：定位活文件 / 二进制备份 + 校验 / 读 `.gil`）。
- **无截图/无模拟器**：只做可计算部分（坐标表/回执字段核对），画面判断交给人截图。
- **无真机试玩**：模拟器 + 静态检查做完后如实标 `pending`，交付附验证建议（要跑的命令）。
