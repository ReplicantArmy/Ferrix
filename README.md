# Ferrix

Ferrix is a high-performance Rust trading system for Pump.fun to PumpSwap migration flow on Solana.

It is not just a "bot binary". It is a full stack made of:
- a live event-driven execution engine (`ferrix`),
- a telemetry and simulation toolchain (`telemetry_tool`, `walk_forward_split`, `backtest_logic`),
- and an always-on autotuning engine (`run_deep_evolution.sh` + `optimize_params`) that continuously improves `best_strategy_params.json` while production trading stays online.

The design goal is simple: fast detection, strict safety, resilient execution, and continuous model adaptation.

## 1. System Overview

At a high level, Ferrix runs two loops:

- Live trading loop:
  - Detect migration events in real time.
  - Analyze token safety and tradability.
  - Enter and manage positions through actor-based state management.
  - Persist trades and portfolio state to SQLite.

- Optimization loop:
  - Transform raw telemetry into deterministic ground-truth simulation data.
  - Perform walk-forward + holdout evaluation at massive parallel scale.
  - Promote new champion params only when blended out-of-sample quality improves.
  - Signal production to restart only when safe.

This separation of concerns is the core strength of the repo.

## 2. Repository Map

Core runtime:
- `src/main.rs` - process bootstrap, logging, metrics endpoint, DB init, app entry.
- `src/app/mod.rs` - command routing and full runtime pipeline wiring.
- `src/migration_watcher.rs` - migration event ingestion and pre-warm logic.
- `src/migrated_checker.rs` - concurrent honeypot + contract safety checks.
- `src/actors/position_manager/` - actor-owned position state, entry observers, sell dispatch.
- `src/price/` and `src/laserstream_client/` - stream clients and price/vault state plumbing.
- `src/services/trading.rs` - trade signal processor, buy/sell execution flow.
- `src/persistence/` - SQLite schema + persistence tasks + caches.
- `src/metrics.rs` - Prometheus metrics catalog.

Research/backtesting/tuning stack:
- `src/bin/telemetry_tool.rs` - converts raw telemetry into clean ground-truth lifecycle data.
- `src/bin/walk_forward_split.rs` - train/WF/holdout split generation.
- `src/bin/optimize_params.rs` - high-throughput evolutionary optimizer.
- `src/bin/backtest_logic.rs` - CLI wrapper around shared simulation engine.
- `src/sim.rs` - hot-path simulator used by optimizer and backtest CLI.
- `run_deep_evolution.sh` - continuous autotuning orchestrator.
- `auto_restart.sh` - safe rollout manager for champion updates.

## 3. Live Runtime Architecture

### 3.1 Entry and Bootstrap

`src/main.rs` does the production boot sequence:
- sets up Prometheus recorder and serves `/metrics` (default `127.0.0.1:9000`),
- initializes dual JSON logging (stdout + rolling file logs),
- loads strategy params from `best_strategy_params.json`,
- loads `known_wallets.json` and starts wallet-file watcher,
- initializes `trades.db`, then calls `app::run(...)`.

### 3.2 Command Modes

CLI subcommands are defined in `src/config.rs`:
- `token --auto-trade` - token creation watcher mode.
- `migration [--dummy-event] [--checks-off]` - migration watcher mode.
- `autobuysell [--checks-off]` - analysis-driven auto execution mode.
- `test-analysis --token-list-path <file>` - batch risk analysis over token list.

### 3.3 Event Pipeline (Migration/Autobuysell)

The main production path in `src/app/mod.rs` is:

1. `MigrationWatcherManager`
- Subscribes to LaserStream transaction feed.
- Extracts migration events.
- Emits telemetry migration events.
- Pre-warms vault state cache for execution realism.

2. `MigratedCheckerManager`
- Runs analysis tasks per event in independent async tasks.
- Uses `tokio::join!` to run:
  - honeypot check,
  - contract safety check.
- For passing events, emits verified trade signal.

3. `PositionManagerActor`
- Owns canonical in-memory position state (single-writer actor model).
- Manages observer states for entry gating.
- Evaluates sell conditions from unified tick snapshots.
- Dispatches `SellOrder` commands.

4. `TradeSignalProcessor` (`src/services/trading.rs`)
- Receives verified events.
- Performs final circuit-breaker gate.
- Executes buys through `PumpSwapTradingClient`.
- Sends `PositionCommand` updates back to actor.

5. `SellOrderProcessor`
- Executes sell transactions.
- Handles slippage retry paths and creator-cache refresh retries.
- Confirms and computes realized outputs.
- Sends `SellConfirmed` or `SellFailed` to actor.

### 3.4 Feature/Tick Engine

`src/actors/position_manager/feature_engine.rs` builds derived microstructure features (RVR, volume windows, momentum, drawdown, red streaks, whale dump flags, creator/exit-liquidity signals), then emits `TickSnapshot` for decision logic.

Sell logic in `src/actors/position_manager/logic.rs` is phase-aware (Chaos/Discovery/Trending/Mature) with adaptive rules for:
- shock-drop exits,
- drawdown caps by run-up tier,
- momentum reversal,
- churn/dud exits,
- adaptive trailing,
- creator/exit-liquidity preemptive exits,
- panic flow exits.

### 3.5 Persistence and Recovery

`trades.db` is initialized and maintained via `src/persistence/`.

Major tables include:
- `trades` - open/closed trade lifecycle records.
- `portfolio_summary` - aggregate portfolio counters and PnL rollups.
- `circuit_breaker_events` - risk-trigger history.
- `sniper_cache` and `creator_reputation_cache` - analysis acceleration.
- `test_analysis` - batch analysis output.
- `address_lookup_tables` - ALT cache.
- metadata and transaction cache tables for data-fetch acceleration.

On restart, open positions are loaded and re-hydrated into actor state.

### 3.6 Observability and Safe Rollout Hooks

Ferrix continuously updates:
- Prometheus metrics via `src/metrics.rs`,
- runtime status file `/dev/shm/ferrix_status.json` via actor scheduled metrics:
  - `open_positions`,
  - `observers`,
  - `active_telemetry`.

That status file is consumed by `auto_restart.sh` so model updates only roll out during safe windows.

## 4. Telemetry -> Ground Truth -> Fold Data

This is the input pipeline for autotuning.

### 4.1 `telemetry_tool`

`src/bin/telemetry_tool.rs`:
- parses raw telemetry event stream (`migration`, `flow`, `tick`),
- groups by mint,
- repairs missing timestamps/age fields for legacy entries,
- reconstructs 1s flow stats and net-flow signals,
- reconstructs pool lifecycle and writes deterministic lifecycle entries to:
  - `tel_out/ground_truth_analysis.jsonl`.

### 4.2 `walk_forward_split`

`src/bin/walk_forward_split.rs`:
- sorts lifecycle records chronologically by slot,
- creates `permanent_holdout.jsonl` from the newest 5 percent of records,
- creates `train_set.jsonl` from the earlier 95 percent,
- generates walk-forward rounds:
  - `wf_round_<n>_train.jsonl`,
  - `wf_round_<n>_test.jsonl`.

Default deep-evo settings use sliding windows with embargo to reduce leakage.

## 5. Deep Autotuning Pipeline (`run_deep_evolution.sh`)

This script is the always-on orchestration layer for model evolution.

It is currently labeled in-script as Data-Driven Nanocap Evolution v9.3.

### 5.1 What It Controls

- Champion file:
  - `best_strategy_params.json`.
- Exploration strategy:
  - normal/explore/chaos modes based on stagnation.
- Data refresh per generation:
  - telemetry reprocessing and WF split regeneration.
- Dynamic throughput policy:
  - auto-calculated minimum entry rate from live migration cadence.
- Promotion policy:
  - blended out-of-sample scoring and holdout gates.
- Production rollout signal:
  - touches `/dev/shm/ferrix_pending_update` when champion is accepted.

### 5.2 Prerequisites

`run_deep_evolution.sh` expects binaries in repository root:
- `./optimize_params`
- `./telemetry_tool`
- `./walk_forward_split`

Build and place them first:

```bash
cargo build --release --bin optimize_params --bin telemetry_tool --bin walk_forward_split --bin backtest_logic
cp target/release/optimize_params .
cp target/release/telemetry_tool .
cp target/release/walk_forward_split .
cp target/release/backtest_logic .
chmod +x optimize_params telemetry_tool walk_forward_split backtest_logic
```

The script also sources `.env_PROD` when present (for values such as `SIM_BUY_AMOUNT_SOL`, slippage thresholds, and other execution realism knobs).

### 5.3 Per-Generation Loop

Each generation does:

1. Choose exploration regime
- `NORMAL`, `EXPLORE`, or `CHAOS` from stagnation counters.
- During first `FORCED_CHAOS_GENS` generations it forces wide exploration.

2. Rotate sweep stage groups
- Chooses one 3-stage `multi-sweep` sequence (entry/trail/dd/phase/churn/etc).

3. Refresh datasets
- `./telemetry_tool telemetry.jsonl`
- `./walk_forward_split --min-train-trades 10 --test-window-trades 450 --embargo-trades 40 --n-rounds 5 --sliding`

4. Compute dynamic entry-rate target
- Tail-reads recent migrations from `telemetry.jsonl` (Python in-script).
- Derives migrations/day via median interval.
- Sets target entry rate to satisfy `TARGET_TRADES_PER_DAY` (default 45/day), with clamps and smoothing.

5. Run optimizer
- Calls `optimize_params` with:
  - large random sample budget (`1.5M` to `3.0M`),
  - `--use-cma` (CMA-ES sampling),
  - realistic execution model (`latency`, `slippage`, fees),
  - WF rounds and holdout blend.

6. Parse and evaluate results
- Reads optimizer output for current champion CV/blended score.
- Detects new champion via `NEW CHAMPION CONFIRMED` marker.
- Tracks best-ever blended score and degradation flags.

7. Fallback search when stuck
- On prolonged stagnation, runs a "hail mary" exploration pass with wider jitter focused on entry/trailing rules.

8. Validate and promote
- Runs minimal sanity validation on new params.
- On pass:
  - keeps champion,
  - appends journal diff,
  - writes update flag `/dev/shm/ferrix_pending_update`.
- On fail:
  - restores backup champion.

### 5.4 Scoring/Promotion Logic Under the Hood

`optimize_params` combines:
- walk-forward CV score,
- holdout score,
- blended score (default 40 percent CV + 60 percent holdout in script via `HOLDOUT_WEIGHT=0.60`).

Promotion is not based on raw in-sample score only. A candidate must:
- pass hard safety/tail-risk gates,
- beat champion beyond minimum improvement threshold,
- pass holdout degradation constraints,
- beat champion on blended score.

This is why the pipeline is robust to overfitting drift.

### 5.5 Non-Blocking Production Update Design

`run_deep_evolution.sh` never directly restarts the live process.

It only drops a signal flag.

`auto_restart.sh` independently:
- watches for update flag,
- polls `/dev/shm/ferrix_status.json`,
- waits until `open_positions + observers + active_telemetry == 0` (or stale status timeout),
- performs service restart,
- clears flag.

This is a strong production-safe rollout model: optimization and execution are decoupled but coordinated.

## 6. Why This Pipeline Is Super Awesome and Can Hit 70,000+ Backtests/sec

The speed does not come from one trick. It comes from stack-wide design choices:

1. In-memory data cache for folds
- `optimize_params` loads all WF/holdout datasets once into RAM.
- No per-candidate disk reads inside the evaluation loop.

2. Zero-I/O hot path
- Candidate evaluation loop is pure compute (`par_iter` over param sets).
- Writes happen only after winner selection.

3. Fast data decode path
- prefers `.rkyv` binary datasets if present,
- otherwise uses `simd-json` with serde fallback.

4. Highly parallel execution
- Rayon parallel candidate scoring.
- Script drives high thread counts (`RAYON_NUM_THREADS`, default 90).

5. Hot simulation engine (`src/sim.rs`)
- precomputed phase thresholds,
- enum-based exit reasons (no per-trade string allocation in hot path),
- tight run loop tuned for repeated invocation.

6. Memory and IO locality
- `tel_out` is linked into `/dev/shm` for RAM-disk speed in tuning runs.

7. Search efficiency
- sensitivity cache guides jitter width,
- CMA-ES + bounded wide exploration reduces wasted search.

8. Compiler/runtime tuning
- release profile uses LTO, single codegen unit, stripped binary, overflow checks off,
- optional CPU-specific and PGO build scripts are provided (`build_release.sh`, `pgo_build.sh`).

### Throughput interpretation

In this repo, "backtests/sec" in evolution context is most meaningfully interpreted as fold-level simulation evaluations per second (candidate-fold evaluations), not full end-to-end generation cycles.

With script defaults at full exploration budget:
- up to 3,000,000 candidates,
- 5 WF rounds,
- 3 sweep stages per optimizer call,
- around 45,000,000 fold evaluations per generation.

On tuned high-core hardware, this architecture is engineered for the 70,000+ fold-backtests/sec class.

## 7. Build and Run

### 7.1 Standard release build

```bash
cargo build --release
```

### 7.2 Recommended validation before shipping patches

```bash
cargo fmt -- --check
cargo clippy --features backtest -- -D warnings
cargo test --features backtest
```

### 7.3 Runtime examples

Migration mode:

```bash
RUST_LOG=info cargo run --release --bin ferrix -- migration
```

Migration mode with checks disabled:

```bash
RUST_LOG=info cargo run --release --bin ferrix -- migration --checks-off
```

Autobuysell mode:

```bash
RUST_LOG=info cargo run --release --bin ferrix -- autobuysell
```

Batch test-analysis mode:

```bash
RUST_LOG=info cargo run --release --bin ferrix -- test-analysis --token-list-path ./test_tokens.txt
```

### 7.4 Continuous optimization workflow

Terminal 1 (optimizer):

```bash
./run_deep_evolution.sh
```

Terminal 2 (safe rollout manager):

```bash
./auto_restart.sh
```

## 8. Important Artifacts and Files

Runtime artifacts:
- `trades.db` - live and analysis persistence.
- `telemetry.jsonl` - continuous event/tick/flow telemetry stream.
- `logs/ferrix.log*` - rolling JSON logs.

Optimization artifacts:
- `best_strategy_params.json` - current champion params consumed by runtime.
- `deep_evo_archives/<timestamp>_DATADRIVEN/` - archived winners.
- `sensitivity_cache.json` - optimizer sensitivity cache.
- `/tmp/ferrix_*` files - blended/cv state and dynamic rate state.
- `/dev/shm/ferrix_pending_update` - rollout signal.
- `/dev/shm/ferrix_status.json` - live safety status for restart manager.

## 9. Why Ferrix Is Well-Designed and Ready for the Next Level

Ferrix is already built with the right production primitives:
- actor-owned trading state to avoid race-condition chaos,
- explicit event channels and clear stage boundaries,
- out-of-sample champion promotion (WF + holdout blend),
- non-blocking model rollout safety guards,
- persistent state and cache layers for fast restart and analysis,
- observability-first design (Prometheus + telemetry + structured logs),
- dedicated optimization stack that can keep adapting strategy DNA continuously.

This is exactly the foundation you want before scaling to larger capital, broader strategy families, or additional execution venues.

## 10. Security and Operational Notes

- Keep secrets in `.env`, `.env_PROD`, or credential files under controlled paths.
- Do not commit wallet keypairs, API keys, or private credentials.
- Use dedicated funded wallets for live trading.
- Monitor circuit-breaker status and degradation flags (`/dev/shm/ferrix_cv_degraded`) during production runs.
