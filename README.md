# Bank Run Replication Archive

This repository contains two related computational economics projects studying bank run dynamics. Together they form a progression from a stylized theoretical model to a full agent-based simulation with social network effects.

## Repository Structure

```
BankRunArchive/
├── diamond-dybvig/    # Theoretical baseline: Diamond-Dybvig parameter sweep
└── bank-runs-abm/     # Extension: networked agent-based bank run model
```

## How the Projects Relate

### 1. `diamond-dybvig` — Theoretical Baseline

Implements the classic Diamond-Dybvig (1983) model as a Monte Carlo simulation. K identical agents deposit resources with a bank. After exogenous shocks force some early withdrawals, remaining agents look ahead stochastically and decide whether to withdraw. The model sweeps over three parameters:

- **Insurance premium** (`insur`) — fraction of deposit returned on failure
- **Bank productivity** (`prod`) — return on invested deposits
- **Exogenous withdrawal probability** (`objP`) — base rate of liquidity shocks

Output is a failure probability surface over this parameter space. This establishes theoretical benchmarks: under what insurance and productivity conditions does a bank run become likely?

### 2. `bank-runs-abm` — Network Extension

Extends the Diamond-Dybvig framework into a heterogeneous agent-based model (ABM) to study the *dynamics* of how a banking system transitions from a stable to a failed state—a question the original theoretical model cannot answer.

Key additions over the baseline:

| Feature | `diamond-dybvig` | `bank-runs-abm` |
|---------|-----------------|-----------------|
| Agents | Identical | Heterogeneous deposits (LogNormal / Pareto) |
| Social structure | None | Watts-Strogatz small-world network |
| Information | Global | Local (agents observe neighbors only) |
| Bank reserves | Implicit | Explicit fractional reserve ratio |
| Insurance | Continuous premium | Quantile-based deposit guarantee |
| Scale | K = 50–500 | K = 1,000 fixed |
| Output | Failure probability surface | Per-agent transaction logs + failure outcome |

Agents in the ABM observe their network neighbors' withdrawal decisions to estimate systemic risk, then run an internal Monte Carlo simulation to compare loss probability if they withdraw now versus if they wait. This bounded-rational decision rule is what generates endogenous cascade dynamics.

### Research Question Progression

1. **`diamond-dybvig`** asks: *Given insurance and productivity levels, what is the equilibrium failure probability?*
2. **`bank-runs-abm`** asks: *Given a realistic social and financial structure, how does a run actually unfold—who withdraws, when, and why?*

The ABM results can be compared against the theoretical benchmarks from `diamond-dybvig` to assess which features of the richer model matter most for failure probability.

## Quick Start

### Prerequisites

- Julia 1.11+
- R 4.0+

Install Julia packages (run once per environment):

```julia
using Pkg
Pkg.add(["Distributions", "Random", "CSV", "DataFrames",
         "Graphs", "StatsBase", "JLD2", "Dates",
         "TreeParzen", "Plots", "Folds"])
```

Install R packages:

```r
install.packages(c("tidyverse", "data.table", "ggplot2", "ggExtra"))
```

### Recommended Replication Order

1. **Run the theoretical model first** to reproduce the failure probability surface:

   ```bash
   mkdir -p DDData
   cd diamond-dybvig
   julia main0001.jl
   Rscript analysis.R
   ```

2. **Run the convergence check** to verify the Monte Carlo estimates are stable:

   ```bash
   julia convergence_test.jl
   ```

3. **Run the ABM** to reproduce network simulation results:

   ```bash
   mkdir -p BankRunData
   cd bank-runs-abm
   ./run_sweep.sh
   Rscript analysis2.R ../BankRunData
   ```

See the `README.md` inside each subdirectory for full parameter documentation, output schemas, and advanced usage (container execution, restart from checkpoint, sysimage compilation).

## Data Directories

Both models write results outside their source directories. Create these before running:

```bash
mkdir -p DDData       # for diamond-dybvig
mkdir -p BankRunData  # for bank-runs-abm
```

The relative paths `../DDData` and `../BankRunData` are hardcoded in the scripts and resolve correctly when run from within the respective subdirectory.

## Citation

If you use this code, please cite:

> Schuler, John S. (forthcoming). "Deposits Are Not Options: Contract Artifacts, Equilibrium Selection, and Network Contagion in Bank Run Models."

The Diamond-Dybvig baseline also builds on:

> Diamond, Douglas W. and Philip H. Dybvig (1983). "Bank Runs, Deposit Insurance, and Liquidity." *Journal of Political Economy*, 91(3), 401–419.

## License

Both `diamond-dybvig` and `bank-runs-abm` are released under the MIT License (see `diamond-dybvig/LICENSE` and `bank-runs-abm/LICENSE`).
