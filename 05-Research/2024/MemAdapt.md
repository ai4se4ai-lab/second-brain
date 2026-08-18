# MemAdapt — Adaptive Memory-Usage Monitoring via Irregularly Sampled Data

**Type:** Full paper | **Venue:** CASCON '24 — 34th Int'l Conf. on Collaborative Advances in Software and COmputiNg (IEEE) **Authors:** Pranjal Chakraborty (Brock), Majid Babaei (McGill), Leila Tahmooresnejad (Brock), Naser Ezzati-Jivan (Brock) **DOI:** 10.1109/CASCON62161.2024.10838037 **Tags:** #AI4SE #systems-monitoring #adaptive-sampling #memory-pressure #ODE-RNN #time-series #irregular-sampling #CASCON2024

---

## 1. One-line summary

MemAdapt **dynamically adjusts how often you sample system memory metrics** by forecasting future memory usage from irregularly-spaced historical data (ODE-RNN), so monitoring stays cheap when things are calm and gets denser right before trouble — without needing constant high-frequency polling.

## 2. Problem it solves

- Memory pressure (OOM, swapping, stalls) needs monitoring to catch — but **monitoring itself has overhead**, and that overhead gets worse exactly when the system is already stressed.
- **Reactive** sampling (increase rate after pressure is detected) always lags behind the actual event.
- **Proactive** sampling (predict pressure ahead of time) needs a forecasting model — but historical monitoring data is naturally **irregularly sampled** (because the sampling rate itself keeps changing), which breaks standard time-series models that assume fixed intervals.
- Core insight: this is fundamentally a "forecasting from irregularly-sampled time series" problem, so the paper borrows from that ML subfield (Kalman filters, Neural ODEs) rather than inventing something bespoke.

## 3. Core contributions

1. **Theoretical framework** linking sampling frequency to expected memory-usage estimation accuracy — a principled way to reason about the sampling-rate/accuracy tradeoff (not just empirical tuning).
2. **A forecasting model comparison for irregular data**, identifying **ODE-RNN** as the best performer for predicting near-future memory usage and, from that, the ideal sampling rate.
3. **A concrete adaptive-sampling algorithm** (with damping to avoid rate-thrashing) that operationalizes the forecasts into real sampling-rate decisions, inspired by Google Dapper / Uber Jaeger's tracing rate-adaptation design.

## 4. Method

### 4.1 Modeling approaches compared (for irregular data forecasting)

- **RNN (baseline)** — plain recurrent model, no explicit handling of irregular gaps besides using Δt as a feature.
- **Kalman Filter (linear, KF-Lin)** — classic state-space recursive estimator: `s_t = F_t s_{t-1} + B_t u_t + w_t`, observation `m_t = M_t s_t + ε_t`.
- **Kalman Filter + LSTM smoothing (KF-LSTM)** — Kalman filter augmented with an LSTM to capture non-linear transitions.
- **ODE-RNN** — extends Neural ODEs into RNNs by modeling continuous-time hidden dynamics between observations; naturally suited to sparse/irregular data since it doesn't assume fixed time steps.

### 4.2 Data

- 5 system metrics collected via `sar`: `%memusg` (memory used), `%swpused` (swap used), `%hugused` (huge-page memory), `%user` (user-level CPU), `1 − %vmeff` (page-reclaim inefficiency).
- Smoothed via an 11-point (±5) moving average to suppress transient noise and capture broader trends.
- Irregularity is **synthetically constructed**: regularly-collected data is resampled at random intervals within a T=900s window to simulate what an adaptively-sampled trace would look like.
- Multiple dataset variants generated per lookback window size `n = 3…7` (number of preceding irregular observations used to predict the next).
- Target variable: `%memusg` at the next timestamp.

### 4.3 Adaptive sampling algorithm (Algorithm 1)

Loop: wait `1/r(n)` → sample metrics → push into circular buffer (size = lookback `n`) → feed buffer into the pretrained forecasting model → predict future memory usage → compute new candidate rate `r'(n+1)` → apply damping function `β` → update rate if change is significant → repeat.

### 4.4 Sampling-rate function `r(t)` — linear mapping from predicted memory usage to sampling rate

Piecewise-linear between a max rate `r_up` (e.g. every 0.5s) at high usage threshold `m_up` (e.g. 80%) and a min rate `r_low` (e.g. every 30–60s) at low usage threshold `m_low`:

```
r(t) = [(r_up − r_low)/(m_up − m_low)] × m + [(m_up·r_low − r_up·m_low)/(m_up − m_low)]
```

clamped at the boundaries (i.e. don't sample faster than `r_up` or slower than `r_low`).

### 4.5 Damping function `β` (prevents rate-thrashing)

Borrowed from Jaeger/Dapper's rate-adaptation design — only accept a rate change if it exceeds a max percentage increment cap `θ`:

```
β(ρ_new, ρ_old, θ) = ρ_old×(1+θ)  if (ρ_new−ρ_old)/ρ_old < 0   [i.e. big relative jump]
                    = ρ_new       otherwise
```

(worked example in paper: θ=50% — a rate change from 20s→5s interval, a 75% relative change, triggers damping/capping; a 5s→4s change, only 20%, passes through untouched.)

## 5. Experimental setup

- ~85,000 consecutive timestamps of system metrics collected over one day.
- Subsampled into irregular sequences of `n+1` points (n = 3–7) within 900s windows to simulate sparse/irregular observation.
- Train/test split: 85% / 15%.
- Evaluated via **extrapolation MSE only** (forecasting forward, not interpolating — matches the real use case).
- Second experiment: real-time evaluation using `sysbench`/`stress` to generate actual memory-usage growth scenarios, comparing the model-suggested sampling rate `r'(t)` against the "ideal" rate computed from ground-truth usage.

## 6. Key results

**Forecasting accuracy — Test MSE (×10⁻¹), lower is better:**

|Model|n=3|n=4|n=5|n=6|n=7|
|---|---|---|---|---|---|
|RNN|8.665|8.07|7.844|7.582|7.426|
|KF-Lin|10.343|10.175|10.048|9.582|9.35|
|KF-LSTM|2.249|2.214|2.178|2.12|1.997|
|**ODE-RNN**|2.304|2.198|**2.055**|**1.991**|**1.768**|

→ ODE-RNN and KF-LSTM both massively outperform plain RNN/linear-KF (~4x lower error); ODE-RNN pulls ahead as lookback `n` grows, and **longer lookback consistently improves forecasting** for all irregular-data-aware models.

**Real-time sampling-rate reliability — % of updates where predicted rate `r'(t)` was within X% of the ideal rate:**

|Model|# rate updates|within 5% (n=7)|within 10% (n=7)|within 15% (n=7)|
|---|---|---|---|---|
|RNN|417|0.573|0.591|0.621|
|KF-LSTM|366|0.701|0.734|0.76|
|**ODE-RNN**|368|**0.785**|**0.825**|**0.834**|

**Headline number: ODE-RNN hits sampling rates within 5% of ideal 78.5% of the time** (at n=7 lookback) — clearly best of the three. RNN baseline made the most rate changes (417, most "twitchy"); KF-LSTM made the fewest (366, most conservative); ODE-RNN sits in between with the best accuracy.

## 7. Limitations (as stated by authors)

- **Sudden/abrupt memory leaks** can fall between sampling events and go undetected entirely — only continuous granular sampling would catch these; this is a fundamental limitation of any adaptive/sparse sampling approach.
- Data is from **simulated** memory-usage variation (sysbench/stress-generated), not necessarily representative of real-world production workloads.
- Framework currently validated only on memory metrics — extension to CPU, network throughput, etc. is future work.

## 8. Why it matters / reusable insight

- **Forecasting-driven resource-monitoring control loop** (predict → compute optimal knob setting → damp to avoid thrashing → act) is a clean, reusable pattern for _any_ adaptive-instrumentation problem, not just memory: applies directly to log verbosity, trace sampling, telemetry frequency, etc.
- The **ODE-RNN > KF-LSTM > plain RNN/Kalman** ordering is a useful prior: whenever you're forecasting from irregularly-timed observations (a recurring theme if you instrument adaptive/self-tuning systems), ODE-RNN is worth trying first.
- The **damping-function trick** (cap relative rate-of-change, borrowed from Dapper/Jaeger) is a simple, well-tested way to stabilize any feedback-controlled parameter that's driven by noisy predictions — reusable well beyond sampling rates (e.g., stabilizing an autoscaler's replica count, or a governance system's audit-frequency knob in MindPortalix).
- Good complementary technique to pair with **SIKG**: both papers are fundamentally about "predict what matters, then act efficiently instead of exhaustively" (test selection there, sampling rate here) — same underlying philosophy of trading exhaustive coverage for informed, adaptive targeting.

## 9. Citation

> P. Chakraborty, M. Babaei, L. Tahmooresnejad, and N. Ezzati-Jivan, "MemAdapt: Adaptive Monitoring of Memory Usage through Irregularly Sampled Data," in _2024 34th International Conference on Collaborative Advances in Software and COmputiNg (CASCON)_, 2024. https://doi.org/10.1109/CASCON62161.2024.10838037