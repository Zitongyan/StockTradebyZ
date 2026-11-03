# 04 · 六大战法规则详解

本文从**业务规则**角度解释每个 Selector，并给出与源码条件的精确映射。策略参数的实际取值以 [configs.json](../configs.json) 为准；构造参数默认值见 [03-api-reference.md](./03-api-reference.md)。

## 0. 公共指标与公共过滤

### 0.1 指标公式

| 指标 | 公式 | 代码 |
| --- | --- | --- |
| BBI | `(MA3 + MA6 + MA12 + MA24) / 4` | `compute_bbi` |
| KDJ | RSV(9)，K/D 以 50 为初值按 `2/3·前值 + 1/3·新值` 递推，`J = 3K - 2D` | `compute_kdj` |
| DIF | `EMA12 - EMA26`（`adjust=False`） | `compute_dif` |
| RSV(N) | `100 × (C - LLV(L,N)) / (HHV(C,N) - LLV(L,N))` | `compute_rsv` |
| MA60 | `close.rolling(60, min_periods=1).mean()` | 各战法内联 |
| ZXDQ（短期线） | `EMA(EMA(C,10),10)` | `compute_zx_lines` |
| ZXDKX（长期线） | `(MA14 + MA28 + MA57 + MA114)/4` | `compute_zx_lines` |

### 0.2 公共当日过滤

`passes_day_constraints_today`（[L190](../Selector.py#L190)）对**当日**施加：

```text
|close_t / close_{t-1} - 1| < 2%       # 涨跌幅绝对值
(high_t - low_t) / low_t   < 7%       # 振幅
```

含义：回避当日已经大幅异动或振幅过大的标的，降低“追高/接飞刀”风险。

**例外**：`BigBullishVolumeSelector` 不调用该过滤，因为它要找的正是放量大阳线。

### 0.3 知行约束

```text
收盘 > ZXDKX           （价格在长期线之上）
ZXDQ   > ZXDKX         （短期线在长期线之上）
```

- 多数战法要求**当日**同时满足；
- `SuperB1Selector` 在历史信号日 `t_m` 要求两条，在当日只要求“短期线 > 长期线”；
- `BigBullishVolumeSelector` 不使用知行约束，仅用 `close < 1.15 × ZXDQ` 判断是否过热。

---

## 1. BBIKDJSelector（少妇战法）

**定位**：在 BBI 整体抬升、KDJ 处于低位、MACD 多头、价格刚站上/守住 MA60 的股票中寻找低吸点。

### 1.1 条件流水线

| # | 条件 | 精确规则 | 源码 |
| --- | --- | --- | --- |
| 1 | 当日过滤 | 涨跌幅绝对值 < 2%，振幅 < 7% | `passes_day_constraints_today` |
| 2 | 价格区间收敛 | 最近 `max_window` 根收盘价：`max/min - 1 ≤ price_range_pct` | [L281-L284](../Selector.py#L281-L284) |
| 3 | BBI 上升 | 在 `[bbi_min_window, max_window]` 中任一窗口，归一化差分的 `bbi_q_threshold` 分位 ≥ 0 | [L287-L293](../Selector.py#L287-L293) |
| 4 | KDJ 低位 | `J < j_threshold` **或** `J ≤ 最近 max_window 根 J 的 j_q_threshold 分位` | [L296-L307](../Selector.py#L296-L307) |
| 5 | MA60 之上 | 当日 `close ≥ MA60` | [L310-L314](../Selector.py#L310-L314) |
| 6 | 有效上穿 | 最近 `max_window` 内存在 `close_{i-1} < MA60_{i-1}` 且 `close_i ≥ MA60_i` | [L317-L319](../Selector.py#L317-L319) |
| 7 | MACD 多头 | `DIF > 0` | [L322-L324](../Selector.py#L322-L324) |
| 8 | 知行当日 | `close > ZXDKX` 且 `ZXDQ > ZXDKX` | [L327-L328](../Selector.py#L327-L328) |

### 1.2 配置语义

```json
{
  "j_threshold": 15,
  "bbi_min_window": 20,
  "max_window": 120,
  "price_range_pct": 1,
  "bbi_q_threshold": 0.2,
  "j_q_threshold": 0.10
}
```

- `price_range_pct = 1` 表示允许 100% 的价格区间（较宽松）；
- 构造函数默认值为 `100.0`（即 10000%），实际等价于不限制。**配置值才是有效约束**；
- `bbi_q_threshold = 0.2` 允许 20% 分位的 BBI 差分为负，即容忍回撤；
- `j_q_threshold = 0.10` 表示当日 J 只要位于近 120 日 J 值的最低 10% 分位即可。

### 1.3 缓冲

`tail(max_window + 20)` → 配置下为 140 根，刚好覆盖 `ZXDKX` 的 114 根预热。

---

## 2. SuperB1Selector（SuperB1 战法）

**定位**：先在一段回看期内找到“曾经满足少妇战法”的启动日 `t_m`，随后股价进入窄幅盘整，最后出现一次缩量/回调式下跌且 J 值极低，属于“强势股回踩”思路。

### 2.1 条件流水线

| # | 条件 | 精确规则 | 源码 |
| --- | --- | --- | --- |
| 1 | 数据与当日过滤 | `len ≥ 2`；当日过滤同 0.2 | [L404-L410](../Selector.py#L404-L410) |
| 2 | 历史长度 | `len ≥ lookback_n + (BBI.max_window + 20)` | [L412-L414](../Selector.py#L412-L414) |
| 3 | 历史信号 `t_m` | 在 `[今日-lookback_n, 昨日]` 中从旧到新寻找一个交易日，其截至该日的完整 `BBIKDJSelector` 条件通过 | [L416-L431](../Selector.py#L416-L431) |
| 4 | 盘整区间 | `t_m` 到昨日收盘价波动率 `max/min - 1 ≤ close_vol_pct`；区间不足 3 天则直接终止搜索 | [L421-L431](../Selector.py#L421-L431) |
| 5 | `t_m` 当日知行 | `close > ZXDKX` 且 `ZXDQ > ZXDKX` | [L435-L438](../Selector.py#L435-L438) |
| 6 | 当日下跌 | `(close_{t-1} - close_t) / close_{t-1} ≥ price_drop_pct` | [L440-L443](../Selector.py#L440-L443) |
| 7 | 当日 J 低位 | `J < j_threshold` 或 `J ≤ 最近 lookback_n 根 J 的 j_q_threshold 分位` | [L445-L451](../Selector.py#L445-L451) |
| 8 | 当日知行 | 仅要求 `ZXDQ > ZXDKX` | [L453-L455](../Selector.py#L453-L455) |

### 2.2 关键实现细节

- 内层 `BBIKDJSelector` 使用 `B1_params`（配置：`bbi_q_threshold=0.3`，比独立运行少妇战法更宽松）；
- 内层判定作用在 `hist.loc[:idx]` 上，因此**会再次施加“候选日”的当日过滤**；
- 候选日按从旧到新扫描：如果某候选日的盘整区间波动过大，会继续尝试更晚的候选日；一旦区间不足 3 天，立即终止整个搜索；
- 配置 `lookback_n=10`，意味着只在小窗口里找信号，命中率取决于近期是否出现过少妇战法信号。

### 2.3 缓冲

`min_len = lookback_n + (B1_params.max_window + 20)` → 配置下 `10 + 140 = 150` 根。

---

## 3. BBIShortLongSelector（补票战法）

**定位**：BBI 上行途中，长期 RSV 持续强势、短期 RSV 经历“冲高 → 回落 → 再冲高”的节奏，配合 MACD 多头，寻找强势股的回踩补仓点。

### 3.1 条件流水线

| # | 条件 | 精确规则 | 源码 |
| --- | --- | --- | --- |
| 1 | 当日过滤 | 同 0.2 | [L622-L623](../Selector.py#L622-L623) |
| 2 | BBI 上升 | 同少妇战法，窗口 `[bbi_min_window, max_window]` | [L626-L632](../Selector.py#L626-L632) |
| 3 | 长期 RSV 强势 | 最近 `m` 天 `RSV_long ≥ upper_rsv_threshold` 全部成立 | [L635-L642](../Selector.py#L635-L642) |
| 4 | 短期 RSV 节奏 | 最近 `m` 天内存在某日 `RSV_short ≥ upper`，且其后存在某日 `RSV_short < lower` | [L644-L658](../Selector.py#L644-L658) |
| 5 | 短期 RSV 收在高位 | 最后一天 `RSV_short ≥ upper_rsv_threshold` | [L660-L662](../Selector.py#L660-L662) |
| 6 | MACD 多头 | `DIF > 0` | [L665-L668](../Selector.py#L665-L668) |
| 7 | 知行当日 | `close > ZXDKX` 且 `ZXDQ > ZXDKX` | [L671-L672](../Selector.py#L671-L672) |

### 3.2 配置语义

```json
{
  "n_short": 5,
  "n_long": 21,
  "m": 5,
  "bbi_min_window": 2,
  "max_window": 120,
  "bbi_q_threshold": 0.2,
  "upper_rsv_threshold": 75,
  "lower_rsv_threshold": 25
}
```

- `m=5`：只观察最近一周的 RSV 结构；
- `bbi_min_window=2`：允许非常短的 BBI 上升窗口，配合 `bbi_q_threshold=0.2` 相当宽松；
- `upper=75 / lower=25` 是 RSV 的强弱分界。

### 3.3 缓冲

```python
need_len = max(n_short, n_long) + bbi_min_window + m   # 配置下 = 21 + 2 + 5 = 28
hist.tail(max(need_len, max_window))                   # → 120
```

`ZXDKX` 需要 114 根，120 根刚好满足。

---

## 4. PeakKDJSelector（填坑战法）

**定位**：寻找“前高之后挖坑、当前价格回到前高附近、KDJ 处于低位”的形态。核心是识别两个峰值 `peak_t`（最新）与 `peak_(t-n)`（参照），并用两者之间的最低收盘价衡量“坑的深度”。

### 4.1 峰值识别

```python
oc_max = max(open, close)
peaks = find_peaks(oc_max, distance=6, prominence=0.5)
```

- `distance=6`：两峰至少间隔 6 根 K 线；
- `prominence=0.5`：绝对价格突出版，**未按价格归一化**；
- 剔除当日峰值后再判断。

### 4.2 条件流水线

| # | 条件 | 精确规则 | 源码 |
| --- | --- | --- | --- |
| 1 | 当日过滤 | 同 0.2 | [L498-L499](../Selector.py#L498-L499) |
| 2 | 至少两个峰 | 剔除当日后 `peaks ≥ 2` | [L505-L516](../Selector.py#L505-L516) |
| 3 | 最新峰更高 | `oc_t > oc_(t-n)` | [L525-L529](../Selector.py#L525-L529) |
| 4 | 中间峰不抬高门槛 | 若总峰数 ≥ 3，`[idx+1, total-2]` 的峰 `oc_max` 必须全部小于 `oc_prev` | [L531-L535](../Selector.py#L531-L535) |
| 5 | 坑深足够 | 两峰之间最低收盘价 `min_close` 存在，且 `oc_prev > min_close × (1 + gap_threshold)` | [L537-L544](../Selector.py#L537-L544) |
| 6 | 回到前高附近 | `|close_t - target_peak.close| / target_peak.close ≤ fluc_threshold` | [L553-L557](../Selector.py#L553-L557) |
| 7 | 当日 J 低位 | 同少妇战法的双重 J 条件 | [L559-L567](../Selector.py#L559-L567) |
| 8 | 知行当日 | `close > ZXDKX` 且 `ZXDQ > ZXDKX` | [L569-L570](../Selector.py#L569-L570) |

### 4.3 配置语义

```json
{
  "j_threshold": 10,
  "max_window": 120,
  "fluc_threshold": 0.03,
  "j_q_threshold": 0.10,
  "gap_threshold": 0.2
}
```

- `gap_threshold=0.2`：参照峰必须比坑底高 20% 以上，强调“深坑”；
- `fluc_threshold=0.03`：当前收盘距离参照峰收盘不超过 3%，即已基本填坑；
- 构造默认 `gap_threshold=0.02`，配置放大了 10 倍，是策略收紧的主要来源。

### 4.4 缓冲

`tail(max_window + 20)` → 140 根。

---

## 5. MA60CrossVolumeWaveSelector（上穿 60 放量战法）

**定位**：股票有效上穿 MA60 后形成放量上涨波段，当前 J 值回落到低位、MA60 仍向上，属于“突破后缩量回踩”的观察点。

### 5.1 条件流水线

| # | 条件 | 精确规则 | 源码 |
| --- | --- | --- | --- |
| 1 | 历史长度 | `≥ max(60 + lookback_n + ma60_slope_days, max_window + 5)` | [L753-L755](../Selector.py#L753-L755) |
| 2 | 当日过滤 | 同 0.2 | [L757-L758](../Selector.py#L757-L758) |
| 3 | 当日 J 低位 | `J < j_threshold` 或 `J ≤ 最近 max_window 根 J 的 j_q_threshold 分位` | [L761-L770](../Selector.py#L761-L770) |
| 4 | MA60 之上 | 当日 `close ≥ MA60` | [L773-L775](../Selector.py#L773-L775) |
| 5 | 有效上穿 | 最近 `lookback_n` 内存在有效上穿 | [L777-L779](../Selector.py#L777-L779) |
| 6 | 波段终点 | `[T, today]` 内 `high` 最大值日为 `Tmax`（并列取第一日） | [L781-L792](../Selector.py#L781-L792) |
| 7 | 波段长度 | `[T, Tmax]` 至少 3 天 | [L794-L798](../Selector.py#L794-L798) |
| 8 | 前置窗口 | `[T - min(wave_len, 10), T-1]`，长度 ≥ `max(5, min(10, wave_len))` | [L800-L804](../Selector.py#L800-L804) |
| 9 | 放量 | 波段均量（忽略 0）`≥ vol_multiple ×` 前置均量 | [L806-L813](../Selector.py#L806-L813) |
| 10 | MA60 向上 | 最近 `ma60_slope_days` 个 MA60 值线性回归斜率 > 0 | [L815-L816](../Selector.py#L815-L816) |
| 11 | 知行当日 | `close > ZXDKX` 且 `ZXDQ > ZXDKX` | [L819-L820](../Selector.py#L819-L820) |

### 5.2 配置语义

```json
{
  "lookback_n": 25,
  "vol_multiple": 1.8,
  "j_threshold": 15,
  "j_q_threshold": 0.10,
  "ma60_slope_days": 5,
  "max_window": 120
}
```

- `vol_multiple=1.8`：上涨波段均量需达到上穿前约 10 日窗口均量的 1.8 倍；
- 前置窗口并非 README 所说的“等长”，实现中**最多 10 天**（`min(wave_len, 10)`）；
- `Tmax` 取波段最高点，要求波段至少 3 天，避免“上穿次日即结束”的噪声。

### 5.3 缓冲

```python
need_len = max(60 + lookback_n + ma60_slope_days, max_window + 20)   # 配置下 max(90, 140) = 140
```

---

## 6. BigBullishVolumeSelector（暴力 K 战法）

**定位**：捕捉“刚刚放量启动的强势长阳，但价格尚未远离短期均线”的个股。

### 6.1 条件流水线

| # | 条件 | 精确规则 | 源码 |
| --- | --- | --- | --- |
| 1 | 基础校验 | 长度足够；OHLC 有限；`close > 0`；`high ≥ max(open, close)`；`low ≤ min(open, close)` | [L888-L911](../Selector.py#L888-L911) |
| 2 | 收阳（可选） | `close ≥ open` | [L913-L915](../Selector.py#L913-L915) |
| 3 | 长阳 | `close_t / close_{t-1} - 1 > up_pct_threshold` | [L917-L920](../Selector.py#L917-L920) |
| 4 | 上影线短 | `(high - max(open, close)) / max(open, close) < upper_wick_pct_max` | [L922-L927](../Selector.py#L922-L927) |
| 5 | 放量 | 当日量 `≥ vol_multiple ×` 前 `vol_lookback_n` 日均量（剔除 0） | [L929-L943](../Selector.py#L929-L943) |
| 6 | 不过热 | `close < ZXDQ × close_lt_zxdq_mult` | [L945-L956](../Selector.py#L945-L956) |

### 6.2 配置语义

```json
{
  "up_pct_threshold": 0.06,
  "upper_wick_pct_max": 0.02,
  "require_bullish_close": true,
  "close_lt_zxdq_mult": 1.15,
  "vol_lookback_n": 20,
  "vol_multiple": 2.5
}
```

这是六大战法中最严格的量价组合：

- 当日涨幅 **> 6%**；
- 上影线 **< 2%**（过滤冲高回落）；
- 成交量 **≥ 2.5 倍** 的 20 日均量；
- 收盘价不得高于 `1.15 × ZXDQ`（尚未加速过热）；
- 必须收阳。

### 6.3 与其他战法的差异

| 差异点 | 说明 |
| --- | --- |
| 不做当日涨跌幅/振幅过滤 | 目标就是大阳线 |
| 不检查知行双条件 | 只用 `ZXDQ` 做过热约束 |
| 不使用 BBI/KDJ/MACD/MA60 | 纯量价 + 短期均线 |
| 缓冲最短 | 默认 `vol_lookback_n + 2 = 22` 根 |

---

## 7. 策略对比

| 特性 | 少妇 | SuperB1 | 补票 | 填坑 | 上穿60 | 暴力K |
| --- | --- | --- | --- | --- | --- | --- |
| 当日涨跌幅/振幅过滤 | 有 | 有 | 有 | 有 | 有 | 无 |
| BBI 上升 | 有 | 内嵌 | 有 | 无 | 无 | 无 |
| KDJ J 低位 | 有 | 有 | 无 | 有 | 有 | 无 |
| MACD DIF > 0 | 有 | 内嵌 | 有 | 无 | 无 | 无 |
| MA60 | 有 | 内嵌 | 无 | 无 | 有 | 无 |
| RSV 短/长期 | 无 | 无 | 有 | 无 | 无 | 无 |
| 峰值形态 | 无 | 无 | 无 | 有 | 无 | 无 |
| 量能比较 | 无 | 无 | 无 | 无 | 有 | 有 |
| 知行双条件（当日） | 有 | 仅短期线 | 有 | 有 | 有 | 无 |
| 大约所需历史 | 140 | 150 | 120 | 140 | 140 | 22 |

## 8. 调参与结果解读建议

1. **先确认数据长度**：任何战法命中为空时，先检查该股票的 CSV 行数是否满足缓冲要求（尤其 `ZXDKX` 需要 114 根）。
2. **`price_range_pct` 是比值**：配置中 `1` 表示允许 100% 区间；若误填 `100`，则几乎不设约束。
3. **J 分位是“或”关系**：只要满足绝对阈值或分位阈值之一即可，放宽 `j_q_threshold` 会显著增加命中数。
4. **SuperB1 的实际宽松度由 `B1_params.bbi_q_threshold` 决定**，而不是外层参数。
5. **填坑战法的 `gap_threshold` 对命中数影响最大**：0.2 表示坑深至少 20%，降低它会明显放宽。
6. **上穿 60 战法的量能前置窗口固定最多 10 天**，调 `lookback_n` 不会改变这一点。
7. **暴力 K 是唯一允许当日大涨的战法**，若与其他战法同时运行，结果集合通常不重叠。
8. **截断窗口会影响指标**：同一只股票在不同战法下的 J/BBI 值可能不同，跨战法比较命中结果时需注意。
