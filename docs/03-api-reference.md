# 03 · API 参考（指标函数与 Selector 类）

本文档描述 [Selector.py](../Selector.py) 的公开 API。所有函数与类均无全局状态、无 I/O。

## 1. 输入数据契约

Selector 层期望的 `DataFrame`：

| 列 | 类型 | 说明 |
| --- | --- | --- |
| `date` | datetime64 | 交易日，**必须已升序**（部分函数内部会再排序，但 `BBIKDJSelector`、`BBIShortLongSelector` 不做） |
| `open` | float | 开盘价 |
| `close` | float | 收盘价 |
| `high` | float | 最高价 |
| `low` | float | 最低价 |
| `volume` | float/int | 成交量 |

缺失列会在取列时抛 `KeyError`；非数值列在比较时可能抛 `TypeError`。`select_stock.load_data` 只保证 `date` 被解析为日期并排序，不检查其余列的类型。

---

## 2. 指标函数

### 2.1 `compute_kdj(df, n=9) -> pd.DataFrame`

位置：[L9](../Selector.py#L9)

计算 KDJ(9,3,3)。

**计算方式**：

- `low_n = low.rolling(n, min_periods=1).min()`
- `high_n = high.rolling(n, min_periods=1).max()`
- `RSV = (close - low_n) / (high_n - low_n + 1e-9) × 100`
- 递推：`K[0] = D[0] = 50`；`K[i] = 2/3·K[i-1] + 1/3·RSV[i]`；`D[i] = 2/3·D[i-1] + 1/3·K[i]`
- `J = 3K - 2D`

**参数/返回**：

| 项 | 说明 |
| --- | --- |
| `n` | 回看窗口，默认 9 |
| 返回 | **新的** DataFrame（`df.assign(K, D, J)`），不修改入参 |
| 空表 | 返回仅含 `K/D/J = NaN` 的副本 |

**注意**：

- 采用 Python 循环递推，时间复杂度 O(n)，大股票池下有可感知耗时；
- 初值固定 50，且从传入窗口的首行开始递推，因此**截断窗口会改变 K/D/J**；
- `min_periods=1` 使窗口不足 9 天时也能计算。

### 2.2 `compute_bbi(df) -> pd.Series`

位置：[L29](../Selector.py#L29)

`BBI = (MA3 + MA6 + MA12 + MA24) / 4`，使用 pandas 默认 `min_periods=window`，因此前 23 行为 `NaN`。

### 2.3 `compute_rsv(df, n) -> pd.Series`

位置：[L37](../Selector.py#L37)

```text
RSV(N) = 100 × (C - LLV(L,N)) / (HHV(C,N) - LLV(L,N))
```

- 低点使用 `low` 的滚动最小值；
- 高点使用 **`close` 的滚动最大值**（不是 `high`），与常见公式略有差异；
- `min_periods=1`，分母加 `1e-9` 防止除零。

### 2.4 `compute_dif(df, fast=12, slow=26) -> pd.Series`

位置：[L52](../Selector.py#L52)

`DIF = EMA(close, fast) - EMA(close, slow)`，使用 `ewm(span=..., adjust=False)`，从首行开始即可产生值（无 NaN 预热）。

### 2.5 `bbi_deriv_uptrend(bbi, *, min_window, max_window=None, q_threshold=0.0) -> bool`

位置：[L59](../Selector.py#L59)

判断 BBI 在某个窗口内“整体上升”。

**算法**：

1. 校验 `0 ≤ q_threshold ≤ 1`；
2. `bbi.dropna()`，长度不足 `min_window` 直接 `False`；
3. `longest = min(len(bbi), max_window or len(bbi))`；
4. 令 `w` 从 `longest` 递减到 `min_window`：
   - 取末尾 `w` 个点，按首值归一化 `norm = seg / seg.iloc[0]`；
   - 一阶差分 `diffs = np.diff(norm)`；
   - 若 `np.quantile(diffs, q_threshold) >= 0`，返回 `True`；
5. 全部窗口都不满足则 `False`。

**参数语义**：

| 参数 | 说明 |
| --- | --- |
| `min_window` | 最小窗口长度（含） |
| `max_window` | 最大窗口长度；`None` 表示不限 |
| `q_threshold` | 允许差分为负的比例。`0` ≈ 全程不下降；越大越宽松（如 `0.2` 表示允许 20% 分位的差分为负） |

**注意**：函数是“从最长窗口开始尝试”，任意一个窗口通过即返回 `True`；并非要求“最长窗口必须通过”。

### 2.6 `_find_peaks(df, *, column="high", distance=None, prominence=None, height=None, width=None, rel_height=0.5, **kwargs) -> pd.DataFrame`

位置：[L107](../Selector.py#L107)

对 `scipy.signal.find_peaks` 的薄封装：

- 校验 `column` 存在，否则抛 `KeyError`；
- 返回峰值所在行的副本，追加 `is_peak=True`；
- SciPy 返回的属性数组若长度与峰数一致，展开为 `peak_{key}` 列；
- **保留原 DataFrame 的索引**（调用方若需要连续索引需自行 `reset_index`）。

### 2.7 `last_valid_ma_cross_up(close, ma, lookback_n=None) -> Optional[int]`

位置：[L144](../Selector.py#L144)

查找最近一次“有效上穿”：`close[i-1] < ma[i-1]` 且 `close[i] >= ma[i]`。

- 返回 **整数位置（iloc 语义）**，不是日期索引标签；
- `lookback_n` 限定只搜索末尾 N 根；`None` 搜索全历史；
- 从后向前找，返回第一个满足的位置；无满足返回 `None`；
- 四个值均需非 `NaN`。

### 2.8 `compute_zx_lines(df, m1=14, m2=28, m3=57, m4=114) -> (pd.Series, pd.Series)`

位置：[L171](../Selector.py#L171)

返回 `(ZXDQ, ZXDKX)`：

| 线 | 公式 | 预热 |
| --- | --- | --- |
| `ZXDQ`（短期线） | `EMA(EMA(close, 10), 10)`，`adjust=False` | 首行即有值 |
| `ZXDKX`（长期线） | `(MA14 + MA28 + MA57 + MA114) / 4`，`min_periods` 等于各自窗口 | 前 113 行为 `NaN` |

> `ZXDKX` 需要至少 114 根 K 线才有效；`zx_condition_at_positions` 遇到 `NaN` 直接判 `False`。因此各 Selector 的缓冲长度必须大于 114（现有配置满足）。

### 2.9 `passes_day_constraints_today(df, pct_limit=0.02, amp_limit=0.07) -> bool`

位置：[L190](../Selector.py#L190)

统一的“当日过滤”：

```text
|close_today / close_yesterday - 1| < pct_limit
(high_today - low_today) / low_today < amp_limit
```

- 行数 `< 2` 返回 `False`；
- `close_yesterday ≤ 0` 或 `low_today ≤ 0` 返回 `False`；
- 两个不等式均为**严格小于**；
- 使用 `iloc[-1]` 与 `iloc[-2]`，依赖行顺序正确。

**适用范围**：6 个战法中除 `BigBullishVolumeSelector` 外均调用此函数。

### 2.10 `zx_condition_at_positions(df, *, require_close_gt_long=True, require_short_gt_long=True, pos=None) -> bool`

位置：[L211](../Selector.py#L211)

在指定位置检查知行约束：

| 条件 | 含义 |
| --- | --- |
| `require_close_gt_long` | `close[pos] > ZXDKX[pos]` |
| `require_short_gt_long` | `ZXDQ[pos] > ZXDKX[pos]` |

- `pos=None` 表示最后一行；越界返回 `False`；
- `ZXDQ` / `ZXDKX` 任一非有限值返回 `False`；
- 每次调用都会重新计算两条线，属于 O(n) 操作。

---

## 3. Selector 类

### 3.1 公共结构

| 成员 | 说明 |
| --- | --- |
| `__init__(**params)` | 从 `configs.json` 的 `params` 直接展开传入；含参数合法性校验的类在构造时抛 `ValueError` |
| `_passes_filters(hist)` | 单只股票判定，返回 `bool`；不修改入参（部分类内部 `copy()`） |
| `select(date, data)` | 批量判定，返回命中的代码列表（保持 `data` 的迭代顺序） |

`select` 的日期语义为 `df[df["date"] <= date]`：若 `date` 不是交易日，会自动使用其之前最近的行情。

---

### 3.2 `BBIKDJSelector`（少妇战法）

位置：[L247](../Selector.py#L247)

```python
BBIKDJSelector(
    j_threshold=-5,
    bbi_min_window=90,
    max_window=90,
    price_range_pct=100.0,
    bbi_q_threshold=0.05,
    j_q_threshold=0.10,
)
```

| 参数 | 默认 | `configs.json` 实际值 | 含义 |
| --- | --- | --- | --- |
| `j_threshold` | -5 | 15 | 当日 J 的绝对阈值 |
| `bbi_min_window` | 90 | 20 | BBI 上升判断的最小窗口 |
| `max_window` | 90 | 120 | 价格波动、J 分位、MA60 回看的最大窗口 |
| `price_range_pct` | 100.0 | 1 | 收盘价区间比例上限（**比值**，`1` = 100%，`0.2` = 20%） |
| `bbi_q_threshold` | 0.05 | 0.2 | BBI 差分允许的负分位比例 |
| `j_q_threshold` | 0.10 | 0.10 | J 的历史分位阈值 |

**过滤流水线**（[L273-L330](../Selector.py#L273-L330)）：

1. 计算 `BBI`；
2. `passes_day_constraints_today`；
3. 最近 `max_window` 根收盘价：`max(close) / min(close) - 1 ≤ price_range_pct`；
4. `bbi_deriv_uptrend(BBI, min_window, max_window, bbi_q_threshold)`；
5. `J_today < j_threshold` **或** `J_today ≤ quantile(J.tail(max_window), j_q_threshold)`；
6. `MA60 = close.rolling(60, min_periods=1).mean()`；当日 `close ≥ MA60`，且最近 `max_window` 内存在有效上穿 MA60；
7. `DIF > 0`；
8. 当日知行：`close > ZXDKX` 且 `ZXDQ > ZXDKX`。

**批量缓冲**：`df[df.date <= date].tail(max_window + 20)`；空表跳过。

---

### 3.3 `SuperB1Selector`（SuperB1 战法）

位置：[L348](../Selector.py#L348)

```python
SuperB1Selector(
    *,
    lookback_n=60,
    close_vol_pct=0.05,
    price_drop_pct=0.03,
    j_threshold=-5,
    j_q_threshold=0.10,
    B1_params=None,          # 必填，否则构造时 ValueError
)
```

| 参数 | 默认 | 配置值 | 含义 |
| --- | --- | --- | --- |
| `lookback_n` | 60 | 10 | 搜索历史 BBIKDJ 信号的天数 |
| `close_vol_pct` | 0.05 | 0.02 | 盘整区间收盘价波动率上限（比值） |
| `price_drop_pct` | 0.03 | 0.02 | 当日相对前一日的最小跌幅 |
| `j_threshold` | -5 | 10 | 当日 J 绝对阈值 |
| `j_q_threshold` | 0.10 | 0.10 | J 分位阈值 |
| `B1_params` | `None` | 见下 | 内嵌 `BBIKDJSelector` 构造参数 |

配置中的 `B1_params`：

```json
{
  "j_threshold": 15,
  "bbi_min_window": 20,
  "max_window": 120,
  "price_range_pct": 1,
  "bbi_q_threshold": 0.3,
  "j_q_threshold": 0.10
}
```

**内部字段**：

- `self.bbi_selector = BBIKDJSelector(**B1_params)`；
- `self._extra_for_bbi = bbi_selector.max_window + 20`，用于保证内层指标有足够历史。

**过滤流水线**（[L404-L457](../Selector.py#L404-L457)）：

1. `len(hist) ≥ 2` 且通过当日过滤；
2. `len(hist) ≥ lookback_n + _extra_for_bbi`；
3. 在 `hist.tail(lookback_n + 1)` 中**排除当日**，从旧到新逐日尝试：
   - 用 `hist.loc[:idx]` 调用内层 `BBIKDJSelector._passes_filters`（因此也会检查该日的当日过滤/知行/MACD 等全条件）；
   - 若通过，取 `[t_m, 昨日]` 收盘价，长度 `< 3` 则放弃并**终止搜索**；
   - 若区间波动率 `max/min - 1 > close_vol_pct`，重置 `t_m` 并继续尝试更晚的候选；
   - 否则确定 `t_m` 并停止；
4. 在 `t_m` 当日检查知行：`close > ZXDKX` 且 `ZXDQ > ZXDKX`；
5. 当日跌幅 `(close_prev - close_today) / close_prev ≥ price_drop_pct`；
6. 当日 J 条件（窗口为最后 `lookback_n` 根）；
7. 当日知行仅要求 `ZXDQ > ZXDKX`（`require_close_gt_long=False`）。

**批量缓冲**：`tail(lookback_n + _extra_for_bbi)`，长度不足直接跳过。

---

### 3.4 `PeakKDJSelector`（填坑战法）

位置：[L474](../Selector.py#L474)

```python
PeakKDJSelector(
    j_threshold=-5,
    max_window=90,
    fluc_threshold=0.03,
    gap_threshold=0.02,
    j_q_threshold=0.10,
)
```

| 参数 | 默认 | 配置值 | 含义 |
| --- | --- | --- | --- |
| `j_threshold` | -5 | 10 | 当日 J 绝对阈值 |
| `max_window` | 90 | 120 | J 分位窗口与批量缓冲基数 |
| `fluc_threshold` | 0.03 | 0.03 | 当日收盘与参照峰收盘的波动率上限 |
| `gap_threshold` | 0.02 | 0.2 | 参照峰必须高于两峰间最低收盘价的比例 |
| `j_q_threshold` | 0.10 | 0.10 | J 分位阈值 |

**峰值识别**：`oc_max = max(open, close)`，调用 `_find_peaks(distance=6, prominence=0.5)`。这两个参数**硬编码**，且 `prominence` 使用绝对价格单位，不同价位股票的识别敏感度不同。

**过滤流水线**（[L494-L572](../Selector.py#L494-L572)）：

1. 当日过滤；
2. `oc_max` 峰值识别，剔除当日峰，至少 2 个峰；
3. 取最新峰 `peak_t`，向前寻找参照峰 `peak_prev`：
   - `oc_t > oc_prev`；
   - 若总峰数 ≥ 3，则中间峰必须全部 `oc < oc_prev`（不能“抬高门槛”）；
   - 两峰之间最低收盘价 `min_close` 存在，且 `oc_prev > min_close × (1 + gap_threshold)`；
4. `|close_today - target_peak.close| / target_peak.close ≤ fluc_threshold`；
5. 当日 J 条件（窗口 `max_window`）；
6. 当日知行双条件。

**批量缓冲**：`tail(max_window + 20)`。

---

### 3.5 `BBIShortLongSelector`（补票战法）

位置：[L591](../Selector.py#L591)

```python
BBIShortLongSelector(
    n_short=3,
    n_long=21,
    m=3,
    bbi_min_window=90,
    max_window=150,
    bbi_q_threshold=0.05,
    upper_rsv_threshold=75,
    lower_rsv_threshold=25,
)
```

| 参数 | 默认 | 配置值 | 含义 |
| --- | --- | --- | --- |
| `n_short` | 3 | 5 | 短期 RSV 窗口 |
| `n_long` | 21 | 21 | 长期 RSV 窗口 |
| `m` | 3 | 5 | 最近观察天数，必须 ≥ 2 |
| `bbi_min_window` | 90 | 2 | BBI 上升最小窗口 |
| `max_window` | 150 | 120 | BBI 上升最大窗口与批量缓冲 |
| `bbi_q_threshold` | 0.05 | 0.2 | BBI 差分负分位比例 |
| `upper_rsv_threshold` | 75 | 75 | RSV 高位阈值 |
| `lower_rsv_threshold` | 25 | 25 | RSV 低位阈值 |

**过滤流水线**（[L618-L674](../Selector.py#L618-L674)）：

1. 计算 BBI；当日过滤；
2. BBI 上升判断；
3. 计算 `RSV_short` / `RSV_long`；`len ≥ m`；
4. 最近 `m` 根：
   - `RSV_long` 全部 `≥ upper_rsv_threshold`；
   - `RSV_short` 存在“先 ≥ upper，之后某日 < lower”的序列；
   - 最后一根 `RSV_short ≥ upper`；
5. `DIF > 0`；
6. 当日知行双条件。

**批量缓冲**：

```python
need_len = max(n_short, n_long) + bbi_min_window + m
hist = tail(max(need_len, max_window))
```

> `bbi_min_window=2` 时 `need_len` 很小，实际仍由 `max_window` 决定缓冲长度。

---

### 3.6 `MA60CrossVolumeWaveSelector`（上穿 60 放量战法）

位置：[L700](../Selector.py#L700)

```python
MA60CrossVolumeWaveSelector(
    *,
    lookback_n=60,
    vol_multiple=1.5,
    j_threshold=-5.0,
    j_q_threshold=0.10,
    ma60_slope_days=5,
    max_window=120,
)
```

| 参数 | 默认 | 配置值 | 含义 |
| --- | --- | --- | --- |
| `lookback_n` | 60 | 25 | 上穿 MA60 的回看天数 |
| `vol_multiple` | 1.5 | 1.8 | 波段均量相对前置窗口的倍数 |
| `j_threshold` | -5.0 | 15 | 当日 J 绝对阈值 |
| `j_q_threshold` | 0.10 | 0.10 | J 分位阈值 |
| `ma60_slope_days` | 5 | 5 | MA60 斜率回归点数 |
| `max_window` | 120 | 120 | J 分位窗口 |

**过滤流水线**（[L743-L822](../Selector.py#L743-L822)）：

1. 排序；`len ≥ max(60 + lookback_n + ma60_slope_days, max_window + 5)`；
2. 当日过滤；
3. 当日 J 条件；
4. `MA60`（`min_periods=1`）且当日 `close ≥ MA60`；最近 `lookback_n` 内有效上穿；
5. 在上穿日 `T` 到当日的区间内，取 `high` 最大值所在日 `Tmax`（并列时取第一日）；
6. 上涨波段 `[T, Tmax]` 长度 `< 3` 则失败；
7. 前置窗口：`pre_start = max(0, T - min(wave_len, 10))`，即最多取 10 天；长度需 `≥ max(5, min(10, wave_len))`；
8. 计算均量（忽略 `volume=0`）：`wave_avg ≥ vol_multiple × pre_avg`；
9. 最近 `ma60_slope_days` 个 MA60 值做一阶线性回归，斜率 `> 0`；
10. 当日知行双条件。

**批量缓冲**：

```python
need_len = max(60 + lookback_n + ma60_slope_days, max_window + 20)
```

> 与 README 描述不同，前置量能窗口上限为 **10 天**，并非完全等长。

---

### 3.7 `BigBullishVolumeSelector`（暴力 K 战法）

位置：[L836](../Selector.py#L836)

```python
BigBullishVolumeSelector(
    *,
    up_pct_threshold=0.04,
    upper_wick_pct_max=0.5,
    vol_lookback_n=20,
    vol_multiple=1.5,
    min_history=None,
    require_bullish_close=True,
    ignore_zero_volume=True,
    close_lt_zxdq_mult=1.0,
)
```

| 参数 | 默认 | 配置值 | 含义 |
| --- | --- | --- | --- |
| `up_pct_threshold` | 0.04 | 0.06 | 当日最小涨幅（`> 0.06` 表示严格大于 6%） |
| `upper_wick_pct_max` | 0.5 | 0.02 | 上影线比例上限（严格小于） |
| `vol_lookback_n` | 20 | 20 | 前置均量天数 |
| `vol_multiple` | 1.5 | 2.5 | 放量倍数 |
| `min_history` | `None` | `None` | 最少历史长度，默认 `vol_lookback_n + 2` |
| `require_bullish_close` | `True` | `true` | 是否要求 `close ≥ open` |
| `ignore_zero_volume` | `True` | `true` | 计算前置均量时是否剔除 0 成交量 |
| `close_lt_zxdq_mult` | 1.0 | 1.15 | `close < ZXDQ × mult` 的过热过滤 |

**过滤流水线**（[L882-L958](../Selector.py#L882-L958)）：

1. 排序；长度检查（`min_history` 与 `vol_lookback_n + 2`）；
2. 取值并校验有限性，`close_prev/close_today > 0`；
3. K 线合法性：`high ≥ max(open, close)` 且 `low ≤ min(open, close)`；
4. 可选收阳：`close ≥ open`；
5. 涨幅 `close_today / close_prev - 1 > up_pct_threshold`；
6. 上影线 `(high - max(open, close)) / max(open, close) < upper_wick_pct_max`；
7. 前置 `n` 日成交量（**不含当日**）剔除 0 后求均值，有效样本需 `≥ max(3, int(n × 0.6))`；
8. `volume_today ≥ vol_multiple × avg_vol`；
9. `ZXDQ` 有限且 `close_today < ZXDQ × close_lt_zxdq_mult`。

**与其他战法的差异**：

- **不调用** `passes_day_constraints_today`（因此允许当日大涨）；
- **不使用** `ZXDKX` 与知行约束，只使用 `ZXDQ` 判断“是否过热”；
- 不使用 BBI / KDJ / MACD / MA60。

**批量缓冲**：`max(min_history, vol_lookback_n + 2)`，长度不足跳过。

---

## 4. 复杂度与性能

| 操作 | 复杂度 | 说明 |
| --- | --- | --- |
| `compute_kdj` | O(n) | Python 循环，常数较大 |
| `compute_bbi` / `compute_rsv` / `compute_dif` / `compute_zx_lines` | O(n) | pandas 向量化 |
| `bbi_deriv_uptrend` | 最坏 O(max_window²) | 外层窗口递减 + 内层 `np.diff`；配置窗口 120 时可接受 |
| `zx_condition_at_positions` | O(n) | 每次调用重算两条线 |
| `SuperB1Selector` | O(n × lookback_n) | 每个候选日都跑一次完整 BBIKDJ（含 KDJ 循环） |
| `PeakKDJSelector` | O(n) | 一次 `find_peaks` |
| 批量选股 | O(股票数 × 单股复杂度) | 单进程串行；未提供并行选股 |

在 5000+ 只股票、`max_window=120` 的规模下，`SuperB1Selector` 与 `BBIKDJSelector` 是主要耗时来源。
