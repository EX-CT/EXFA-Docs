# EXFA 许可与数据来源政策 / Licensing policy

状态：生效　·　2026-10-05　·　取代旧的 `eve-fit-docs/LICENSING.md`（存档于 `history/eve-fit-docs/LICENSING.md`）

## 1. 各仓库许可

| 仓库 | 许可 | 说明 |
|---|---|---|
| EXFA-Engine | LGPL-3.0-or-later | `LICENSE` + `LICENSE.GPL-3.0`（LGPL v3 是在 GPL v3 之上的附加许可） |
| EXFA-App | LGPL-3.0-or-later | Desktop / Web / Cloud / MCP |
| EXFA-Data | LGPL-3.0-or-later | 代码；生成的 EVE 数据附 `LICENSE.EVE`（见第 3 节） |
| EXFA-Bench | LGPL-3.0-or-later | 用例、运行器、计分；**`oracle/` 为 GPL-3.0-or-later**（导入 Pyfa，见第 2 节） |
| EXFA-Docs | CC-BY-4.0 | 本仓库全部文档 |

- 从旧 `eve-*` 仓库迁移来的代码保留原有版权声明；EX-CT 自有的原 MIT 代码在迁移时改为 LGPL-3.0-or-later。
- 第三方依赖须与 LGPL-3.0 兼容（MIT、BSD、Apache-2.0、LGPL 等）。引入 GPL 依赖须先评审，且只能出现在 EXFA-Bench `oracle/`。
- LGPL 允许任何前端（包括闭源前端）以进程边界、动态链接或 WASM 模块的方式调用引擎。

## 2. Pyfa

| Pyfa 部分 | 许可 |
|---|---|
| 应用（`gui/`、`service/`、`graphs/`、根目录） | GPL-3.0-or-later |
| `eos/`（引擎） | LGPL-2.0-or-later（文件头）；`eos/calc.py` 为 GPL-3.0 |

政策：
1. **Pyfa 只作为黑盒对照和行为参考。** 比较数值输出不是复制。任何 EXFA 仓库都不复制、不逐行翻译 Pyfa 代码。
2. **调用 Pyfa 的代码只存在于 EXFA-Bench `oracle/`**（GPL-3.0-or-later，单独的 `LICENSE`）。其他仓库只使用它产出的期望值数据。
3. **行为按"观察到的结果"实现。** 依据：CCP 数据、公开公式（EVE University wiki、CCP 开发者博客）、公开文件格式（EFT/DNA/XML/ESI），
   以及 Pyfa 黑盒输出。Pyfa 的特殊行为照着结果实现，并在引擎 `DESIGN.md` 的 Provenance 中记录，不复制代码。
4. **不使用 Pyfa 的数据表**：内置伤害模式、目标配置、缩写表（jargon）都不使用。
   - 预设只用由 SDE 推导的数据（EXFA-Data `presets/`）；SDE 无法推导的部分由 EXFA 依据公开游戏机制自行整理。
   - 缩写表由 EXFA 依据社区常识自行编写（EXFA-Data `aliases/`），中英双语，与 Pyfa 无关。

## 3. EVE 数据

- SDE、图标、市场数据 © CCP hf.，按 CCP 第三方开发者许可（Developer License Agreement）使用。
- 生成的数据（数据包、预设、价格快照）只作为 Release 资源发布，附 `LICENSE.EVE`，不提交进 git。
- 市场数据来自 CCP ESI；可选的 Fuzzwork 聚合数据须注明来源并控制请求量。
- EVE Online 及相关商标归 CCP hf. 所有；EXFA 与 CCP 无关联。所有产品的关于页须包含这一声明。

## 4. 产品命名与商标

- 产品名不包含 "Windows" 等第三方商标；平台写在发布包名称里，如 "EXFA Desktop for Windows"。

---

**English summary.** All EXFA code (Engine, App, Data, Bench) is LGPL-3.0-or-later; EXFA-Bench `oracle/` (which imports
Pyfa) is GPL-3.0-or-later; documentation is CC-BY-4.0. Pyfa is used only as a black-box oracle and behaviour reference;
no Pyfa code is copied or translated, and Pyfa's data tables (built-in damage patterns, target profiles, jargon) are not
used. EVE data © CCP hf., used under the CCP Developer License, shipped only as release assets with `LICENSE.EVE`.
