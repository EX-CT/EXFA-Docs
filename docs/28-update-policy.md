# 28 · 更新策略：SDE / Pyfa / 引擎 / 价格（2026-11）

取代 24、27 中关于"`sde-update` 每 6 小时轮询并自动发版"的描述。

## 结论

| 对象 | 怎么更新 | 谁来做 |
|---|---|---|
| **价格** | EXFA-Data `prices-snapshot.yml` 每日定时生成 `prices-jita44-*` release；App `pages.yml` 每小时拉最新 | CI 自动（纯数据，失败不影响代码） |
| **SDE 数据集** | EXFA-Data `sde.yml` 每 6 小时自动构建 `sde-<build>-r<rev>` release（含 `CHANGELOG`/`diff` 报告）。**引擎不自动跟随** | CI 构建数据；引擎采纳由本地手动 |
| **引擎采纳新 SDE** | 本地运行 `python ci/sde_update.py`（EXFA-Engine） | 人 / agent，本地 |
| **Pyfa 参照** | 冻结的配对（Pyfa 版本 + 其 `eve.db` build）；Pyfa 发新版时手动换一对 | 人 / agent，本地（见下） |
| **发版** | PR 合并后打 `v<version>` tag，`release.yml` 构建并上传 | tag 由人推，构建 CI |

## 为什么不用 CI 自动更新

采纳新 SDE 要做判断：失败是 CCP 改了数据、Bench 里写死的夹具过期，还是引擎 bug。CI 做不了这个判断，只能开 issue。
`sde-update` 曾连续失败 86 次（原因是价格夹具写死了 SDE build），产生 80+ 个重复 issue，没人响应。

## 测试分两层

1. **逻辑层（Pyfa 作客观来源）**：引擎用与 Pyfa 数据对应的 SDE 对比 Pyfa 的输出（`EXFA-Bench/oracle/`，期望值已提交，CI 不运行 Pyfa）。
   不随 SDE 更新而变化；只在更换 Pyfa 配对时重新生成。已知的 Pyfa 数据滞后差异登记在 `oracle/data/pyfa-data-drift.{md,json}`。
2. **数据层（新 SDE）**：新 SDE 上跑全部 Bench 套件（"只能升不能降"的基线）、引擎单元测试、模糊测试；
   `EXFA-Data` 的 `CHANGELOG-<build>.md` / `diff-<build>.json` 说明这次 SDE 改了什么。数值变化由人审。

注意：Bench 全绿说明引擎在这些用例上与 Pyfa 当时的数据一致且无回归，**不等于**在最新 SDE 上已被 Pyfa 验证。

## 流程

### 采纳新 SDE（EXFA-Engine）

```
python ci/sde_update.py [--tag sde-<build>-r<rev>] [--bench-sha <sha>] [--gate-only]
```

下载数据集和 Pyfa 预设 → 固定 `_bench`（`bench.lock`）→ `cargo build --locked` + `cargo test --workspace`
→ `ci/run_suites.sh native`（跳过 round-1 sha，新数据合法改变字节）→ 嵌入最新价格快照 → 更新 `sde.lock`、patch 版本、
round-1 sha → 提交 `Update SDE to <tag>`。然后推分支开 PR（CI 在原生和 WASM 上再跑一遍），合并，打 tag 发版。
门禁失败时脚本退出并指向 `ci-results/native/nrg.txt`；逐条归类后再决定改 Bench 夹具、改引擎，或接受数据变化。

### 更换 Pyfa 配对（EXFA-Bench）

1. 取新版 Pyfa 及其 `eve.db`，运行 `tools/pyfa_drift.py --eve-db PATH --dataset dist/dataset-*.json.gz`，更新 drift 文件、删除已消除的条目。
2. 运行 `oracle/pyfa_oracle.py` 重新生成 `expected/`，与旧值逐条比对：是 Pyfa 行为变化，还是引擎要改。
3. 更新基线（`baselines/exfa-engine.json`）、presets-pyfa release 与 `PRESETS_TAG`，提交。

没有自动检测 Pyfa 新版本的任务；需要时可以加一个只提醒、不改代码的定时检查。

## CI 保留范围

- Engine / Bench / Data / App / Format 的 push 与 PR 检查。
- Bench `nightly.yml`：固定 SDE tag 上的套件与模糊自洽检查（不再浮动到最新 SDE）。
- Data：6 小时 SDE 数据集构建、每日价格快照、App 每小时部署（取最新价格）。
- 已移除：Engine `sde-update.yml`、Data 向引擎发的 `repository_dispatch`。
