> **本仓库已迁移。** canton-sim 现在位于
> [Rocky-DEX/Canton-Assurance-Layer](https://github.com/Rocky-DEX/Canton-Assurance-Layer)
> 的 `rust/sim-*`，文档在 `docs/simulator/`，Development Fund 提案在
> `docs/grant/simulator/`，托管控制台里也有对应页面。完整的提交历史已一并迁移。
> 这里不再维护；issue 与 pull request 请到新仓库提交。

<h1 align="center">canton-sim</h1>

<p align="center">
  Canton Network 交易的提交前模拟、失败原因解释与费用预估工具。
</p>

<p align="center">
  <a href="https://github.com/Rocky-DEX/Rocky.simulator/actions/workflows/ci.yml"><img src="https://github.com/Rocky-DEX/Rocky.simulator/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-blue.svg" alt="License"></a>
  <img src="https://img.shields.io/badge/rust-1.88%2B-orange.svg" alt="Rust 1.88+">
  <img src="https://img.shields.io/badge/canton-3.4%20Ledger%20API%20v2-informational.svg" alt="Canton 3.4">
</p>

<p align="center">
  <a href="README.md">English</a> | 简体中文
</p>

---

## 目录

- [概述](#概述)
- [功能特性](#功能特性)
- [工作原理](#工作原理)
- [快速开始](#快速开始)
- [使用说明](#使用说明)
  - [命令行](#命令行)
  - [HTTP API](#http-api)
  - [Rust crate](#rust-crate)
- [配置项](#配置项)
- [报告字段参考](#报告字段参考)
- [费用模型与精度](#费用模型与精度)
- [范围边界与相关工作](#范围边界与相关工作)
- [仓库结构](#仓库结构)
- [开发指南](#开发指南)
- [安全说明](#安全说明)
- [路线图](#路线图)
- [参与贡献](#参与贡献)
- [社区](#社区)
- [许可证](#许可证)

## 概述

canton-sim 在一条 Canton Ledger API 命令**提交之前**回答三个问题：

| 问题 | canton-sim 给出的答案 |
|---|---|
| 这笔交易会做什么？ | participant 将要提交的确切账本效果：每一个 create、消耗性或非消耗性 exercise、fetch 与 rollback，附带模板、choice、参数、signatory、stakeholder、informee、输入合约与有效期窗口。 |
| 它为什么会被拒绝？ | 结构化诊断：失败所在阶段、Canton 错误码及基金会官方的解释与解决建议、从 cause 中抽取的事实（断言消息、缺失的授权方、模板、choice、合约 ID）、所引用合约是已归档还是未知，以及具体的下一步操作。 |
| 它要花多少钱？ | 同步器流量字节数，按网络实时配置折算成 USD 与 Canton Coin；当命令涉及 Canton Coin 转账时，另给出 Splice 的转账费、创建费与锁持有费。 |

它建立在 Canton 已经提供的原语之上：Ledger API 交互式提交流程中的 **prepare** 步骤。`prepare` 会在 participant 上完整执行 Daml 解释与授权检查，返回「将会被提交」的交易与流量成本估算，但不会进入排序（sequencing）。canton-sim 只需要读权限，从不提交，从不持有密钥，所有数据都留在运营方自己的节点上。

## 功能特性

- **提交前模拟**在将要提交的那个 participant 上完成，因此与真实执行的一致性是天然保证的。token 能以读身份代表的任何一方都可以模拟。
- **账本效果预览**：解码 `PreparedTransaction` protobuf，在终端以树状呈现，或输出 JSON。
- **失败原因解释**：覆盖 228 个 Canton 错误码，目录文本从 Canton 3.4 源码重新生成，并对开发者最常遇到的错误提供逐码提示。可接受 JSON 错误体、HTTP 客户端错误串或 gRPC 风格日志行。
- **合约状态查询**：遇到合约类错误时区分「活跃」「已在 offset N 归档」「本 participant 从未见过」。
- **费用预估**：基于 participant 自身的成本估算与从 Scan 加载的 Splice 实时定价。全程定点小数运算。
- **三种使用形态**：带 `--json` 输出与 CI 退出码的命令行、会转发调用方 bearer token 的 HTTP 服务、可嵌入的 Rust crate。
- **无系统依赖**：内置的 Canton proto 用 `protox` 编译；构建与测试不需要 `protoc`、JVM 或 Canton 节点。

## 工作原理

```
SimulationRequest
   │  构造 JsPrepareSubmissionRequest（全新 commandId，启用 estimateTrafficCost）
   ▼
POST /v2/interactive-submission/prepare ─────────────────────────────────┐
   │ 2xx                                                                 │ 4xx / 5xx  JsCantonError
   ▼                                                                     ▼
解码 PreparedTransaction（prost）                                   解析 LedgerError
   ├─ 节点、informee、输入合约、有效期窗口                                  ├─ 阶段、目录条目、抽取事实、提示
   ├─ costEstimation ─▶ 流量报价（字节 → USD → CC）                       └─ CONTRACT_* / LOCAL_VERDICT_* ─▶ /v2/events/events-by-contract-id
   └─ Amulet 转账 ─▶ Splice 费用报价                                                                        ▼
   ▼                                                                                        outcome: would_fail + diagnosis
outcome: would_succeed + effects + traffic + fees
```

传输层失败会得到 `outcome: inconclusive`；工具绝不会报告一个它没有亲眼见到的成功。

**`prepare` 看不到的部分**会写在每份报告的 caveats 里：输入合约的并发争用、对手方 participant 的包 vetting 状态、以及排序时刻的时间检查，都只在确认阶段才会验证。详见 [docs/design/architecture.md](docs/design/architecture.md)。

## 快速开始

### 环境要求

| 要求 | 说明 |
|---|---|
| Rust 1.88 或更高 | `rustup update stable` |
| 提供 JSON Ledger API v2 的 Canton participant | 流量成本估算需要 Canton 3.4 及以上；3.3 可用，但没有成本估算 |
| bearer token | 需要对每个 `actAs` 方拥有*读*权限。不需要签名密钥。 |
| （可选）Splice Scan 地址 | 用于实时费用定价；不配置时使用带标注的参考默认值 |

### 安装

```bash
git clone git@github.com:Rocky-DEX/Rocky.simulator.git
cd Rocky.simulator
cargo install --path cli/canton-sim            # 安装 canton-sim
cargo install --path server/canton-sim-server  # 安装 canton-sim-server（可选）
```

或构建容器镜像：

```bash
docker build -t canton-sim .
```

### 运行第一次模拟

```bash
export CANTON_SIM_LEDGER_URL=https://validator.example/api/json-api
export CANTON_SIM_TOKEN_FILE=./token.jwt
export CANTON_SIM_SCAN_URL=https://scan.sv-1.global.canton.network.sync.global   # 可选

canton-sim simulate --act-as 'exchange::1220…' fixtures/perp-custody/platform-account-debit.json
```

命令会被接受时的输出（数值仅为示意）：

```
canton-sim — WOULD SUCCEED
participant: https://validator.example/api/json-api/v2   commandId: canton-sim-9f1b…   214 ms
actAs: exchange::1220…

Effects: 1 create, 1 exercise (1 consuming), 0 fetch, 0 rollback   on global-synchronizer::1220…
  Exercise PerpCustody:PlatformAccount.Debit on 00a1b2c3d4…f0e1   by exchange::1220…   [consuming]
    arg: {"delta":"25.0","chain_tx_id":"canton-sim-dry-run"}
    Create PerpCustody:PlatformAccount → 00c4d5e6f7…a2b3   signatories: exchange::1220…
Informees: exchange::1220…
Input contracts: 1 (1 consumed)
Prepared transaction: 1,912 bytes, hash 3f4deaf145a15cdc…, HASHING_SCHEME_VERSION_V2

Traffic: 4,512 bytes (request 4,212 + response 300) ≈ 0.27 USD ≈ 54 CC   [pricing: scan:https://scan…]

Caveats:
  - prepare covers Daml interpretation and authorization on this participant; contention on input
    contracts, package vetting on counterparties' participants and sequencer-time checks are only
    verified at confirmation.
```

同一条命令会被拒绝时的输出：

```
canton-sim — WOULD FAIL at Daml interpretation
UNHANDLED_EXCEPTION — Daml AssertionFailed: "Insufficient balance"
  An `assert`/`assertMsg` in the choice body evaluated to False with message "Insufficient balance".
  The transaction would be rejected before reaching the sequencer; nothing is charged.
  template a1b2c3d4…:PerpCustody:PlatformAccount · choice Debit · contracts 00a1b2…
  contract state: 00a1b2c3d4…f0e1 ACTIVE (a1b2c3d4…:PerpCustody:PlatformAccount)
  Canton: This error occurs when a user throws an error and does not catch it with try-catch.
  Next steps:
   - Contract 00a1b2c3d4…f0e1 is ACTIVE and visible to the acting parties, so the failure is not
     about its existence; check authorization and the choice arguments.
   - Read the assertion message: it names the business rule that rejected the input …
  retryable: no   category: InvalidGivenCurrentSystemStateOther   http: 400   correlationId: 9f1c2d
```

请把 [`fixtures/perp-custody/`](fixtures/perp-custody/README.md) 中的占位合约 ID 与 party ID 替换为你 participant 上的真实值。

## 使用说明

### 命令行

| 命令 | 用途 | 是否需要 participant |
|---|---|---|
| `canton-sim simulate [COMMANDS_JSON]` | 对命令做 dry-run，输出效果、诊断与费用。输入可以是一个 JSON Ledger API `Command` 对象、对象数组或完整的模拟请求对象；`-` 表示从标准输入读取（默认）。 | 需要 |
| `canton-sim explain <INPUT>` | 解释已有的错误：错误码、JSON 错误体、日志行，或 `-` 读标准输入。 | 不需要 |
| `canton-sim contract <CONTRACT_ID> --party <PARTY>…` | 报告合约在该 participant 上是活跃、已归档还是未知。 | 需要 |
| `canton-sim effects <INPUT>` | 离线解码 base64 的 `PreparedTransaction` 或 prepare 响应 JSON 为账本效果。 | 不需要 |
| `canton-sim fee` | 在没有 participant 的情况下为流量估算（`--request-bytes`、`--response-bytes`）和/或 Amulet 转账（`--transfer-cc`，可重复）定价。 | 不需要 |
| `canton-sim catalog [--filter TEXT]` | 列出错误目录。 | 不需要 |

`simulate` 的常用选项：

| 选项 | 含义 |
|---|---|
| `--act-as <PARTY>`（可重复，必填） | 以哪些方的身份解释命令 |
| `--read-as <PARTY>`（可重复） | 用于合约可见性的额外方 |
| `--synchronizer-id <ID>` | 指定同步器；省略则由 participant 自动路由 |
| `--command-id <ID>` | 命令 ID；省略时自动生成 `canton-sim-<uuid>` |
| `--no-lookup` | 失败时不做合约状态查询 |
| `--no-arguments` | 效果中不包含解码后的 create / choice 参数 |
| `--include-prepared` | JSON 输出中包含原始 base64 的 prepared transaction |
| `--json` | 以 JSON 输出完整报告 |
| `--fail-on-reject` | 结果不是 `would_succeed` 时以状态码 2 退出（CI 门禁） |
| `--scan`、`--traffic-usd-per-mb`、`--cc-usd` | 费用定价来源与覆盖值 |

退出码：`0` 成功；`1` 参数或传输错误；`2` 在 `--fail-on-reject` 下模拟结果为 `would_fail` 或 `inconclusive`。

示例：

```bash
canton-sim simulate --act-as 'alice::1220…' --json - < cmd.json | jq .outcome
canton-sim simulate --act-as 'alice::1220…' --fail-on-reject cmd.json
canton-sim explain 'JSON Ledger API POST … returned 400 Bad Request: {"code":"UNHANDLED_EXCEPTION", …}'
canton-sim explain DAML_AUTHORIZATION_ERROR
canton-sim contract 00a1b2… --party 'alice::1220…'
canton-sim effects prepare-response.json
canton-sim fee --request-bytes 4200 --response-bytes 300 --transfer-cc 10000
canton-sim catalog --filter VERDICT
```

### HTTP API

```bash
canton-sim-server --ledger $CANTON_SIM_LEDGER_URL --token-file token.jwt --scan $CANTON_SIM_SCAN_URL
```

| 方法与路径 | 请求体 | 返回 |
|---|---|---|
| `POST /v1/simulate` | `SimulationRequest` JSON（见[报告字段参考](#报告字段参考)） | `SimulationReport` JSON；带 `Accept: text/plain` 时返回纯文本 |
| `POST /v1/explain` | `{"error": "<错误码 | JSON 错误体 | 日志行>"}` | `Diagnosis` JSON |
| `GET /v1/catalog` | – | 全部目录条目 |
| `GET /v1/catalog/{code}` | – | 单条目录条目，未知时返回 `404` |
| `GET /v1/fee-schedule` | – | 当前使用的费用表及其来源 |
| `GET /healthz` | – | `ok` |

调用方可以携带自己的 `Authorization: Bearer …` 头，服务会原样转发给 participant，使模拟以调用方的权限运行（可用 `--forward-auth=false` 关闭）。

```bash
curl -s localhost:8787/v1/simulate -H 'content-type: application/json' \
  -d '{"act_as":["alice::1220…"],"commands":[{"ExerciseCommand":{"templateId":"#pkg:Mod:Tmpl","contractId":"00…","choice":"Accept","choiceArgument":{}}}]}' | jq .
curl -s localhost:8787/v1/explain -H 'content-type: application/json' -d '{"error":"CONTRACT_NOT_FOUND"}'
```

### Rust crate

| crate | 用途 |
|---|---|
| `canton-sim-core` | `Simulator`、`JsonLedgerClient`、`SimulationRequest` / `SimulationReport`、prepared transaction 解码、文本渲染 |
| `canton-sim-diagnose` | `LedgerError::from_text`、`diagnose`、错误目录。纯函数、同步。 |
| `canton-sim-fee` | `quote_traffic`、`quote_amulet_transfer`、`FeeSchedule::from_scan_json` |
| `canton-sim-proto` | 上述功能用到的 Canton Ledger API v2 消息的 prost 类型 |

```rust
use canton_sim_core::{JsonLedgerClient, SimulationRequest, Simulator, TokenSource};
use canton_sim_fee::FeeSchedule;
use std::sync::Arc;

let client = JsonLedgerClient::new("https://validator.example/api/json-api", TokenSource::File("token.jwt".into()))?;
let sim = Simulator::new(Arc::new(client), FeeSchedule::splice_defaults());
let report = sim.simulate(SimulationRequest::new(vec!["alice::1220…".into()], vec![command_json])).await;
println!("{}", canton_sim_core::render::report_to_text(&report));
```

## 配置项

所有配置既可以用命令行参数，也可以用环境变量；参数优先级高于环境变量。

| 变量 | 适用 | 默认值 | 说明 |
|---|---|---|---|
| `CANTON_SIM_LEDGER_URL` | CLI、server | –（必填） | JSON Ledger API 基础地址，带不带 `/v2` 后缀均可 |
| `CANTON_SIM_TOKEN` | CLI、server | – | bearer token 值。CLI 中与 `CANTON_SIM_TOKEN_FILE` 互斥。 |
| `CANTON_SIM_TOKEN_FILE` | CLI、server | – | 存放 bearer token 的文件；每次请求重新读取，token 轮换后无需重启 |
| `CANTON_SIM_USER_ID` | CLI、server | – | Ledger API 用户 ID，仅当 token 中不含用户 ID 时需要 |
| `CANTON_SIM_SCAN_URL` | CLI、server | – | Splice Scan 基础地址，用于实时费用定价。未设置时使用 Splice 参考值，且每份报价都会标注 `splice-defaults`。 |
| `CANTON_SIM_LISTEN` | server | `127.0.0.1:8787` | 监听地址 |
| `CANTON_SIM_FORWARD_AUTH` | server | `true` | 是否把调用方携带的 bearer token 转发给 participant |
| `RUST_LOG` | CLI、server | CLI 为 `warn`，server 为 `info` | 日志过滤；CLI 日志输出到 stderr |

模板见 [`.env.example`](.env.example)。

## 报告字段参考

`SimulationReport`（`simulate` 与 `POST /v1/simulate` 的 JSON 输出）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `outcome` | `would_succeed` \| `would_fail` \| `inconclusive` | `prepare` 的结果；`inconclusive` 表示无法连接或无法理解 participant 的响应 |
| `ledger.base_url` | string | 本次模拟所针对的 participant |
| `command_id` | string | `prepare` 使用的命令 ID；canton-sim 绝不会用它做真实提交 |
| `effects` | object | 成功时存在：`nodes[]`、`counts`、`informees[]`、`input_contracts[]`、`synchronizer_id`、`max_record_time`、`prepared_size_bytes`、`prepared_transaction_hash_hex` |
| `traffic` | object | participant 返回 `costEstimation` 时存在：`cost.{confirmation_request,confirmation_response,total}`、`usd`、`cc`、`pricing.source` |
| `amulet_fee` | object | 命令为 Canton Coin 转账时存在：`transfer_fee_usd`、`create_fee_usd`、`lock_holder_fee_usd`、`total_usd`、`total_cc` |
| `diagnosis` | object | 失败时存在：`code`、`phase`、`title`、`summary`、`explanation`、`resolution`、`extracted`、`hints[]`、`retryable`、`definite_answer`、`error` |
| `contract_states[]` | array | 错误中涉及合约的查询结果：`active`、`archived`、`unknown`、`lookup_failed` |
| `caveats[]` | array | 本次模拟无法验证的事项。务必阅读。 |
| `elapsed_ms`、`simulated_at` | number、string | 耗时与时间戳 |

`SimulationRequest` 字段：`act_as[]`（必填）、`commands[]`（必填，JSON Ledger API `Command` 对象）、`read_as[]`、`user_id`、`command_id`、`disclosed_contracts[]`、`synchronizer_id`、`package_id_selection_preference[]`、`min_ledger_time_rel_secs`、`expected_signatures[]`、`lookup_contracts`（默认 `true`）、`include_arguments`（默认 `true`）、`include_prepared_transaction`（默认 `false`）。完整示例见 [`fixtures/perp-custody/request-full.json`](fixtures/perp-custody/request-full.json)。

## 费用模型与精度

| 组成 | 来源 | 公式 |
|---|---|---|
| 流量字节数 | participant 的 `costEstimation`（Canton 3.4） | 确认请求 + 确认响应，由将要付费的节点自己计算 |
| 流量单价 | Scan 中的 `AmuletRules.transferConfig.extraTrafficPrice`（USD/MB）与最新开放轮次的 `amuletPrice`（USD/CC） | `usd = bytes / 1e6 × price`，`cc = usd / amuletPrice` |
| Canton Coin 转账费 | Scan 中的 `AmuletRules.transferConfig`（Splice `TransferConfigUSD`） | 对转给他方的每笔输出按阶梯费率计费，每个输出（含找零）一笔创建费，每个锁持有人一笔锁持有费 |

精度说明（每份报告中也会打印）：

- 流量报价是**上限**：未建模 participant 的免费基础流量额度；不包含请求放大与跨同步器 reassignment。
- 费用报价对其所依据的配置是精确的；配置会随治理投票和轮次变化，生产环境请务必用 `--scan` 定价。
- 不带 `--scan` 时使用 Splice 参考默认值，并标注 `splice-defaults`。

## 范围边界与相关工作

canton-sim 刻意保持窄范围，以便与其他 Canton 工具组合使用：

- **交易树可视化与 prepared/committed 对比**属于 Walnut 已获批的 `dpm trace`（canton-dev-fund #327），其中包含 `dpm trace prepare`。canton-sim 接受相同的命令 payload，并补上 `dpm trace` 没有的部分：基于目录的诊断与合约状态查询、费用与流量定价、面向 CI / bot / 钱包的机器可读门禁。
- **在脱离 participant 的环境里基于 ACS 快照重放执行**是 Tenderly 的路线（#481）。canton-sim 则让将要提交的 participant 在自己的状态上解释命令：一致性天然保证，数据不出节点，不需要密钥。
- **节点本地的取证查询**属于 Daml Shell（#752）。

完整的重叠分析见 [docs/proposal/landscape-2026-09.md](docs/proposal/landscape-2026-09.md)。

## 仓库结构

```
crates/canton-sim-proto      内置的 Canton 3.4 Ledger API proto → prost 类型（protox 编译）
crates/canton-sim-diagnose   错误目录（data/canton-error-catalog.json）、JsCantonError 解析、分类器
crates/canton-sim-fee        流量定价与 Splice Amulet 费用模型；Scan JSON 加载器
crates/canton-sim-core       Ledger API 客户端、PreparedTransaction → Effects、Simulator、文本渲染
cli/canton-sim               命令行工具
server/canton-sim-server     axum HTTP 服务
fixtures/perp-custody        针对 Rocky 托管合约包的命令样例
docs/design                  架构说明
docs/error-catalog.md        带来源标注的生成式错误目录
docs/proposal                Canton Development Fund 提案与竞品分析
docs/reference               Development Fund 模板与规则快照
```

## 开发指南

```bash
cargo build --all
cargo test --all                                # 31 个单元与集成测试，不需要 Canton 节点
cargo clippy --all-targets -- -D warnings       # CI 中任何 warning 都会失败
cargo fmt --all
```

集成测试使用进程内的 axum mock 模拟 JSON Ledger API（`crates/canton-sim-core/tests/simulate.rs`）。若要对真实 participant 测试，把 `CANTON_SIM_LEDGER_URL` 指向 LocalNet 或 DevNet validator 的 JSON API 并运行 fixtures。

为新的 Canton 版本重新生成错误目录：抽取脚本与来源规则见 [docs/design/architecture.md](docs/design/architecture.md#error-catalog-provenance)；每条目录条目都记录了来源文件。

## 安全说明

- canton-sim 从不调用 `execute` 或 `submit`，从不签名，从不持有密钥。缺陷的最坏后果只是一份错误的报告。
- token 每次请求时读取，从不写入日志。
- HTTP 服务会把调用方的 bearer token 原样转发给 participant。请把它部署在与 participant JSON API 相同的访问控制之后，或使用 `--forward-auth=false` 搭配只读的服务端 token。
- 报告中包含合约参数，请按与账本相同的保密级别处理。

漏洞报告方式见 [SECURITY.md](SECURITY.md)。

## 路线图

| 里程碑 | 状态 |
|---|---|
| 核心工具：CLI、HTTP 服务、crate、错误目录、费用定价、mock 测试的 CI | 已交付 |
| 在 LocalNet、DevNet 与 MainNet 上的实机验证；流量估算校准；带标注的拒绝语料与公开精度指标 | 计划中 |
| DPM 组件 `dpm fee` 与 `dpm explain`；TypeScript 客户端；对接 `dpm trace` 与 DPM Debug 的适配器 | 计划中 |
| 随 Canton 版本重新生成目录；采纳报告；移交 | 计划中 |

细节与验收标准见 [docs/proposal/submission/](docs/proposal/submission/)。

## 参与贡献

欢迎贡献。请先阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 了解工作流程、编码规范，以及如何新增目录提示或 fixture。每个 Pull Request 都必须通过 `cargo fmt`、`cargo clippy -- -D warnings` 与 `cargo test --all`。

## 社区

- 问题与功能请求：[GitHub Issues](https://github.com/Rocky-DEX/Rocky.simulator/issues)
- Rocky DEX：[Discord](https://discord.gg/Wu5VmFfjSn) · [@Rocky_exchange](https://x.com/Rocky_exchange)
- Canton Development Fund 讨论：grants-discuss@lists.sync.global

## 许可证

canton-sim 基于 [Apache License 2.0](LICENSE) 开源。
