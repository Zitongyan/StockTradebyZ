# 05 · 数据规范、配置与运行方式

## 1. 数据契约

### 1.1 行情 CSV

路径：`data/{6位代码}.csv`（如 `data/000001.csv`），由 `fetch_kline.py` 生成。

| 列 | 类型 | 来源 |
| --- | --- | --- |
| `date` | `datetime64` | Tushare `trade_date` |
| `open` | float | 前复权开盘价 |
| `close` | float | 前复权收盘价 |
| `high` | float | 前复权最高价 |
| `low` | float | 前复权最低价 |
| `volume` | float/int | Tushare `vol` |

约定：

- 日期**升序**；
- 无重复日期；
- 不含未来日期；
- 每次抓取**全量覆盖**，不做增量合并；
- 停牌/无数据时可能生成**只有表头**的空文件；
- 复权方式固定为 **qfq（前复权）**，历史价格会随新数据变化，因此不适合做增量拼接。

### 1.2 股票清单 `stocklist.csv`

| 列 | 说明 |
| --- | --- |
| `ts_code` | 带交易所后缀，如 `000001.SZ` |
| `symbol` | 6 位代码，如 `000001` |
| `name` | 股票名称 |
| `area` | 地区 |
| `industry` | 行业（`SectorShift.py` 依赖此列） |

当前仓库约 **5429 条**记录。`fetch_kline` 只依赖 `ts_code` 与 `symbol`；`SectorShift` 依赖 `symbol`（或 `ts_code`/`code`）与 `industry`。

### 1.3 配置 `configs.json`

顶层结构：

```json
{
  "selectors": [
    {
      "class": "BBIKDJSelector",
      "alias": "少妇战法",
      "activate": true,
      "params": { "...": "..." }
    }
  ]
}
```

| 字段 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- |
| `class` | 是 | — | `Selector.py` 中的类名 |
| `alias` | 否 | 类名 | 结果展示用 |
| `activate` | 否 | `true` | 显式 `false` 时跳过 |
| `params` | 否 | `{}` | 原样展开传给构造函数 |

兼容的顶层写法：数组、`{"selectors": [...]}`、单个对象。**未知参数**会触发构造函数的 `TypeError`，`select_stock` 捕获后跳过该战法。

### 1.4 配置参数总表

以下“配置值”来自 [configs.json](../configs.json)，“默认值”来自类构造函数。

#### BBIKDJSelector

| 参数 | 默认 | 配置值 | 说明 |
| --- | --- | --- | --- |
| `j_threshold` | -5 | 15 | J 绝对阈值 |
| `bbi_min_window` | 90 | 20 | BBI 最小窗口 |
| `max_window` | 90 | 120 | 最大窗口 |
| `price_range_pct` | 100.0 | 1 | 收盘价区间（比值） |
| `bbi_q_threshold` | 0.05 | 0.2 | BBI 差分负分位 |
| `j_q_threshold` | 0.10 | 0.10 | J 分位 |

#### SuperB1Selector

| 参数 | 默认 | 配置值 |
| --- | --- | --- |
| `lookback_n` | 60 | 10 |
| `close_vol_pct` | 0.05 | 0.02 |
| `price_drop_pct` | 0.03 | 0.02 |
| `j_threshold` | -5 | 10 |
| `j_q_threshold` | 0.10 | 0.10 |
| `B1_params` | 无（必填） | 见下 |

`B1_params` 配置值：`j_threshold=15`、`bbi_min_window=20`、`max_window=120`、`price_range_pct=1`、`bbi_q_threshold=0.3`、`j_q_threshold=0.10`。

#### BBIShortLongSelector

| 参数 | 默认 | 配置值 |
| --- | --- | --- |
| `n_short` | 3 | 5 |
| `n_long` | 21 | 21 |
| `m` | 3 | 5 |
| `bbi_min_window` | 90 | 2 |
| `max_window` | 150 | 120 |
| `bbi_q_threshold` | 0.05 | 0.2 |
| `upper_rsv_threshold` | 75 | 75 |
| `lower_rsv_threshold` | 25 | 25 |

#### PeakKDJSelector

| 参数 | 默认 | 配置值 |
| --- | --- | --- |
| `j_threshold` | -5 | 10 |
| `max_window` | 90 | 120 |
| `fluc_threshold` | 0.03 | 0.03 |
| `gap_threshold` | 0.02 | 0.2 |
| `j_q_threshold` | 0.10 | 0.10 |

#### MA60CrossVolumeWaveSelector

| 参数 | 默认 | 配置值 |
| --- | --- | --- |
| `lookback_n` | 60 | 25 |
| `vol_multiple` | 1.5 | 1.8 |
| `j_threshold` | -5.0 | 15 |
| `j_q_threshold` | 0.10 | 0.10 |
| `ma60_slope_days` | 5 | 5 |
| `max_window` | 120 | 120 |

#### BigBullishVolumeSelector

| 参数 | 默认 | 配置值 |
| --- | --- | --- |
| `up_pct_threshold` | 0.04 | 0.06 |
| `upper_wick_pct_max` | 0.5 | 0.02 |
| `vol_lookback_n` | 20 | 20 |
| `vol_multiple` | 1.5 | 2.5 |
| `min_history` | `None` | `None` |
| `require_bullish_close` | `true` | `true` |
| `ignore_zero_volume` | `true` | `true` |
| `close_lt_zxdq_mult` | 1.0 | 1.15 |

---

## 2. 运行方式

### 2.1 环境准备

```bash
# 推荐 Python 3.11 / 3.12
pip install -r requirements.txt
```

依赖：`pandas==2.3.0`、`tqdm==4.66.4`、`tushare==1.4.21`、`scipy==1.14.1`。

### 2.2 环境变量

| 变量 | 是否必需 | 说明 |
| --- | --- | --- |
| `TUSHARE_TOKEN` | 抓取阶段必需 | `fetch_kline.py` 读取后调用 `ts.set_token` |
| `NO_PROXY` / `no_proxy` | 脚本强制设置 | 覆盖为 `api.waditu.com,.waditu.com,waditu.com` |

> `fetch_kline.py` 会**直接覆盖**已有的 `NO_PROXY` 值。如果所在网络必须通过代理访问 Tushare，需要修改这段逻辑，否则请求可能无法走代理。

### 2.3 抓取行情

```bash
python fetch_kline.py \
  --start 20240101 \
  --end today \
  --stocklist ./stocklist.csv \
  --exclude-boards gem star bj \
  --out ./data \
  --workers 6
```

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--start` | `20190101` | `YYYYMMDD` 或 `today` |
| `--end` | `today` | `YYYYMMDD` 或 `today` |
| `--stocklist` | `./stocklist.csv` | 清单路径 |
| `--exclude-boards` | 空 | `gem` / `star` / `bj` 可多选 |
| `--out` | `./data` | 输出目录 |
| `--workers` | `6` | 线程数 |

产物：`data/{code}.csv`、`fetch.log`。

### 2.4 运行选股

```bash
python select_stock.py \
  --data-dir ./data \
  --config ./configs.json \
  --date 2025-09-10

# 只跑指定股票
python select_stock.py --tickers 000001,600000 --date 2025-09-10
```

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--data-dir` | `./data` | 行情目录（`all` 模式只读一级 CSV） |
| `--config` | `./configs.json` | 配置路径 |
| `--date` | 数据最新交易日 | `pd.to_datetime` 可解析的格式 |
| `--tickers` | `all` | `all` 或逗号分隔代码 |

产物：`select_results.log`（追加）+ 控制台输出。

### 2.5 行业分布统计

```bash
python SectorShift.py \
  --data_dir ./data \
  --stocklist stocklist.csv \
  --j_threshold 15 \
  --trade_date 2025-09-10
```

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--data_dir` | `./data` | 行情目录 |
| `--stocklist` | `stocklist.csv` | 清单路径 |
| `--j_threshold` | `15.0` | 日线 J 阈值 |
| `--trade_date` | `None` | `YYYYMMDD` / `YYYY-MM-DD`；为空则取每只股票最后一行 |

输出：`industry_counts` 的 JSON 数组。若需 Excel，调用函数并传 `export_excel_path`：

```python
from SectorShift import compute_j_industry_distribution

compute_j_industry_distribution(
    data_dir="./data",
    stocklist_path="stocklist.csv",
    j_threshold=15.0,
    trade_date="2025-09-10",
    export_excel_path="./industry.xlsx",
)
```

### 2.6 历史价格反查

```bash
python find_stock_by_price_concurrent.py 15.80 \
  --data-dir ./data \
  --price-type close \
  --start-date 2025-01-01 \
  --end-date 2025-09-10 \
  --tolerance 0.02 \
  --benchmark
```

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `price` | 必填（位置参数） | 目标价格 |
| `--data-dir` | `./data` | 递归查找 CSV |
| `--price-type` | `close` | `close` / `high` / `low` |
| `--start-date` | 无 | 只给一个边界时，该值同时作为起止 |
| `--end-date` | 无 | 同上 |
| `--tolerance` | `0.00` | 闭区间 `[price-tol, price+tol]` |
| `--max-workers` | CPU 核数 | 进程数 |
| `--benchmark` | 关闭 | 输出耗时统计 |

---

## 3. 日志与产物

| 文件 | 写入方 | 模式 | 内容 |
| --- | --- | --- | --- |
| `fetch.log` | `fetch_kline.py` | 追加 | 抓取进度、重试、封禁冷却、失败股票 |
| `select_results.log` | `select_stock.py` | 追加 | 每个战法的交易日、命中数、股票列表 |
| `data/*.csv` | `fetch_kline.py` | 覆盖 | 单只股票行情 |
| `industry.xlsx` | `SectorShift.py` | 覆盖 | 仅在函数调用传参时生成 |

两个日志都没有轮转策略，长期运行需自行清理或改名。

---

## 4. 错误处理与退出行为

### 4.1 `fetch_kline.py`

| 场景 | 行为 |
| --- | --- |
| 缺少 `TUSHARE_TOKEN` | `raise ValueError`，进程终止 |
| `stocklist.csv` 不存在 | `pd.read_csv` 抛 `FileNotFoundError` |
| 缺少 `symbol` / `ts_code` 列 | `KeyError` |
| 板块过滤后为空 | `logger.error` + `sys.exit(1)` |
| 单只命中限流 | 冷却约 600 秒，最多 3 次尝试 |
| 单只其它异常 | 等待 `15 × 尝试次数` 秒后重试 |
| 3 次全失败 | 记录 error，**不写文件**（旧文件保留） |
| 抓到空数据 | 写入只含表头的 CSV |
| 数据含未来日期/缺失日期 | `validate` 抛 `ValueError` → 进入重试流程 |

### 4.2 `select_stock.py`

| 场景 | 行为 |
| --- | --- |
| 数据目录不存在 | error + `sys.exit(1)` |
| 股票池为空 | error + `sys.exit(1)` |
| 全部 CSV 加载失败 | error + `sys.exit(1)` |
| 配置文件不存在 | error + `sys.exit(1)` |
| 配置 JSON 语法错误 | `json.load` 抛异常，进程终止 |
| `selectors` 为空 | error + `sys.exit(1)` |
| 单个 Selector 构造/类加载失败 | 记录 error 后 `continue` |
| 某只股票 CSV 缺失 | warning 后跳过该股票 |

### 4.3 `SectorShift.py` / `find_stock_by_price_concurrent.py`

| 场景 | 行为 |
| --- | --- |
| 缺少行业列 | `SectorShift` 抛 `ValueError` |
| `trade_date` 无法解析 | `SectorShift` 抛 `ValueError` |
| 行情目录不存在 | `find` 记录 error，返回空结果（退出码 0） |
| `price_type` 非法 | `find` 抛 `ValueError`，`main` 捕获后记录 error 并返回 |

---

## 5. 性能特征

| 环节 | 并发模型 | 主要开销 | 备注 |
| --- | --- | --- | --- |
| 抓取 | 线程池（默认 6） | 网络请求、Tushare 限流 | 命中限流会整体串行冷却 |
| 选股 | 单进程串行 | 每只股票的指标计算 | `SuperB1` 会对每个候选日重跑完整 BBIKDJ，最慢 |
| KDJ | Python 循环 | O(n)，常数大 | 每个战法都会重新计算 |
| 上穿60 | pandas + polyfit | 量能窗口、回归 | 可接受 |
| 价格反查 | 进程池 | 读取 + DataFrame 跨进程序列化 | 大股票池内存占用高 |
| 行业统计 | 单进程 | 全量读取 + KDJ | 无并发 |

实用的缩短路径：先 `--tickers` 限制股票池；或减少 `max_window` / `lookback_n`；数据抓取阶段用 `--exclude-boards` 缩小范围。

---

## 6. 排错速查

| 症状 | 可能原因 | 处理 |
| --- | --- | --- |
| 选股结果为 0，且所有战法都为空 | `data/` 无 CSV、日期晚于数据、或历史长度不足 | 检查 `data/*.csv` 行数与最后日期 |
| 只有个别战法为空 | 该战法条件严格或参数过紧 | 对照 04 的条件表，先放宽 `j_q_threshold` / `gap_threshold` / `price_range_pct` |
| `KeyError: 'date'` | CSV 列名不符 | 重新用 `fetch_kline.py` 生成，或补齐列 |
| `TypeError: ... unexpected keyword argument` | 配置中 `params` 有拼写错误/多余字段 | 对照 03 的构造签名；该战法会被跳过并记录 error |
| `ValueError: 请先设置环境变量 TUSHARE_TOKEN` | 未设置 token | `export TUSHARE_TOKEN=...` |
| 抓取长时间卡住 | 命中 Tushare 限流，处于 600 秒冷却 | 查看 `fetch.log`，等待或降低 `--workers` |
| 某只股票行情停留在旧日期 | 最近一次抓取 3 次失败，旧文件未被覆盖 | 查看日志，单独重跑该代码 |
| 行业统计缺少某些股票 | `stocklist.csv` 无对应代码，或 `data` 中缺少 CSV | 检查代码匹配与文件 |
| 价格反查无结果 | 容差过小或日期边界限制过窄 | 增大 `--tolerance`，检查日期 |
| `select_stock` 找不到子目录行情 | `all` 模式只扫描一级目录 | 用 `--tickers` 指定，或把 CSV 放到 `data/` 根目录 |

---

## 7. 已知限制与坑

### 7.1 文档与实现不一致

| 项 | 说明 |
| --- | --- |
| 策略数量 | [README.md](../README.md#L45) 写“现包含 5 个策略”，实际 `Selector.py` 与 `configs.json` 为 **6 个** |
| 暴力K 的 JSON 示例 | [README.md](../README.md#L330-L347) 的 JSON 代码块**未闭合**，会影响 Markdown 渲染 |
| `--tickers` | `select_stock.py` 支持，但 README 未记录 |
| 新增脚本 | README 的“项目结构”未包含 `SectorShift.py`、`find_stock_by_price_concurrent.py`、`appendix.json` |
| 量能窗口 | README 称前置窗口“等长或截断”，实现中上限固定为 10 天 |
| 价格容差 | `find_stock_by_price_concurrent.py` 的 help 写默认 0.01，argparse 实际默认 `0.00` |

### 7.2 行为性限制

| 项 | 影响 | 说明 |
| --- | --- | --- |
| 指标在截断窗口内重算 | 同一股票在不同战法下 J/BBI 值可能不同 | `select` 先 `tail(n)` 再计算，KDJ 初值 50、EMA/MA 预热都从窗口首行开始 |
| 缓冲长度依赖参数 | 修改 `max_window` 会同时改变指标值和样本长度 | 调参时结果变化可能来自“窗口变化”而非“阈值变化” |
| `ZXDKX` 需要 114 根 | 历史不足时知行条件恒为 `False` | 现有配置缓冲 120/140 可满足 |
| 全量覆盖无增量 | 每次抓取都重新拉全历史 | 复权一致性更好，但请求量大 |
| 抓取失败保留旧文件 | 下游可能使用过期数据 | 见 6 的排错表 |
| `select` 只扫描一级目录 | 子目录 CSV 不会被 `all` 模式发现 | `find_stock` 是递归的，二者不一致 |
| `SectorShift` 扫描多格式但只读 CSV | `.feather/.parquet` 或子目录文件会被跳过 | 代码列表与可加载数据可能不一致 |
| `_find_peaks` 的 `prominence=0.5` | 绝对价格阈值，低价股/高价股敏感度不同 | 未做价格归一化，跨价位比较不稳定 |
| `MA60` 使用 `min_periods=1` | 数据不足 60 天时使用部分均值 | 与严格 MA60 有差异；选股缓冲通常已覆盖 |
| `peaks` 保留原索引 | 库复用时若索引重复，`get_loc` 可能返回切片 | `select_stock` 的 CSV 索引是唯一 RangeIndex，不受影响 |
| 无测试与打包 | 无自动化回归保障 | 仓库无 `tests/`、`setup.py`、`pyproject.toml` |
| `appendix.json` 未被引用 | 修改它不会影响任何流程 | 可视为遗留文件 |

### 7.3 扩展时的注意事项

1. 新增 Selector 时，`select` 的缓冲长度必须 ≥ 114（若使用知行约束）；
2. 若构造函数新增必填参数，必须在 `configs.json` 中提供，否则该战法会被跳过；
3. 复用 `_passes_filters` 时，确保传入的 `DataFrame` 按日期升序且索引唯一；
4. 不要假设 `hist` 的索引从 0 开始；`select_stock` 读取 CSV 后是连续索引，但外部调用不一定。
