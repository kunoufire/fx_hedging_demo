# FX Forward Curve Analysis & Hedging Framework

> **An institutional-grade Python framework for FX forward curve analysis, multi-currency portfolio management, and systematic hedging strategies.**

Thanks to the Abu Dhabi hot summer, I had no place to hide from the heat but staying at home on this project.

[![Python](https://img.shields.io/badge/python-3.8%2B-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-Internal-red.svg)](LICENSE)

## Table of Contents

1. [Overview](#overview)
2. [Architecture & Design Philosophy](#architecture--design-philosophy)
3. [Core Components](#core-components)
4. [Object Interactions](#object-interactions)
5. [Design Patterns](#design-patterns)
6. [Data Flow](#data-flow)
7. [Quick Start](#quick-start)
8. [Jupyter Notebooks](#jupyter-notebooks)
9. [Project Structure](#project-structure)
10. [API Reference](#api-reference)

---

## Overview

This framework provides a complete solution for institutional FX hedging workflows:

1. **Forward Curve Retrieval** - Fetch and cache multi-tenor forward curves
2. **Portfolio Management** - Track multi-currency portfolios with NAV history and rebalancing/inflow/outflow
3. **Hedging Strategies** - Implement systematic FX hedging using the Strategy Pattern

### Key Features

- ✅ **Strategy Pattern** - Pluggable hedging algorithms with runtime switching
- ✅ **Persistent Caching** - Intelligent caching layer for Bloomberg API calls
- ✅ **Multi-Currency Support** - Handle both direct (EURUSD) and indirect (USDJPY) quotations
- ✅ **Historical Analysis** - Backtesting support with `generate_trade_history()`
- ✅ **Rich Context** - Strategies access historical data via lazy-loaded properties
- ✅ **Clean Separation** - Orchestration (HedgeManager) separated from decisions (Strategies)

---

## Architecture & Design Philosophy

### Design Principles

This framework follows SOLID principles and modern software engineering practices:

```
┌─────────────────────────────────────────────────────────────────┐
│                    DESIGN PHILOSOPHY                            │
├─────────────────────────────────────────────────────────────────┤
│ 1. Separation of Concerns                                       │
│    → Data fetching (ForwardCurve) ≠ Business logic (HedgeManager)│
│    → Orchestration (HedgeManager) ≠ Decisions (Strategies)      │
│                                                                  │
│ 2. Open/Closed Principle                                        │
│    → Open for extension (new strategies via ABC)                │
│    → Closed for modification (HedgeManager unchanged)           │
│                                                                  │
│ 3. Dependency Inversion                                         │
│    → HedgeManager depends on HedgingStrategy interface          │
│    → Not on concrete strategy implementations                   │
│                                                                  │
│ 4. Single Responsibility                                        │
│    → ForwardCurve: Data retrieval + simple analysis             │
│    → Portfolio: NAV tracking + rebalancing                      │
│    → HedgeManager: Workflow orchestration                       │
│    → Strategies: Hedge ratio calculation only                   │
└─────────────────────────────────────────────────────────────────┘
```

### Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        LAYER 1: DATA                            │
│  ForwardCurve + Portfolio                                       │
│  • Fetch and cache market data from Bloomberg                   │
│  • Track portfolio NAV and exposures                            │
│  • Provide simple analysis methods (term structure, carry)      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 2: ORCHESTRATION                       │
│  HedgeManager                                                   │
│  • Identify FX exposures from portfolio                         │
│  • Build rich context (HedgingContext) with all decision data   │
│  • Delegate to strategy for hedge ratio calculations           │
│  • Generate trade recommendations and execution notionals       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     LAYER 3: STRATEGY                           │
│  HedgingStrategy (ABC) + Concrete Implementations               │
│  • Calculate hedge ratios based on context                      │
│  • Encapsulate hedging logic and decision rules                 │
│  • Return recommendations with optional confidence/metadata     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. ForwardCurve - Data Retrieval Layer

**Responsibility**: Fetch, cache, and provide basic analysis of FX forward curves.

**Design Rationale**:
- **Class-based API** (`ForwardCurve.fetch()`) for ergonomic usage
- **Functional API** (`fetch_and_build_forward_curve()`) for flexibility
- **Persistent caching** to minimize expensive Bloomberg API calls
- **Validation** ensures data integrity (QUOTATION_BASE_CURRENCY, FWD_SCALE)

```python
class ForwardCurve:
    """
    Immutable forward curve object with analysis methods.

    Design choices:
    - Fetched via class method (Builder Pattern)
    - Immutable after construction (data integrity)
    - Lazy-loaded analysis methods (performance)
    - Plotting methods for quick visualization
    """

    @classmethod
    def fetch(cls, config, currency, start_date, end_date, periodicity='D'):
        """Builder pattern - returns fully initialized ForwardCurve"""
        # 1. Validate tickers (QUOTATION_BASE_CURRENCY, SECURITY_TYP)
        # 2. Get FWD_SCALE for proper point scaling
        # 3. Fetch historical data (with caching)
        # 4. Calculate forward rates: Spot + Points/10^SCALE
        # 5. Return ForwardCurve instance

    def term_structure(self, as_of_date=None):
        """Latest term structure - simple analysis"""

    def carry_metrics(self):
        """Full carry panel across all tenors"""

    def plot_term_structure(self):
        """Interactive visualization"""
```

**Key Innovation**: Two-tier caching system
- **BDP Cache** (reference data): 7-day expiry for static fields
- **BDH Cache** (historical data): Smart date range extension, only fetches missing dates

### 2. Portfolio - Multi-Currency NAV Tracking

**Responsibility**: Track portfolio NAV across multiple currencies with rebalancing and cash flows.

**Design Rationale**:
- **Base currency normalization** - All NAVs expressed in base currency (e.g., USD)
- **Auto-add currencies** - Rebalancing automatically adds new currencies
- **Flexible construction** - From scratch or from existing NAV history CSV
- **Periodic frequency** - Support daily, monthly, quarterly rebalancing

```python
class Portfolio:
    """
    Multi-currency portfolio with NAV tracking.

    Design choices:
    - Base currency as single source of truth
    - Weights represent USD value of foreign assets
    - Returns must be in USD terms (FX effects embedded)
    - CashFlow and RebalanceEvent as separate dataclasses (composition)
    """

    def __init__(self, base_currency, init_date, initial_nav, weights, periodic_freq='M'):
        """Explicit initialization for clarity"""

    @classmethod
    def from_nav_history(cls, nav_df, base_currency):
        """Alternative constructor from existing NAV data"""

    def add_rebalance(self, date, target_weights):
        """Auto-add new currencies when they appear in target_weights"""
        # Key feature: No need to manually add currencies first!

    def simulate_history(self, returns_df):
        """Generate NAV history given returns"""
```

**Key Innovation**: `add_rebalance()` automatically adds new currencies
- **Before**: Must call `add_currency()` before rebalancing to new currency
- **After**: Just pass target weights including new currencies - they're added automatically

### 3. HedgeManager - Workflow Orchestration

**Responsibility**: Orchestrate the hedging workflow WITHOUT implementing hedging logic.

**Design Rationale** (Separation of Concerns):
- **What HedgeManager DOES**: Identify exposures, build context, generate trades, calculate notionals
- **What HedgeManager DOESN'T DO**: Decide hedge ratios (delegated to Strategy)
- **Auto-filtering**: Only loads forward curves needed by portfolio currencies
- **Convention handling**: Automatically resolves EURUSD vs USDJPY quoting conventions

```python
class HedgeManager:
    """
    Hedging workflow orchestrator.

    Design choices:
    - Strategy injected at construction (Dependency Injection)
    - Delegates ratio calculation to strategy (Strategy Pattern)
    - Returns trade data only (no strategy metadata in main output)
    - Runtime strategy switching via set_strategy()
    """

    def __init__(self, portfolio, forward_curves, strategy=None):
        """
        Args:
            strategy: Any HedgingStrategy subclass (polymorphism!)
                     Defaults to CarryCostStrategy if not provided
        """
        self.strategy = strategy or CarryCostStrategy()

    def identify_exposures(self):
        """Extract foreign currency exposures from portfolio NAV"""

    def analyze_carry(self, currency_pair, foreign_currency, tenor):
        """4-step carry calculation: Asset/Liability → Convention → Trade Direction → Carry"""

    def generate_trades(self, tenor=None):
        """
        Workflow:
        1. Build HedgingContext with all decision-making data
        2. Delegate to strategy.calculate_ratios(context)  ← POLYMORPHISM!
        3. Merge strategy output with trade data
        4. Return unified trade dictionary
        """
        context = HedgingContext(
            exposures=self.identify_exposures(),
            carry_analyses={...},
            forward_curves=self.forward_curves,  # Pass for rich strategies
            ...
        )
        recommendations = self.strategy.calculate_ratios(context)  # ← Key line!
        return self._build_trade_data(recommendations, context)

    def set_strategy(self, new_strategy):
        """Switch strategy at runtime (Strategy Pattern benefit!)"""
        self.strategy = new_strategy
```

**Key Innovation**: HedgeManager is **strategy-agnostic**
- Works with ANY strategy implementing `HedgingStrategy` interface
- No if/else logic for different strategies
- Add new strategies without modifying HedgeManager code

### 4. HedgingStrategy - Abstract Interface (Strategy Pattern)

**Responsibility**: Define the contract for all hedging strategies.

**Design Rationale**:
- **Abstract Base Class** enforces interface compliance
- **HedgingContext** provides rich decision-making data
- **StrategyRecommendation** allows strategies to return confidence, ranges, metadata
- **Lazy properties** (carry_quantiles, carry_history) for performance

```python
class HedgingStrategy(ABC):
    """
    Abstract interface for all hedging strategies.

    Design choices:
    - ABC ensures subclasses implement required methods
    - Context object (vs many parameters) for clean interface
    - default_tenor as strategy property (strategies control defaults)
    - Optional rich output via StrategyRecommendation (v2.7.0+)
    """

    @abstractmethod
    def calculate_ratios(self, context: HedgingContext, **kwargs) -> Dict[str, StrategyOutput]:
        """
        The only method strategies MUST implement.

        Args:
            context: HedgingContext with exposures, carry_analyses, forward_curves, etc.

        Returns:
            Dict mapping currency to recommendation (dict or StrategyRecommendation)
        """
        pass

    @abstractmethod
    def name(self) -> str:
        """Return strategy name for logging"""
        pass
```

**Concrete Strategies**:

1. **ConstantRatioStrategy** - Fixed ratios (simplest)
2. **CarryCostStrategy** - Tiered ratios based on carry (default)
3. **PolicyDrivenStrategy** - Pre-defined institutional policies
4. **QuantileBasedStrategy** - Historical carry distribution-based (advanced)

**Key Innovation**: Rich context via lazy properties
```python
class HedgingContext:
    # Core data (always populated)
    exposures: Dict[str, Dict[str, Any]]
    carry_analyses: Dict[str, CarryAnalysis]

    # Rich data (optional)
    forward_curves: Optional[Dict[str, ForwardCurve]] = None

    @property
    def carry_quantiles(self):
        """Lazy-computed historical carry percentiles"""
        # Access historical data from forward_curves
        # Return {ccy: {p10, p25, p50, p75, p90, percentile, z_score}}
```

---

## Object Interactions

### Workflow 1: Loading Forward Curves

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ 1. ForwardCurve.fetch(config, 'EURUSD', start, end)
       ↓
┌─────────────────────────────────────────────────────┐
│              ForwardCurve (Builder)                 │
├─────────────────────────────────────────────────────┤
│ 2. validate_currency_forward_curve()                │
│    └→ Check QUOTATION_BASE_CURRENCY consistency     │
│    └→ Verify SECURITY_TYP for forward instruments   │
│                                                      │
│ 3. get_currency_forward_scales()                    │
│    └→ cached_bdp() ← BDP Cache (7-day expiry)       │
│    └→ Returns {1M: 4, 3M: 4, 6M: 4, ...}            │
│                                                      │
│ 4. fetch_forward_curve_history()                    │
│    └→ cached_bdh() ← BDH Cache (smart date range)   │
│    └→ Returns spot + forward points DataFrame       │
│                                                      │
│ 5. build_forward_curve()                            │
│    └→ Forward_Rate = Spot + Points/10^FWD_SCALE     │
│    └→ Add FWD_{tenor}_Rate columns                  │
└─────────────────────────────────────────────────────┘
       │
       ↓
┌─────────────────┐
│  ForwardCurve   │ (immutable instance with analysis methods)
│  - data         │
│  - spot         │
│  - rate(tenor)  │
│  - plot_*()     │
└─────────────────┘
```

### Workflow 2: Creating Portfolio

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ Option A: from_nav_history(nav_df, 'USD')
       │ Option B: __init__(base_currency, init_date, ...)
       ↓
┌─────────────────────────────────────────────────────┐
│                   Portfolio                         │
├─────────────────────────────────────────────────────┤
│ • Infer initial weights from NAV columns            │
│ • Infer periodic_freq from date spacing            │
│ • Initialize nav_history from DataFrame             │
│                                                      │
│ Key features:                                       │
│ - add_rebalance() auto-adds new currencies          │
│ - All NAVs in base currency (USD)                  │
│ - Returns must embed FX effects                     │
└─────────────────────────────────────────────────────┘
       │
       ↓
┌─────────────────┐
│   Portfolio     │ (with NAV history ready for analysis)
│  - currencies   │
│  - nav_history  │
│  - weights      │
└─────────────────┘
```

### Workflow 3: Hedging with Strategy Pattern

This is the **core interaction** demonstrating the Strategy Pattern:

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ 1. Create strategy
       │    strategy = CarryCostStrategy()
       │
       │ 2. Inject into HedgeManager
       │    hm = HedgeManager(portfolio, forward_curves, strategy=strategy)
       ↓
┌─────────────────────────────────────────────────────────────────┐
│                         HedgeManager                            │
├─────────────────────────────────────────────────────────────────┤
│ 3. hm.generate_trades(tenor='3M')                               │
│    │                                                             │
│    ├──→ Step 1: Identify exposures                              │
│    │    └─ exposures = identify_exposures()                     │
│    │       Returns: {EUR: {nav, weight, currency_pair}, ...}    │
│    │                                                             │
│    ├──→ Step 2: Analyze carry for each currency                 │
│    │    └─ carry_analyses = {ccy: analyze_carry(...), ...}      │
│    │       Returns: {EUR: CarryAnalysis(carry_bps, ...), ...}   │
│    │                                                             │
│    ├──→ Step 3: Build rich context                              │
│    │    └─ context = HedgingContext(                            │
│    │           exposures=exposures,                             │
│    │           carry_analyses=carry_analyses,                   │
│    │           forward_curves=self.forward_curves,  ← Rich data │
│    │           tenor='3M',                                      │
│    │           ...                                              │
│    │       )                                                    │
│    │                                                             │
│    └──→ Step 4: DELEGATE TO STRATEGY (polymorphism!)            │
│         └─ recommendations = self.strategy.calculate_ratios(    │
│                context                                          │
│            )                                                    │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  │ 4. Calculate ratios
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│              HedgingStrategy (Abstract Interface)               │
│              CarryCostStrategy (Concrete Implementation)        │
├─────────────────────────────────────────────────────────────────┤
│ def calculate_ratios(self, context: HedgingContext):           │
│     results = {}                                                │
│     for currency in context.exposures.keys():                   │
│         carry = context.carry_analyses[currency]                │
│         carry_bps = carry.carry_bps_ann                         │
│                                                                  │
│         # Strategy-specific logic                               │
│         if carry_bps > 100:                                     │
│             ratio = 0.90  # Strong positive carry               │
│         elif carry_bps > 20:                                    │
│             ratio = 0.70  # Moderate positive carry             │
│         ...                                                     │
│                                                                  │
│         results[currency] = {                                   │
│             'recommended_ratio': ratio,                         │
│             'rationale': '...',                                 │
│             'metadata': {}                                      │
│         }                                                       │
│     return results                                              │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  │ 5. Return recommendations
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│                         HedgeManager                            │
├─────────────────────────────────────────────────────────────────┤
│ 6. Merge strategy output with trade data                       │
│    └─ trades = _build_trade_data(recommendations, context)      │
│       Combines:                                                 │
│       - Strategy ratios + rationale                             │
│       - Exposure NAV                                            │
│       - Carry analysis (bps, direction, interpretation)         │
│       - Forward pricing (spot, forward_rate, tenor)             │
│       - Trade metadata (currency_pair, forward_ticker)          │
│                                                                  │
│ 7. Return unified trade dictionary                             │
└─────────────────────────────────────────────────────────────────┘
       │
       ↓
┌─────────────┐
│   Client    │ Receives: {
└─────────────┘    'EUR': {
                      'recommended_ratio': 0.90,
                      'rationale': 'Strong positive carry',
                      'exposure_nav': 1024692,
                      'carry_bps_ann': 164.3,
                      'spot_rate': 1.0354,
                      'forward_rate': 1.0396,
                      ...
                   }
                }
```

**Key Insight**: The Strategy Pattern enables:
1. **HedgeManager** doesn't know which strategy is active (polymorphism)
2. **Strategy** doesn't know about trade data assembly (separation of concerns)
3. **Client** can switch strategies at runtime via `set_strategy()`
4. **New strategies** added without modifying HedgeManager (Open/Closed Principle)

---

## Design Patterns

### 1. Builder Pattern (ForwardCurve)

**Problem**: Creating a ForwardCurve requires multiple steps (validation, scaling, fetching, calculation).

**Solution**: Class method `ForwardCurve.fetch()` encapsulates construction:

```python
# Client code is simple
curve = ForwardCurve.fetch(config, 'EURUSD', start, end)

# Instead of:
# 1. Validate tickers
# 2. Get FWD_SCALE
# 3. Fetch historical data
# 4. Calculate forward rates
# 5. Create instance
```

### 2. Strategy Pattern (HedgingStrategy)

**Problem**: Multiple hedging algorithms with different logic, need to switch at runtime.

**Solution**: Abstract interface + concrete implementations:

```python
# Define interface
class HedgingStrategy(ABC):
    @abstractmethod
    def calculate_ratios(self, context): pass

# Implement strategies
class CarryCostStrategy(HedgingStrategy): ...
class QuantileBasedStrategy(HedgingStrategy): ...

# Use polymorphically
hm = HedgeManager(portfolio, curves, strategy=CarryCostStrategy())
trades = hm.generate_trades()  # Uses CarryCostStrategy

hm.set_strategy(QuantileBasedStrategy())  # Switch at runtime
trades = hm.generate_trades()  # Now uses QuantileBasedStrategy
```

**Benefits**:
- ✅ Add new strategies without modifying HedgeManager
- ✅ Test strategies independently
- ✅ Switch algorithms at runtime
- ✅ Client code doesn't change when adding strategies

### 3. Lazy Evaluation (HedgingContext properties)

**Problem**: Computing carry quantiles is expensive, not all strategies need it.

**Solution**: `@property` with caching:

```python
class HedgingContext:
    @property
    def carry_quantiles(self):
        """Computed only when accessed, cached thereafter"""
        if self._carry_quantiles_cache is not None:
            return self._carry_quantiles_cache

        # Expensive computation
        result = self._compute_quantiles()
        self._carry_quantiles_cache = result
        return result
```

**Benefits**:
- ✅ ConstantRatioStrategy doesn't pay cost (never accesses)
- ✅ QuantileBasedStrategy computes once, reuses
- ✅ Clean API (looks like attribute access)

### 4. Dependency Injection (HedgeManager constructor)

**Problem**: HedgeManager needs a strategy, but shouldn't create it (tight coupling).

**Solution**: Inject strategy at construction:

```python
class HedgeManager:
    def __init__(self, portfolio, forward_curves, strategy=None):
        self.strategy = strategy or CarryCostStrategy()  # Default
```

**Benefits**:
- ✅ Easy to test with mock strategies
- ✅ Client controls which strategy to use
- ✅ No hard dependency on specific strategy class

---

## Data Flow

### Complete System Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    1. DATA ACQUISITION                          │
└─────────────────────────────────────────────────────────────────┘
         Bloomberg Terminal
                │
                │ xbbg API calls
                ↓
         ┌──────────────┐
         │ Caching Layer│
         ├──────────────┤
         │ BDP: 7-day   │ → Reference data (FWD_SCALE, BASE_CCY)
         │ BDH: smart   │ → Historical data (spot, fwd points)
         └──────────────┘
                │
                ↓
         ┌──────────────┐
         │ ForwardCurve │ → Calculate forward rates
         └──────────────┘

         ┌──────────────┐
         │   NAV CSV    │
         └──────────────┘
                │
                ↓
         ┌──────────────┐
         │  Portfolio   │ → Track multi-currency NAV
         └──────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  2. EXPOSURE IDENTIFICATION                     │
└─────────────────────────────────────────────────────────────────┘
         ┌──────────────┐
         │  Portfolio   │
         └──────┬───────┘
                │ Latest NAV by currency
                ↓
         ┌──────────────────┐
         │  HedgeManager    │
         │ identify_exposures()│
         └──────┬───────────┘
                │
                ↓ exposures = {
                      'EUR': {nav, weight, currency_pair},
                      'GBP': {nav, weight, currency_pair}
                  }

┌─────────────────────────────────────────────────────────────────┐
│                    3. CARRY ANALYSIS                            │
└─────────────────────────────────────────────────────────────────┘
         ┌──────────────┐        ┌──────────────────┐
         │ ForwardCurve │        │   Exposures      │
         └──────┬───────┘        └────────┬─────────┘
                │                         │
                │ Spot, Forward Rate      │ Trade Direction
                └────────────┬────────────┘
                             ↓
                      ┌──────────────┐
                      │ CarryAnalysis│
                      ├──────────────┤
                      │ • carry_bps  │
                      │ • direction  │
                      │ • is_benefit │
                      └──────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  4. STRATEGY DECISION                           │
└─────────────────────────────────────────────────────────────────┘
         ┌────────────────────────┐
         │   HedgingContext       │
         ├────────────────────────┤
         │ • exposures            │
         │ • carry_analyses       │
         │ • forward_curves       │ ← Rich data for advanced strategies
         │ • tenor                │
         └───────────┬────────────┘
                     │
                     │ context passed to strategy
                     ↓
         ┌─────────────────────────┐
         │   HedgingStrategy       │
         │   calculate_ratios()    │
         └───────────┬─────────────┘
                     │
                     │ Strategy logic:
                     │ - CarryCost: Use carry_bps tiers
                     │ - Quantile: Use carry_quantiles (lazy property)
                     │ - Constant: Use predefined ratios
                     ↓
         ┌─────────────────────────┐
         │  Recommendations        │
         ├─────────────────────────┤
         │ {                       │
         │   'EUR': {              │
         │     'recommended_ratio', │
         │     'rationale',        │
         │     'metadata'          │
         │   }                     │
         │ }                       │
         └─────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   5. TRADE GENERATION                           │
└─────────────────────────────────────────────────────────────────┘
         ┌──────────────────┐      ┌──────────────────┐
         │ Recommendations  │      │  Context Data    │
         └────────┬─────────┘      └────────┬─────────┘
                  │                         │
                  └──────────┬──────────────┘
                             ↓
                   ┌──────────────────┐
                   │ HedgeManager     │
                   │ _build_trade_data()│
                   └────────┬─────────┘
                            │
                            │ Merge:
                            │ - Strategy output (ratio, rationale)
                            │ - Exposure data (NAV, weight)
                            │ - Carry data (bps, direction)
                            │ - Pricing (spot, forward, tenor)
                            ↓
         ┌─────────────────────────────────────┐
         │           Trade Data                │
         ├─────────────────────────────────────┤
         │ {                                   │
         │   'EUR': {                          │
         │     'recommended_ratio': 0.90,      │
         │     'rationale': 'Strong carry',    │
         │     'exposure_nav': 1024692,        │
         │     'carry_bps_ann': 164.3,         │
         │     'spot_rate': 1.0354,            │
         │     'forward_rate': 1.0396,         │
         │     'forward_ticker': 'EUR3M Curncy',│
         │     'currency_pair': 'EURUSD',      │
         │     'trade_direction': 'SELL'       │
         │   }                                 │
         │ }                                   │
         └─────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                6. EXECUTION NOTIONALS                           │
└─────────────────────────────────────────────────────────────────┘
         ┌─────────────────┐
         │   Trade Data    │
         └────────┬────────┘
                  │ Extract ratios
                  ↓
         ┌──────────────────────────┐
         │ HedgeManager             │
         │ calculate_hedge_notionals()│
         └──────────┬───────────────┘
                    │
                    │ For each currency:
                    │ - notional_base = exposure_nav × ratio
                    │ - notional_foreign = notional_base / fwd_rate
                    │ - direction = SELL/BUY
                    │ - maturity_date = today + tenor
                    ↓
         ┌─────────────────────────┐
         │   HedgePosition         │
         ├─────────────────────────┤
         │ • currency_pair         │
         │ • direction (SELL/BUY)  │
         │ • notional_base_ccy     │
         │ • notional_foreign_ccy  │
         │ • forward_rate          │
         │ • maturity_date         │
         └─────────────────────────┘
                    │
                    ↓
         Ready for execution!
```

---

## Quick Start

### Installation

```bash
# Clone repository
git clone <repository-url>
cd fx_hedging_linear

# Install dependencies
pip install -r requirements.txt

# Requires Bloomberg Terminal with xbbg API access
```

### Basic Usage

#### 1. Fetch Forward Curve

```python
from src import ForwardCurve, load_forward_curve_config
from datetime import datetime, timedelta

# Load configuration
config = load_forward_curve_config('data/raw/forward_curve_tickers.csv')

# Fetch forward curve
curve = ForwardCurve.fetch(
    config=config,
    currency='EURUSD',
    start_date=datetime(2015, 1, 1),
    end_date=datetime.now(),
    periodicity='Q'  # Quarterly
)

# Analysis
print(curve.term_structure())  # Latest term structure
print(curve.carry_metrics())   # Carry across all tenors
curve.plot_term_structure()    # Interactive plot
```

#### 2. Create Multi-Currency Portfolio

```python
from src import Portfolio
import pandas as pd

# Option A: From existing NAV history
nav_df = pd.read_csv('data/raw/nav_history.csv', parse_dates=['Date'], index_col='Date')
portfolio = Portfolio.from_nav_history(nav_df, base_currency='USD')

# Option B: From scratch
portfolio = Portfolio(
    base_currency='USD',
    init_date=datetime(2020, 1, 1),
    initial_nav=1_000_000,
    weights={'USD': 0.6, 'EUR': 0.3, 'GBP': 0.1},
    periodic_freq='M'
)

# Add rebalancing (auto-adds new currencies!)
portfolio.add_rebalance(
    datetime(2022, 1, 1),
    {'USD': 0.5, 'EUR': 0.25, 'GBP': 0.1, 'JPY': 0.15}  # JPY auto-added!
)

print(portfolio.summary())
```

#### 3. Implement FX Hedging with Strategy Pattern

```python
from src import HedgeManager, CarryCostStrategy, QuantileBasedStrategy

# Load forward curves
curves = {
    'EURUSD': ForwardCurve.fetch(config, 'EURUSD', start, end),
    'GBPUSD': ForwardCurve.fetch(config, 'GBPUSD', start, end),
    'USDJPY': ForwardCurve.fetch(config, 'USDJPY', start, end)
}

# Create hedge manager with strategy
strategy = CarryCostStrategy(default_tenor='3M')
hm = HedgeManager(portfolio, curves, strategy=strategy)

# Generate hedge recommendations
trades = hm.generate_trades()

for ccy, trade in trades.items():
    print(f"{ccy}: {trade['recommended_ratio']:.0%} hedge - {trade['rationale']}")
    print(f"  Carry: {trade['carry_bps_ann']:+.1f} bps")

# Switch strategy at runtime
hm.set_strategy(QuantileBasedStrategy(base_ratio=0.5, sensitivity=0.4))
trades_quantile = hm.generate_trades()

# Calculate execution notionals
hedge_ratios = {ccy: t['recommended_ratio'] for ccy, t in trades.items()}
positions = hm.calculate_hedge_notionals(hedge_ratios, tenor='3M')

for ccy, pos in positions.items():
    print(f"{pos.currency_pair}: {pos.direction} ${pos.notional_base_ccy:,.0f}")
```

---

## Jupyter Notebooks

### [01_forward_curve_analysis.ipynb](notebooks/01_forward_curve_analysis.ipynb)

**Demonstrates**: ForwardCurve class and data retrieval layer

- ✅ Bloomberg data fetching with caching
- ✅ Forward rate calculation (Spot + Points/10^SCALE)
- ✅ Validation (QUOTATION_BASE_CURRENCY, FWD_SCALE)
- ✅ Term structure analysis
- ✅ Carry metrics and visualization

**Key Concepts**:
- Builder Pattern (`ForwardCurve.fetch()`)
- Two-tier caching system
- Lazy-loaded analysis methods

### [02_portfolio_demo.ipynb](notebooks/02_portfolio_demo.ipynb)

**Demonstrates**: Multi-currency portfolio management

- ✅ Portfolio construction from NAV history
- ✅ Auto-inference (weights, periodic frequency)
- ✅ Rebalancing with auto-add currencies
- ✅ Cash flow events
- ✅ Performance attribution

**Key Concepts**:
- Base currency normalization
- `from_nav_history()` alternative constructor
- `add_rebalance()` auto-add feature

### [03_hedge_manager_demo.ipynb](notebooks/03_hedge_manager_demo.ipynb)

**Demonstrates**: End-to-end FX hedging with Strategy Pattern

- ✅ **Section 4**: Abstract interface explanation (HedgingStrategy ABC)
- ✅ **Section 5**: Polymorphism demonstration (multiple strategies)
- ✅ Exposure identification
- ✅ 4-quadrant carry analysis
- ✅ Strategy-based hedge recommendations
- ✅ Runtime strategy switching
- ✅ Trade generation and execution notionals
- ✅ Multi-tenor analysis
- ✅ Historical backtesting with `generate_trade_history()`

**Key Concepts**:
- Strategy Pattern (Abstract Base Class + Concrete Implementations)
- Dependency Injection (strategy passed to HedgeManager)
- Polymorphism (HedgeManager works with any strategy)
- Separation of Concerns (orchestration vs decisions)
- Lazy Evaluation (HedgingContext properties)

**Special Section**:
- **Section 4** provides a deep dive into the abstract interface design, explaining:
  - Interface contract (what strategies must implement)
  - Benefits of abstract base classes
  - HedgingContext dataclass structure
  - StrategyRecommendation enhanced output
  - Simple vs advanced strategy examples
  - How HedgeManager uses polymorphism

---

## Project Structure

```
fx_hedging_linear/
├── src/
│   ├── __init__.py                      # Public API exports
│   ├── forward_curve.py                 # ForwardCurve class + caching
│   ├── portfolio.py                     # Portfolio + CashFlow + RebalanceEvent
│   ├── hedge_manager.py                 # HedgeManager + CarryAnalysis + HedgePosition
│   ├── hedging_strategies.py            # HedgingStrategy ABC + implementations
│   │                                    #   - HedgingContext (input dataclass)
│   │                                    #   - StrategyRecommendation (output dataclass)
│   │                                    #   - ConstantRatioStrategy
│   │                                    #   - CarryCostStrategy
│   │                                    #   - PolicyDrivenStrategy
│   │                                    #   - QuantileBasedStrategy
│   ├── pnl_attribution.py               # P&L attribution (future)
│   └── days_to_mty.py                   # Days to maturity analysis
│
├── notebooks/
│   ├── 01_forward_curve_analysis.ipynb  # Data layer demonstration
│   ├── 02_portfolio_demo.ipynb          # Portfolio management
│   └── 03_hedge_manager_demo.ipynb      # Strategy Pattern + complete workflow
│
├── data/
│   ├── raw/
│   │   ├── forward_curve_tickers.csv    # Bloomberg ticker configuration
│   │   └── nav_history.csv              # Sample NAV data
│   ├── cache/                           # Auto-generated
│   │   ├── bdp/                         # Reference data cache (pickle)
│   │   └── bdh/                         # Historical data cache (parquet)
│   └── processed/                       # Export outputs
│       └── hedge_trade_history.csv
│
├── tests/
│   ├── __init__.py
│   ├── demo_hedge_manager.py            # Comprehensive HedgeManager demo
│   ├── test_hedging_strategies.py       # Strategy unit tests
│   ├── test_carry_logic.py              # 4-quadrant carry tests
│   └── ...
│
├── docs/
│   ├── HEDGE_MANAGER_GUIDE.md           # Detailed usage guide
│   └── portfolio_design.md              # Portfolio module design
│
├── CLAUDE.md                            # AI assistant instructions
├── README.md                            # This file
└── requirements.txt                     # Dependencies
```

---

## API Reference

### ForwardCurve Class

```python
class ForwardCurve:
    """Immutable forward curve with analysis methods."""

    @classmethod
    def fetch(cls, config, currency, start_date, end_date, periodicity='D',
              force_refresh=False, validate=True, verbose=False) -> 'ForwardCurve':
        """Builder method - returns initialized ForwardCurve instance."""

    @property
    def data(self) -> pd.DataFrame:
        """Full DataFrame with spot and forward rates."""

    @property
    def spot(self) -> pd.Series:
        """Spot rate time series."""

    def rate(self, tenor: str) -> pd.Series:
        """Forward rate for specific tenor."""

    def term_structure(self, as_of_date=None) -> pd.Series:
        """Latest term structure."""

    def carry_metrics(self) -> pd.DataFrame:
        """Carry metrics panel across all tenors."""

    def ann_premium(self, tenor: str) -> pd.Series:
        """Annualized forward premium for tenor."""

    def plot_term_structure(self, as_of_date=None):
        """Plot term structure."""

    def plot_carry_history(self, tenor='3M'):
        """Plot historical carry."""
```

### Portfolio Class

```python
class Portfolio:
    """Multi-currency portfolio with NAV tracking."""

    def __init__(self, base_currency, init_date, initial_nav, weights, periodic_freq='M'):
        """Explicit constructor."""

    @classmethod
    def from_nav_history(cls, nav_df, base_currency) -> 'Portfolio':
        """Alternative constructor from NAV DataFrame."""

    def add_currency(self, currency, initial_weight=0.0):
        """Manually add new currency."""

    def add_rebalance(self, date, target_weights):
        """Schedule rebalancing (auto-adds new currencies)."""

    def add_cash_flow(self, cash_flow: CashFlow):
        """Add cash flow event."""

    def simulate_history(self, returns_df):
        """Generate NAV history from returns."""

    def summary(self) -> str:
        """Performance summary."""

    def total_return(self) -> float:
        """Cumulative return."""

    def plot_comprehensive_analysis(self, figsize=(16, 12)):
        """Plot NAV history + allocations + returns."""
```

### HedgeManager Class

```python
class HedgeManager:
    """Hedging workflow orchestrator."""

    def __init__(self, portfolio, forward_curves, strategy=None, auto_filter=True):
        """
        Args:
            portfolio: Portfolio instance
            forward_curves: Dict[str, ForwardCurve]
            strategy: HedgingStrategy subclass (defaults to CarryCostStrategy)
            auto_filter: Auto-filter curves to portfolio currencies
        """

    def identify_exposures(self) -> Dict[str, Dict[str, Any]]:
        """Extract foreign currency exposures."""

    def analyze_carry(self, currency_pair, foreign_currency, tenor) -> CarryAnalysis:
        """4-quadrant carry analysis."""

    def generate_trades(self, tenor=None) -> Dict[str, Dict[str, Any]]:
        """Generate hedge trade recommendations."""

    def calculate_hedge_notionals(self, hedge_ratios, tenor) -> Dict[str, HedgePosition]:
        """Calculate execution notionals."""

    def generate_hedge_summary(self, tenor) -> pd.DataFrame:
        """Comprehensive summary table."""

    def generate_trade_history(self, frequency='QE', tenor='3M') -> pd.DataFrame:
        """Historical backtesting analysis."""

    def set_strategy(self, new_strategy: HedgingStrategy):
        """Switch strategy at runtime."""
```

### HedgingStrategy Abstract Base Class

```python
class HedgingStrategy(ABC):
    """Abstract interface for hedging strategies."""

    def __init__(self, default_tenor='3M'):
        """Base constructor."""

    @abstractmethod
    def calculate_ratios(self, context: HedgingContext, **kwargs) -> Dict[str, StrategyOutput]:
        """Calculate hedge ratios (MUST IMPLEMENT)."""

    @abstractmethod
    def name(self) -> str:
        """Return strategy name (MUST IMPLEMENT)."""
```

### Concrete Strategies

```python
class ConstantRatioStrategy(HedgingStrategy):
    """Fixed hedge ratios."""
    def __init__(self, ratios: Dict[str, float], default_ratio=0.5, default_tenor='3M'):
        ...

class CarryCostStrategy(HedgingStrategy):
    """Carry-based tiered ratios."""
    def __init__(self, cost_thresholds=None, default_ratio=0.5, default_tenor='3M'):
        ...

class PolicyDrivenStrategy(HedgingStrategy):
    """Pre-defined policy ratios."""
    def __init__(self, policies: Dict[str, float], default_ratio=0.5, default_tenor='3M'):
        ...

class QuantileBasedStrategy(HedgingStrategy):
    """Historical carry distribution-based."""
    def __init__(self, base_ratio=0.5, sensitivity=0.4, min_ratio=0.1, max_ratio=0.95, default_tenor='3M'):
        ...
```

### HedgingContext Dataclass

```python
@dataclass
class HedgingContext:
    """Rich context passed to strategies."""

    # Core data (always populated)
    exposures: Dict[str, Dict[str, Any]]
    carry_analyses: Dict[str, CarryAnalysis]
    tenor: str
    as_of_date: datetime
    exposure_type: str

    # Rich data (optional)
    forward_curves: Optional[Dict[str, ForwardCurve]] = None
    metadata: Dict[str, Any] = field(default_factory=dict)

    # Computed properties (lazy evaluation)
    @property
    def carry_history(self) -> pd.DataFrame:
        """Historical carry DataFrame."""

    @property
    def carry_quantiles(self) -> Dict[str, Dict[str, float]]:
        """Carry percentiles for regime detection."""
```

---

## Requirements

- Python 3.8+
- Bloomberg Terminal with xbbg API access
- pandas, numpy, matplotlib, seaborn
- xbbg (Bloomberg API wrapper)

See `requirements.txt` for full dependency list.

---

## Version History

- **v2.7.0**  - Enhanced strategy output with StrategyRecommendation, lazy carry_quantiles
- **v2.6.0**  - Default tenor support in strategies
- **v2.5.0**  - Strategy Pattern implementation, HedgingStrategy ABC
- **v2.0.0**  - HedgeManager and multi-currency portfolio
- **v1.0.0**  - Initial release with ForwardCurve

---

## License

Internal use only.

---

## Author

kunoufire@gmail.com

For questions or contributions, please contact the repository owner.
