---
layout: post
title: "Mathematical approaches toward alerts"
date: 2025-12-28 09:00:00 +0000
categories: [math, observability]
tags: [alerts, prometheus, math]
---

# Statistical Anomaly Detection Techniques

A comprehensive guide to statistical methods for detecting outliers, anomalies, and shifts in time-series data. This reference details the mathematical mechanics, manual calculation examples, and specific use cases for monitoring infrastructure and application metrics.

## Table of Contents

1. [Square Root Law: Adaptive Thresholding](#square-root-law-adaptive-thresholding)
2. [Z-Score (3-Sigma Rule)](#1-z-score-3-sigma-rule)
3. [Interquartile Range (IQR)](#2-interquartile-range-iqr)
4. [Median Absolute Deviation (MAD)](#3-median-absolute-deviation-mad)
5. [Rolling Percentiles](#4-rolling-percentiles)
6. [Change-Point Detection (CUSUM)](#5-change-point-detection-cusum)
7. [Grubbs' Test](#6-grubbs-test)
8. [Generalized Extreme Studentized Deviate (ESD)](#7-generalized-extreme-studentized-deviate-esd)
9. [Kolmogorov-Smirnov (K-S) Test](#8-kolmogorov-smirnov-k-s-test)
10. [Seasonal-Trend Decomposition (STL)](#9-seasonal-trend-decomposition-stl)
11. [Exponentially Weighted Moving Average (EWMA)](#10-exponentially-weighted-moving-average-ewma)
12. [GARCH (Volatility Modeling)](#11-garch-volatility-modeling)
13. [Moving Averages & Smoothing (Simple & Exponential)](#12-moving-averages--smoothing-simple--exponential)
14. [Holt-Winters (Triple Exponential Smoothing)](#13-holt-winters-triple-exponential-smoothing)
15. [ARIMA (Autoregressive Integrated Moving Average)](#14-arima-autoregressive-integrated-moving-average)
16. [Derivative (Velocity & Acceleration)](#15-derivative-velocity--acceleration)
17. [Seasonal Derivative (Context-Aware Velocity)](#16-seasonal-derivative-context-aware-velocity)
18. [Partial Derivatives (Multi-Dimensional Anomaly Detection)](#17-partial-derivatives-multi-dimensional-anomaly-detection)

---

## Square Root Law: Adaptive Thresholding

### Origin

This approach is derived from the fundamental **Law of Large Numbers** in probability theory and the statistical concept of **Standard Error**. introduce by **Jacob Bernoulli** in the 18th century

### Core Philosophy

> "The more data you have, the less tolerance you should have for error."

* **Low Traffic (High Uncertainty):** When you only have 5 users, 1 error is statistically insignificant (20% error rate). It is likely just noise. The system should relax and not alert.
* **High Traffic (High Certainty):** When you have 50,000 users, 1 error is still negligible, but a 1% error rate is a massive disaster. The system should tighten up and alert immediately.

**The Goal:** Create a dynamic "Confidence Band" that expands and contracts automatically based on real-time volume.

---

### How It Works (The Math) — Square Root Law

The statistical noise (volatility) of a metric decreases in proportion to the **Square Root** of the sample size.

If you increase your traffic by a factor of **4**, your data becomes **2** times more reliable (). Therefore, your alert threshold can be **2** times stricter.

### The Equation

The formula to calculate the **Dynamic Error Threshold** is:

$$T(n) = \frac{k}{\sqrt{n}} + B$$

Where
$k$: **Sensitivity Factor**  Controls the "Strictness." Higher = Wider bands (fewer alerts).
$n$: **Sample Size** The number of requests in the current window
$B$: **Floor (Buffer)** The minimum error rate you will *ever* accept, even at infinite traffic. Prevents alerting on 0.001% errors.

---

### Math Example

Let's prove why this works using **** and **Window = 30m**.

#### Scenario A: Night Mode (Low Traffic)

* **Traffic:** 4 requests.
* **One Error:** Occurs. Actual Error Rate =  (**25%**).

##### Threshold Calc

* **Result:** . **NO ALERT.** (Correct, because 1 error in 4 requests is statistically meaningless).

#### Scenario B: Day Mode (High Traffic)

* **Traffic:** 10,000 requests.
* **One Error:** Occurs. Actual Error Rate = .

##### Threshold Calc

* **Result:** The threshold has automatically tightened from **76%** down to **2.5%**.

---

### Implementation (PromQL) — Square Root Law: Adaptive Thresholding

This query calculates the **Dynamic Threshold** (the Red Line in your graph).

```promql
# Logic: k / sqrt(traffic) + Floor

(
  # k (Sensitivity Factor)
  1.5 
  / 
  # Square Root of Traffic
  sqrt(
    sum by (dh_geid) (
      increase(metric{}[$time_window])
    ) 
    + 1 # Safety +1 to prevent division by zero if traffic is 0
  )
) 
+ 
# Floor (Minimum acceptable error rate)
0.01

```

---

### Tuning & Configuration — Square Root Law: Adaptive Thresholding

You can adjust the **Sensitivity Factor ()** to change the behavior of the system.

| Value | Mode | Behavior | Use Case |
| --- | --- | --- | --- |
| **1.0** | **Aggressive** | Very tight bands. Will alert faster but risks false positives on "bad luck" single errors. | Critical payments, Login services. |
| **1.5** | **Balanced** | **Recommended Standard.** Ignores single-error noise but catches real drifts. | General APIs, browsing traffic. |
| **2.0** | **Conservative** | Very loose bands. Requires strong evidence (multiple errors) to trigger. | Background jobs, Non-critical features. |
| **3.0** | **3-Sigma** | Statistical certainty. Will almost never false positive, but might miss slow-burning issues. | PagerDuty wake-up calls. |

## 1. Z-Score (3-Sigma Rule)

### The "Standard" Test

* **Origin:** Rooted in the work of Carl Friedrich Gauss (early 1800s); formalized for quality control by Walter Shewhart in 1924.
* **Core Philosophy:** "In a normal world, 99.7% of things happen near the average. Anything else is suspicious."

### How It Works (The Math) — Z-Score (3-Sigma Rule)

It assumes data follows a Gaussian (Normal) distribution. It measures the distance of a data point from the mean in units of standard deviation.

#### The Equation

$$Z = \frac{x - \mu}{\sigma}$$

Where:

* $x$ = The current data point.
* $\mu$ (Mu) = The average (mean) of the dataset.
* $\sigma$ (Sigma) = The standard deviation (spread) of the dataset.

#### Detection Threshold

* If
$$|Z| > 3$$,
it is an anomaly.

### Manual Math Example (Z-Score)

Dataset: `[10, 12, 10, 11, 12, 50]`

* **Mean ($\mu$):** $17.5$
* **Std Dev ($\sigma$):** $14.6$
  
#### Test Point (50)

$$Z = \frac{50 - 17.5}{14.6} = 2.22$$

* **Result:** Since $2.22 < 3$, the Z-score **fails** to detect this outlier because the outlier itself inflated the standard deviation (masking effect).

### Implementation (PromQL) — Z-Score (3-Sigma Rule)

```promql
(
    abs(metric - avg_over_time(metric[1h])) 
    / 
    stddev_over_time(metric[1h])) > 3
```

### Tuning & Configuration — Z-Score (3-Sigma Rule)

* Standard: 3 Sigma (99.7% confidence).

* Best For: Stable, flat metrics (e.g., Disk Usage, Fan Speed).

## 2. Interquartile Range (IQR)

### The "Robust" Fence

* **Origin:** Introduced by John Tukey in 1977 (*Exploratory Data Analysis*).
* **Core Philosophy:** "The average is easily corrupted by a single bad data point. The middle of the pack is the only thing we can trust."

### 1. How It Works (The Math)

It ignores the tails of the distribution and builds a safe zone based on the middle 50% of the data.

#### The Equation

$$IQR = Q_3 - Q_1$$

#### The Fences

$$\text{Lower Fence} = Q_1 - (1.5 \times IQR)$$
$$\text{Upper Fence} = Q_3 + (1.5 \times IQR)$$

Where:

* $Q_1$ = 25th Percentile (The bottom of the box).
* $Q_3$ = 75th Percentile (The top of the box).

### Manual Math Example (IQR)

Dataset: `[10, 12, 13, 15, 16, 1000]`

* **Sort Data:** `10, 12, 13, 15, 16, 1000`
* **Q1 (25th):** 12
* **Q3 (75th):** 16
* **IQR:** $16 - 12 = 4$
* **Upper Fence:** $16 + (1.5 \times 4) = 22$

#### Test Point (1000)

$1000 > 22$. **Detected.**

* **Result:** Unlike Z-Score, IQR detects the anomaly easily because the 1000 didn't skew the median.

### Implementation (PromQL) — Interquartile Range (IQR)

```promql
# Upper Fence Check
quantile(0.75, metric) + 
(
    1.5 * (quantile(0.75, metric) - quantile(0.25, metric))
) < metric 
```

### Tuning & Configuration — Interquartile Range (IQR)

* Multiplier:
  * 1.5: Standard Tukey fence. Captures moderate outliers.
  * 3.0: "Far out" fence. Only captures extreme anomalies.

* Best For: Noisy data with occasional spikes (Latency, Error Rates, Page Load Time).

## 3. Median Absolute Deviation (MAD)

#### The "Chaotic Data" Test

* **Origin:** Promoted by Peter Hampel (1974) and John Tukey for robust statistics.
* **Core Philosophy:** "Even the standard deviation is too sensitive. We need a deviation metric that ignores the magnitude of the disaster."

### How It Works (The Math) — Interquartile Range (IQR)

It calculates the median of the distances from the median. It is mathematically the most robust scale estimator possible (it has a 50% breakdown point, meaning 50% of our data can be infinite noise and the MAD won't budge).

#### The Equation

1. Calculate the Median of the raw data.
2. Calculate the absolute difference between every point and that median.
3. Take the Median of those differences.

$$MAD = \text{median}(|x_i - \text{median}(X)|)$$

#### The Threshold (Modified Z-Score)

To make MAD comparable to a standard deviation, we scale it by a constant ($k \approx 1.4826$) for normal distributions.
$$\text{Score} = \frac{x - \text{median}(X)}{1.4826 \times MAD}$$

### Manual Math Example (MAD)

Dataset: `[1, 1, 2, 2, 4, 6, 900]`

1. **Find Median:** The middle number is **2**.
2. **Calculate Deviations:**
    * $$|1-2| = 1$$
    * $$|1-2| = 1$$
    * $$|2-2| = 0$$
    * $$|2-2| = 0$$
    * $$|4-2| = 2$$
    * $$|6-2| = 4$$
    * $$|900-2| = 898$$
3. **List Deviations:** `[1, 1, 0, 0, 2, 4, 898]`
4. **Sort Deviations:** `[0, 0, 1, 1, 2, 4, 898]`
5. **Find MAD:** The median of this new list is **1**.
6. **Result:** The huge outlier (900) had **zero impact** on the MAD. If we used Standard Deviation, the 900 would have exploded the variance.

### Implementation (PromQL) — Median Absolute Deviation (MAD)

Note: PromQL does not natively support recursive iterations, so we approximate MAD using quantiles.

```promql
# 1. Calculate the Median
avg_over_time(quantile_over_time(0.5, metric[1h])) 

# 2. Use that to find anomalies (Simplified approximation)
abs(metric - quantile_over_time(0.5, metric[1h])) 
> 
(
    3 * 1.4826 * quantile_over_time(
        0.5, abs(metric - quantile_over_time(0.5, metric[1h])
    )[1h]))
```

### Tuning & Configuration — Median Absolute Deviation (MAD)

* The Constant (1.4826): Keep this. It aligns MAD with the Normal Distribution so we can still use "3 Sigma" logic.

* Threshold: A Modified Z-Score > 3.5 is typically considered an outlier.

* Best For: Highly volatile data (e.g., Microservice Latency, Database Query Duration) where spikes are frequent but short-lived.

## 4. Rolling Percentiles

#### The "Dynamic" Test

* **Origin:** Derived from Moving Window Statistics (widely adopted in APM tools in the 1990s).
* **Core Philosophy:** "Normal is relative. Normal at 3 PM is different from Normal at 3 AM. We only care about the *local* context."

### How It Works (The Math) — Rolling Percentiles

Instead of comparing a data point to a fixed threshold (like "Alert if > 100"), it creates a sliding window of time (e.g., the last 1 hour) and calculates a dynamic threshold based on rank order (percentiles).

#### The Equation

1. Define a window $W$ (e.g., $t-60m$ to $t$).
2. Calculate the $P_{99}$ (99th percentile) of that window.
3. Set the Trigger condition.

$$P_{99} = \text{The value below which 99\% of recent observations fall}$$
$$\text{Trigger} = \text{Current Value} > (P_{99} \times \text{Sensitivity})$$

### Manual Math Example — Rolling Percentiles

**Window (Last 5 mins):** `[100, 102, 98, 105, 101]`

1. **Sort the Window:** `[98, 100, 101, 102, 105]`
2. **Calculate P99:** For a small sample, the max value (105) is roughly the P99.
3. **New Incoming Value:** `110`
4. **Comparison:** Is $110 > 105$? **Yes.**
5. **Result:** The system adapts. If the values next week naturally rise to `200`, the P99 will also rise to `200`, so `210` will trigger an alert, but `110` (which is now low) won't.

### Implementation (PromQL) — Rolling Percentiles

This query alerts if the current value is **20% higher** than the 99th percentile of the last hour.

```promql
# Current value vs Historical Window
metric > 
(
  quantile_over_time(0.99, metric[1h]) * 1.2
)
```

### Tuning & Configuration — Rolling Percentiles

* Window Size:

  * 10m: Hyper-reactive. Good for high-frequency trading or real-time gaming.

  * 1h - 4h: Stable. Good for standard web traffic.

* Percentile:

  * P99: Very strict. Only the top 1% sets the bar.

  * P50 (Median): Can be used to detect general shifts in trend, not just spikes.

* Best For: **Capacity planning and metrics with "Slow Drift"** (e.g., User Signups, where 100/min is normal today but 500/min might be normal next year) also it would make sense to use it for **Canary rollout monitor** as the traffic is rolling in increments X% over time.

## 5. Change-Point Detection (CUSUM)

#### The "Slow Leak" Test

* **Origin:** Invented by E.S. Page in 1954 (Biometrika).
* **Core Philosophy:** "A huge spike is easy to spot. But a small error that happens *forever* is worse than a big error that happens once. We need to track the 'Debt' of the error."

### How It Works (The Math) — Change-Point Detection (CUSUM)

CUSUM (Cumulative Sum) does not look at the raw value at a specific moment. Instead, it accumulates the deviations between the observed value and a target mean over time.

#### The Equation
$$S_t = \max(0, S_{t-1} + (x_t - \mu - C))$$

Where:

* $S_t$ = The Cumulative Sum at time $t$ (The "Debt").
* $S_{t-1}$ = The previous sum.
* $x_t$ = The current data point.
* $\mu$ (Mu) = The target mean (expected value).
* $C$ = Slack parameter (allowable noise, usually 0.5 standard deviations).

#### The Logic
If $x_t$ is close to the mean $\mu$, the term $(x_t - \mu - C)$ is negative, so the `max(0, ...)` resets the sum to zero.
If $x_t$ is slightly above the mean, the sum grows. If it *stays* above the mean, the sum grows exponentially until it crosses a threshold.

### Manual Math Example — Change-Point Detection (CUSUM)

**Scenario:** Target Mean = 100. Allowable slack ($C$) = 0. Threshold = 5.
**Incoming Data:** `[100, 101, 101, 101, 101, 101]`

1. **t=0:** Value 100. Diff 0. **Sum = 0.**
2. **t=1:** Value 101. Diff +1. **Sum = 1.** (Z-score would ignore this tiny change).
3. **t=2:** Value 101. Diff +1. Sum = $1 + 1$ = **2.**
4. **t=3:** Value 101. Diff +1. Sum = $2 + 1$ = **3.**
5. **t=4:** Value 101. Diff +1. Sum = $3 + 1$ = **4.**
6. **t=5:** Value 101. Diff +1. Sum = $4 + 1$ = **5.** -> **ALARM!**

**Result:** Even though the value `101` is visually indistinguishable from `100` on a graph, CUSUM detected that the mean had *permanently shifted*.

### Implementation (PromQL) — Change-Point Detection (CUSUM)

Note: True recursive CUSUM is difficult in standard PromQL. We simulate it by summing deviations over a window.

```promql
# Calculate the Cumulative Sum of deviations over the last hour
sum_over_time(
  (
    # The deviation of the current point from the daily average
    metric - avg_over_time(metric[1d])
  )[1h:1m] # Resolution of 1 minute
) 
> 
# Threshold: If the accumulated error exceeds 50
50
```

### Tuning & Configuration — Change-Point Detection (CUSUM)

* Threshold (h): typically set to 5σ. This is the "Decision Interval."

* Slack (k or C): typically 0.5σ. This filters out normal background noise.

* Best For:
  * **Memory Leaks**: A slow, constant increase in RAM usage.

  * **Disk Usage**: Detecting when a disk will fill up based on a subtle change in write rate.

  * **Slight Regressions**: API usually responds in 100ms, now it responds in 105ms. Z-score won't catch it; CUSUM will.

## 6. Grubbs' Test

#### The "Single Outlier" Test

* **Origin:** Published by Frank E. Grubbs in 1950 (*Sample Criteria for Testing Outlying Observations*).
* **Core Philosophy:** "Is the single worst data point in this set statistically impossible?"

### How It Works (The Math) — Grubbs' Test

It calculates a specific Z-score for the most extreme value (min or max) and compares it to a critical value derived from the [T-Distribution](https://en.wikipedia.org/wiki/Student%27s_t-distribution) which is the standard for [Cauchy Distribution](https://en.wikipedia.org/wiki/Cauchy_distribution)

#### The Equation

1. Find the mean ($\bar{x}$) and standard deviation ($s$).
2. Identify the value furthest from the mean (the "Suspect").
3. Calculate the Grubbs Statistic ($G$):
   $$G = \frac{|\text{Suspect} - \bar{x}|}{s}$$
4. Compare $G$ to the Critical Value (from a [standard statistical table](https://fizyka.p.lodz.pl/media/filer_public/2c/2e/2c2e4c9c-796b-4054-9e85-ac7fc91071ec/oth2.pdf) based on sample size $N$).

### Manual Math Example (Grubbs' Test)

**Dataset (N=5):** `[10, 10, 10, 10, 50]`

1. **Calculate Mean:** $(10+10+10+10+50) / 5 = \mathbf{18}$
2. **Calculate Std Dev:** $\approx \mathbf{17.9}$
3. **Identify Suspect:** The value **50** is furthest from 18.
4. **Calculate G:**
    $$G = \frac{|50 - 18|}{17.9} = \mathbf{1.78}$$
5. **Check Critical Value:**
    * For $N=5$, the critical value (at 95% confidence) is **1.67**.
6. **Result:** Since $1.78 > 1.67$, the value **50** is statistically confirmed as an outlier.

### Implementation (PromQL) — Grubbs' Test

```promql
(
  abs(
    metric_name 
    - 
    avg_over_time(metric_name[1h]) # The Mean
  )
)
/ 
stddev_over_time(metric_name[1h])  # The StdDev

# COMPARISON:
# replace '3.2' with the Critical Value from the table 
# based on how many scrape points are in our window.
# Example: 1h window with 1m interval = 60 samples. 
# Table says Critical Value for N=60 is approx 3.0
> 3.0
```

### Tuning & Configuration — Grubbs' Test

* **Constraint:** Assumes the underlying data is Normally Distributed (Gaussian). If our data is naturally skewed (like latency), Grubbs will give false positives.
* **Limitation:** Can only detect **exactly one** outlier.
  * **The "Masking" Problem:** If the dataset was `[10, 10, 10, 50, 50]`, the Mean would shift to **26**, and the Std Dev would explode. Both 50s would "hide" each other, and the test would fail to flag either.
* **Best For:** Small, consistent datasets like **Batch Job Runtimes** or **Daily Report Totals**.

## 7. Generalized Extreme Studentized Deviate (ESD)

#### The "Gold Standard" for Spikes

* **Origin:** Bernard Rosner (1983).
* **Core Philosophy:** "Don't stop at one. Peel the outliers off the dataset like layers of an onion to prevent them from hiding each other."

### How It Works (The Math) — Generalized ESD

ESD is essentially **Grubbs' Test on a loop**. It addresses the biggest weakness of Z-Score and Grubbs: **Masking**.

* **Masking:** If we have *two* massive outliers (e.g., 100 and 101), they both inflate the standard deviation so much that *neither* looks like an anomaly mathematically. ESD solves this by removing them one by one.

#### The Algorithm

1. Define $k$ (the maximum number of outliers we suspect, e.g., 5).
2. Run the following loop $k$ times:
    * Find the most extreme value (furthest from mean).
    * Calculate the test statistic $R_i$ (same formula as Grubbs).
    * **Remove** that value from the dataset.
    * **Recalculate** the Mean and Standard Deviation (which will now be smaller and more accurate).
3. Check the Critical Value table for each Removed Point to see which ones were actually outliers.

### Manual Math Example (The "Masking" Problem)

**Dataset:** `[10, 10, 10, 50, 100]`

#### Attempt 1: Standard Z-Score

* **Mean:** 36
* **Std Dev:** $\approx 41$ (Huge!)
* **Check 50:** $(50 - 36) / 41 = \mathbf{0.34}$. (Looks totally normal).
* **Check 100:** $(100 - 36) / 41 = \mathbf{1.56}$. (Looks normal).
* **Result:** **FAIL.** The 100 "masked" the 50.

#### Attempt 2: ESD (Iterative)

* **Pass 1:** Mean=36, Std=41. Max is **100**. $R_1 = 1.56$. Remove 100.
* **Pass 2:** Dataset is now `[10, 10, 10, 50]`.
  * **New Mean:** 20.
  * **New Std Dev:** 17.
  * Max is **50**.
  * $R_2 = (50 - 20) / 17 = \mathbf{1.76}$.
* **Verdict:** Now that 100 is gone, the "noise" is gone. We see clearly that 50 is an outlier (Score 1.76 is significant for N=4).

### Implementation (PromQL) — Generalized ESD

```promql
# The Distance of the current point...
(
  metric
  - 
  # ...from the Median (instead of Mean)
  quantile_over_time(0.5, metric[1h])
)
> 
# ...is greater than X times the Interquartile Range
(
  # The spread of the middle 50% (Q3 - Q1)
  (
    quantile_over_time(0.75, metric[1h]) 
    - 
    quantile_over_time(0.25, metric[1h])
  ) 
  # The "Significance" Multiplier (Adjust this up/down)
  * 3.0
)
```

### Tuning & Configuration — Generalized ESD

* **Parameter $k$:** we must select the *maximum* number of outliers to look for (e.g., "Find up to 10 outliers").

#### Best For

* **Application Metrics (RPS, Errors):** Incidents usually cause clusters of spikes (e.g., 5 bad minutes). ESD finds all 5.
* **Cleaning Training Data:** Removing bad data points before training a Machine Learning model.

## 8. Kolmogorov-Smirnov (K-S) Test

#### The "Shape Shifter" Test

* **Origin:** Developed by Andrey Kolmogorov (1933) and Nikolai Smirnov (1948).
* **Core Philosophy:** "I don't care about the average. I care if the *shape* of the data has changed."

### How It Works (The Math) — Kolmogorov-Smirnov (K-S) Test

This is a **Non-Parametric** test. It does not assume our data is a Bell Curve. Instead, it compares the **Cumulative Distribution Function (CDF)** of two datasets:

1. **Reference Window:** (e.g., "The last 7 days").
2. **Current Window:** (e.g., "The last 10 minutes").

It calculates the maximum vertical distance between these two curves.

#### The Equation
$$D = \max_x |F_{reference}(x) - F_{current}(x)|$$

Where:

* $D$ = The K-S Statistic (The "Distance" score).
* $F(x)$ = The cumulative probability (0 to 1) at value $x$.

### Manual Math Example — Kolmogorov-Smirnov (K-S) Test

**Scenario:** Comparing API Latency between "Normal" (Baseline) and "Issue" (Current).

#### The Data

* **Baseline (Normal):** 5 requests spread evenly: `[100ms, 120ms, 140ms, 160ms, 180ms]`

* **Current (Issue):** 5 requests with a "lag spike": `[100ms, 180ms, 190ms, 190ms, 190ms]`

#### Step 1: Calculate Cumulative Probability (CDF)
We calculate what percentage of requests are less than or equal to specific thresholds ($x$).

| Threshold ($x$) | Baseline CDF ($F_b$) | Current CDF ($F_c$) | Difference ($F_b$ - $F_c$)     |
| :-------------- | :------------------- | :------------------ | :----------------------------- |
| **100ms**       | $1/5 = 0.2$          | $1/5 = 0.2$         | $\|0.2 - 0.2\| = 0.0$          |
| **120ms**       | $2/5 = 0.4$          | $1/5 = 0.2$         | $\|0.4 - 0.2\| = 0.2$          |
| **140ms**       | $3/5 = 0.6$          | $1/5 = 0.2$         | $\|0.6 - 0.2\| = 0.4$          |
| **160ms**       | $4/5 = 0.8$          | $1/5 = 0.2$         | $\|0.8 - 0.2\| = \mathbf{0.6}$ |
| **180ms**       | $5/5 = 1.0$          | $2/5 = 0.4$         | $\|1.0 - 0.4\| = 0.6$          |
| **190ms**       | $5/5 = 1.0$          | $5/5 = 1.0$         | $\|1.0 - 1.0\| = 0.0$          |

#### Step 2: Find the Statistic ($D$)
The K-S Statistic is the **maximum distance** found in the table.
$$D = 0.6$$

#### Result

* **Z-Score:** The average only moved slightly (140ms $\to$ 170ms), which might not trigger a 3-sigma alert if variance is high.

#### K-S Test Result

A distance of **0.6 (60%)** indicates a massive change in the traffic pattern. **Alert Triggered.**

### Implementation (PromQL) — Kolmogorov-Smirnov (K-S) Test

```promql
# K-S Test Approximation in PromQL
max(
  abs(
    # Part 1: Current CDF (Normalized to 0-1)
    (
      # Calculates the per-second rate for each specific bucket. 
      sum by (le) (rate(metric[10m]))
      / ignoring(le) group_left
      #Calculates the Total Rate of all requests combined.
      sum by () (rate(metric[10m]))
    )
    -
    # Part 2: Baseline CDF (Last Week)
    (
      sum by (le) (rate(metric[10m] offset 1w))
      / ignoring(le) group_left
      sum by () (rate(metric[10m] offset 1w))
    )
  )
)
```

### Tuning & Configuration — Kolmogorov-Smirnov (K-S) Test

#### The Threshold

* **$D > 0.1$:** Minor drift.
* **$D > 0.3$:** Significant change in behavior.
* **$D > 0.5$:** Totally different distribution.

#### Best For
  
* **Canary Deployments:** Did the new code version change the latency profile?
* **Bot Detection:** Humans have random latency. Bots often have "robotic" (identical) latency distributions. K-S spots the shape change instantly.

## 9. Seasonal-Trend Decomposition (STL)

#### The "Time Traveler" Test

* **Origin:** Developed by Robert Cleveland et al. (1990) at Bell Labs.
* **Core Philosophy:** "Traffic isn't random. It has a heartbeat (Seasonality) and a direction (Trend). If we surgically remove those, whatever is left is the anomaly."

### How It Works (The Math) — Seasonal-Trend Decomposition (STL)

STL assumes every metric is actually three different signals added together. It uses LOESS (Locally Estimated Scatterplot Smoothing) to separate them.

#### The Equation
$$Y_t = T_t + S_t + R_t$$

Where:

* $Y_t$ = The raw data we see.
* $T_t$ = **Trend** (The long-term direction: "Users are growing 10% per year").
* $S_t$ = **Seasonality** (The repeating pattern: "Traffic always drops at 3 AM").
* $R_t$ = **Remainder** (The Noise/Anomaly).

#### The Detection
We only care about $R_t$.
$$\text{Anomaly Score} = |Y_t - (T_t + S_t)|$$

### Manual Math Example — Seasonal-Trend Decomposition (STL)

**Scenario:** we run an ordering app.

* **Trend ($T$):** user base is slowly growing. The baseline is **1,000 users**.
* **Seasonality ($S$):** we have a massive spike every morning at **9:00 AM** when people ordering before work. The pattern adds **+500 users**.
* **Noise ($R$):** Random fluctuation is usually around $\pm 50$ users.

#### The Equation
$$Y_{observed} = \text{Trend} + \text{Seasonality} + \text{Remainder}$$

### Case 1: A Normal Tuesday (9:00 AM)

1. **Calculate Expected Value:**
    * **Trend:** 1,000
    * **Seasonality:** +500 (It is 9 AM!)
    * **Prediction:** $1,000 + 500 = \mathbf{1,500}$
2. **Measure Reality:** we observe **1,520** users.
3. **Find the Remainder:**
    $$R = 1,520 - 1,500 = \mathbf{+20}$$
4. **Verdict:** $+20$ is within normal noise ($\pm 50$). **System Healthy.**

#### Case 2: The Anomaly (Wednesday 9:00 AM)

A bug in the app is preventing 50% of users from logging in.

1. **Measure Reality:** we observe **1,050** users.
2. **The "Static Threshold" Trap:**
    * If we had a standard alert set to *"Page me if users drop below 800"*, **we would NOT get alerted.**
    * Why? Because 1,050 is technically a "healthy" number (it is higher than 800).
    * The system thinks everything is fine, but we are losing revenue.
3. **The STL Calculation:**
    * **Prediction:** Still **1,500** (The model knows it is 9 AM).
    * #### Remainder
    $$R = 1,050 - 1,500 = \mathbf{-450}$$
4. **Verdict:** $-450$ is **9x** larger than normal noise ($9\sigma$). **CRITICAL ALERT.**

#### Summary Comparison

| Metric               | Normal Day | Anomaly Day     | Logic                                           |
| :------------------- | :--------- | :-------------- | :---------------------------------------------- |
| **Observed ($Y$)**   | 1,520      | **1,050**       | What the dashboard shows.                       |
| **Expected ($T+S$)** | 1,500      | 1,500           | What *should* happen at 9 AM.                   |
| **Remainder ($R$)**  | **+20**    | **-450**        | The Deviation ($Y - \text{Expected}$).          |
| **Z-Score Alert**    | Safe       | **Safe (FAIL)** | 1050 looks "Average" compared to the whole day. |
| **STL Alert**        | Safe       | **ALARM**       | It noticed the *missing* 450 users.             |

### Implementation (PromQL) — Seasonal-Trend Decomposition (STL)

Note: True LOESS decomposition is too complex for standard PromQL. We cannot do full LOESS decomposition in PromQL, but we can build a very strong approximation using **Historical Averaging**.

#### The Logic

1. **Calculate Baseline ($T+S$):** Instead of just looking at "Last Week," we take the average of the last 3 weeks at this exact specific minute. This smooths out one-off outliers from history.
2. **Calculate Deviation ($R$):** `Current Value - Baseline`.
3. **Dynamic Threshold:** Triggers if the deviation is bigger than **2 Standard Deviations** of the recent data.

```promql
# 1. Calculate the expected value (Trend + Seasonality)
# We use the value from exactly one week ago to capture the "Season"
(
  metric 
  - 
  metric offset 1w
) 
> 
# 2. Threshold
# Allow for some variance (e.g., 20% drift is okay)
(metric offset 1w) * 0.2
```

**Advanced "Smooth" Version**: Instead of just one data point (which might be noisy), average the last 3 weeks:

```promql
# 1. DEFINE THE EXPECTED VALUE (The Baseline)
# We average the metrics from 1, 2, and 3 weeks ago to find the "Normal" level for this specific time.
with (
  baseline = (
    metric offset 1w + 
    metric offset 2w + 
    metric offset 3w
  ) / 3
)

# 2. THE ALERT CONDITION
# Trigger if the absolute difference is huge
abs(metric - baseline)

> 

# 3. THE DYNAMIC THRESHOLD
# We check if the deviation is greater than 2 Standard Deviations (95% confidence)
# We calculate stddev over the last hour to see how "noisy" the data is right now.
(
  2 * stddev_over_time(metric[1h])
)
# OR (Alternative): Trigger if off by more than 20% of the baseline
# (
#   0.20 * baseline
# )
```

### Tuning & Configuration — Seasonal-Trend Decomposition (STL)

* Seasonality Window:
  * Daily: Compare to offset 1d (Good for simple apps).
  * Weekly: Compare to offset 1w (Critical for business apps where Monday $\ne$ Sunday)
* Best For:
  * Business Metrics: Orders per minute, Login rates.
  * Cyclical Traffic: Identifying a dip in traffic on "Black Friday" (where high traffic is expected) versus a dip on a random Tuesday.

## 10. Exponentially Weighted Moving Average (EWMA)

#### The "Short Term Memory" Test

* **Origin:** C.C. Holt (1957).
* **Core Philosophy:** "Not all data is equal. What happened 1 minute ago is more important than what happened 10 minutes ago."

### How It Works (The Math) — Exponentially Weighted Moving Average (EWMA)

A standard moving average gives every point the same weight. EWMA applies a "decay factor" ($\alpha$) so that older data fades away exponentially. It adapts faster to shifts than a simple average.

#### The Equation
$$S_t = \alpha \cdot x_t + (1 - \alpha) \cdot S_{t-1}$$

Where:

* $S_t$ = The new EWMA value.
* $x_t$ = The current raw data point.
* $S_{t-1}$ = The previous EWMA value.
* $\alpha$ (Alpha) = The smoothing factor ($0 < \alpha < 1$).
  * High $\alpha$ (e.g., 0.9): Fast reaction (ignores history).
  * Low $\alpha$ (e.g., 0.1): Slow reaction (smooths out noise).

### Manual Math Example — Exponentially Weighted Moving Average (EWMA)

**Scenario:** Detecting a CPU spike.
**Params:** $\alpha = 0.5$ (Balanced).
**Data:** `[10, 10, 10, 50]`

1. **t=1 (Value 10):** Start at 10. $EWMA = 10$.
2. **t=2 (Value 10):** $0.5(10) + 0.5(10) = \mathbf{10}$.
3. **t=3 (Value 10):** $0.5(10) + 0.5(10) = \mathbf{10}$.
4. **t=4 (Value 50):**
    $$EWMA = 0.5(50) + 0.5(10) = 25 + 5 = \mathbf{30}$$

#### Comparison

* **Simple Average (last 4):** $(10+10+10+50)/4 = \mathbf{20}$.
* **EWMA:** $\mathbf{30}$.
* **Result:** EWMA reacted much faster to the spike (jumped to 30) than the simple average (only reached 20).

### Implementation (PromQL) — Exponentially Weighted Moving Average (EWMA)

Prometheus uses the `holt_winters` function, which is based on EWMA logic (specifically, double exponential smoothing).

```promql
# Smooth the metric using Holt-Winters to remove jitter
# 0.5 = Smoothing Factor (Old data is less important)
# 0.5 = Trend Factor (React to changes in slope)
holt_winters(metric[10m], 0.5, 0.5)
```

**Custom EWMA Simulation:** We can also simulate a pure EWMA by comparing a "Fast" average against a "Slow" average (MACD style).

```promql
# 1. Calculate the Divergence (The Gap)
# We compare the "Short Term Memory" (5m) vs "Long Term Memory" (1h)
abs(
  rate(http_requests_total[5m]) 
  - 
  rate(http_requests_total[1h])
)

> 

# 2. Dynamic Threshold (The Guardrail)
# We trigger if the Gap is bigger than 3 Standard Deviations of the recent history.
# This prevents alerts during "noisy" times but catches sudden spikes.
(
  3 * stddev_over_time(rate(http_requests_total[1h])[1h:])
)
```

### Tuning & Configuration — Exponentially Weighted Moving Average (EWMA)

* The Alpha ($\alpha$):
  * Use 0.1 for very noisy data (smoothing).
  * Use 0.8 for critical alerts where every second counts.
* Best For:
  * Network Traffic: Smoothing out "bursty" packet data to see the true trend.
  * FinOps: Detecting cost anomalies in real-time billing.

## 11. GARCH (Volatility Modeling)

### The "Panic Detector"

* **Origin:** Robert Engle (1982) and Tim Bollerslev (1986). (Won the Nobel Prize in Economics).
* **Core Philosophy:** "Panic clusters together. If the market was crazy yesterday, it will likely be crazy today, even if there is no new news."

#### How It Works (The Math) — GARCH (Volatility Modeling)

Standard statistics assume that "Risk" (Standard Deviation) is constant. GARCH assumes that Risk changes over time.

It models the **Variance** ($\sigma^2$) rather than the value itself.
The volatility of *today* depends on three things:

1. **Baseline ($\omega$):** The minimum background noise.
2. **Recent Shocks ($\alpha$):** "Did something explode yesterday?"
3. **Recent Volatility ($\beta$):** "Was everyone already panicking yesterday?" (The Memory).

#### The Equation

$$\sigma^2_t = \omega + \alpha \cdot \epsilon^2_{t-1} + \beta \cdot \sigma^2_{t-1}$$

##### Manual Math Example — GARCH (Volatility Modeling)

**Scenario:** Monitoring Latency Jitter.

* **Normal Variance ($\omega$):** 1.
* **Alpha ($\alpha$):** 0.1 (Reaction to new shocks).
* **Beta ($\beta$):** 0.8 (Memory of old panic).

###### Day 1 (Calm):

Variance is low (5). Everything is fine.

###### Day 2 (The Incident):

A database fails. A huge shock occurs ($\epsilon^2 = 100$).

* **New Variance:** $1 + 0.1(100) + 0.8(5) = 1 + 10 + 4 = \mathbf{15}$.
* Result: System enters "High Alert" mode.

###### Day 3 (The Aftermath):

The database is fixed. The shock is gone ($\epsilon^2 = 0$).

* **Standard Prediction:** Variance should drop instantly to 1.
* **GARCH Prediction:** $1 + 0.1(0) + 0.8(15) = 1 + 0 + 12 = \mathbf{13}$.
* **Result:** Even though the issue is fixed, GARCH keeps the risk level high (13) because it knows systems take time to stabilize.

### Implementation (PromQL) — GARCH (Volatility Modeling)

Note: True GARCH requires recursive loops (using yesterday's result to calculate today's). PromQL cannot do this. We approximate it by comparing "Current Volatility" vs "Long-Term Average Volatility".

We want to detect if we are in a **High Volatility Cluster**.

```promql
# 1. Calculate Current Volatility (The last 10 minutes)
with (
  current_volatility = stddev_over_time(rate(metric[1m])[10m:])
)

# 2. Compare to Baseline Volatility (The last 1 hour)
# If current jitter is 2x higher than normal jitter
current_volatility 
> 
2 * avg_over_time(
  stddev_over_time(rate(metric[1m])[10m:])[1h:]
)
```

### 4. Tuning & Configuration

* The Threshold (2x or 3x):

  * In Infrastructure (CPU), volatility should be low. If it triples, something is wrong.

* Best For:

  * FinOps

  * Latency Jitter: Detecting "Micro-bursts" where the average latency looks fine, but the variance has exploded (meaning users are experiencing unpredictable slowness).

# Timeseries Anomaly Detection Techniques

## 12. Moving Averages & Smoothing (Simple & Exponential)

### The "Noise Filter"

* **Origin:** Standard statistical tool used since the early 20th century, notably in finance and signal processing.
* **Core Philosophy:** "The world is noisy. Don't react to every bump in the road; look at the path the road is taking."

### How It Works (The Math) — Moving Averages & Smoothing

#### Simple Moving Average (SMA)

Calculates the unweighted mean of the previous $n$ data points. It is "slow" because old data drags it down.

* **Formula:** $SMA = \frac{p_1 + p_2 + \dots + p_n}{n}$

#### Exponential Moving Average (EMA)

Gives more weight to recent data. It reacts faster to new trends while still smoothing out noise.

* **Formula:** $EMA_t = \alpha \cdot x_t + (1 - \alpha) \cdot EMA_{t-1}$
  * $\alpha$ (Alpha): The smoothing factor ($0 < \alpha < 1$). Higher = Faster reaction.

### Manual Math Example — Moving Averages & Smoothing

**Scenario:** CPU Load spikes.
**Data:** `[10, 10, 10, 90]` (Sudden spike to 90%).

#### SMA (Simple Average of 4 points)

* Calculation: $(10+10+10+90) / 4 = 120 / 4 = \mathbf{30}$.
* *Verdict:* The spike is diluted heavily.

#### EMA (Alpha = 0.5)

* Step 1 (Value 10): EMA = 10.
* Step 2 (Value 10): $0.5(10) + 0.5(10) = 10$.
* Step 3 (Value 10): $0.5(10) + 0.5(10) = 10$.
* Step 4 (Value 90): $0.5(90) + 0.5(10) = 45 + 5 = \mathbf{50}$.
* *Verdict:* The EMA jumped to **50**, reacting much faster than the SMA (30).

### Implementation (PromQL) — Moving Averages & Smoothing

#### Simple Moving Average (SMA)

This is the standard `rate()` or `avg_over_time()` over a window.

```promql
# Calculate the average over the last 10 minutes
avg_over_time(process_cpu_seconds_total[10m])
```

#### Exponential Moving Average (EMA)

Exponential Moving Average (EMA): Prometheus uses `holt_winters` for exponential smoothing.

```promql
# Smooth the metric using Holt-Winters
# 0.5 = Smoothing Factor (Alpha)
# 0.5 = Trend Factor (Beta)
holt_winters(process_cpu_seconds_total[10m], 0.5, 0.5)
```

The "MACD" Strategy (Compare SMA vs EMA): Detect anomalies by comparing a fast-moving average against a slow-moving one.

```promql
# Trigger if the "Fast" average (5m) is 20% higher than the "Slow" average (1h)
rate(metric[5m]) 
> 
rate(metric[1h]) * 1.2
```

### Tuning & Configuration — Moving Averages & Smoothing

* Window Size:

  * Short (1m - 5m): Good for real-time alerting but noisy.
  * Long (1h - 4h): Good for capacity planning and trend analysis.

* Best For:

  * Visualizing Trends: Making jagged graphs readable.
  * Removing Jitter: Preventing alerts from firing on single-second spikes.

## 13. Holt-Winters (Triple Exponential Smoothing)

#### The "Seasonality Expert"

* **Origin:** Charles Holt (1957) and Peter Winters (1960).
* **Core Philosophy:** "A simple average is blind. To predict the future, you must understand three things: Where we are (Level), where we are going (Trend), and what time of year it is (Seasonality)."

### How It Works (The Math) — Holt-Winters (Triple Exponential Smoothing)

While Simple EWMA tracks one line, Holt-Winters tracks three separate components and combines them to make a prediction.

#### The Core Components

1. **Level ($\ell_t$):** The baseline value (smoothed) right now.
2. **Trend ($b_t$):** The speed of growth (slope) per time unit.
3. **Seasonality ($s_t$):** The repeating cyclical pattern (e.g., "Mondays are always +20%").

#### The Time Variables

1. **Current Time ($t$):** The specific moment "now" when the calculation runs.
2. **Horizon ($h$):** The number of steps into the future we are predicting.
3. **Period ($m$):** The length of one full cycle (e.g., 24 hours).

#### The Equation (simplified)

$$y_{t+h} = \ell_t + h b_t + s_{t+h-m}$$

*Meaning:* The forecast = Current Level + (Trend $\times$ Future Time) + The Seasonal Adjustment from the last cycle.

### Manual Math Example — Holt-Winters (Triple Exponential Smoothing)

**Scenario:** Predicting Web Traffic for 8:00 PM (3 hours from now).

* **Level ($\ell$):** 1000 users (Current baseline).
* **Trend ($b$):** +10 users per hour (Growth speed).
* **Seasonality ($s$):** Historically, 8:00 PM is "Prime Time" and adds **+500 users**.
* **Horizon ($h$):** 3 (Predicting 3 hours ahead).

#### Prediction for 3 hours from now ($h=3$)

1. **Base Prediction:** $1000 + (3 \times 10) = 1030$.
2. **Add Seasonality:** $1030 + 500 = \mathbf{1530}$.

#### Why this matters

A simple linear trend would predict **1010**.
Holt-Winters predicts **1530**.
If the actual traffic is **1500**, the simple model would scream "ANOMALY!" (Unexpected Spike). Holt-Winters says, "Relax, this is exactly what I expected."

### 3. Implementation (PromQL)

Note: **Double Exponential Smoothing** (Level + Trend). It does not natively handle the "Seasonal" (Gamma) component in the function arguments. To handle Seasonality, we usually combine it with `offset` logic.

#### Standard Trend Smoothing

```promql
# Smooth the metric based on Level and Trend
# sf (Smoothing Factor): 0.5 (Focus on recent data)
# tf (Trend Factor): 0.1 (Assume trends change slowly)
double_exponential_smoothing(metric[1h], 0.5, 0.1)
```

Full "Triple" Simulation (Trend + Seasonality): To get the full power of Holt-Winters in PromQL, we compare the current value against a "Seasonal Baseline" that is also smoothed.

```promql
# Alert if Current Value deviates from (Last Week's smoothed trend)
rate(metric[5m])
>
# Calculate the expected value using Holt-Winters on last week's data
double_exponential_smoothing(rate(metric[5m] offset 1w)[1h], 0.5, 0.1) 
* 1.2 # Allow 20% deviation
```

### Tuning & Configuration — Holt-Winters (Triple Exponential Smoothing)

The function holt_winters(range, sf, tf) takes two parameters:

* Smoothing Factor (sf): * Low (0.1): Very smooth, ignores spikes.

  * High (0.9): Jittery, hugs the raw data closely.

* Trend Factor (tf):

  * Low (0.1): Assumes the trend (slope) is constant/stable.

  * High (0.9): Assumes the trend changes rapidly (e.g., huge variance in traffic during short period of time).

Best For:

* Capacity Planning: "Based on the last 3 months, when will the disk fill up?"

* Noisy Metrics: Smoothing out CPU usage that spikes every time a cron job runs

## 14. ARIMA (Autoregressive Integrated Moving Average)

#### The "Time Traveler"

* **Origin:** George Box and Gwilym Jenkins (1970).
* **Core Philosophy:** "The future is an echo of the past. By analyzing the echoes (Autoregression) and the mistakes we made yesterday (Moving Average), we can pinpoint exactly where we will be tomorrow."

### How It Works (The Math) — ARIMA (Autoregressive Integrated Moving Average)

ARIMA is a heavy statistical model that combines three distinct techniques to tame messy data. It is usually denoted as **ARIMA(p, d, q)**.

1. **AR (Autoregression - $p$):** "History repeats itself."
    * Today's value depends on yesterday's value.
2. **I (Integrated - $d$):** "Flatten the hill."
    * We subtract consecutive values (Differencing) to remove trends and make the data "Stationary" (flat).
3. **MA (Moving Average - $q$):** "Learn from mistakes."
    * Today's value depends on the *prediction error* (noise) of yesterday.

#### The Equation (Simplified AR(1) model)

$$Y_t = C + \phi_1 Y_{t-1} + \theta_1 \epsilon_{t-1} + \epsilon_t$$

* $\phi_1 Y_{t-1}$: The "Echo" from yesterday (AR).
* $\theta_1 \epsilon_{t-1}$: The "Correction" from yesterday's error (MA).

### Manual Math Example — ARIMA (Autoregressive Integrated Moving Average)

**Scenario:** Memory Usage Growth.

* **Day 1:** 40% (Prediction was 38%. Error $\epsilon = +2$).
* **Day 2:** 42% (Prediction was 41%. Error $\epsilon = +1$).

#### Predicting Day 3

We believe the memory grows based on the previous day ($0.9 \times Y_{t-1}$) plus a correction for yesterday's error ($0.5 \times \epsilon_{t-1}$).

1. **AR Component:** $0.9 \times 42\% = 37.8\%$.
2. **MA Component:** $0.5 \times 1\% \text{ (Yesterday's error)} = 0.5\%$.
3. **Total Prediction:** $37.8 + 0.5 = \mathbf{38.3\%}$.

*Result:* If Day 3 usage comes in at **50%**, ARIMA screams "Anomaly!" because it expected 38.3%.

### Implementation (PromQL) — ARIMA (Autoregressive Integrated Moving Average)

Note: True ARIMA requires complex iterative solving (maximum likelihood estimation) which PromQL **cannot** do natively. However, we can approximate the "Integrated" (Trend) and "Autoregressive" parts using Linear Regression.

We use `predict_linear` to simulate the forecasting capability of ARIMA.

```promql
# 1. The Alert Condition
# Trigger if the Actual Value deviates from the Linear Prediction
abs(
  # A. The Actual Current Value
  sum by (XX) (rate(metric{...}[5m]))
  
  -
  
  # B. The "ARIMA-lite" Prediction
  # Predict the future value based on the trend of the last hour
  predict_linear(
    sum by (XX) (rate(metric{...}[5m]))[1h:5m], 
    0 # Predict 0 seconds into the future (Compare trend vs reality NOW)
  )
)

> 

# 2. The Threshold (Standard Deviation)
# Allow deviation up to 2 standard deviations of the recent history
2 * stddev_over_time(
  sum by (XX) (rate(metric{...}[5m]))[1h:5m]
)
```

Why this works as an approximation:

* AR (AutoRegressive): predict_linear looks at the past points [1h] to draw a line.

* I (Integrated): rate() handles the differencing, turning counters into stationary speed.

* Validation: If the data suddenly jumps off the trend line, the difference (A - B) becomes huge.

### Tuning & Configuration — ARIMA (Autoregressive Integrated Moving Average)

* The Window [1h:5m]:

  * Short (15m): Catches extremely sharp, sudden turns.

  * Long (6h): Catches slow drifts (like a memory leak).

* The Prediction Delta (The 0 in predict_linear):

  * 0: Checks "Does the current value match the current trend?" (Anomaly Detection).

  * 3600 (1h): Checks "Will we run out of disk space in an hour?" (Forecasting).

Best For:

* Disk Space / Memory: Metrics that have strong, linear trends (filling up).

* Organic Growth: User registration counters.

# Using Derivative

## 15. Derivative (Velocity & Acceleration)

### The "Car Crash" Detector

* **Core Philosophy:** "I don't care if the car is fast (High Value). I care if it stops suddenly (Negative Derivative) or accelerates uncontrollably (Positive Derivative)."
* **The Math:** The derivative measures the **Slope** (Rate of Change).
  * **First Derivative (Velocity):** How fast is the value changing?
  * **Second Derivative (Acceleration):** Is the problem getting worse?

### How It Works

Most alerts trigger on **Value** (e.g., "CPU > 90%").
Derivative alerts trigger on **Speed** (e.g., "CPU jumped 50% in 1 minute").

This is critical for **"Flash Crowds"** or **"Crash Loops"** where the absolute number might still be low, but the *trend* is terrifying.

#### The Equation

$$\frac{dy}{dt} \approx \frac{y_{current} - y_{previous}}{\Delta time}$$

### Manual Math Example — Derivative (Velocity & Acceleration)

**Scenario:** A Memory Leak.

* **Minute 1:** 20% used.
* **Minute 2:** 21% used. (Change = +1%. Boring.)
* **Minute 3:** 80% used. (Change = **+59%**. PANIC!)

* **Standard Alert (>90%):** Sleeping. (80% is "Safe").
* **Derivative Alert:** **FIRES IMMEDIATELY**. It saw the massive jump.

### Implementation (PromQL) — Derivative (Velocity & Acceleration)

#### For Gauges (Memory, Queue Size, Thread Count)

Use the native `deriv()` function. It calculates the slope per second.

```promql
# Alert if Memory Usage is changing faster than 1% per second
# (Which means it will fill up in less than 2 minutes)
abs(
  deriv(process_resident_memory_bytes[5m])
) 
> 
# Threshold: 10 MB per second (Adjust unit as needed)
10 * 1024 * 1024
```

#### For Counters (Error Rates - The "Second Derivative")

Since metric is a Counter, its "Rate" is already the First Derivative. To see if the Error Rate is Accelerating (getting worse), we calculate the Difference in Rates.

Note: Since your Prometheus might struggle with subqueries `(deriv(rate(...)[...]))`, we use the robust "Delta" comparison method.

```promql
# Calculate the "Acceleration" of errors
# (Current Error Speed - Error Speed 2 minutes ago)

(
  # 1. Speed NOW
  sum(rate(metric{http_status_code=~"5.*"}[2m]))
  - 
  # 2. Speed 2 minutes ago
  sum(rate(metric{http_status_code=~"5.*"}[2m] offset 2m))
)
> 
# 3. The Threshold (Acceleration Limit)
# Trigger if the error rate jumps by more than 5 errors/sec in just 2 minutes.
5
```

### Tuning & Configuration — Derivative (Velocity & Acceleration)

* The Window [5m] (in deriv(metric[5m])):

  * Short (1m): Extremely sensitive "Twitch" response. Catches instant spikes but is very noisy.

  * Medium (5m): The standard. Measures sustainable speed over a few minutes.

  * Long (1h): Measures long-term momentum. Useful for detecting slow but unstoppable memory leaks.

* The Threshold (The Limit):

  * High Value: Only alerts on catastrophic explosions (e.g., Queue grows by 10k/sec).

  * Low Value: Alerts on any sudden movement. Good for stable systems where any change is suspicious.

* Best For:

  * Queue Monitoring: SQS / RabbitMQ / Kafka lag exploding.

  * Flash Crowds: Sudden traffic spikes that haven't hit the absolute "Total" limit yet.

  * Crash Detection: When a metric drops to zero instantly (Negative Derivative).

Here are the two advanced Derivative-based approaches formatted in Markdown.

## 16. Seasonal Derivative (Context-Aware Velocity)

#### The "Morning Rush" Detector

* **Core Philosophy:** "Fast growth is fine if it happens every day at this time (e.g., 9 AM login spike). It is only bad if it happens when the system should be quiet (e.g., 3 AM)."
* **The Math:** Compare **Current Velocity** vs. **Historical Velocity** (same time last week).

#### The Logic
Instead of checking `Velocity > 10`, we check:
`Current Velocity > (Historical Velocity + Buffer)`

#### Implementation (PromQL)

```promql
# Alert if RPS is growing much faster than it did last week
(
  # 1. Current Acceleration (RPS Now - RPS 2 mins ago)
  (
    sum(rate(metric[2m])) 
    - 
    sum(rate(metric[2m] offset 2m))
  )
  - 
  # 2. Historical Acceleration (Same time last week)
  (
    sum(rate(metric[2m] offset 1w)) 
    - 
    sum(rate(metric[2m] offset 1w2m))
  )
)
> 
# 3. Threshold: Allowable difference in acceleration
# "Growing by 500 req/sec faster than usual is scary"
500
```

### Tuning & Configuration — Seasonal Derivative (Context-Aware Velocity)

#### The Window [10m]

* **Short (5m):** Highly sensitive to small timing mismatches between weeks.
* **Long (30m):** Better for matching broad trends (e.g., the general "morning ramp-up").

#### Best For

* **User Login Storms:** Distinguishing a DDoS attack from a normal Monday morning login rush.
* **Batch Jobs:** Ignoring the massive CPU spike that happens every night at 2 AM during backup.

### Statistical Derivative (Z-Score of Velocity)

#### The "Unusual Movement" Detector

* **Core Philosophy:** "I don't know what the 'Speed Limit' is. Just tell me if the metric is moving *weirdly* fast compared to the last hour."
* **The Math:** Apply **Z-Score (Standard Deviation)** to the **Derivative**.

#### The Logic
We calculate the Standard Deviation of the *Derivative* itself.

* If the metric usually changes slowly (StdDev is low), a small jump triggers it.
* If the metric is naturally chaotic (StdDev is high), it waits for a massive jump.

#### Implementation (PromQL)

```promql
# 1. Calculate how fast the Error Rate is changing (Absolute Change)
# We use idelta() on the rate to see the instant jump between samples
abs(
  idelta(
    sum(rate(metric{http_status_code=~"5.*"}[$time_window]))[$time_window:]
  )
)
> 
# 2. Dynamic Threshold
# "Is this jump 3x bigger than the standard volatility of the last hour?"
3 * stddev_over_time(
  idelta(
    sum(rate(metric{http_status_code=~"5.*"}[$time_window]))[$time_window:]
  )[1h:]
)
```

### Tuning & Configuration — Statistical Derivative (Z-Score of Velocity)

#### The Sigma Multiplier (3 * ...)

* **2 Sigma:** Sensitive. Catches minor deviations in speed.
* **3 Sigma:** Robust. Only catches truly "alien" behavior.

#### Best For

* **Memory Leaks:** Detecting a leak that suddenly accelerates (e.g., a "slow leak" turns into a "fast leak").
* **Stock/Crypto Prices:** Detecting market volatility rather than high prices.

## 17. Partial Derivatives (Multi-Dimensional Anomaly Detection)

#### The "Root Cause" Detector

* **Core Philosophy:** "In a complex system, nothing happens in a vacuum. If CPU goes up, is it because Traffic went up (Normal), or because Traffic stayed flat (Anomaly)?"
* **The Math:** A **Partial Derivative** ($\frac{\partial f}{\partial x}$) measures how a function changes when you move *one* variable while holding others constant.
  * $\frac{\partial \text{CPU}}{\partial \text{Requests}}$: "How much does CPU increase for every 1 new request?"

### How It Works

Standard monitoring looks at metrics in isolation (1D).
Partial Derivatives look at the **relationship** between metrics (2D or 3D).

#### The Equation (Simplified)

$$\text{Efficiency Index} = \frac{\Delta \text{Resource Usage}}{\Delta \text{Workload}}$$

### Manual Math Example — Partial Derivatives (Multi-Dimensional Anomaly Detection)

**Scenario:** A Database Server.

* **Normal Behavior:** Every 100 queries consume 1% CPU.
  * Ratio: $1\% / 100 = 0.01$.
* **Anomaly (Bad Query):** Traffic is steady (100 queries), but CPU jumps to 50%.
  * Ratio: $50\% / 100 = 0.5$.

* **Standard Alert (CPU > 80%):** Does not fire (50% is "safe").
* **Standard Alert (Traffic drop):** Does not fire (Traffic is normal).
* **Partial Derivative Alert:** **FIRES**. The *relationship* broke. The "Cost per Query" jumped 50x.

### Implementation (PromQL) — Partial Derivatives (Multi-Dimensional Anomaly Detection)

We don't need complex calculus in PromQL. We simply calculate the **Ratio of Rates**.

#### The "Cost Per Request" Alert (CPU vs RPS)

Detects inefficient code, bad queries, or "Death Spirals" where CPU rises without traffic growth.

```promql
# Alert if CPU usage per Request becomes too high
# (Efficiency degrades)

(
  # 1. CPU Rate (Resource)
  rate(container_cpu_usage_seconds_total[5m])
  / 
  # 2. Request Rate (Workload)
  rate(metric[5m])
)

> 

# 3. Dynamic Threshold
# "Is the cost per request 2x higher than it was last week?"
2 * (
  rate(container_cpu_usage_seconds_total[5m] offset 1w)
  / 
  rate(metric[5m] offset 1w)
)
```

#### The Latency vs Throughput Check (Little's Law violation)

increases, Latency usually increases slightly. If Throughput is flat but Latency explodes, you have a downstream bottleneck (Database/Disk).

```promql
# Alert if Latency increases WITHOUT a corresponding increase in traffic
# (This isolates the issue to the app/backend, ruling out "high load")

(
  # 1. Change in Latency
  deriv(metric[5m]) 
  > 0 
)
and
(
  # 2. Traffic is stable or dropping
  deriv(metric[5m]) <= 0
)
```

### Why this is powerful

**Eliminates "Noise"**: It won't alert you just because traffic spiked and CPU went up (that's normal). It only alerts if CPU goes up disproportionately.

**Finds "Invisible" Regressions**: A developer pushes code that makes the JSON parser 10% slower. CPU rises slightly. No thresholds are breached. But the Partial Derivative (CPU/RPS) instantly shifts by 10%.

### Best For

* **Efficiency Monitoring**: "Are we getting the same performance per dollar?"

* **DDoS Detection**: Distinguishing "High Traffic" (High RPS, Normal CPU/RPS ratio) from "Attack" (Low RPS, Massive CPU usage due to complex attack vectors).

* **Database Profiling**: Detecting "Table Scans" (Low query count, High IOPS).

## Benfits of Derivative comparing to threashold based ops

### Speed vs. Position (Reaction Time)Plain

* **Comparison (Position)**: Alert me if the car hits the wall.
* **Derivative (Velocity)**: Alert me if we are driving 100 mph towards the wall.

The Problem with Normal Rates: If you set an alert for Disk Usage > 90%, you are waiting until the disaster has almost happened.

* Scenario: A log file goes rogue and fills the disk at 1 GB per second.
  * Normal Alert: Waits 10 minutes until disk hits 90%. You have 10 seconds to fix it. Too late.
  * Derivative Alert: Sees the slope jump from 0 GB/s to 1 GB/s instantly. It alerts you when the disk is still at 40%, giving you 10 minutes to fix it.2.

### Scale Independence (The "Baseline" Problem)

* **Plain Comparison:** Requires you to know the "Magic Number" (Threshold).
* **Derivative:** Only cares about the Change.

#### The Problem with Percentages

Imagine a queue.

* Day: You process 1,000 items/sec. A threshold of Queue > 500 is useless (always firing).
* Night: You process 10 items/sec. A threshold of Queue > 500 is useless (never fires, even if the queue is stuck).

The Derivative Solution:You alert on deriv(queue_size) > 50.

* This asks: "Is the queue growing rapidly?"
* It works perfectly during the day (1000 $\to$ 1050) and perfectly at night (10 $\to$ 60).
* It ignores the size of the queue and focuses on the health of the consumer
  
### Detecting "Frozen" States (The Zero Derivative)

Sometimes, the most dangerous anomaly is Silence (or artificial perfection). Real distributed systems are "jittery." If a metric becomes perfectly flat, it usually means the monitoring agent has crashed, the logic is stuck, or the data is stale.

#### Error Rate (The "False Success" Freeze)

**Scenario:** Your logging sidecar crashes. The main app continues running but stops reporting errors. Prometheus sees "0 errors" forever.

* **Normal Alert (Error Rate > 1%)**: Sees 0%. Thinks "Great job, system is perfect!"

* **Derivative Alert**: Detects that the natural "background noise" of errors has vanished.

#### RPS (The "Load Balancer Zombie" Freeze)

**Scenario:** A load balancer misconfiguration sends traffic to a cached page, or a "Zombie" process keeps reporting the last known value (e.g., "100 RPS") repeatedly.

* **Normal Alert (RPS < 10)**: The zombie reports "100 RPS", so the low-traffic alert never fires.
* **Derivative Alert**: Organic traffic is chaotic. If RPS is exactly 100.0 for 5 minutes, it is likely synthetic or cached.
* **The Logic**: "Is the speed of change (derivative) effectively zero?"

```promql
# Alert if the "Velocity" of traffic is zero (unnaturally flat line)
abs(
  deriv(
    rate(metric[1m])[5m]
  )
) < 0.0001
```
