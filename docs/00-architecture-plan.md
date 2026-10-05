# 00 — EXFA 整体架构与仓库规划

状态：**草案，待审核**　·　日期：2026-10-05

EXFA · 精密装配助理（Exactitude Fitting Assistant）是 EX-CT 系列中的 EVE Online 配船产品。本文确定 EXFA 的
品牌命名、分层架构、仓库划分、许可政策和从旧 `eve-*` 仓库迁移的步骤。

---

## 1. 命名

### 1.1 产品

| 产品 | 英文名 | 中文名 | 说明 |
|---|---|---|---|
| 桌面应用 | **EXFA Desktop** | EXFA 桌面版 | 主力产品，Electron。发布包写作 "EXFA Desktop for Windows / for macOS" |
| 网页应用 | **EXFA Web** | EXFA 网页版 | 公开入口，引擎在浏览器内运行（WASM） |
| 云端解算服务 | **EXFA Cloud** | EXFA 云端 | 解算 HTTP API + 网络 MCP |
| MCP | **EXFA MCP** | — | 本地版随 Desktop，网络版随 Cloud |
| 命令行 | `exfa` | — | 引擎 CLI |

完整品牌名 **EXFA · 精密装配助理** 用于窗口标题、网页标题和关于页。产品名中不使用 "Windows"
（平台中立，且避免与微软 "Windows App" 产品及其商标规范冲突）。

### 1.2 仓库（GitHub 组织 `EX-CT`）

| 仓库 | 作用 |
|---|---|
| `EXFA-Engine` | 解算引擎（Rust） |
| `EXFA-App` | Desktop / Web / Cloud / MCP（TypeScript monorepo） |
| `EXFA-Data` | SDE 管线、预设、缩写表、价格快照工具 |
| `EXFA-Bench` | 契约测试用例 + Pyfa 对照 |
| `EXFA-Docs` | 本仓库：设计文档 |

EXFA 不单独建 `.github` 仓库：EX-CT 组织主页面向整个 EXCT 系列，只增加一段 EXFA 介绍。

### 1.3 包与标识

- Rust crate（小写）：`exfa-core`、`exfa-sde`、`exfa-codegen`、`exfa-capsim`、`exfa-model`、`exfa-formats`、
  `exfa-optimizer`、`exfa-cli`（二进制 `exfa`）、`exfa-wasm`、`exfa-formats-wasm`。
- npm（小写，scope `@ex-ct`）：`@ex-ct/exfa-engine-client`、`@ex-ct/exfa-ui`、`@ex-ct/exfa-fit-core`、
  `@ex-ct/exfa-platform`、`@ex-ct/exfa-mcp`、`@ex-ct/exfa-prices`。
- 桌面应用 ID：`com.ex-ct.exfa`。
- 引擎输出 `provenance.engine`：`exfa-engine <version>`。

---

## 2. 设计原则

1. **引擎是唯一的计算与数据权威。** 前端和 MCP 不自行解析 SDE；搜索、市场树、物品详情、缩写检索全部通过
   引擎 RPC 获取。同一个引擎版本 = 同一份数据，所有入口结果一致。
2. **引擎无状态、不联网、不存储。** 同一个 `FitRequest` 永远得到逐字节相同的 `FitStats`。联网（ESI、价格、
   更新下载）和存储（配置库、角色、设置）全部由宿主（Desktop / Web / Cloud）负责。
3. **SDE 编译进引擎。** CCP 每月只更新几次 SDE；有新 SDE 构建时自动重新编译、测试并发布引擎，不做每日发版，
   也不做运行时 SDE 解释器（旧 docs/22 的 `.edp` 运行时注入方案作废）。
4. **价格由宿主注入。** 引擎内嵌一份发布时的价格快照；宿主可注入新的价格文件或在请求中带价格表和覆盖价
   （优先级：请求覆盖价 > 请求价格表 > 注入文件 > 内嵌快照）。不同前端按需选择价格来源和更新频率。
5. **引擎以版本化产物发布。** 原生二进制（Windows / macOS / Linux）、WASM + TS 封装、formats WASM。
   Desktop、Web、Cloud 只消费 Release 产物，不在各自 CI 中从源码编译 Rust。
6. **功能完整与计算正确优先，覆盖只增不减，然后才是性能。** 以 Pyfa 为对照基准，Pyfa 只作为黑盒对照使用。

---

## 3. 分层架构

```
┌─ 宿主层（都基于引擎，可以有很多个）──────────────────────────────────────┐
│  EXFA Desktop (Electron, 主力)     EXFA Web (公开入口)     第三方前端          │
│  职责：ESI 登录与技能获取、配置保存、价格来源与注入、引擎更新                  │
└───────────────┬──────────────────────────┬──────────────────────────────────┘
                │ 本机：原生引擎子进程           │ 浏览器：WASM
┌─ 接入层 ──────┴──────────────────────────┴──────────────────────────────────┐
│  EXFA Cloud：引擎进程池 + HTTP API + 网络 MCP（Streamable HTTP）               │
│  本地 MCP：由 Desktop 托管，连本机引擎，并在本机暴露 HTTP / MCP 接口             │
└───────────────┬─────────────────────────────────────────────────────────────┘
┌─ 引擎层 ──────┴─────────────────────────────────────────────────────────────┐
│  EXFA-Engine：计算、图表、批量、价格层、搜索/物品/市场/缩写查询、导入导出格式   │
└───────────────┬─────────────────────────────────────────────────────────────┘
┌─ 数据层 ──────┴─────────────────────────────────────────────────────────────┐
│  EXFA-Data：SDE → 数据包 + 预设 + 缩写表；价格快照工具与每日公共快照            │
└─────────────────────────────────────────────────────────────────────────────┘
  旁路：EXFA-Bench（契约用例 + Pyfa 对照）　　EXFA-Docs（文档）
```

### 3.1 数据与发布流程

```
CCP 新 SDE 构建 ─► EXFA-Data 发布数据包 ─► 触发 EXFA-Engine：编译 + Bench 门禁 + Release
                                                     │ 原生二进制 / WASM / formats WASM
Jita 市场 ─► EXFA-Data 每日价格快照 ─┐                ▼
                                     └──────► EXFA-App：Desktop（引擎作为可单独更新的组件）
                                                         Web（WASM）
                                                         Cloud（原生引擎进程池）
```

---

## 4. 仓库详细规划

### 4.1 EXFA-Engine（LGPL-3.0-or-later）

来源：`EX-CT/eve-dogma` main（`d70e371`），**保留 git 历史**（LGPL 与"未使用 Pyfa 代码"的来源审计需要）。

```
crates/
  exfa-codegen        构建期：数据包 → 静态表 + 效果代码
  exfa-sde            编译进来的静态数据（物品、属性、分组、市场树、中英文名、缩写表…）
  exfa-model          FitRequest / FitStats 类型
  exfa-capsim         电容模拟
  exfa-core           引擎：计算、统计、图表、批量、价格层、查询 RPC（只接受结构化输入）
  exfa-formats        EFT / DNA / ESI / XML / multibuy / shipstats 等导入导出（不依赖 exfa-core）
  exfa-optimizer      配船优化器（保持现状，不作为重点）
  exfa-cli            二进制 exfa：calc、batch、serve-stdio、graph、search、type、version…
  exfa-wasm           浏览器用引擎（C ABI，后续加 TS 类型绑定）
  exfa-formats-wasm   浏览器用导入导出
contract/             契约文档 + JSON Schema（引擎拥有自己的接口定义）
```

迁移要点：
- crate、二进制和输出中的引擎标识改名，round-1 逐字节基线重新生成一次（预期内变更）。
- 新增/补齐查询 RPC：市场树、物品详情、缩写检索，替代旧网页和旧 MCP 各自的数据索引。
- 契约与 Schema 从旧的三处（`eve-dogma-rs/docs/contract.md`、`eve-dogma-bench/CONTRACT.md`、
  `eve-fit-docs/schema/`）合并到 `contract/`。
- 保留原有版权声明（LGPL 要求），新增 EXFA 品牌说明。
- C++ 备份实现 J 不迁移，只保留在归档的旧仓库中。

### 4.2 EXFA-App（LGPL-3.0-or-later）

来源：`eve-fit-web` + `eve-fit-mcp`，干净的初始提交（注明来源仓库与 commit）。

```
apps/
  desktop   EXFA Desktop：Electron；原生引擎子进程；配置存本地文件/SQLite；ESI 登录拿技能；
            价格拉取与注入；引擎组件更新；本地 MCP + 本机 HTTP 接口
  web       EXFA Web：Vite 静态站；浏览器 WASM；配置存 IndexedDB
  cloud     EXFA Cloud：引擎进程池 + HTTP API + 网络 MCP；鉴权与限流
packages/
  engine-client   统一 Engine 接口：wasm | 原生子进程 | http
  mcp             MCP 工具定义（只有一份，Desktop 与 Cloud 共用）
  fit-core        配置模型、指标、what-if、配置库（存储接口可替换）
  ui              React 组件（Desktop 与 Web 共用）
  platform        平台差异：存储 / 联网 / ESI / 文件
```

迁移要点：
- 去掉旧 Web 和旧 MCP 自建的 dataset 索引，改为调用引擎查询 RPC。
- 只保留主线引擎后端（WASM、原生、http）；旧的 TS 引擎（variant D）和 C++ 引擎（J）后端不迁移。
- 不迁移"可选加载 Pyfa 预设"功能（见第 5 节）。

### 4.3 EXFA-Data（LGPL-3.0-or-later；生成的数据附 CCP 数据许可）

来源：`eve-sde-pipeline` + `eve-market-prices`，干净的初始提交。

```
sde/       SDE 管线（Python 标准库）：CCP JSONL SDE → 数据包；新 SDE 构建时发布并触发引擎重新编译
presets/   预设生成：通用/弹药/NPC 伤害模式与目标配置、植入体套装、技能预设（全部由 SDE 推导）
aliases/   EXFA 中英双语缩写表（见 5.3）
prices/    价格快照工具与库（TypeScript，发布为 @ex-ct/exfa-prices）；每日公共快照
```

### 4.4 EXFA-Bench（LGPL-3.0-or-later；`oracle/` 为 GPL-3.0-or-later）

来源：`eve-dogma-bench` 的 `pending-1.11` 分支内容（它才是最新的，不是 main），干净的初始提交。

- 契约测试用例、期望值、计分与"不许倒退"门禁。
- `oracle/` 直接调用 Pyfa，必须保持 GPL-3.0，单独目录、单独许可文件；其他仓库只使用它生成的期望值数据。

### 4.5 EXFA-Docs（CC-BY-4.0）

来源：`eve-fit-docs` 整理后迁入，干净的初始提交。包括本规划、功能清单（原 docs/19）、接口与设计文档、交接文档。

---

## 5. 许可与数据来源政策

### 5.1 许可

| 内容 | 许可 |
|---|---|
| 所有代码（Engine、App、Data、Bench） | LGPL-3.0-or-later |
| Bench 的 `oracle/`（调用 Pyfa） | GPL-3.0-or-later（单独标注） |
| 文档（EXFA-Docs） | CC-BY-4.0 |
| EVE 数据（SDE、市场数据） | © CCP hf.，按 CCP 第三方开发者许可使用，数据产物附 `LICENSE.EVE` |

迁移代码时保留原有版权声明。EVE Online 及相关商标归 CCP hf. 所有，EXFA 与 CCP 无关联。

### 5.2 Pyfa

- Pyfa 只作为黑盒对照（Bench `oracle/`）和行为参考；任何仓库都不复制、不翻译 Pyfa 代码。
- 不使用 Pyfa 的预设数据（内置伤害模式、目标配置）和 Pyfa 的缩写表。预设只用 SDE 推导的数据，
  不足部分（如深渊按天气/等级的配置）由 EXFA 依据公开的游戏机制自行整理。

### 5.3 EXFA 缩写表

由 EXFA 依据社区常识自行编写的中英双语扩展版，与 Pyfa 无关：

| 能力 | 说明 |
|---|---|
| 中英双语 | 英文社区缩写 + 国服中文叫法 |
| 规则生成变体 | 尺寸（s/m/l/xl/c）、科技等级（t1/t2）、势力前缀（cn/rf/fn/in…）按规则组合 |
| 结构化匹配 | 匹配物品组、市场分组、物品 ID，而不是对物品名套正则 |
| 一词多义排序 | 按搜索场景（槽位、物品类别）排序候选 |
| 拼音辅助 | 可选：拼音首字母与全拼检索 |
| 测试兜底 | 每条缩写必须命中当前 SDE 中至少一个物品，否则 CI 失败 |
| 分发 | 维护在 EXFA-Data，随 SDE 编译进引擎，经引擎搜索接口统一提供 |

---

## 6. 迁移步骤

1. **EXFA-Docs**：本规划与命名审核。（当前步骤）
2. **EXFA-Engine**：从 `eve-dogma` main 带历史迁移；改名、许可声明、契约合并；重建 CI 与基线；发布首个版本产物。
3. **EXFA-Data**：合并 SDE 管线与价格工具；预设去掉 Pyfa 部分；新缩写表；新 SDE → 触发引擎发布。
4. **EXFA-Bench**：迁入 `pending-1.11` 内容，指向 EXFA-Engine 产物。
5. **EXFA-App**：先拆共享包和 `apps/web`，再迁 MCP 到 `packages/mcp`（去掉自建索引），然后 `apps/cloud`，
   最后 `apps/desktop`。
6. **收尾**：EX-CT 组织主页增加 EXFA 介绍；新仓库 CI 全部通过后，归档 9 个旧仓库。

### 6.1 归档范围

`eve-dogma`、`eve-dogma-rs`、`eve-dogma-lab`、`eve-dogma-bench`、`eve-sde-pipeline`、`eve-market-prices`、
`eve-fit-web`、`eve-fit-mcp`、`eve-fit-docs`。

不在范围内（不属于 EXFA，不动）：`.github`、`Vexor`、`Shuttle`、`eve-incursions`。

---

## 7. 待定事项

1. EXFA Cloud 的部署位置（服务器 / 云平台）与对外域名。
2. EXFA Web 的托管方式（GitHub Pages 或与 Cloud 同域）。
3. ESI 应用注册（Desktop 与 Web 的登录回调方式）。
4. Desktop 的自动更新渠道（应用本体与引擎组件分开更新）。
5. 各新仓库公开时间（目前 EXFA-Docs 为私有，审核通过后公开）。
