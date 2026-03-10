# Online Metastability Detection for Intraday Markets

This project explores whether short-horizon market dynamics can be described by a changing local **potential landscape**, and whether features derived from that landscape help improve forecasting.

The pipeline estimates a nonparametric local drift and diffusion from rolling windows of high-frequency price data, reconstructs an implied potential \(U(x)\), labels the window as `single_well`, `double_well`, or `unclear`, and then tests whether those regime features add value to simple prediction tasks.

**The current public notebook run uses synthetic data to validate the end-to-end research pipeline.** The goal at this stage is to confirm that the full workflow behaves sensibly, produces interpretable structural outputs, and supports downstream modeling before moving to real market data.

The current notebook run should therefore be viewed as a **pipeline validation and research prototype**, not a final trading result.

---

## Project idea

Most intraday models use lagged returns, volume, and volatility as inputs. This project asks a different question:

> Can we estimate the local dynamical structure of the market itself, and use that structure as a predictive feature?

For each rolling window, the code:

1. computes log-price \(x_t = \log P_t\)
2. estimates conditional drift and diffusion from price increments
3. reconstructs an implied potential
   \[
   U(x) = -\int \mu(x)\,dx
   \]
4. extracts structural features such as:
   - number of wells
   - barrier height
   - well distance
   - well asymmetry
   - drift magnitude
   - diffusion level
5. uses those features in forecasting models alongside standard baseline features

The intuition is:

- a **single-well** potential suggests one dominant local equilibrium
- a **double-well** potential suggests two competing local states
- an **unclear** potential suggests weak or unstable structure

---

## Forecasting tasks

The current notebook evaluates two simple tasks.

### 1. Future realized volatility
Predict realized volatility over the next fixed horizon.

### 2. Continuation vs reversal
Predict whether short-horizon direction continues or reverses relative to recent momentum.

These are intentionally simple first targets. They let us test whether potential-based regime features are adding information beyond standard lagged return, volume, and liquidity features.

---

## Data

The current public version uses synthetic high-frequency data to validate the estimation and modeling pipeline. This choice was practical: the most relevant real high-frequency datasets either require paid access, additional API setup, or more data-engineering infrastructure for reliable ingestion than was available for this initial prototype. The goal of this stage was to validate the end-to-end research workflow first, then move to real market data in a later version. 

This synthetic setup is useful because it allows the project to test:
- whether rolling drift and diffusion estimation works
- whether the implied potential can be reconstructed cleanly
- whether regime labels are stable and interpretable
- whether the downstream ML pipeline runs end to end

At this stage, the synthetic run is serving as a **research sandbox**, not as evidence of live trading performance or real-market alpha.

The intended next step is to apply the same framework to real high-frequency datasets such as:
- Uniswap v3 pool data
- centralized limit-order-book data
- other trade-level intraday market data

---

## Current run summary

### Feature table
The pipeline successfully generated a feature table with rolling structural features and supervised targets.

Example rows include:
- future realized volatility
- future return
- continuation label
- past volatility and returns
- signed volume and liquidity features
- potential-derived features such as `barrier_height`, `U_range`, `drift_abs_mean`, and `current_drift`

### Regime distribution
The estimated regime counts were:

- `single_well`: **580**
- `unclear`: **102**
- `double_well`: **7**

This means the estimated dynamics were overwhelmingly classified as **single-well**, with occasional unclear windows and very few double-well detections.

### Model comparison

#### Future volatility regression
- baseline RMSE: **0.001747**
- baseline + potential RMSE: **0.001751**
- baseline \(R^2\): **-0.579**
- baseline + potential \(R^2\): **-0.585**

#### Continuation classification
- baseline accuracy: **0.565**
- baseline + potential accuracy: **0.580**
- baseline AUC: **0.603**
- baseline + potential AUC: **0.600**

---

## Interpretation of the results

### 1. The structural estimation pipeline works
The project successfully:
- built rolling drift and diffusion estimates
- reconstructed implied potentials
- produced interpretable regime labels
- generated features suitable for downstream ML
- completed the full train/evaluation loop

That matters because the first hurdle in a project like this is not performance. It is getting the structural pipeline to behave consistently and produce plausible intermediate outputs.

### 2. Most windows are single-well
The regime distribution is dominated by `single_well`, and the potential snapshots shown in the plots are smooth bowl-shaped curves. That suggests the estimator is mostly seeing a single local equilibrium rather than a strong two-state structure.

This is important because it means the current configuration is not yet producing rich metastable behavior often enough for the `double_well` label to become a major predictive signal.

### 3. Double-well detection is currently too rare
Only **7** windows were labeled `double_well`. That is too sparse to support a strong claim that metastability is driving the predictions. At this stage, the double-well component is more of a detected edge case than a central explanatory regime.

This is probably the main reason the potential-based feature set does not yet materially improve the volatility model.

### 4. The volatility model is not yet strong
Both regression models have negative \(R^2\), which means neither model is outperforming a trivial mean-style benchmark in a meaningful way on this run. The two prediction lines also look smoother than the realized volatility series and miss some spikes.

That tells us the current feature set and model class are not yet sufficient for a strong volatility-forecasting result.

### 5. Potential features add only weak incremental value so far
The full model does **not** improve volatility prediction. Its RMSE is marginally worse than the baseline.

For continuation classification, the full model gives a **small increase in accuracy** from 0.565 to 0.580, but the AUC is essentially unchanged and slightly worse. That suggests the potential features may be affecting the decision threshold behavior a bit, but they are not yet creating a clearly better ranking signal.

### 6. Liquidity dominates the signal
The feature-importance chart shows that `past_liquidity_mean` is by far the most important variable in the full regression model. Potential-derived features such as `diffusion_std`, `drift_abs_mean`, `current_drift`, and `current_diffusion` do appear, but they are much smaller contributors.

That is a useful result. It suggests:
- the potential features are not pure noise
- but they are not yet the main driver of predictive performance
- the current setup is still mostly behaving like a liquidity-state model with structural overlays

---

## What the plots show

### Price series
The price chart shows clear shifts between broad price levels, which is useful for testing whether the estimator reacts differently across market states.

### Estimated regime over time
The regime timeline is dominated by `single_well`, with clusters of `unclear` windows and only a few `double_well` classifications.

### Regime counts
The bar chart confirms that the current setup is strongly imbalanced toward the single-well regime.

### Potential snapshots
The example potential curves are clean, convex, and visually interpretable. In the current run, the displayed examples are all single-well shapes, which matches the regime counts.

### Future volatility: actual vs predictions
The regression models broadly track the average level of future volatility but fail to capture larger realized moves very well. The full model and baseline look similar.

### Feature importances
The full regression model is dominated by `past_liquidity_mean`, while structural features play a secondary role.

---

## What this means for the project

This is a **good prototype** but **not yet a strong final result**.

The current run supports the claim that:

> the pipeline can estimate and use potential-based regime features in a stable and interpretable way

But it does **not yet support** the stronger claim that:

> metastability-based features clearly improve short-horizon forecasting

So the project has moved beyond the stage of being only an interesting idea, but it still needs stronger empirical lift before it becomes a standout quantitative research result.

---

## Main limitations of the current version

The current run highlights several limitations:

1. **Synthetic-data evaluation only**  
   The current public results are from synthetic data used for pipeline validation, not real market data.

2. **Regime imbalance**  
   Double-well states are too rare to drive strong results.

3. **Weak predictive uplift**  
   Potential features do not materially improve the volatility model, and only marginally affect classification accuracy.

4. **Simple model class**  
   Random forests are convenient for a prototype, but they are not necessarily the best model for structured time-series forecasting.

5. **No strict walk-forward tuning**  
   This version uses a straightforward time split rather than a more careful walk-forward cross-validation design.

6. **Structural signal may need richer state inputs**  
   The current setup uses price-based local structure plus simple liquidity/volume variables. A stronger version should condition more explicitly on market state.

---

## Recommended next steps

The clearest next improvements are:

### Improve the state representation
Add richer inputs such as:
- order-flow imbalance
- local liquidity shape
- jump indicators
- rolling spread-like proxies
- regime duration
- transition indicators

### Make regime detection less brittle
Experiment with:
- different window sizes
- different bin counts
- more smoothing choices
- stricter or looser double-well thresholds

### Strengthen evaluation
Use:
- walk-forward validation
- stronger baseline models
- calibration checks
- horizon sensitivity analysis

### Move to real data
Run the same framework on:
- Uniswap v3 data
- centralized order-book data
- other high-frequency trade-level datasets

### Add a better economic target
Instead of only forecasting volatility and continuation, test whether regime features help predict:
- adverse selection
- reversal probability after a passive fill
- transition risk
- local turbulence or liquidity deterioration

---

## Takeaway

The most important outcome of the current run is not raw performance. It is that the project now has a working end-to-end research pipeline:

- rolling structural estimation
- implied potential reconstruction
- regime classification
- feature extraction
- downstream prediction
- result visualization

The current evidence suggests that the structural features are **plausible and partially informative**, but not yet strong enough to carry the forecasting task on their own.

In one sentence:

> This project has crossed the “interesting idea” stage and reached the “working research prototype” stage, but it still needs real-data validation, stronger regime detection, and stronger empirical lift to become a standout quant result.

---

## Files produced by the notebook

The notebook saves:
- `metastability_features.pdf`
- `metastability_model_results.pdf`

It also generates plots for:
- price series
- estimated regime over time
- regime counts
- potential snapshots
- future volatility predictions
- feature importances
