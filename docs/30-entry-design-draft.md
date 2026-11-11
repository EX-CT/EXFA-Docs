# 30 · 引擎入口设计草案：check / canfit / 严重度（2026-11，草案）

接 29。状态：**草案，尚未实现**。标「待确认」的是还没拍板的点。

## 原则

1. 一个入口 = 一种输入形状 = 一种输出形状；不用 `operation` 之类的字段切换。
2. 引擎窄而严格：不补默认值、不认识"组"、不认识模式。Format 定义请求与结果的**形状**和**模式**，引擎入口执行**判断**。
3. 判断规则只在引擎里写一份（引擎有 SDE 和属性计算，Format 没有）。
4. 入口名是单个小写词，不用下划线。

## 入口分档

| 档 | 性质 | 入口 |
|---|---|---|
| A 核心计算 | 无状态，一入口一形状 | `calc` `batch` `check` `canfit` `graph` |
| B 查询 | 无状态，查数据 | `search` `type` `market.*` `item.*` `names.resolve` `implant_sets.list` …（命名后议） |
| C 格式互转 | 无状态，属 `exfa-formats` | `eft_parse` `eft_export` `format_*` `fits.backup` `character.import_evemon`（命名后议） |
| D 会话/运维 | **有状态**（改进程全局状态） | `prices_load` `sde_override` `pyfa_data_load` `pyfa_data_status` `version` `meta` |
| E 搜索/规划 | 输入输出复杂 | `optimize`（本轮不设计，A 档稳定后再议） |

## 严重度

每条 violation 带 `severity`，**写死在引擎里，请求不能改**。

| severity | 含义 | 前端 | agent |
|---|---|---|---|
| `fatal` | 配装不合法，结果无意义 | 退回这次改动，显示原因 | 不要用这个配装 |
| `overload` | 合法，资源超了，数值有意义 | 照常计算，标红 | 可调整 |

请求本身有问题（JSON 坏、`type_id` 不存在、字段类型错）不是 violation，是最外层的 `error`；前端同样退回。

### 现有 violation code 的归类

| severity | code |
|---|---|
| fatal | `SLOTS_EXCEEDED` `TURRET_HARDPOINTS` `LAUNCHER_HARDPOINTS` `NOT_FITTABLE` `SHIP_RESTRICTION` `RIG_SIZE` `CHARGE_GROUP` `CHARGE_SIZE` `CHARGE_CAPACITY` `MAX_GROUP_FITTED` `MAX_TYPE_FITTED` |
| fatal（待确认，见下） | `MAX_GROUP_ONLINE` `MAX_GROUP_ACTIVE` |
| overload | `CPU_OVERLOAD` `POWER_OVERLOAD` `CALIBRATION_OVERLOAD` `DRONE_BANDWIDTH` `DRONE_BAY` `FIGHTER_BAY` `FIGHTER_TUBES` `CARGO_OVERLOAD` `MISSING_SKILL` |

### 需要新增的 violation

实测发现引擎现在**不查**这些（见本轮讨论）：

| 新 code | severity | 说明 |
|---|---|---|
| `SLOT_MISMATCH` | fatal | 请求显式写的 `slot` 与模块按效果推断出的槽位不符；现在显式值直接覆盖推断值，没有任何提示 |
| `SUBSYSTEM_DUPLICATE` | fatal | 两个子系统 `subSystemSlot` 相同（核心 125 / 防御 126 / 进攻 127 / 推进 128）。`maxSubSystems` 是 5 但只有 4 个类别，数量检查拦不住 |
| `SUBSYSTEM_MISSING` | fatal | T3 缺少某一类子系统；判定是"4 个类别各一个"，不是"数量够 5" |
| `ACTIVE_DRONES_EXCEEDED` | fatal（待确认） | 出动无人机数超过 `maxActiveDrones`；现在只在输出里显示，不检查 |

### 自动调整的归类

| 现有调整 | 归类 | `check` | `calc` |
|---|---|---|---|
| `STATE_CLAMPED` | 轻微 | 返回修正后的配装 | 自己改，记入 `adjustments` |
| `SECURITY_DEFAULTED` | 轻微 | 同上 | 同上 |
| `MODE_DEFAULTED`（T3 驱逐舰没写模式） | **fatal** | 报错 | 拒绝；前端负责默认选一个模式 |
| `FIGHTER_QUANTITY_CLAMPED` | **fatal** | 报错 | 拒绝 |
| `SLOT_OCCUPIED_SKIPPED` | **fatal** | 报错 | 拒绝 |

原则：**只要引擎改了用户明确放进去的东西，就不算轻微。** `check` 与 `calc` 共用同一个规整函数。
`compute` 的 `DEFAULTED` 调整已随信封移除。

## `calc` / `batch` 的行为变化

- 默认遇到 fatal 或 error → 拒绝：最外层 `error`，`code: "INVALID_FIT"`，带完整 fatal 列表（`code`、`message`、位置）。overload 照算。
- `batch`：逐项同样处理，失败项留在各自位置（现有行为），不影响其他项。
- **放行开关** `options.restrictions: "off"`：不拒绝，照算，`violations` 里仍列出问题。对应 Pyfa 的 "Disable Fitting Restrictions"。
  Bench 里输入故意不合法、Pyfa 仍出数的用例（目前 15 个：13 个 fatal 类、2 个 state 类）加此选项继续有效。
- 影响面：`MODE_DEFAULTED` 变 fatal 后，Bench 里没写模式的战术驱逐舰用例会被拒绝，需补显式模式。个数实现时跑套件确认。

## `check`

输入：一个配装（FitRequest）。输出：

```jsonc
{
  "ok": true,                 // 没有 fatal
  "violations": [ {"code": "SHIP_RESTRICTION", "severity": "fatal", "message": "...", "module_index": 0} ],
  "adjusted": { /* 修正后的配装；只有存在轻微调整时才有 */ },
  "adjustments": [ /* 做了哪些轻微调整 */ ]
}
```

前端流程（每次编辑后）：
1. `error` 或 `ok:false` → 退回这次改动，显示原因。
2. 有 `adjusted` → 无感替换成它。
3. 只剩 overload → 进入 `calc`，标红。

`check` 不输出统计，不做完整计算以外的事；`fix` 不再单设，修正并入 `check`。

## `canfit`

输入：一个配装 + 候选列表 `[{type_id, slot?, charge_type_id?}]` + `mode`。只判断 **fatal**（不含 overload，不含技能）。

- `mode: "filter"`：只返回能放的候选 id 列表，用于直接过滤装备列表（最紧凑、最快）。
- `mode: "explain"`：每个候选返回通过与否和 fatal 原因，用于灰显并提示原因。

性能：基础配装只处理一次（槽位数含子系统加成、挂点、已装物品的组/类型计数），每个候选只做查表：槽位是否有空、舰船限制、改装件尺寸、
`maxGroupFitted` / `maxTypeFitted`、挂点、子系统类别、弹药匹配。不涉及技能、加成和电容模拟。具体耗时实现后实测。

## 信封（RPC 外层包装）

- 成功只有 `result`，失败只有 `error`，都在最外层（现在错误有最外层和 `result` 内两处）。
- `method` 必须写，缺省不再当作 `calc`（倾向，**待确认**）。

## 待确认

1. `MAX_GROUP_ONLINE` / `MAX_GROUP_ACTIVE` / `ACTIVE_DRONES_EXCEEDED` 是并入 fatal，还是单列一档（"能装但状态不允许"，前端表现为拒绝切换状态而不是拒绝放置）。
2. `method` 缺省是否报错。
3. B / C / D 档的命名规则。
4. 模式与显式标记的最终名字（见 29）。
5. 价格与预算怎么接（`calc` 的价格块、`optimize` 的预算约束）。
6. `optimize` 的设计。

## 实施顺序

1. 合并 29 对应的 `compute` 移除（Engine #100、Format #2、Bench #13、Docs #4，随后 App）。
2. 信封规则 + 严重度 + 新增 violation + `restrictions` 开关，`calc` / `batch` 拒绝 fatal；Bench 同步补显式模式与开关。
3. `check`。
4. `canfit`。
5. 命名收敛（B / C / D 档）。
