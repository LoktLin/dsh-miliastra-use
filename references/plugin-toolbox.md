# 插件工具箱（dsh-miliastra）—— 什么时候用哪个 · 回执看什么 · 坑在哪

> **这份解决的问题**：技能里有铁律、有代码形状、有代码风格，但**"插件本身怎么用"一直散在各处** ——
> 结果就是每次都靠现场翻工具 schema。
> ⇒ 这里按 **"我要干什么"** 组织，每条给：**用哪个 op → 回执看哪几个字段 → 踩过的坑**。
>
> ⚠️ **只写 schema 表达不了的东西**（决策、看哪几个字段、语义坑）。
> 参数名/取值一律以**工具 schema** 为准，不要从这里抄 —— 也不要把本文当成 API 文档。

---

## 目录

| 我在干什么 | 去哪一节 |
|---|---|
| 改完代码，**能不能部署** | **§1 交付闸门** ★ 最常用 |
| 要**生成**东西（粒子 / 像素画 / 流光文字 / 结构体） | §2 生成器 |
| 想知道**跑没跑 / 建没建 / 崩没崩** | §3 取证 |
| 要**现读**交接值（模板/容器索引） | §4 交接值 |
| 要**挑素材**（图片号 / 音效） | §5 素材 |
| 出问题了，**先看这张表** | **§6 语义坑** ★ |

---

## 1. ★ 交付闸门：deploy 之前 / 之后各跑什么

### 1.1 部署前（**一次调用搞定大部分**）

```
miliastra_code {op:"preflight", dir:"<只放可部署产物的目录>"}
```

**回执看两个字段**：`checkedItems`（查了什么）与 `notCheckedItems`（**没查什么**）——
后者才是你要记住的：

| ✅ 它查 | ⛔ 它**不查**（还得自己跑） |
|---|---|
| Lua **真语法**（fengari 真编译，报错带行号） | 作用域（漏 `local` / 声明晚于使用）→ `tools/check-lua-scope.mjs` |
| 结构配对 | 风格基线 → `tools/check-lua-style.mjs --baseline` |
| 全局写（真机运行期给全局赋值不生效） | 类型 / 运行时语义 |
| 图片有图源（`IMAGE_WITHOUT_SOURCE`） | 平台 UI 铁律 → `miliastra_code {op:"lint-ui"}` |

★ **这是本工程"改完必跑门"的第一道**，因为它**首次把"Lua 真语法"接进了门禁** ——
在此之前，两道语言门禁都只做词法扫描，**语法错能一路绿到真机**（实测栽过 3 次）。

> ⚠️ **坑：`dir` 模式会把"中间产物"也算进去。**
> 实测：在**构建源片段**那一层跑，报了 **60 个"不过"**、`verdict` 直接说"别部署" ——
> 但那 60 个全是像素画生成器的**中间产物**（只从里面抠数据表，**根本不部署它们**）✗
> ⇒ **要喂"只放可部署产物"的那个目录**。
> ★ 成品与片段**同在** `案子/<地图>/2.代码/`（旧的 `code/`+`sources/` 已不存在）
> ⇒ 混层时**先拷一份只含产物的临时目录**再喂 `dir`，别直接把整个 `2.代码/` 喂进去。

### 1.2 部署前（模拟器，**不占真机、不用人点**）

```
miliastra_sim {op:"bind", containerId:<容器节点>, templates:[…], scripts:[…], runForMs:16000}
```

- **`runForMs` 是"真跑"**（固定步长 1/30 照抄引擎 `FIXED_DT`）⇒ **15 秒级的事件也能一次绑完就看到**
  （死亡 → 重生 → 换人）。在此之前只能看 3~8 秒，长流程够不到 ✗
- **回执默认精简档**：只给 `controlCount`(+峰值) / `logs` / `handover.missing` / 会话字段。
  要全文（`sources[]` / `scripts[]` / `simAssumptions` 正文）传 `withMeta:true` ✓
- **三样核心一个不少**：`controlCount`（控件建没建、建了多少）/ `run.logs`（脚本跑没跑）/ 有没有 `lua-error`

**判据**：`run.logs` 里**不许**出现 `attempt to call` / `nil value` / `首错=` —— **有错就修完再 deploy**。

### 1.3 部署（**必须显式 `level` + `file`**）

```
miliastra_code {op:"deploy", level:"<关卡ID>", file:"<活文件名>", source:"<本地产物>"}
```

- ⚠️ **`level` 不传 = 写"当前关卡"**，而**换图会改掉"当前关卡"** ⇒ 实测事故：在 A 图 deploy 把 B 图的同名活文件覆盖了 ✗
- ⚠️ **多脚本工程 `file` 不传 = 按"最近改动"挑** ⇒ 曾把 `input.lua` 的内容写进 `view.lua` ✗
- **批量**：`files:[{file,source},…]`（**`files` 参数在 schema 里没有但实现了** ⇒ 只能"猜"出来，见 §6）
- **看回执的 `dest`** 再往下走 ✓

### 1.4 存盘之后、试玩之前

```
miliastra_health {op:"sha", all:true, mirror:"<工作区镜像目录>"}
```

**一张表看全部活文件**：`liveSha` / `mirrorSha` / `embeddedSha` 三列 + `verdict` + `summary`。
- `三方一致` = 可以直接试玩 ✓
- `该存盘了` = 活文件比 `.gil` 里嵌的新（**试玩跑的是嵌的那份**）
- ⚠️ **`.gil` 是"存盘那一刻的快照"，不是实时的** ⇒ 别拿它当"现在磁盘上是什么"

> 单文件版口径完全一致（同一个比较函数），`all:true` 只是一次列全 ✓

### 1.5 部署后、试玩中

| 要什么 | 用什么 |
|---|---|
| 现在到底在不在试玩 / 开跑到第几秒 | `miliastra_playtest {op:"status"}`（信号来自 `output_log.txt`，**延迟 0.07~0.18 秒**） |
| 本局 `.gia` 落盘了没有 | 同上回执里的 `localGia`（`landed`/`missing`/`running`/`none`） |
| 局中画面 | **只能截图**（`.gia` 一局结束才落盘） |
| 真机帧率 / 长帧 | `miliastra_probe {op:"deploy", template:"perf"}` → **人点一次试玩** → `op=collect` → **`miliastra_code {op:"restore"}` 还原脚本** ⚠️ 别忘了最后一步 |

---

## 2. 生成器：`miliastra_gen` 的四个 op

**共同点**：都是**离线生成**、零平台 API、**不写任何文件** ⇒ 产出交给你自己落盘/接入 ✓
**共同点 2**：交接值（模板索引 / 容器索引）**它会先自动读当前关卡的 `.gil`**，
**只有唯一候选才采用**；拿不到就**报错点名**（`needsHandover[]`）—— **绝不编** ✓

| op | 生成什么 | 什么时候用 |
|---|---|---|
| **`vfx-lua`** | 可部署的**粒子/图元**客户端 Lua | 特效、弹道；**`preset:"list"` 先挑**（13 粒子 + **3 图元**） |
| **`pixel-art`** | 可部署的**像素画** Lua（图片控件**矩形块拼图**，不是一像素一控件） | 参考图/头像 → 控件；要 `cols`/`rows`/`maxSide` + `pixelSize` |
| **`text-gradient`** | **逐帧刷字**的客户端 Lua（`EnableUpdate` + `OnUpdate` 换帧） | 流光标题、渐变文字 |
| **`struct-json`** | 可导入千星的**变量 JSON** | 自定义变量/结构体 |

★ **实战用法**（夏日祭法师）：`pixel-art` 生成 74 个头像（cols=6 ⇒ 每人 ~27 块）；
`text-gradient` 生成标题帧表注入主脚本；`vfx-lua` 生成 26 套粒子外观 + 3 类图元 ✓

> ★ **`vfx-lua` 的 `output:"data"` 优先** —— 它给**结构化层**（`layers[]`，含每个字段/曲线/弧长），
> 直接喂构建脚本即可。**别用 `output:"lua"` 再正则解析自己的产物**（产物格式一变就静默错位）。
> ⚠️ **图元层**（`shapeKind:"sprite"`）**打破**"控件数 = 层数 × 每层池"的旧口径：
> **一个形状 = 1 个控件**（+ `trailCount` 个残影）⇒ 比用粒子拼便宜一个数量级 ✓
> 详见技能 `miliastra-code` 里的极端粒子案例（`case-extreme-particles.md`）§4。

> ⚠️ **两个硬规则**（生成前校验，不过就报错）：结构体 ID 必须 **10 位数字**、单条文本 **≤500 字符**。
> ⚠️ **未验证项恒带 `unverified[]`**：`<size=N>`、4bit、图元层的真机渲染（旋转正负号 / 拉伸采样）——
> **`preflight[]` 里 `ok:null` 的意思是"判不了"，不是"通过"** ✓ 别把它读成绿灯。

---

## 3. 取证：跑没跑 / 建没建 / 崩没崩

| 问题 | 用什么 | 看什么 |
|---|---|---|
| **有没有报错** | `miliastra_log {op:"errors"}` ★ **先跑这个** | 按**形态**捞（stack traceback / `attempt to index` / nil value / 文件:行号），**不看标签** —— 真机报错行**可能没有任何 `[…]` 前缀**，按 tag grep **一条都捞不到** |
| 这一局跑成什么样 | `miliastra_log {op:"runs"}` | 每局一行摘要 + 与上一局的 diff |
| 日志里的指标分布 | `miliastra_log {op:"metrics", evt:"…"}` | `core`（集中区）/ `hotBin`（热区）；要求日志写成 `k=v` 才好抽 |
| 脚本运行时建了哪些控件 | `miliastra_sim {op:"controls", runtime:true, geom:true}` | 需先 `op=play action=start` 且 `keepRunning:true` |
| 画面是什么样 | `miliastra_shot {op:"capture"}` | **画面本工具不识别**；`pid/process/title` + **候选窗口清单**要核对 |
| 等开跑再连拍 | `miliastra_shot {op:"burst", awaitPlaytest:true, startAfterSec:1, untilGone:true}` | 短局别"等 8 秒再拍"（会全落局外） |

> ⚠️ **"试玩了却没有新日志"有两种**：① 这一局一条 `print` 都没有；② **`.gia` 落盘因时序失败**。
> 后者时**"没有日志"不能当唯一判据** ⇒ 判画面改用截图 ✓

> ⚠️ **`miliastra_log` 是"模拟器日志"还是"真机 `.gia`"要看工具回执的 `logScopeNote`** ——
> 两者取证路径不同（模拟器随时可读，真机要等一局结束）✓

---

## 4. 交接值：怎么**现读** + 台账

**铁律**：容器节点索引 / 控件模板索引 / 图片资产 ID **必须来自创作者或 `.gil`，禁止编造**（R8）。

**能自动读的**（先读，别问人）：
- `miliastra_map {op:"clientui", summaryOnly:true}` —— 本关**能被脚本动态创建**的模板（**只有"无父节点"的独立模板算**）
- `miliastra_sim {op:"handover"}` —— 从源码里抽
- `miliastra_gen` —— 它自己就会读 `.gil`（唯一候选才采用）

**台账**（确认一次，长期免传）：
```
miliastra_health {op:"handover", action:"set", handover:{container:…, imageTemplate:…}}
```
之后 `miliastra_gen` **自动带上**（回执里标 `handoverFrom` 含 `ledger`，逐项带 `from:"ledger"`）✓
**只有显式 `set` 才写盘**；`clear` 要 `confirm` ✓

⚠️ **写死 = 换图即废**：实测 `1073741838` 换到 `1073741839` 后旧号全不适用 ✓

---

## 5. 素材

| 要什么 | 用什么 |
|---|---|
| 挑**图片号**（按语义） | `miliastra_asset {op:"icon-search", q:"…"}`（1543 条；名字是**识图推断**，恒带 `nameSource:"vision-inferred"` + `confidence`） |
| 看**平台图片资源库**（按分类/色档） | `miliastra_asset {op:"catalog", category:"…"}` |
| 找**音效** | `miliastra_asset {op:"sound-search", q:"宝箱 开启"}`（中英名模糊搜；⛔ **不支持拼音**） |
| 把图**存下来反复引用** | `miliastra_asset {op:"add", source:"<绝对路径>"}`（**按内容寻址**，同图只存一份；落**插件数据目录**，不碰活文件） |

★ **真机可用全部 1543 个图片号；模拟器只画 `100001~100006`**
（方块 / 圆 / 三角 / 四角星 / 五角星 / **圆环**）⇒ 挑素材时**预览用 `100001~100006`，真机再换真号** ✓

---

## 6. ★ 语义坑（每条都是我当时踩的现场）

| # | 坑 | 现场 |
|---|---|---|
| 1 | **`step {dt:N}` 只跳时钟、不补跑中间逻辑** | 我以为"推进 16 秒"，结果 `time` 跳到 18.13s 但 `frame` 只有 65、日志全挤在一个时间戳、HP 只掉十几点 ⇒ **试了 4 次才明白**。**要长流程用 `bind` 的 `runForMs`**（那是真跑） |
| 2 | **`bind` 的 `settleSec` 与 `runForMs` 同时给，以 `runForMs` 为准** | 回执里 `runMode` 会明说；别以为两个都生效 |
| 3 | **`files:[…]` 批量部署实现了，但 schema 里没有** | 我猜了三次参数名才投成 9 个文件；回执 `mode:"multi"` 是成功标志 |
| 4 | **`op=deploy` 的 lint 现在会真编译**（fengari）—— 但要看 `lintChecks`/`lintSkips` 自述边界 | 早期它**不查语法**却没有任何限定语 ⇒ "deploy 成功"被我误当成"语法正确"（栽 3 次） |
| 5 | **`preflight` 的 `ok:null` = 判不了**，不是通过 | `unverified[]` 里的项（`<size=N>` / 4bit / 图元真机渲染）一律 `ok:null` ⇒ **别读成绿灯** |
| 6 | **交接值随换图变化** ⇒ 每次 `deploy` 显式 `level` | 事故：在 A 图 deploy 覆盖了 B 图同名活文件（靠自动备份+`op=restore` 救回） |
| 7 | **`.gil` 是存盘当时的快照**，不是实时 | 判"游戏跑的是哪版"用 `miliastra_map {op:"script"}` 的 `match` / `sha all:true` 的 `embeddedSha` |
| 8 | **模拟器通过 ≠ 真机通过** | 官方素材渲染 / 联机 / 手感 / **帧率**都不覆盖 ⇒ 如实标 `pending` |
| 9 | **试玩探针用完必须还原** | `miliastra_probe` 会**临时覆盖活文件** ⇒ `op=collect` 之后务必 `miliastra_code {op:"restore"}` |
| 10 | **回执都带 `ok`，缺 `ok` 不能当成功** | 判回执**先看 `ok`**；没有这个字段就按"未知"处理 —— 缺 `ok` 却按成功走，会把失败读成通过 |
| 11 | **`bind` 的 `templates` / `containerId` 不能空** | 空着也能跑完，但**建不出控件、画面不变**，且**没有报错**（看不出原因）。预览链路：`miliastra_gen op=vfx-lua`（生成）→ `miliastra_sim op=bind`（投工程）→ 试玩页看画面 |

---

## 7. 一句话记住的优先级

> **改完先过工作区门禁（作用域 + 风格两道，见 SKILL.md §8）→ 再 `preflight`（含真语法）→ 模拟器 `bind runForMs`（看控件数+首错）→ `deploy`（显式 level+file）
> → 编辑器存盘 → `sha all:true` 确认三方一致 → 人点试玩 → `log op=errors` 取证。**
>
> 这套里**只有"存盘"和"点试玩"必须人做**，其余都能自动 ✓
