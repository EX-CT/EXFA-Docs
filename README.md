# EXFA-Docs

**EXFA · 精密装配助理** — Exactitude Fitting Assistant

EXFA 是 EX-CT 系列中的 EVE Online 配船产品：一个无状态、可复现、对 AI 友好的解算引擎，以及基于它的
EXFA Desktop、EXFA Web、EXFA Cloud 和 EXFA MCP。本仓库存放 EXFA 的架构、设计与规划文档。

EXFA is the EVE Online fitting product of the EX-CT series: a stateless, reproducible, AI-friendly fitting engine and
the products built on it (EXFA Desktop, EXFA Web, EXFA Cloud, EXFA MCP). This repository holds the design documents.

## 仓库 / Repositories

| 仓库 | 作用 |
|---|---|
| `EXFA-Engine` | 解算引擎（Rust，原生 + WASM） |
| `EXFA-App` | EXFA Desktop / Web / Cloud / MCP（TypeScript monorepo） |
| `EXFA-Data` | SDE 管线、预设、缩写表、价格快照工具 |
| `EXFA-Bench` | 契约测试用例与 Pyfa 对照 |
| `EXFA-Docs` | 设计文档（本仓库） |

## 文档 / Documents

| # | 文档 | 内容 |
|---|---|---|
| 00 | [architecture-plan](docs/00-architecture-plan.md) | 命名、设计原则、分层架构、仓库规划、许可政策、迁移步骤 |

## 许可 / License

文档采用 [CC-BY-4.0](LICENSE)。EVE Online 及相关商标归 CCP hf. 所有，EXFA 与 CCP 无关联。

Documentation is licensed under [CC-BY-4.0](LICENSE). EVE Online and all related trademarks are the property of
CCP hf.; EXFA is not affiliated with CCP.
