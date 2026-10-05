# EXFA-Docs

**EXFA · 精密装配助理** — Exactitude Fitting Assistant

EXFA 是 EX-CT 系列中的 EVE Online 配船产品：一个无状态、可复现、对 AI 友好的解算引擎，以及基于它的
EXFA Desktop、EXFA Web、EXFA Cloud 和 EXFA MCP。本仓库存放 EXFA 的架构、设计与规划文档。

EXFA is the EVE Online fitting product of the EX-CT series: a stateless, reproducible, AI-friendly fitting engine and
the products built on it (EXFA Desktop, EXFA Web, EXFA Cloud, EXFA MCP). This repository holds the design documents.

## 仓库 / Repositories

| 仓库 | 作用 |
|---|---|
| [EXFA-Engine](https://github.com/EX-CT/EXFA-Engine) | 解算引擎（Rust，原生 + WASM），契约与 Schema 在其 `contract/` |
| [EXFA-App](https://github.com/EX-CT/EXFA-App) | EXFA Desktop / Web / Cloud / MCP（TypeScript monorepo） |
| [EXFA-Data](https://github.com/EX-CT/EXFA-Data) | SDE 管线、预设、缩写表、价格快照工具 |
| [EXFA-Bench](https://github.com/EX-CT/EXFA-Bench) | 契约测试用例与 Pyfa 对照 |
| EXFA-Docs | 设计文档（本仓库） |

## 文档 / Documents

`00` 与 `LICENSING.md` 为 EXFA 新写；`01`–`23` 从 `EX-CT/eve-fit-docs@7fa6adb` 原样迁入，**保留原编号**（代码和其他仓库按编号引用），
文中的旧名称见下方"名称对照"。

状态：**现行** = 当前规范；**参考** = 仍有参考价值；**已取代** = 被其他文档取代；**历史** = 过程记录。

| # | 文档 | 状态 | 说明 |
|---|---|---|---|
| 00 | [architecture-plan](docs/00-architecture-plan.md) | 现行 | 命名、原则、分层架构、仓库规划、迁移步骤 |
| — | [LICENSING](LICENSING.md) | 现行 | 许可与数据来源政策 |
| 01 | [pyfa-analysis](docs/01-pyfa-analysis.md) | 参考 | Pyfa/eos 计算方式分析 |
| 02 | [dogma-engine-analysis](docs/02-dogma-engine-analysis.md) | 参考 | EVEShipFit dogma-engine 分析 |
| 03 | [feature-parity-checklist](docs/03-feature-parity-checklist.md) | 已取代 | 被 19 取代 |
| 04 | [sde-pipeline](docs/04-sde-pipeline.md) | 现行 | 数据包格式与管线设计 |
| 05 | [api-schema](docs/05-api-schema.md) | 已取代 | 被 EXFA-Engine `contract/` 取代 |
| 06 | [mcp-design](docs/06-mcp-design.md) | 现行 | MCP 工具/资源/提示设计 |
| 07 | [performance-plan](docs/07-performance-plan.md) | 历史 | |
| 08 | [roadmap](docs/08-roadmap.md) | 已取代 | 被 00 取代 |
| 09 | [engine-round-1-evaluation](docs/09-engine-round-1-evaluation.md) | 历史 | 引擎方案 A–K 评测 |
| 10 | [round-2-graphs-plan](docs/10-round-2-graphs-plan.md) | 历史 | |
| 11 | [import-export-parity](docs/11-import-export-parity.md) | 参考 | 导入导出格式对齐 |
| 12 | [repo-hygiene-audit](docs/12-repo-hygiene-audit.md) | 历史 | |
| 13 | [engine-round-2-evaluation](docs/13-engine-round-2-evaluation.md) | 历史 | 图表方案 G1–G4 评测 |
| 14 | [mutated-modules](docs/14-mutated-modules.md) | 现行 | 改装件与植入体/增效剂组合 |
| 15 | [fuzz-triage](docs/15-fuzz-triage.md) | 历史 | |
| 17 | [fuzzing-and-ci](docs/17-fuzzing-and-ci.md) | 参考 | 差分模糊测试与 CI 流程 |
| 18 | [f-vs-j](docs/18-f-vs-j.md) | 历史 | F 与 J 引擎对比 |
| 19 | [pyfa-feature-inventory](docs/19-pyfa-feature-inventory.md) | 现行 | Pyfa 功能清单（范围基线，YAML 为源，CI 使用） |
| 20 | [rust-architecture-plan](docs/20-rust-architecture-plan.md) | 参考 | crate 布局部分已由 00 更新 |
| 21 | [optimizer-and-character-input](docs/21-optimizer-and-character-input.md) | 现行 | 优化器接口与角色/技能输入格式 |
| 22 | [embedded-sde-and-prices](docs/22-embedded-sde-and-prices.md) | 现行（部分） | §2 运行时 SDE 注入（`.edp`）作废，见 00 原则 3；价格快照格式 §3–§5 现行 |
| 23 | [batch-api-and-prices](docs/23-batch-api-and-prices.md) | 现行 | 批量 API 与价格层 |

旧项目的交接文档、进度记录、草稿和工具存档于 [`history/eve-fit-docs/`](history/eve-fit-docs/)。
`tools/render_inventory.py` 由 19 的 YAML 生成其 Markdown 表格与 CSV。

## 名称对照 / Name mapping

| 旧名称（01–23 文中） | EXFA 名称 |
|---|---|
| `EX-CT/eve-dogma`（引擎 F）、crate `eve-dogma` | `EX-CT/EXFA-Engine`、crate `exfa-core` |
| 二进制 `eve-fit`、crate `eve-cli` | 二进制 `exfa`、crate `exfa-cli` |
| `eve-wasm` / `eve-fit-formats` / `eve-fit-formats-wasm` | `exfa-wasm` / `exfa-formats` / `exfa-formats-wasm` |
| `eve-sde` / `eve-fit-model` / `eve-capsim` / `eve-optimizer` / `eve-dogma-codegen` | `exfa-sde` / `exfa-model` / `exfa-capsim` / `exfa-optimizer` / `exfa-codegen` |
| `EVE_DOGMA_*` 环境变量 | `EXFA_*` |
| `eve-dogma-bench` | `EX-CT/EXFA-Bench` |
| `eve-sde-pipeline`、`eve-market-prices` | `EX-CT/EXFA-Data`（`sde/`、`prices/`） |
| `eve-fit-web`、`eve-fit-mcp` | `EX-CT/EXFA-App`（`apps/web`、`packages/mcp`） |
| `eve-fit-docs` | `EX-CT/EXFA-Docs` |
| `eve-dogma-rs`（方案 A）、`eve-dogma-lab`（方案 B–K、J） | 已归档，不迁移 |

## 许可 / License

文档采用 [CC-BY-4.0](LICENSE)。政策见 [LICENSING.md](LICENSING.md)。EVE Online 及相关商标归 CCP hf. 所有，EXFA 与 CCP 无关联。

Documentation is licensed under [CC-BY-4.0](LICENSE); see [LICENSING.md](LICENSING.md). EVE Online and all related
trademarks are the property of CCP hf.; EXFA is not affiliated with CCP.
