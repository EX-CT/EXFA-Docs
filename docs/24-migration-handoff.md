# 24 — 迁移交接 / Migration handoff

状态：迁移进行中　·　更新：2026-10-05　·　总规划见 [00-architecture-plan.md](00-architecture-plan.md)

## 当前状态

| 仓库 | 状态 |
|---|---|
| [EXFA-Engine](https://github.com/EX-CT/EXFA-Engine)（公开） | 已迁移 `EX-CT/eve-dogma@d70e371`（**带历史**）。改名完成（crate `exfa-*`、二进制 `exfa`、引擎标识 `exfa-engine 0.1.0`、env `EXFA_*`）；`contract/` 收编契约与 schema；release.yml 就绪（v* tag + dry-run dispatch）。round-1 归一化后逐字节一致，证据：`D:\AIWorkspace\EXFA\_migration\engine\` |
| [EXFA-Bench](https://github.com/EX-CT/EXFA-Bench)（公开） | 干净迁移 `eve-dogma-bench@01e6724`（pending-1.11）。固定套件已拍平到 `suites/{graphs,cap,mutated,formats,round1}`（各含 `SOURCE`）；删除 bake-off 机器（bench.py / variants.yaml / evaluate.py / results-*）；Pyfa 工具集中到 `oracle/`（GPL-3.0，自己的 LICENSE）；基线 `baselines/exfa-engine.json` 已提高到当前成绩（commit `b2ae949`） |
| [EXFA-Data](https://github.com/EX-CT/EXFA-Data)（公开） | 干净迁移 `eve-sde-pipeline@83ba879` + `eve-market-prices@8d65963`。目录：`sde/`、`presets/`（Pyfa 预设已移除）、`aliases/`（114 条中英缩写表，待扩展）、`prices/`（`@ex-ct/exfa-prices`，CLI `exfa-prices`）。已发布 `sde-3569502-r5`（Latest），dataset sha256 与旧 release 逐字节一致。价格 release 用 `--latest=false`，取最新价格须按 `prices-` tag 前缀查找 |
| [EXFA-Docs](https://github.com/EX-CT/EXFA-Docs)（公开） | 旧 docs 01–23 原编号迁入 `docs/`（19 的 YAML 在 docs/19-pyfa-feature-inventory.yaml）；`LICENSING.md` 新政策（代码 LGPL-3.0+、`oracle/` GPL-3.0、文档 CC-BY-4.0、不用 Pyfa 数据表）；交接/进度/草稿在 `history/eve-fit-docs/` |
| `EX-CT/EXFA-App` | **尚未创建** |
| 9 个旧仓库 | 未归档；**archived 前需用户最终确认** |

## 未完成：EXFA-Engine CI 切换（被打断在中途）

已推：`5ef28d9`（CI/release 切到 EXFA-Data + EXFA-Bench、sde.lock + bench.lock、sde-update workflow）、`a6266ee`（workflow_call 输入修正）。

### 待办
1. **修红 CI**：run `37350249094` 失败于 Windows build：`eft_mutations_round_trip_exactly` —— fixtures 检出为 CRLF，导出期望 LF。修法二选一：`.gitattributes` 强制 fixtures `eol=lf`（对齐 EXFA-Bench 做法），或测试按行比较/归一化行尾。修后确认 native + wasm 套件在 EXFA-Bench 基线下全绿。
2. **发 v0.1.0**：CI 绿后打 tag `v0.1.0` → release.yml 正式发版（资产：4 平台原生包、`exfa-wasm32-wasip1.wasm`、`exfa_wasm.wasm`、`exfa_formats_wasm.wasm`、`release.json`、`SHA256SUMS`）。EXFA-App 要用这些产物。
3. **sde-update 验证**：workflow_dispatch 传 `sde-3569502-r5` 应直接退出（已验证过一次：`37350293803` ✓）。

## 待用户操作
- `EXFA_DISPATCH_TOKEN`：fine-grained PAT（owner EX-CT，repo 仅 EXFA-Engine，Contents: Read+Write），`gh secret set EXFA_DISPATCH_TOKEN -R EX-CT/EXFA-Data`。不配则新 SDE 不自动触发引擎重建（可 workflow_dispatch 手动跑 sde-update）。

## 下一步：EXFA-App 迁移（未开始）
- `EX-CT/EXFA-App` monorepo：`apps/web`（迁入 eve-fit-web）+ `packages/mcp`（迁入 eve-fit-mcp）；apps/desktop、apps/cloud 和其他 packages 后续按 00 规划新建。
- **不迁**：`engines/`、`lab-g1/`（525 个构建产物文件，~123 MB，进了 git 历史）；ts-worker（variant-d）与 wasm-j-worker（J）后端；Pyfa 预设功能（`src/data/pyfaPresets.*`、`tools/slim-presets-pyfa.mjs`，按许可政策）。
- **必须改**：价格"latest"解析——`/releases/latest` 会拿到 sde release，要改为列 releases 按 `prices-` tag 前缀取最新（MCP `src/price-file.ts`、Web pages.yml）；引擎 WASM 从 EXFA-Engine Release 下载（不再 CI 编译 Rust）；MCP 数据集索引暂时保留（等引擎查询 RPC 后再拆）；品牌字符串（"EXFA · 精密装配助理" / EXFA Web / About / i18n）。
- Pages 部署地址会变成 `ex-ct.github.io/EXFA-App/`；私有窗口已过，repo 公开可部署。web.json 基线（EXFA-Bench `baselines/web.json`）对应浏览器套件。

## 本地工作区
`D:\AIWorkspace\EXFA\`：旧 9 仓库克隆 + `EXFA-Engine`、`EXFA-Bench`、`EXFA-Data`、`EXFA-Docs` + `data/`（dataset-3569502-r5.json.gz）+ `_migration/`（迁移证据：round1 对比、套件对比、sha 比对）。本机工具链：cargo 1.96（wasm32-unknown-unknown、wasm32-wasip1 target 已装）、node 24、python 3.14（pyyaml + `Temp\pyshim` sitecustomize，本地跑 bench 脚本需要）。rustup 网络问题可用 `RUSTUP_DIST_SERVER=https://rsproxy.cn`。
