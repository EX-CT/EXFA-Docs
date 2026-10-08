# 27 — v2 交接 / v2 handoff

状态：App v2 规格 A–D 已合并上线；§5 为下一阶段待实现的 Format/Engine/配置组方案　·　更新：2026-10-08　·　前一份交接（迁移，已完成）见 [24](24-migration-handoff.md)

这份文档汇总 EXFA v2 大升级的进度，覆盖全部仓库，换环境（云端 ↔ 本地）后从这里接手。
App 各阶段的细节规格放在 EXFA-App 的 `docs/v2/`（设计方案 `design.md`、实现规格 `app-v2-spec.md`、格式规格 `format-v1-spec.md`、App 侧状态 `HANDOFF.md`），均在 main 上。

## 1. 总表

| 仓库 | 状态 | 最新 |
|---|---|---|
| [EXFA-Engine](https://github.com/EX-CT/EXFA-Engine) | v2 所需功能**已完成** | release `v0.2.1`（契约 1.5.0，内置 `sde-3586130-r7`），main CI 绿 |
| [EXFA-Format](https://github.com/EX-CT/EXFA-Format) | v1 **已完成** | `v1.0.0`（schema + TS `@exfa/format` + Rust `exfa-format` + migrate/resolve + 目录导出布局） |
| [EXFA-Data](https://github.com/EX-CT/EXFA-Data) | 正常运行 | Latest = `sde-3586130-r7`；价格快照每天发布（`prices-jita44-*`） |
| [EXFA-Bench](https://github.com/EX-CT/EXFA-Bench) | CI 已修复，bench-ci 绿（见 §3.2）；预期 SDE build 改从 `$EXFA_DATASET` 读取（见 §3.5） | presets `presets-pyfa-3569502-r6` |
| [EXFA-App](https://github.com/EX-CT/EXFA-App) | **Stage A–D 全部合并上线**（C：`1b208a7`，D：`77e7ee6`）；Pages 已恢复部署（见 §3.1） | 引擎 WASM `v0.2.1`（`ba9c3a4`）；配置库已迁到 `@exfa/format`（`77e7ee6`） |
| EXFA-Docs | 本文档 | |

## 2. 已完成

- **Engine v0.2.0**（[PR #6](https://github.com/EX-CT/EXFA-Engine/pull/6)）
  - `adjustments[]` `{code,path,from,to}`：引擎自动纠正过的输入逐条列出。
  - `scenarios[]` → `scenario_results`：固定敌人、距离、速度、角度的有效 dps，与图表上对应点的数值相同。
  - 图表新增攻击方速度、攻击方角度、目标角度三个横轴。
  - 格式转换 RPC 并进主 `exfa_wasm.wasm`。
  - 反馈分四级：`adjustments`（已纠正，计算继续）/ `violations`（游戏里装不上或超限）/ `warnings`（真缺口：技能缺失、不支持的投影种类、未知 buff）/ `error`。
- **Engine v0.1.2 / v0.1.3**
  - 可以自动纠正的请求静默归一，不再发警告（被动件 active→online 等）。
  - `type` RPC 新增 `allowed_states`。
  - 新增 violations：`FIGHTER_TUBES`、`FIGHTER_BAY`、`DRONE_BAY`、`CARGO_OVERLOAD`。
- **EXFA-Format v1.0.0**：分三层，Fit → FitDocument（可替代件、命名分支、版本历史）→ Library（文件夹、场景、编队）。EFT、DNA、ESI、XML 只是外部转换格式。
- **App Stage A**（[PR #1](https://github.com/EX-CT/EXFA-App/pull/1)，squash `cfa5245`）
  - 新 logo、PWA，暖黑底配金色。
  - 单屏三栏布局，底部停靠区。
  - toast 提示带撤销，去掉了浏览器原生弹窗。
  - 网页只用 WASM 引擎（HTTP 引擎留给 MCP/API）。

## 3. 未完成 / 已知问题

### 3.1 App Pages 部署自 Stage A 合并后一直失败（已修复，EXFA-App #3）

Stage A 合并后，`pages` workflow 的 bench 门禁报 `formats` 0/4779，部署没有执行，线上站点还是 Stage A 之前的版本。

原因在测试工具，产品代码没问题。Stage A 删掉了 TS 解析器，格式转换改走引擎 RPC。`apps/web/tools/browser-rpc.mjs` 把 `rpcRaw` 返回的 `{id,result}` 信封又包了一层 `result`，bench 读到的结构因此全部对不上。修复分支是 `devin/1791378812-fix-pages-formats-rpc`。已合并（EXFA-App #3），合并后 `pages` 的 build/deploy 均成功，线上站点已是 Stage A。

### 3.2 EXFA-Bench bench-ci 红（已修复，EXFA-Bench #4）

`nightly.yml` 还在从已删除的 `EX-CT/eve-sde-pipeline`、`EX-CT/eve-fit-docs` 下载数据。修复分支 `devin/1791378812-ci-exfa-data-sources` 把来源改成：
- dataset：EXFA-Data `sde-3569502-r7`；
- presets：EXFA-Bench `presets-pyfa-3569502-r6`；
- 功能清单：EXFA-Docs。

已合并（EXFA-Bench #4），main 上 bench-ci 已转绿，issue #1 已关闭。

### 3.3 App Stage B（[PR #2](https://github.com/EX-CT/EXFA-App/pull/2)，已合并 `8a6d84d`，已上线）

紧凑市场、智能/替换/添加模式、单击预览与双击应用、对比条（变种、弹药、离线、角色），每一行都由引擎真算。

验收已完成：e2e 105/105（修掉了从未跑通过的测试脚本问题：已移除的 puppeteer API、`{clickCount}`、`'Control+a'` 组合键写法、单击装件改为双击 Add 模式、Windows 路径 `fileURLToPath`）；1920×1080 与 1600×900 截图目视通过（`tools/shots.mjs`）；CI 与 pages 部署均绿。

细节见 App 的 `docs/v2/HANDOFF.md`（已在 main）。

### 3.4 App Stage D（[PR #4](https://github.com/EX-CT/EXFA-App/pull/4)，已合并 `77e7ee6`）

配置库已迁到 `@exfa/format`（`github:EX-CT/EXFA-Format#v1.0.0`）：

- 内部模型换成 `FitDocument`/`Library@1`，引擎请求一律走 `requestFor` → format `resolveFit`（覆盖 refs/links/fleets/scenarios）。
- IndexedDB `eve-fit-web` 升 v2：`docs` + `kv.index`；旧 `fits`/`kv` 和 localStorage 经 `migrate()` 迁移并原样留作备份。
- FitBrowser：真·文件夹树（任意层级、拖拽）、编队节点含成员/角色徽章、fit 展开分支+版本历史。
- fit 头部分支下拉（分歧时带 `*`）+ 保存为分支；对比条新增可替代 tab，模块行 `⇄n` 徽章。
- 导入导出：单 fit `.exfa.json`；整库 `.zip`（fflate + `toFiles` 目录布局）；`showDirectoryPicker` 文件夹导入导出（不支持时按钮隐藏）；`.exfa.json`/`.zip`/文件夹/旧备份 JSON 全部走 `migrate`/`fromFiles`，按 id+`modified` 合并并显示 导入/更新/跳过 计数。
- 验收：`tsc`+`vitest` 54/54+`vite build` 绿；smoke-d3 7/7、d4 5/5、d5 4/4；e2e 108/108（新增 `.exfa.json` 导出、`.zip` 导出、zip 合并导入三项）。

### 3.5 App Stage C（[PR #5](https://github.com/EX-CT/EXFA-App/pull/5)，已合并 `1b208a7`）

场景系统 + 图表坞重写，全部走引擎 graph RPC：

- 场景编辑器在左侧「配置文件」tab：列表 + 内联编辑（名称；目标 = 目标抗性 / 库中配置+resist_mode / 自定义数值；距离 km；目标/我方速度 m/s 或 % 切换；角度；目标信号半径覆盖；设置含 ignore_resists 默认关、apply_projected、ignore_lock_range、ignore_drone_control_range、mobile_drone_mode）。内置场景「当前目标 · 10 km 环绕」随库播种（builtin 不落盘）。
- 右栏「场景」区：勾选写 `doc.refs.scenario_ids`，引擎 `scenario_results` 显示有效 dps/齐射 + 占纸面 %，错误内联。
- 图表坞：横轴 = `graph_specs` 全部轴（含攻击方速度/角度新轴），每轴单位+默认范围（跃迁距离按 AU 显示）；Y 多选；线源 = 配置多选（分支单列）× 场景多选；按 fit 上 8 色调色板、按场景分线型；图例点击开关、悬停十字线取值、CSV/PNG 导出、上限 12 条（超出有提示）。
- 细节：扫 `tgt_speed_pct` 时会把场景 params 里的 `tgt_speed_mps` 一并删掉（引擎规则绝对值优先，不删则轴无效）；ewar/remote_reps 只提供 fit 目标的场景（profile 目标在引擎侧被忽略）。
- 验收：`tsc`+`vitest` 61/61+`vite build` 绿；smoke-c 13/13；e2e 108/108（target-fit / ecm-damage 检查改为场景驱动）。

至此 app-v2-spec 的 A–D 四个阶段全部完成。

### 3.6 SDE 版本不一致（已修复：引擎 `v0.2.1` 内置 `sde-3586130-r7`）

已修。引擎 `v0.2.1`（提交 `f2dc7e7`）内置 `sde-3586130-r7`，与界面数据集一致；App 已钉用 v0.2.1 WASM 并部署（`ba9c3a4`）。

这次升级暴露并修掉了三个流水线问题：

1. **Bench fixture 钉死旧 build**（EXFA-Bench #5）：`sde`/`price_inject`/`batch` 套件的 runner 和生成数据硬编码 `SDE_BUILD=3569502`，门禁对 3586130 引擎报了 ~100 个元数据失败。现在 `SDE_BUILD` 从 `$EXFA_DATASET` 的 `sde.build` 读取，fixture 用 `d22/tools/gen_d22.py`、`batch/tools/gen_gap_cases.py` 重新生成；`snap-other-build` 仍固定 3500000（故意测外来 build 警告）。
2. **bench.lock 与 sde.lock 的更新顺序死锁**：`ci.yml`/`sde-update.yml` 新增 `bench_sha` 输入——门禁可以用未合并的 Bench 提交跑，`sde-update` 的 update 任务把该 sha 和 `sde.lock` 写进同一个提交，main 上每个提交都自洽。
3. **release.yml artifact 污染**：sde-update → ci 门禁 → release 同处一个 workflow run，`manifest` 任务的无 pattern `merge-multiple` 下载把门禁的 `ci-results-*`（`out/<suite>/` 目录）混进 `dist/`，`sha256sum -- *` 遇目录报错。包产物改名 `pkg-*` 并按 pattern 下载。

另外 `EXFA_DISPATCH_TOKEN` 确实没配（EXFA-Data 侧日志：`dispatch skipped`）。没有再依赖 PAT：`sde-update` 现在每 6 小时轮询 EXFA-Data 最新 `sde-*` release（仓库是 public，GITHUB_TOKEN 可读），自动走完升级流水线；repository_dispatch 通道保留为快车道。手动触发时不填 tag 也会取最新 release。

### 3.7 杂项

- EXFA-Engine 本地有一份旧 stash（`WIP on 1791223779-gitattributes-eol-lf`），是 crate 合并前的 Cargo.lock 改动。修复早已合并，可以丢弃。
- 不做的事：后台自动目录同步、桌面端外壳（Tauri）。目前只做手动导入导出；目录结构与将来的桌面端保持一致。

## 4. 接手须知（本地或云端）

- App 的测试要在 `apps/web` 目录里跑：`npx tsc -b && npx vitest run && npx vite build`。在仓库根目录跑 vitest 会扫到 `packages/mcp/dist`。
- 浏览器工具（`tools/smoke.mjs`、`e2e.mjs`、`screenshot.mjs`、`browser-rpc.mjs`）需要用 `CHROME` 环境变量指定 Chrome 路径，默认值是 `/usr/bin/google-chrome`。
- 本地预览：`npx vite build && npx vite preview --port 4173`，然后打开 `http://127.0.0.1:4173/EXFA-App/`。
- 本地构建 Engine 需要设置 `EXFA_DATASET=<dataset-3586130-r7.json.gz>`（或 `sde.lock` 里钉的那版）。不要对整个仓库跑 `cargo fmt`：仓库本身没有按 rustfmt 格式化，跑一次会改动几十个文件。
- 改了 MCP 工具描述后要跑 `npm run schemas`，CI 会比对 schema 快照。
- 发布 Engine 时先合并 PR，`fetch origin/main` 之后再在合并提交上打 tag。另外 App 的 `mcp-engine` CI 用的是最新的 Engine release，所以发版前先在 `packages/mcp` 用新的二进制跑一遍测试。
- Commit 结尾的署名：`Co-Authored-By: Devin AI <158243242+devin-ai-integration[bot]@users.noreply.github.com>`。

## 5. Format、批量引擎与配置组（已实现；本节为契约与验收的记录）

> **性质和范围**：本节 2026-10-08 与用户讨论形成，按 §5.6 顺序已在 Format v1.1.0 / Engine v0.2.2 / App Stage E / Bench 落地（实现状态见 §5.7）。用户明确允许重新设计 EXFA-Format，**不要求保持此前 Format v1 的兼容性**。本节不是授权删除当前用户数据；存储切换、数据清除等操作须另行明确。价格快照与价格表机制维持现状，仅由宿主按已有方式适配。后续改动须维持本节的契约与验收，不要把现有 `exfa/fit@1` 结构当成目标设计的限制。

### 5.1 用户确认的硬约束与现状依据

1. 引擎保持计算与 SDE 数值权威；前端/宿主维护工作区、配置组、配置之间的有向关系，解析引用并编排计算。引擎不得因配置组而持有工作区、用户配置库或舰队会话状态。
2. **一份计算输入对应一份计算输出，不代表一次只算一艘船**：一份批量请求对应一份包含 N 条结果的批量响应。引擎原有 `batch` 是必要能力，不允许以 Web 循环 `calc` 取代。
3. 机器人需要的“舰船＋装备、默认全 5 技能”是**同一种计算格式的简写／语法糖**，不是与详细配置平行的 `quick` 模式或另一套内核。显式全 0、角色档案、模块状态和其他设置必须覆盖省略时的默认值。
4. 投射来源可以是需要解算的**完整舰船**：来源船自身的技能、装备、弹药和可用加成参与引擎计算。宿主决定来源、目标、模块分配；引擎决定技能修正、实际投射量、距离衰减及目标侧效果。
5. 引擎仍提供图表、搜索、格式转换及批量／参数扫描等现有能力；不能因为新计算媒介而砍掉这些操作。价格机制不列为重构内容。

核对代码的事实：`EXFA-Engine/crates/exfa-core/src/batch.rs` 已实现 `fits`、`base+variants`、`base+product`、`base+sweep`、上限、差值、字段提取、筛选、排序、逐条错误；`EXFA-Docs/docs/23-batch-api-and-prices.md` 是现行批量语义。原生引擎实测：单个 BatchRequest JSON 得到单个 BatchResponse JSON；`fits` 中的两条完整 `stats` 分别与独立 `calc` 一致。当前直接调用引擎时，省略技能实际等于全 0、省略模块状态等于 `online`，而 Web/Format/MCP 有自己的默认全 5/活动状态补全，需统一。当前 Format 单文档的角色/场景/其他配置主要是 ID 引用，单独导出后可能回落默认值或丢失依赖；当前舰队仅有成员及指挥/成员角色，并非关系图。Web 的 `Compare` 已优先调用 `batch`，MCP 的 `compute_batch` 主要做友好输入转换，然后把组合计算交给引擎。

### 5.2 对象分层、格式所有权和输入输出边界

以下标识是**建议名称**，新格式可按实施阶段细化，但必须保持语义和边界：

| 对象 | 消费方 | 含义 |
|---|---|---|
| EFT/DNA/ESI 等外部格式 | 游戏和第三方 | 仅导出选定的实际装配；不假装能容纳 EXFA 场景或编队关系 |
| `exfa/fit-plan@1` | 宿主 | 完整单配置：装配、装备/弹药变体、角色技能、场景/假想敌、外部条件及显式设置 |
| `exfa/group@1` | 宿主 | 配置模板的多个舰船**实例**、实例覆盖和有向投射/指挥/目标关系 |
| `exfa/workspace@1` | 宿主 | 隔离配置库、编队、角色、场景、默认值和编辑状态 |
| `exfa/package@1` | 宿主之间 | 带依赖闭包的可交换包，根对象为单配置、组或工作区 |
| **`exfa/compute@1`** | **引擎** | **一次计算请求；`operation=calc` 或 `operation=batch`** |
| **`exfa/compute-result@1`** | **调用者** | **与上述一次请求配对的一个响应对象** |

引擎**只接受已经确定的计算内容**，不接受需要查询宿主库才能解释的 `fit_id`、`character_id`、`workspace_id`、`group_id`。宿主文档可以保留这些引用，但送进引擎前必须完成解析、字段作用域合成及依赖校验。外层计算格式包含单算或批量的判别字段，内层共享一个 `FitSpec`；引擎内部可以继续使用规范化的 Rust `FitRequest` 和已有 `calc`/`batch` 内核。图表继续有其独立、带版本的 GraphRequest/GraphResult；图表的一份输入可以包含多个 X 样本，`graph-batch` 不应被删除。

EXFA-Format 仓库是**交换对象及计算输入外壳的 schema/类型权威**：维护 JSON Schema、TS/Rust 类型、纯引用解析和共享测试向量；不得重新实现 dogma。Engine 负责使用其内嵌 SDE 做数值相关的归一化和计算，负责实际 FitStats/BatchResponse 的数值语义；对外交互使用 Format 定义的计算媒介，双方用同一批 fixtures 和 CI 防止请求/结果形状漂移。App/MCP 可将物品名称、EFT、DNA 转成数字 `type_id`，但计算媒介须确定、无模糊名称；不能让 Web 和 MCP 各自定义语法糖默认值。

### 5.3 计算格式、语法糖与批量（关键契约）

拟议的顶层结构：

```ts
type ComputeRequest =
  | { format: 'exfa/compute@1'; operation: 'calc'; fit: FitSpec }
  | { format: 'exfa/compute@1'; operation: 'batch'; batch: BatchSpec };
type ComputeResponse =
  | { format: 'exfa/compute-result@1'; operation: 'calc'; result: FitStats }
  | { format: 'exfa/compute-result@1'; operation: 'batch'; result: BatchResult };
```

机器人或简单调用者可提交合法的**同一种格式的简写**：

```json
{"format":"exfa/compute@1","operation":"calc","fit":{"ship":{"type_id":587},"modules":[{"type_id":438}]}}
```

新的计算格式应明确规定省略技能＝全 5、Alpha＝否、省略植入体/药剂/投射/舰队增益＝无；省略可激活装备状态＝可激活则 `active`、否则 `online`；省略目标＝无指定敌人（纸面数值不能被标为针对敌人的有效 DPS）。无人机/舰载机、默认环境、验证和 `null`/显式 `false` 的语义也须在 schema、规范化测试里逐字段定稿。**这不是当前引擎默认行为**，不能仅在 Web 端补值：归一化应由引擎入口执行，用同一份引擎 SDE 判定装备能否激活；宿主允许预览，但无权定义另一套数值默认。显式提供全 0 技能、`online` 等值时原样生效，不因“省略默认全 5”而被覆盖。对未知物品、不合法状态或不支持的输入，返回结构化错误/明确的 `adjustments`，不静默替换。

拟议 batch 写法：

```json
{
  "format":"exfa/compute@1", "operation":"batch",
  "batch": {
    "base":{"ship":{"type_id":587},"modules":[{"type_id":438}]},
    "variants":[
      {"id":"active","patch":[]},
      {"id":"online","patch":[{"op":"replace","path":"/modules/0/state","value":"online"}]}
    ],
    "fields":["navigation.max_velocity"], "deltas":true
  }
}
```

`BatchSpec` 必须保留现有四种来源 `fits` / `base+variants` / `base+product` / `base+sweep` 及 JSON Patch、`swap_type`（替换所有同类型件）、`fields`、`deltas`、`delta_ref`、`filter`、稳定 `sort_by`、`top_n`、per-item 错误、价格覆盖和组合数上限。引擎仍负责**批量展开及运算**，不是前端展开 N 次独立 `calc`。请求中这四种来源只能选一种；一组不同实例通常编成 `batch.fits`，单配置的弹药/技能方案适合 `variants`、`product`、`sweep`。同时比较不同模板与方案时，宿主可将有限组合编成 `fits` 或分成若干批次，不能把批量能力取消。现有默认组合限制 2,000，硬上限 100,000；新外壳不得绕过上限。

**归一化与 patch 顺序**：先用引擎入口的统一规则将 `base` 变成完整规范化结构，再按轴顺序应用 patch，逐项验证和计算；持久化的候选装备/弹药使用稳定实例 ID，宿主针对本次规范化结构映射为 JSON Pointer，避免数组移动或 `swap_type` 误换所有同型号件。`calc` 与 `batch` 对同一个展开后的规范化配置、相同引擎/SDE/价格条件，未裁剪的统计输出应一致；字段裁剪只影响返回投影，不影响求解。全批格式错误返回一个顶层错误；单条配置计算错误留在该条 `id/index`，不牵连其他条。batch 响应保留源顺序索引、计算总数、匹配数、排序/筛选后列表及 provenance。CLI JSONL 每一行仍可以是一份独立请求和响应，不和“一进一出”冲突。

### 5.4 宿主数据：作用域、组、导入导出

`FitPlan` 持有当前装配和可替代装备/弹药、命名方案、所需角色/技能与目标/场景引用。装备实例有稳定 ID，候选是包括 `type_id`、装填弹药及数量的完整选择，不能只按 `type_id` 去重。技能和会影响求解的假想敌、投射来源等对象在单方案导出时须被装进 `Package` 的依赖闭包，空工作区导入后无需依赖发送者私有库；原生 EFT/DNA 只投影实际选定装配，明确提示附加条件丢失。导出的角色技能和角色身份/ESI 令牌必须分开；令牌绝不进入交换包。

工作区默认条件、配置内设置、组公共条件、舰船实例覆盖、场景条件和临时试算，按**逐字段定义的覆盖规则**合成，区分“继承”“明确设值”“明确清除”，不作任意 JSON 深合并。建议优先级：格式默认→工作区→组→配置→实例→场景→临时覆盖；哪些字段允许在哪层出现及是替换、集合合并还是引擎游戏规则聚合，需要逐项契约化。宿主显示最终条件的来源；应用语言/布局属于 UI 设置，不应改变计算。一个工作区切换到另一个工作区须隔离配置、角色、场景、价格适配状态及旧请求的异步响应。价格注入、快照及覆盖层机制按现状保留，本阶段只处理宿主在切换环境时的缓存和会话适配。

`Group` 中**模板不等于实例**：攻击船模板可产生 D01…D10 十艘实例，后勤模板产生 L01，指挥模板产生 C01。关系记录稳定 `source_actor_id`、`target_actor_id` 或目标集合、关系种类、来源装备实例 ID、数量/距离/启用状态和场景条件。指挥覆盖与单目标修甲分配是不同语义：宿主必须校验同一单目标修甲器不会被完整分配给十个不同目标；不能仅靠“该舰为后勤”标签推断具体支援。当前 `Library.fleets` 的 `fit_id + command/member` 关系与单配置 `links.projected_fits` 不足以表达此图，新格式可重建而不必照搬。

**组求解流程**：宿主选定配置组及场景→解析每个实例的模板、技能、方案、环境→解析作用到该实例的有向关系→为每个实例生成不含宿主 ID 引用的 `FitSpec`→将多个实例放入**一个** `exfa/compute@1 operation=batch / batch.fits[]`→引擎计算→宿主按 `id/index` 映回实例、显示结果与异常。组层是编排与展示，而数值求解、批量筛选/排序/差值仍在引擎。组结果的聚合只能对语义上可相加的指标进行；不能把每舰 EHP 等数据无条件相加后称为舰队生存时间。

### 5.5 完整舰船投射：引擎所需的最小扩展

当前引擎支持 `projected[kind=fit]`，会构建来源船（包括技能/装配），但随后将其所有可投射活动装备作用到目标；无法只选修甲器 #1、#2 给 D01，把 #3 给 D02。只给引擎裸模块又丢来源船技能和整船修正。因此扩展**单次 `FitSpec` 的投射项**，不扩展引擎去读取 `Group`：

```json
{
  "kind": "fit",
  "fit": {
    "ship": { "type_id": 634 },
    "character": { "skills": { "default_level": 5 } },
    "modules": [
      { "id": "rep-1", "type_id": 11355, "state": "active" },
      { "id": "rep-2", "type_id": 11355, "state": "active" }
    ]
  },
  "select": { "module_ids": ["rep-1", "rep-2"] },
  "amount": 1,
  "distance_m": 8000
}
```

这是拟议结构示意（送葬者级与小型远程装甲维修器 I 的 `type_id` 已用引擎查询核对），并非当前引擎接受的投射格式；其他必需字段及装备合法性还须按新 schema 校验。`fit` 是一个自足的来源船计算输入，`module_ids` 仅选择该来源输入中的装备实例；宿主从关系图确定来源/目标和选择，引擎**先按完整来源配置求得修正属性，再仅将被选中的活动装备投射到目标**。来源整船的技能、装备、可用指挥条件须生效；需要补 Bench 用例确认来源接受加成时没有被当前嵌套解析器的深度截断吞掉。项目中可以按相同原则扩展无人机/舰载机选择。不得靠删除来源船其他装备来模拟选择，因为会改变来源自己的计算条件。多目标分配互斥与唯一性由宿主编排校验；引擎校验选择 ID 存在、来源装备合法及可投射。上限与来源递归深度必须定稿，引用循环明确报错，不静默省略。

第一阶段只承诺**静态、有确定外部输入的单舰/批量评估**。当多艘船互相传电、缺电影响支援、目标状态反馈给来源等形成耦合或时间闭环，现有独立 FitRequest 批算没有精确解；宿主可以保存关系但必须标注未精确支持，不能假定递归展开就算对。后续若需要耦合求解，另行定义输入物理模型，而不是隐式变成引擎持久化工作区。

### 5.6 跨仓实施顺序与验收门槛

1. **EXFA-Format：定契约。** 先写上述对象的精确 JSON Schema/TS/Rust 类型、语法糖逐字段默认值、作用域合成、交换包闭包与 golden fixtures；破坏性改造无需为 v1 新增兼容逻辑，但实施不得自动删除浏览器用户数据。
2. **EXFA-Engine：统一入口。** 让 CLI/RPC/WASM 消费同一种计算外壳，在**引擎内**规范化简写后调用现有单算/批量内核；保留 `fits`、变体、积、扫描、比较功能及图表能力。TS 宿主与 Rust 实现用同一 fixtures 校验。
3. **EXFA-Engine/Bench：来源选择。** 完整来源船解算＋按装备稳定 ID 选择投射，验证技能/增益/距离/叠加/数量以及非法引用；原生和 WASM 等价，并验证大批量下的内存/时间限制。当前 batch 原生并行、WASM 顺序的差异不改变输出次序。
4. **EXFA-App：编排。** Web/MCP 采用统一输入规范；将组的有向关系解析为 batch.fits，方案候选编译为 variants/product/sweep；导入有依赖预览和冲突处理；工作区切换隔离状态。不可再在 Web/MCP 为同一计算字段私自维护不同默认值。
5. **EXFA-Docs/Bench：门禁。** 固化计算与交换契约，纳入 JSON Schema、Format TS/Rust、Engine native/WASM、MCP、Web E2E 与现有 Bench 门禁；产物版本及 SDE 必须与已有发布链对齐。

必测验收样例：①只有 `ship+modules` 的机器人简写与显式全 5/明确活动状态在同一引擎数据上结果相等，显式全 0 不被覆盖；② batch 的每条未裁剪结果＝逐条单算，并保留 2,000/100,000 上限、差值/筛选/排序及逐条错误隔离；③ 单配置连同技能、两套弹药、假想敌、后勤、场景完整导出，在空工作区导入后计算条件不丢失；④ 1 后勤＋10 攻击＋1 指挥经一次 `batch.fits` 求解，按实例得到结果，单目标模块不被重复分配；⑤ 完整来源船经技能及指挥修正后仅把选中装备投射给目标；⑥ 循环、不存在的依赖、超限或不支持的动态反馈均明确报错/提示，不伪造精确结果；⑦ 原生、WASM、Web、MCP、格式 TS/Rust 共用契约并通过回归。

**实现时的首要禁令**：不要制造“简单/复杂两套引擎”、不要以 Web 循环 `calc` 替代引擎 `batch`、不要让 Engine 读 Workspace/Group、不要在宿主复制引擎 dogma 公式、不要把不支持的关系静默当作已计算、不要借本方案改造价格系统。

### 5.7 实施状态（2026-10-08 落地）

| 仓 | 版本/提交 | 内容 |
|---|---|---|
| EXFA-Format | **v1.1.0**（PR #1, `c191f3a`） | `exfa/compute@1`/`compute-result@1` 信封与类型；`Group`/`GroupActor`/`GroupRelation`（project/command、source_item_ids、targets、amount、distance_m、enabled）与 `compileGroup`（一次 batch；深克隆、issues 不抛异常）；`packageFit/packageGroup/mergePackage` 依赖闭包与 rename/replace/skip 合并；装备条目 `id?`；`Library.groups`；schema+fixtures+TS(32)/Rust(9) 测试 |
| EXFA-Engine | **v0.2.2**（PR #10, `5d487c2`） | `compute` RPC/`compute_json`/`exfa compute [FILE]`；FitSpec 简写引擎内归一化（技能缺省 5、模块状态按可激活性、无人机 active=quantity，递归进嵌套 fit；显式值优先，calc 记 `DEFAULTED` 调整）；batch base/fits[].fit 先归一化再展开；`projected[kind=fit].select` 三类白名单、未命中告警；CONTRACT.md `compute` 一节 |
| EXFA-Bench | PR #6, `1239972` | `ext/unit` 7 个 `unit_sel_*` 用例（单/双/空/未命中/drone_ids；手推导+引擎实测值；远修非线性 85.33≠170.65/2） |
| EXFA-App | Stage E（PR #6, `ceb0218`） | `@exfa/format` v1.1.0；装备 `id` 赋值与 `ensureItemIds` 回填；`ui/Groups.tsx` 组编辑器+逐实例结果表；工作区注册表+`ws` 标记分库+切换器+陈旧响应代际防护；`engineCompute`/`computeCalc`/`computeBatch` 能力探测+回退；单配置导出为 `exfa/package@1` 依赖闭包；内置 WASM→v0.2.2 |

已覆盖的验收项：①（compute.rs `shorthand_equals_explicit`、`explicit_values_always_win`）、②（`batch_fits_normalize_and_echo_ids`、variants 归一化基线）、⑤（`projected_fit_select_whitelist` + bench `unit_sel_*`）、⑦（Engine CI 原生+WASM 套件；Format TS/Rust 校验；App 70 vitest + CI web/mcp/mcp-engine）。③④⑥ 的端到端情形（完整包导入空工作区、1+10+1 组、动态闭环提示）由格式/App 单测覆盖编译与合并语义；UI 侧动态闭环仍以 issues/告警呈现，未实现耦合求解（符合第一阶段边界）。
