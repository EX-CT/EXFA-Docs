# 27 — v2 交接 / v2 handoff

状态：进行中（App Stage B 待验收）　·　更新：2026-10-07　·　前一份交接（迁移，已完成）见 [24](24-migration-handoff.md)

这份文档汇总 EXFA v2 大升级的进度，覆盖全部仓库，换环境（云端 ↔ 本地）后从这里接手。
App 各阶段的细节规格放在 EXFA-App 的 `docs/v2/`（设计方案 `design.md`、实现规格 `app-v2-spec.md`、格式规格 `format-v1-spec.md`、App 侧状态 `HANDOFF.md`）。
目前这些文件只在 Stage B 分支上，[EXFA-App#2](https://github.com/EX-CT/EXFA-App/pull/2) 合并后进入 main。

## 1. 总表

| 仓库 | 状态 | 最新 |
|---|---|---|
| [EXFA-Engine](https://github.com/EX-CT/EXFA-Engine) | v2 所需功能**已完成** | release `v0.2.0`（契约 1.5.0），main CI 绿 |
| [EXFA-Format](https://github.com/EX-CT/EXFA-Format) | v1 **已完成** | `v1.0.0`（schema + TS `@exfa/format` + Rust `exfa-format` + migrate/resolve + 目录导出布局） |
| [EXFA-Data](https://github.com/EX-CT/EXFA-Data) | 正常运行 | Latest = `sde-3579973-r7`；价格快照每天发布（`prices-jita44-*`） |
| [EXFA-Bench](https://github.com/EX-CT/EXFA-Bench) | CI 修复中（见 §3.2） | presets `presets-pyfa-3569502-r6` |
| [EXFA-App](https://github.com/EX-CT/EXFA-App) | Stage A 已合并；**Stage B 进行中**；Stage C/D 未开始；Pages 部署修复中（见 §3.1） | PR #1 已合并，PR #2 开着 |
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

### 3.1 App Pages 部署自 Stage A 合并后一直失败（修复中）

Stage A 合并后，`pages` workflow 的 bench 门禁报 `formats` 0/4779，部署没有执行，线上站点还是 Stage A 之前的版本。

原因在测试工具，产品代码没问题。Stage A 删掉了 TS 解析器，格式转换改走引擎 RPC。`apps/web/tools/browser-rpc.mjs` 把 `rpcRaw` 返回的 `{id,result}` 信封又包了一层 `result`，bench 读到的结构因此全部对不上。修复分支是 `devin/1791378812-fix-pages-formats-rpc`。合并后要确认 Pages 部署成功，并确认线上站点已是 Stage A。

### 3.2 EXFA-Bench bench-ci 红（修复中）

`nightly.yml` 还在从已删除的 `EX-CT/eve-sde-pipeline`、`EX-CT/eve-fit-docs` 下载数据。修复分支 `devin/1791378812-ci-exfa-data-sources` 把来源改成：
- dataset：EXFA-Data `sde-3569502-r7`；
- presets：EXFA-Bench `presets-pyfa-3569502-r6`；
- 功能清单：EXFA-Docs。

CI 转绿后关掉 Bench issue #1 "Nightly bench failure"。

### 3.3 App Stage B（[PR #2](https://github.com/EX-CT/EXFA-App/pull/2)，分支 `devin/1791318840-v2-market`）

代码已基本写完：紧凑市场、智能/替换/添加模式、单击预览与双击应用、对比条（变种、弹药、离线、角色），每一行都由引擎真算。

验收还缺：
1. 完整跑一遍 `tools/e2e.mjs`（上次被中断）。
2. 1920×1080 和 1600×900 下截图并目视检查。
3. 修完问题后合并。

具体清单见 App 的 `docs/v2/HANDOFF.md`。

### 3.4 App Stage D、Stage C（未开始，按这个顺序做）

- **Stage D**：配置库改为 EXFA-Format 存储，包括文件夹、可替代件、分支、历史、编队/指挥、投影配置；IndexedDB 迁移要保留旧的 `eve-fit-web*` 存储；手动导入导出支持单个 fit、文件夹和 zip。
- **Stage C**：场景编辑器加图表坞。多个配置乘多个目标画多条线，最多 12 条；支持引擎全部横轴、图例开关、十字准线、CSV/PNG 导出。

### 3.5 SDE 版本不一致（引擎侧，未做）

EXFA-Data 的 Latest 已是 `sde-3579973-r7`，但引擎 `sde.lock` 仍是 `sde-3569502-r7`。新 SDE 发布后没有触发引擎重建：可能是 `EXFA_DISPATCH_TOKEN` 没配，这一点还没核实。

后果：网页界面层用的是 3579973 的 dataset，WASM 引擎内置的是 3569502。两者之间新增的物品，界面上能看到，引擎却算不了。

修法：
1. 手动运行 EXFA-Engine 的 `sde-update` workflow，输入 `sde-3579973-r7`；或者本地改 `sde.lock`。
2. 重算 `ci/round1.sha256` 和 `ci/round1-base.sha256`，并把版本号升到 0.2.1 后再计算 hash。
3. 在 `packages/mcp` 用新的二进制跑一遍 `npm test`。
4. 发布 `v0.2.1`。

另一种做法：Pages 改为固定使用引擎 `release.json` 里的 `sde_tag`，保证界面层和引擎是同一版 SDE。

### 3.6 杂项

- EXFA-Engine 本地有一份旧 stash（`WIP on 1791223779-gitattributes-eol-lf`），是 crate 合并前的 Cargo.lock 改动。修复早已合并，可以丢弃。
- 不做的事：后台自动目录同步、桌面端外壳（Tauri）。目前只做手动导入导出；目录结构与将来的桌面端保持一致。

## 4. 接手须知（本地或云端）

- App 的测试要在 `apps/web` 目录里跑：`npx tsc -b && npx vitest run && npx vite build`。在仓库根目录跑 vitest 会扫到 `packages/mcp/dist`。
- 浏览器工具（`tools/smoke.mjs`、`e2e.mjs`、`screenshot.mjs`、`browser-rpc.mjs`）需要用 `CHROME` 环境变量指定 Chrome 路径，默认值是 `/usr/bin/google-chrome`。
- 本地预览：`npx vite build && npx vite preview --port 4173`，然后打开 `http://127.0.0.1:4173/EXFA-App/`。
- 本地构建 Engine 需要设置 `EXFA_DATASET=<dataset-3569502-r7.json.gz>`。不要对整个仓库跑 `cargo fmt`：仓库本身没有按 rustfmt 格式化，跑一次会改动几十个文件。
- 改了 MCP 工具描述后要跑 `npm run schemas`，CI 会比对 schema 快照。
- 发布 Engine 时先合并 PR，`fetch origin/main` 之后再在合并提交上打 tag。另外 App 的 `mcp-engine` CI 用的是最新的 Engine release，所以发版前先在 `packages/mcp` 用新的二进制跑一遍测试。
- Commit 结尾的署名：`Co-Authored-By: Devin AI <158243242+devin-ai-integration[bot]@users.noreply.github.com>`。
