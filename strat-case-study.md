# C Backtesting Platform — Case Study

TL;DR
- Delivered a C-based backtesting engine and Python reporting pipeline during a 4-week internship. The system loads historical market/macro data from CSV, computes signals, executes configurable backtests with optional transaction-cost and funding adjustments, and exports results for plotting and metrics. Outputs are deterministic and reproducible.

Context & my role
- Team: Financial analysis (internship, 4 weeks).
- Role: Sole developer for the backtesting engine and reporting pipeline; collaborated with analysts for requirements and with data owners for the historical dataset.
- Stack: C (core engine), GSL (optimization utilities), Python (reporting), CSV feeds, Git, VS Code, Make.

Problem statement & goals
- Problem: Ad‑hoc spreadsheets and notebooks slowed iteration and made it hard to run consistent experiments across multiple strategies and parameter sets.
- Goals:
  - Single CLI to run backtests over the full history and emit comparable CSVs per strategy.
  - Deterministic outputs and reproducible metrics (Sharpe/Calmar/Drawdown) across runs, to compare to the literature.
  - Simple plotting/reporting step for side‑by‑side comparisons.

Constraints & assumptions
- Data sensitivity: proprietary dataset; all strategy and signal specifics redacted. 
- Runtime/memory targets: 10k strategies/minute to be able to optimize them computationally.
- Build environment: local dev box; no external services required.

High‑level architecture
- Components: CSV Loader → Indicators → Signal/Regime → Backtest Engine → Metrics → CSV/Plots.

  [CSV] -> [Loader] -> [Indicators] -> [Signal/Regime]
                                 \            |
                                  \           v
                                   -> [Backtest Engine] -> [Metrics] -> [CSV + Plots]

Key implementation choices (sanitized)
- Deterministic, single-threaded core for reproducibility; parameter sweeps done serially in the default build to preserve determinism (parallel execution is a very straightforward extension).
- Separation of concerns:
  - Data ingest in a dedicated parser: [`parse_csv`](src/parser.c).
  - Indicators and regime/signal computation in dedicated modules (redacted).
  - Execution/backtest in two engines: [`run_backtest`](src/backtest.c), [`run_proportional_backtest`](src/backtest.c).
  - Metrics isolated: Sharpe/Calmar/Drawdown in [`calculate_sharpe_ratio`](src/performance.c) and related helpers.
- Outputs in stable CSV format for downstream analysis and for Python plotting ([plotting.py](plotting.py)).

Core flow (sanitized)
- Load time series and macro data from CSV into typed structs.
- Compute rolling indicators and standardized signals over a configurable lookback (details redacted).
- Map signals into daily/weekly allocations or regimes (details redacted).
- Simulate portfolio with configurable rebalancing cadence and optional costs/funding adjustments.
- Persist timeseries to CSV; compute per-strategy metrics; generate plots and tabular summaries.

Selected files and symbols
- Data ingest: [`parse_csv`](src/parser.c) writes into the in‑memory dataset.
- Orchestration/entrypoint: [src/main.c](src/main.c) (runs indicators, signals, backtests, and optimization loops).
- Backtesting:
  - Header/API: [include/backtest.h](include/backtest.h)
  - Engines: [`run_backtest`](src/backtest.c), [`run_proportional_backtest`](src/backtest.c)
- Metrics: [`calculate_sharpe_ratio`](src/performance.c) and related drawdown/calmar helpers.
- Reporting: [plotting.py](plotting.py) reads engine CSVs, computes summary metrics, and plots comparisons.

Sanitized code sketch (control flow)
```c
// high-level sketch (redacted internals)
int main(void) {
  parse_csv("data/data.csv");                    // load inputs
  calculate_indicators(/*weights, window*/);     // redacted
  determine_signals();                         // redacted
  // run a suite of backtests with/without adjustments
  pResult a = run_backtest(/*weights...*/);
  pResult b = run_proportional_backtest(/*...*/);
  // emit CSVs -> plotting.py aggregates metrics and figures
}
```

Backtest engine details (sanitized)
- Rebalancing: configurable period (weeks).
- Transaction costs: optional adjustment using recent volatility and trade size (internal formula redacted).
- Funding adjustment: optional excess‑return mode subtracting a risk‑free proxy at the engine step.
- Metrics:
  - Annualized return/volatility derived from the per‑period series.
  - Drawdown and Calmar computed on the selected series (value vs. excess returns).
  - Sharpe computed on excess returns via `calculate_sharpe_ratio`.

Build, run, and reproduce
- Build
```sh
make
# outputs: strat.exe (Windows) or ./strat (POSIX) depending on toolchain
```
- Run (produces CSVs under csv/ and logs under log/)
```sh
# default run using data/data.csv
./strat        # or .\strat.exe on Windows
```
- Plot and export comparison table
```sh
python plotting.py
# writes strategy_metrics.csv and figures under images/ (when enabled)
```
- Inputs/outputs
  - Input: data/data.csv (sanitized historical series).
  - Outputs: per‑strategy CSVs (csv/*.csv), log/quadrants.csv (diagnostics), images/* (figures when enabled), strategy_metrics.csv.

Performance and correctness
- Deterministic results by construction (single-threaded execution and fixed math).
- CSV outputs are stable across runs for the same inputs and parameters.
- Runtime baselines: 10k strategies/minute.

What changed vs. initial state
- Unified and reproducible pipeline replacing one‑off spreadsheet/notebook experiments.
- Automated CSV exports per strategy and a single plotting/reporting step.
- Optional transaction‑cost and funding adjustments integrated in‑engine rather than in notebooks.

Results & impact (redacted/representative)
- Example outputs and figures are produced by the pipeline (`README.md`); concrete numbers depend on the datasets and parameters in use.
- The platform delivers an optimized strategy based from the macroecnomic signals in less than 1 hour on a single core, with all parameters and metrics avaialble to compare to exising strategies.

Reliability & testing
- Smoke tested on the full history to ensure stable CSV outputs.
- Parameter sweeps executed in batch  for sensitivity checks (details redacted).
- Basic unit testing of parser and backtesting functions.

Roadmap (next steps)
- Parallel parameter sweeps while preserving a deterministic mode for publishing.
- Config‑driven runs (JSON/YAML) rather than compiled configs.
- Extend reporting with richer diagnostics (contribution by asset, turnover, and cost attribution).

Notes on sanitization
- Strategy logic, signal definitions, weights, and parameterizations intentionally redacted.
- Dataset and figures shown are from sanitized inputs; replace with anonymized or synthetic data as needed.
