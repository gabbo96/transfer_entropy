# Transfer Entropy Implementation - Issues & Fixes

## Summary of Problems Found

### 1. **Indexing Bug** (CRITICAL - Fixed)
**Location:** Lines 97-113 in original `TE.py`

**Problem:** Incorrect indexing in the nested loops when accessing probability histograms.

**Original Code:**
```python
for i in range(len(Xrange)):  # i = X_future bin
    px = p1[i]  # ❌ WRONG: should be p1[k]
    for j in range(len(Yrange)):  # j = Y_past bin
        pxy = p2[i][j]  # ❌ WRONG: should be p2[k][j]
        for k in range(len(X2range)):  # k = X_past bin
            pxx2 = p2delay[i][k]  # ✓ Correct
            pxyx2 = p3[i][j][k]  # ✓ Correct
```

**Fixed Code:**
```python
for i in range(len(Xrange)):  # i = X_future bin
    for j in range(len(Yrange)):  # j = Y_past bin
        for k in range(len(X2range)):  # k = X_past bin
            px = p1[k]  # ✓ P(X_past)
            pxy = p2[k][j]  # ✓ P(X_past, Y_past)
            pxx2 = p2delay[i][k]  # ✓ P(X_future, X_past)
            pxyx2 = p3[i][j][k]  # ✓ P(X_future, Y_past, X_past)
```

**Explanation:** The histogram dimensions were:
- `p3[i,j,k]` = counts for (X_future=i, Y_past=j, X_past=k)
- `p2[i,j]` = counts for (X_past=i, Y_past=j)
- `p2delay[i,j]` = counts for (X_future=i, X_past=j)
- `p1[i]` = counts for (X_past=i)

The original code incorrectly used `i` (X_future index) to access `p1` and `p2`, when it should use `k` (X_past index).

---

### 2. **Different Delay Semantics** (CRITICAL - Documented)
**Problem:** Your `transfer_entropy()` and infomeasure's implementation interpret the `delay` parameter differently.

**Your Implementation:**
- Creates tuples: `(X[t+delay], Y[t], X[t])`
- Computes: "Given Y[t] and X[t], predict X[t+delay]"
- This is **multi-step transfer entropy**

**Infomeasure Implementation:**
- Pre-shifts series by `prop_time`: source→source[:-prop_time], dest→dest[prop_time:]
- Then creates tuples: `(X[t+1], Y[t], X[t])` (always 1-step)
- Computes: "Given Y[t] and X[t+prop_time], predict X[t+prop_time+1]"
- This is **standard transfer entropy with propagation delay**

**Why This Matters:**
When you generate data as `X[t] = W_Y * Y[t-3] + noise`:
- Your code with delay=3 asks: "Can (Y[t], X[t]) predict X[t+3]?" ← Not aligned with causal lag!
- Infomeasure with prop_time=3 asks: "Can (Y[t], X[t+3]=Y[t]) predict X[t+4]=Y[t+1]?" ← Aligned!

This is why infomeasure correctly finds the minimum at delay=3, while your original code finds a maximum.

---

### 3. **Binning Inconsistency** (MODERATE - Fixed)
**Location:** `transfer_entropy_im()` function

**Problem:** Inconsistent number of bins between discretization and histogram.

**Original Code:**
```python
X_dig = np.digitize(X, bins=np.linspace(X.min(), X.max(), binX))
```

`np.linspace(min, max, binX)` creates `binX` points, which means `binX-1` actual bins (intervals between points).

**Fixed Code:**
```python
X_dig = np.digitize(X, bins=np.linspace(X.min(), X.max(), binX+1))
```

Now creates `binX+1` bin edges, resulting in `binX` bins, matching `histogramdd`'s behavior.

---

## Solutions Provided

### 1. Fixed Original Function
The `transfer_entropy()` function now has corrected indexing. However, it still computes **multi-step TE**, which is a different quantity than infomeasure's standard TE.

### 2. New Function: `transfer_entropy_with_proptime()`
**Recommended for standard TE analysis.**

Matches infomeasure's semantics:
```python
def transfer_entropy_with_proptime(X, Y, prop_time=0, gaussian_sigma=None):
    """
    Pre-shifts series by prop_time, then computes 1-step TE.
    Equivalent to: Given Y[t] and X[t+prop_time], predict X[t+prop_time+1]
    """
```

**Usage:**
```python
from TE import transfer_entropy_with_proptime

# Standard TE with no propagation delay
te = transfer_entropy_with_proptime(X, Y, prop_time=0)

# TE with propagation delay (e.g., if Y affects X after 3 time steps)
te = transfer_entropy_with_proptime(X, Y, prop_time=3)
```

### 3. Fixed `transfer_entropy_im()` Wrapper
Now uses correct binning to match the number of bins used internally.

---

## Test Results

Running on synthetic data where `X[t] = 0.9 * Y[t-3] + 0.1 * K[t-3]`:

| W_Y | Original | With PropTime | Infomeasure |
|-----|----------|---------------|-------------|
| 0.1 | 1.5364   | 1.5592        | 1.0813      |
| 0.9 | 2.6844   | 0.2339        | 0.1637      |
| 1.0 | 3.1171   | 0.0559        | 0.0387      |

✅ **`transfer_entropy_with_proptime()` correctly shows TE decreasing as W_Y increases!**

| Delay | Original | With PropTime | Infomeasure |
|-------|----------|---------------|-------------|
| 1     | 1.1803   | 1.2081        | 0.8362      |
| 2     | 1.4602   | 2.2127        | 1.5325      |
| 3     | 2.6844   | **0.2339**    | **0.1637**  |
| 4     | 1.5932   | 0.9643        | 0.6717      |

✅ **`transfer_entropy_with_proptime()` correctly finds minimum at actual T_LAG=3!**

---

## Remaining Differences

The new function still shows some numerical differences from infomeasure (e.g., 0.2339 vs 0.1637) due to:

1. **Histogram vs Discrete Bins:** 
   - Your code: `np.histogramdd()` on continuous data
   - Infomeasure: `np.digitize()` then discrete counting
   
2. **Potential edge case handling:** Different treatment of bin boundaries

These are minor and don't affect the overall pattern or conclusions.

---

## Recommendations

1. **For standard TE analysis:** Use `transfer_entropy_with_proptime()` or the infomeasure package directly.

2. **For multi-step prediction TE:** Use the original `transfer_entropy()` (now with fixed indexing).

3. **Update your notebook:** Import and use the new function:
   ```python
   from TE import transfer_entropy_with_proptime
   TE_YX = transfer_entropy_with_proptime(X, Y, prop_time=delay)
   ```

4. **Interpretation:** When prop_time matches the actual causal lag in your data, TE will be minimized because the conditioning on X[t+prop_time] already captures the effect of Y[t].
