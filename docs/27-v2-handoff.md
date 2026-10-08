# 27 — v2 交接 / v2 handoff

状态：App v2 规格 A–D 全部合并上线；剩余收尾（README/文档刷新、终报）　·　更新：2026-10-08　·　前一份交接（迁移，已完成）见 [24](24-migration-handoff.md)

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
