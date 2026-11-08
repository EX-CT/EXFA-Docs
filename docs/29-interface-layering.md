# 29 · 接口分层与 `compute` 信封的移除（2026-11）

取代 27 §5.3 中关于 `exfa/compute@1` / `exfa/compute-result@1` 信封及"语法糖在引擎内归一化"的描述。27 的其余部分
（配置组、工作区、`select`、`exfa/package@1`）不变。

## 结论

1. **引擎没有 `compute` 入口了。** `exfa/compute@1` 信封、`compute` RPC 方法、`exfa compute` 命令、`compute-result@1` 全部删除
   （Engine、Format 1.2.0、App 同步）。引擎只保留按功能区分的入口：`calc`、`batch` 等，每个入口一种输入形状、一种输出形状。
2. **引擎不补默认值。** 省略 `character.skills.default_level` 即 0 级，省略模块 `state` 即 `online`，与 Pyfa 对照时一致。
3. **默认值、配置组等便利属于 Format 层**（`exfa/fit@1` 文档 → `resolve` → FitRequest；`compileGroup` → BatchRequest）。
   `compileGroup` 现在直接返回引擎的 BatchRequest：`{ batch_version: 1, fits: [{ id, label, fit }] }`，不再包信封（读 `request.fits`，不是 `request.batch.fits`）。
4. `select` 白名单与装备条目的 `id` 是普通 FitRequest 字段，保留，行为不变。

## 为什么

信封当初（27 §5.3）要解决的是 Web / Format / MCP 各自在 TypeScript 里补默认值、三处实现会不一致。它的做法是把补默认值收进引擎，
并用 `operation` 字段把"算一个配装"和"批量算"挤进同一个入口。这两点都不合适：

- 一个入口两种输入输出形状，靠字段切换，`check` / `place` / `fix` 这类新入口再塞进去只会让 schema 越来越松。
- 同一件事（算一个配装）出现了两套默认值：严格的 `calc`（Bench 对 Pyfa 用）和宽松的 `compute`。

## 接下来的方向（已商定，细节待定）

- **入口分四层**：A 核心计算（无状态，一入口一形状）、B 查询（无状态）、C 格式互转（`exfa-formats`）、D 会话/运维（有状态：`prices_load`、`sde_override`、`pyfa_data_load`）。
- **Format 只定义形状与"模式"，引擎只执行显式标记。** 严格模式下 Format 发出的请求字段写全、缺字段在 Format 层报错；
  自动模式下 Format 发出显式标记，由引擎（唯一有 SDE 的一侧）解释：
  - 技能：`default_level: 5`（全 5）；`default_level: 4` + `raise_required: true`（全 4，配装所需技能取所需等级，连前置链）。显式写出的等级永远优先。
  - 模块状态：`state: "auto"`（能激活则 active，否则 online）。
  标记的种类后续再补充；本文档不规定其最终名字。
- **验证规则只在引擎入口里执行**（不在 Format 里复制一份）。`violations` 将带 `severity`（`fatal` / `state` / `overload`），
  并新增 `check`（验）、`canfit`（放）、`fix`（修）入口；它们的请求与结果形状由 Format 定义。尚未实现。

## 迁移

- Engine：删除 `compute.rs` 与 `compute` RPC；`select` 测试改走 `calc`（`tests/select.rs`）。
- Format 1.2.0（**破坏性**）：删除 `computeRequest` / `compute_request`、`Compute*` 类型与 `schema/compute.schema.json`；`compileGroup` 返回值变化见上。
- App：网页直接调 `calc` / `batch`；MCP 的 `compute_fit` 等工具名不变（它们是 MCP 工具，不是引擎入口），内部改为调引擎的 `calc` / `batch`。
