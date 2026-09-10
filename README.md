> **This repository has moved.** canton-sim now lives in
> [Rocky-DEX/Canton-Assurance-Layer](https://github.com/Rocky-DEX/Canton-Assurance-Layer)
> under `rust/sim-*`, with its documentation at `docs/simulator/`, the
> Development Fund proposals at `docs/grant/simulator/` and a page in the
> hosted console. The full commit history was carried over. Nothing here is
> maintained any more; please open issues and pull requests there.

<h1 align="center">canton-sim</h1>

<p align="center">
  Pre-submit simulation, failure explanation and fee estimation for Canton Network transactions.
</p>

<p align="center">
  <a href="https://github.com/Rocky-DEX/Rocky.simulator/actions/workflows/ci.yml"><img src="https://github.com/Rocky-DEX/Rocky.simulator/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-blue.svg" alt="License"></a>
  <img src="https://img.shields.io/badge/rust-1.88%2B-orange.svg" alt="Rust 1.88+">
  <img src="https://img.shields.io/badge/canton-3.4%20Ledger%20API%20v2-informational.svg" alt="Canton 3.4">
</p>

<p align="center">
  English | <a href="README.zh-CN.md">简体中文</a>
</p>

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [How It Works](#how-it-works)
- [Quick Start](#quick-start)
- [Usage](#usage)
  - [CLI](#cli)
  - [HTTP API](#http-api)
  - [Rust crates](#rust-crates)
- [Configuration](#configuration)
- [Report Reference](#report-reference)
- [Fee Model and Accuracy](#fee-model-and-accuracy)
- [Scope and Related Work](#scope-and-related-work)
- [Repository Layout](#repository-layout)
- [Development](#development)
- [Security](#security)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Community](#community)
- [License](#license)

## Overview

canton-sim answers three questions about a Canton Ledger API command **before** it is submitted:

| Question | Answer canton-sim gives |
|---|---|
| What will this transaction do? | The exact ledger effects the participant would commit: every create, consuming or non-consuming exercise, fetch and rollback, with template, choice, arguments, signatories, stakeholders, informees, input contracts and validity window. |
| Why would it be rejected? | A structured diagnosis: the failing phase, the Canton error code with the Foundation's own explanation and resolution, the facts extracted from the cause (assertion message, missing authorizers, template, choice, contract ids), whether a referenced contract is archived or unknown, and concrete next steps. |
| What will it cost? | Synchronizer traffic in bytes, priced in USD and Canton Coin from live network configuration, plus Splice transfer, create and lock-holder fees when the command moves Canton Coin. |

It is built on a primitive Canton already ships: the interactive-submission **prepare** step of the Ledger API. `prepare` runs full Daml interpretation and authorization on the participant, returns the transaction that *would* be committed together with a traffic-cost estimate, and never sequences anything. canton-sim needs only read rights, never submits, never holds keys, and keeps all data on the operator's node.

## Features

- **Pre-submit simulation** on the participant that would submit, so parity with real execution is by construction. Works for any party the token can *read* as.
- **Ledger-effects preview** decoded from the `PreparedTransaction` protobuf, rendered as a tree in the terminal or as JSON.
- **Failure explanation** for 228 Canton error codes, with catalog text regenerated from the Canton 3.4 sources and per-code hints for the errors developers hit most. Accepts a JSON error body, an HTTP-client error string, or a gRPC-style log line.
- **Contract-state lookup** on contract errors: *active*, *archived at offset N*, or *never seen by this participant*.
- **Fee estimation** from the participant's own cost estimate and live Splice pricing loaded from Scan. Fixed-point decimal arithmetic throughout.
- **Three surfaces**: a CLI with `--json` output and a CI exit code, an HTTP service that forwards the caller's bearer token, and embeddable Rust crates.
- **No system dependencies**: vendored Canton protos are compiled with `protox`; no `protoc`, no JVM, no Canton node needed to build or test.

## How It Works

```
SimulationRequest
   │  build JsPrepareSubmissionRequest (fresh commandId, estimateTrafficCost enabled)
   ▼
POST /v2/interactive-submission/prepare ─────────────────────────────────┐
   │ 2xx                                                                 │ 4xx / 5xx  JsCantonError
   ▼                                                                     ▼
decode PreparedTransaction (prost)                               parse LedgerError
   ├─ nodes, informees, input contracts, validity window                 ├─ phase, catalog entry, extracted facts, hints
   ├─ costEstimation ─▶ traffic quote (bytes → USD → CC)                 └─ CONTRACT_* / LOCAL_VERDICT_* ─▶ /v2/events/events-by-contract-id
   └─ Amulet transfer ─▶ Splice fee quote                                                                       ▼
   ▼                                                                                          outcome: would_fail + diagnosis
outcome: would_succeed + effects + traffic + fees
```

Transport failures produce `outcome: inconclusive`; the tool never reports a success it has not seen.

**What `prepare` cannot see** is stated in every report's caveats: contention on input contracts, package vetting on counterparties' participants, and sequencer-time checks are only verified at confirmation. See [docs/design/architecture.md](docs/design/architecture.md).

## Quick Start

### Prerequisites

| Requirement | Notes |
|---|---|
| Rust 1.88 or newer | `rustup update stable` |
| A Canton participant with the JSON Ledger API v2 | Canton 3.4 or newer for traffic-cost estimation; 3.3 works without it |
| A bearer token | Needs *read* rights on every `actAs` party. No signing keys are required. |
| (Optional) A Splice Scan URL | For live fee pricing; otherwise labelled reference defaults are used |

### Install

```bash
git clone git@github.com:Rocky-DEX/Rocky.simulator.git
cd Rocky.simulator
cargo install --path cli/canton-sim            # installs `canton-sim`
cargo install --path server/canton-sim-server  # installs `canton-sim-server` (optional)
```

Or build a container image:

```bash
docker build -t canton-sim .
```

### Run your first simulation

```bash
export CANTON_SIM_LEDGER_URL=https://validator.example/api/json-api
export CANTON_SIM_TOKEN_FILE=./token.jwt
export CANTON_SIM_SCAN_URL=https://scan.sv-1.global.canton.network.sync.global   # optional

canton-sim simulate --act-as 'exchange::1220…' fixtures/perp-custody/platform-account-debit.json
```

Output when the command would be accepted (values illustrative):

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

Output when the same command would be rejected:

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

Replace the placeholder contract and party ids in [`fixtures/perp-custody/`](fixtures/perp-custody/README.md) with values from your participant.

## Usage

### CLI

| Command | Purpose | Needs a participant |
|---|---|---|
| `canton-sim simulate [COMMANDS_JSON]` | Dry-run a command; print effects, diagnosis and fees. Input is a JSON Ledger API `Command` object, an array of them, or a full simulation request object; `-` reads stdin (default). | Yes |
| `canton-sim explain <INPUT>` | Explain an error you already have: a code, a JSON error body, a log line, or `-` for stdin. | No |
| `canton-sim contract <CONTRACT_ID> --party <PARTY>…` | Report whether a contract is active, archived or unknown to the participant. | Yes |
| `canton-sim effects <INPUT>` | Decode a base64 `PreparedTransaction` or a prepare response JSON into ledger effects, offline. | No |
| `canton-sim fee` | Price a traffic estimate (`--request-bytes`, `--response-bytes`) and/or an Amulet transfer (`--transfer-cc`, repeatable) without a participant. | No |
| `canton-sim catalog [--filter TEXT]` | List the error catalog. | No |

Common options for `simulate`:

| Option | Meaning |
|---|---|
| `--act-as <PARTY>` (repeatable, required) | Parties on whose behalf the command is interpreted |
| `--read-as <PARTY>` (repeatable) | Additional parties for contract visibility |
| `--synchronizer-id <ID>` | Prescribe a synchronizer; omit to let the participant route |
| `--command-id <ID>` | Command id; a fresh `canton-sim-<uuid>` is generated when omitted |
| `--no-lookup` | Skip contract-state lookups on failure |
| `--no-arguments` | Omit decoded create/choice arguments from the effects |
| `--include-prepared` | Include the raw base64 prepared transaction in JSON output |
| `--json` | Emit the full report as JSON instead of text |
| `--fail-on-reject` | Exit with status 2 unless the outcome is `would_succeed` (CI gate) |
| `--scan`, `--traffic-usd-per-mb`, `--cc-usd` | Fee pricing source and overrides |

Exit codes: `0` success, `1` usage or transport error, `2` the simulation reported `would_fail` or `inconclusive` under `--fail-on-reject`.

Examples:

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

| Method and path | Body | Returns |
|---|---|---|
| `POST /v1/simulate` | `SimulationRequest` JSON (see [Report Reference](#report-reference)) | `SimulationReport` JSON, or plain text with `Accept: text/plain` |
| `POST /v1/explain` | `{"error": "<code | json body | log line>"}` | `Diagnosis` JSON |
| `GET /v1/catalog` | – | All catalog entries |
| `GET /v1/catalog/{code}` | – | One catalog entry, `404` if unknown |
| `GET /v1/fee-schedule` | – | The fee schedule in use and its source |
| `GET /healthz` | – | `ok` |

A caller may send its own `Authorization: Bearer …` header; it is forwarded to the participant unchanged so the simulation runs with the caller's rights (disable with `--forward-auth=false`).

```bash
curl -s localhost:8787/v1/simulate -H 'content-type: application/json' \
  -d '{"act_as":["alice::1220…"],"commands":[{"ExerciseCommand":{"templateId":"#pkg:Mod:Tmpl","contractId":"00…","choice":"Accept","choiceArgument":{}}}]}' | jq .
curl -s localhost:8787/v1/explain -H 'content-type: application/json' -d '{"error":"CONTRACT_NOT_FOUND"}'
```

### Rust crates

| Crate | Use it for |
|---|---|
| `canton-sim-core` | `Simulator`, `JsonLedgerClient`, `SimulationRequest` / `SimulationReport`, prepared-transaction decoding, text rendering |
| `canton-sim-diagnose` | `LedgerError::from_text`, `diagnose`, the error catalog. Pure and synchronous. |
| `canton-sim-fee` | `quote_traffic`, `quote_amulet_transfer`, `FeeSchedule::from_scan_json` |
| `canton-sim-proto` | prost types for the Canton Ledger API v2 messages used above |

```rust
use canton_sim_core::{JsonLedgerClient, SimulationRequest, Simulator, TokenSource};
use canton_sim_fee::FeeSchedule;
use std::sync::Arc;

let client = JsonLedgerClient::new("https://validator.example/api/json-api", TokenSource::File("token.jwt".into()))?;
let sim = Simulator::new(Arc::new(client), FeeSchedule::splice_defaults());
let report = sim.simulate(SimulationRequest::new(vec!["alice::1220…".into()], vec![command_json])).await;
println!("{}", canton_sim_core::render::report_to_text(&report));
```

## Configuration

All settings are available as flags and as environment variables. Flags take precedence.

| Variable | Used by | Default | Description |
|---|---|---|---|
| `CANTON_SIM_LEDGER_URL` | CLI, server | – (required) | JSON Ledger API base URL, with or without the `/v2` suffix |
| `CANTON_SIM_TOKEN` | CLI, server | – | Bearer token value. Mutually exclusive with `CANTON_SIM_TOKEN_FILE` in the CLI. |
| `CANTON_SIM_TOKEN_FILE` | CLI, server | – | File containing the bearer token; re-read on every request so rotated tokens are picked up |
| `CANTON_SIM_USER_ID` | CLI, server | – | Ledger API user id, needed only when the token does not carry one |
| `CANTON_SIM_SCAN_URL` | CLI, server | – | Splice Scan base URL for live fee pricing. When unset, Splice reference values are used and every quote is labelled `splice-defaults`. |
| `CANTON_SIM_LISTEN` | server | `127.0.0.1:8787` | Listen address |
| `CANTON_SIM_FORWARD_AUTH` | server | `true` | Forward a caller-supplied bearer token to the participant |
| `RUST_LOG` | CLI, server | `warn` (CLI), `info` (server) | Log filter; the CLI logs to stderr |

A template is provided in [`.env.example`](.env.example).

## Report Reference

`SimulationReport` (JSON output of `simulate` and `POST /v1/simulate`):

| Field | Type | Description |
|---|---|---|
| `outcome` | `would_succeed` \| `would_fail` \| `inconclusive` | Result of `prepare`; `inconclusive` means the participant could not be reached or understood |
| `ledger.base_url` | string | Participant the simulation ran against |
| `command_id` | string | Command id used for `prepare`; never reused for a real submission by canton-sim |
| `effects` | object | Present on success: `nodes[]`, `counts`, `informees[]`, `input_contracts[]`, `synchronizer_id`, `max_record_time`, `prepared_size_bytes`, `prepared_transaction_hash_hex` |
| `traffic` | object | Present when the participant returned `costEstimation`: `cost.{confirmation_request,confirmation_response,total}`, `usd`, `cc`, `pricing.source` |
| `amulet_fee` | object | Present when the command is a Canton Coin transfer: `transfer_fee_usd`, `create_fee_usd`, `lock_holder_fee_usd`, `total_usd`, `total_cc` |
| `diagnosis` | object | Present on failure: `code`, `phase`, `title`, `summary`, `explanation`, `resolution`, `extracted`, `hints[]`, `retryable`, `definite_answer`, `error` |
| `contract_states[]` | array | Lookup results for contracts named in the error: `active`, `archived`, `unknown`, `lookup_failed` |
| `caveats[]` | array | What the simulation could not verify. Always read them. |
| `elapsed_ms`, `simulated_at` | number, string | Timing |

`SimulationRequest` fields: `act_as[]` (required), `commands[]` (required; JSON Ledger API `Command` objects), `read_as[]`, `user_id`, `command_id`, `disclosed_contracts[]`, `synchronizer_id`, `package_id_selection_preference[]`, `min_ledger_time_rel_secs`, `expected_signatures[]`, `lookup_contracts` (default `true`), `include_arguments` (default `true`), `include_prepared_transaction` (default `false`). A complete example is in [`fixtures/perp-custody/request-full.json`](fixtures/perp-custody/request-full.json).

## Fee Model and Accuracy

| Component | Source | Formula |
|---|---|---|
| Traffic bytes | Participant `costEstimation` (Canton 3.4) | Confirmation request + confirmation response, as computed by the node that would pay |
| Traffic price | Scan `AmuletRules.transferConfig.extraTrafficPrice` (USD/MB) and the newest open round's `amuletPrice` (USD/CC) | `usd = bytes / 1e6 × price`, `cc = usd / amuletPrice` |
| Canton Coin transfer fees | Scan `AmuletRules.transferConfig` (Splice `TransferConfigUSD`) | Step-function transfer fee on outputs to other parties, create fee per output including change, lock-holder fee per holder |

Accuracy notes, also printed in every report:

- The traffic quote is an **upper bound**: the participant's free base-rate allowance is not modelled. Request amplification and reassignments are excluded.
- Fee quotes are exact for the configuration they were computed from; the configuration changes per governance vote and per round, so always price with `--scan` for production use.
- Without `--scan`, values are Splice reference defaults and are labelled `splice-defaults`.

## Scope and Related Work

canton-sim is deliberately narrow so that it composes with other Canton tooling:

- **Transaction visualisation and prepared-vs-committed diffs** belong to Walnut's approved `dpm trace` (canton-dev-fund #327), which includes `dpm trace prepare`. canton-sim accepts the same command payloads and adds what `dpm trace` does not: a catalog-backed diagnosis with contract-state lookup, fee and traffic pricing, and a machine-readable gate for CI, bots and wallets.
- **Off-participant re-execution on a hydrated ACS** is Tenderly's approach (#481). canton-sim instead asks the participant that would submit to interpret the command on its own state: parity by construction, no data leaves the node, no keys.
- **Node-local forensic querying** is Daml Shell (#752).

The full overlap analysis is in [docs/proposal/landscape-2026-09.md](docs/proposal/landscape-2026-09.md).

## Repository Layout

```
crates/canton-sim-proto      vendored Canton 3.4 Ledger API protos → prost types (compiled with protox)
crates/canton-sim-diagnose   error catalog (data/canton-error-catalog.json), JsCantonError parsing, classifier
crates/canton-sim-fee        traffic pricing and Splice Amulet fee model; Scan JSON loader
crates/canton-sim-core       Ledger API client, PreparedTransaction → Effects, Simulator, text renderer
cli/canton-sim               command-line interface
server/canton-sim-server     axum HTTP service
fixtures/perp-custody        command fixtures against Rocky's custody package
docs/design                  architecture
docs/error-catalog.md        generated catalog with provenance
docs/proposal                Canton Development Fund proposals and landscape analysis
docs/reference               snapshots of the Development Fund template and rules
```

## Development

```bash
cargo build --all
cargo test --all                                # 31 unit and integration tests, no Canton node needed
cargo clippy --all-targets -- -D warnings       # CI fails on warnings
cargo fmt --all
```

Integration tests run against an in-process axum mock of the JSON Ledger API (`crates/canton-sim-core/tests/simulate.rs`). To test against a real participant, point `CANTON_SIM_LEDGER_URL` at a LocalNet or DevNet validator's JSON API and run the fixtures.

Regenerating the error catalog for a new Canton release: the extraction script and provenance rules are described in [docs/design/architecture.md](docs/design/architecture.md#error-catalog-provenance); every entry records its source file.

## Security

- canton-sim never calls `execute` or `submit`, never signs, and never holds keys. The worst outcome of a defect is a wrong report.
- Tokens are read per request and never logged.
- The HTTP service forwards a caller's bearer token to the participant unchanged. Deploy it behind the same access control as the participant's JSON API, or run it with `--forward-auth=false` and a read-only server token.
- Reports contain contract arguments. Treat them with the same confidentiality as the ledger.

To report a vulnerability, see [SECURITY.md](SECURITY.md).

## Roadmap

| Milestone | Status |
|---|---|
| Core tool: CLI, HTTP service, crates, catalog, fee pricing, mock-tested CI | Delivered |
| Live validation on LocalNet, DevNet and MainNet; traffic-estimate calibration; labelled rejection corpus with published precision | Planned |
| DPM components `dpm fee` and `dpm explain`; TypeScript client; adapters for `dpm trace` and DPM Debug | Planned |
| Catalog regeneration per Canton release; adoption report; handover | Planned |

Details and acceptance criteria are in [docs/proposal/submission/](docs/proposal/submission/).

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) for the workflow, coding rules and how to add catalog hints or fixtures. Every pull request must pass `cargo fmt`, `cargo clippy -- -D warnings` and `cargo test --all`.

## Community

- Issues and feature requests: [GitHub Issues](https://github.com/Rocky-DEX/Rocky.simulator/issues)
- Rocky DEX: [Discord](https://discord.gg/Wu5VmFfjSn) · [@Rocky_exchange](https://x.com/Rocky_exchange)
- Canton Development Fund discussion: grants-discuss@lists.sync.global

## License

canton-sim is licensed under the [Apache License 2.0](LICENSE).
