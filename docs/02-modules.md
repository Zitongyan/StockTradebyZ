# 02 · 模块职责与接口

本仓库共 5 个 Python 模块。按“入口脚本 / 策略库 / 分析工具”分类说明。

---

## 1. `fetch_kline.py` — 行情抓取

**职责**：把股票池的日线行情从 Tushare 落到本地 CSV，供离线选股使用。

### 1.1 全局常量与日志

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `LOG_FILE` | [L22](../fetch_kline.py#L22) | 固定为当前工作目录下 `fetch.log` |
| `logger` | [L31](../fetch_kline.py#L31) | 名称 `fetch_from_stocklist`，同时输出 stdout 与文件 |
| `COOLDOWN_SECS` | [L34](../fetch_kline.py#L34) | 限流冷却基数 600 秒 |
| `BAN_PATTERNS` | [L35-L40](../fetch_kline.py#L35-L40) | 识别“访问频繁 / 429 / 403 / forbidden / max retries exceeded”等限流特征 |
| `pro` | [L57](../fetch_kline.py#L57) | 模块级 `ts.pro_api` 会话；`set_api` 可注入外部会话（便于 GUI 复用） |

日志为**追加模式**，无轮转；多次运行会持续增长。

### 1.2 函数清单

| 函数 | 位置 | 输入 | 输出/行为 |
| --- | --- | --- | --- |
| `_looks_like_ip_ban(exc)` | [L42](../fetch_kline.py#L42) | 异常对象 | 异常消息小写后是否包含任一 `BAN_PATTERNS` |
| `RateLimitError` | [L46](../fetch_kline.py#L46) | — | `RuntimeError` 子类，标记需要长冷却的错误 |
| `_cool_sleep(base_seconds)` | [L50](../fetch_kline.py#L50) | 冷却基数 | 叠加 `U(0.9,1.2)` 抖动后 `time.sleep`，并记录 warning |
| `set_api(session)` | [L59](../fetch_kline.py#L59) | 已创建的 pro 会话 | 赋值全局 `pro`，供外部注入 |
| `_to_ts_code(code)` | [L65](../fetch_kline.py#L65) | 6 位代码 | 补零后按前缀映射：`60/68/9 → .SH`，`4/8 → .BJ`，其余 `.SZ` |
| `_get_kline_tushare(code,start,end)` | [L75](../fetch_kline.py#L75) | 代码、起止日期 `YYYYMMDD` | 调 `ts.pro_bar(adj="qfq", freq="D")`；返回 6 列升序 DataFrame；空数据返回空 DF；命中限流抛 `RateLimitError` |
| `validate(df)` | [L102](../fetch_kline.py#L102) | 行情 DF | 去重（按 `date`）、升序、重置索引；缺失日期或未来日期抛 `ValueError`；空 DF 原样返回 |
| `_filter_by_boards_stocklist(df, exclude_boards)` | [L114](../fetch_kline.py#L114) | stocklist DF、排除集合 | 依据 `symbol` / `ts_code` 过滤创业板、科创板、北交所 |
| `load_codes_from_stocklist(stocklist_csv, exclude_boards)` | [L134](../fetch_kline.py#L134) | 清单路径 | 过滤板块 → `symbol` 补 6 位 → 保序去重 → 代码列表 |
| `fetch_one(code,start,end,out_dir)` | [L144](../fetch_kline.py#L144) | 单只代码等 | 最多 3 次抓取，成功则覆盖写 `{code}.csv`；3 次均失败仅记录 error |
| `main()` | [L173](../fetch_kline.py#L173) | CLI | 解析参数、校验 token、创建输出目录、并发调度 |

### 1.3 关键行为细节

**板块过滤**（[L114-L132](../fetch_kline.py#L114-L132)）：

| 取值 | 规则 |
| --- | --- |
| `gem` | `symbol` 以 `300` / `301` 开头 |
| `star` | `symbol` 以 `688` 开头 |
| `bj` | `ts_code` 以 `.BJ` 结尾，或 `symbol` 以 `4` / `8` 开头 |

**限流与重试**（[L144-L170](../fetch_kline.py#L144-L170)）：

| 情形 | 处理 |
| --- | --- |
| 命中封禁特征 | 记录 error → 冷却约 600 秒（含抖动）→ 进入下一次尝试 |
| 其它异常（网络、解析等） | 等待 `15 × 尝试次数` 秒后重试 |
| 成功 | `to_csv(index=False)` **直接覆盖**，`break` |
| 返回空数据 | 写入仅含表头的空 CSV |
| 3 次均失败 | 记录 `三次抓取均失败，已跳过`，**不写文件** |

> 注意：3 次失败时不会删除既有文件。若之前已有旧 CSV，本次失败会留下**过期行情**，下游无法感知。详见 [05 · 已知限制](./05-data-config-runtime.md#7-已知限制与坑)。

**输出 CSV 契约**：

```
date,open,close,high,low,volume
```

- `date` 为 `datetime64`，升序；
- 其余 5 列为数值型；
- 前复权（qfq），每次全量覆盖，不做增量拼接。

**环境变量处理**（[L193-L200](../fetch_kline.py#L193-L200)）：

- 强制设置 `NO_PROXY` / `no_proxy` 为 `api.waditu.com,.waditu.com,waditu.com`；
- 缺少 `TUSHARE_TOKEN` 时抛 `ValueError`，进程退出。

### 1.4 CLI

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--start` | `20190101` | 起始日期 `YYYYMMDD`，支持 `today` |
| `--end` | `today` | 结束日期，支持 `today` |
| `--stocklist` | `./stocklist.csv` | 股票清单路径 |
| `--exclude-boards` | 空列表 | 多选：`gem` / `star` / `bj` |
| `--out` | `./data` | 输出目录，自动创建 |
| `--workers` | `6` | 并发线程数 |

---

## 2. `select_stock.py` — 选股编排入口

**职责**：加载行情与配置，动态实例化 Selector，逐个执行并输出结果。

### 2.1 日志

| 符号 | 位置 | 说明 |
| --- | --- | --- |
| `logger` | [L23](../select_stock.py#L23) | 名称 `select`；`select_results.log` 追加写入，同时输出 stdout |

### 2.2 函数清单

| 函数 | 位置 | 说明 |
| --- | --- | --- |
| `load_data(data_dir, codes)` | [L28](../select_stock.py#L28) | 逐个读取 `{code}.csv`，`parse_dates=["date"]` 并升序排序；缺失文件 warning 后跳过 |
| `load_config(cfg_path)` | [L40](../select_stock.py#L40) | 解析 JSON；缺失文件/空配置直接 `sys.exit(1)` |
| `instantiate_selector(cfg)` | [L62](../select_stock.py#L62) | 校验 `class` 字段；动态导入 `Selector` 并 `getattr`；返回 `(alias, instance)` |
| `main()` | [L80](../select_stock.py#L80) | 参数解析、数据加载、交易日推断、循环执行 |

### 2.3 配置解析规则

`load_config` 兼容三种顶层结构（[L44-L53](../select_stock.py#L44-L53)）：

```jsonc
// 1) 数组
[ { "class": "...", ... }, ... ]

// 2) 带 selectors 键
{ "selectors": [ { "class": "...", ... } ] }

// 3) 单个选择器对象
{ "class": "...", ... }
```

每个配置项字段：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `class` | 是 | `Selector.py` 中的类名；缺失抛 `ValueError` |
| `alias` | 否 | 结果展示名，缺省用类名 |
| `activate` | 否 | 默认 `true`；显式为 `false` 时跳过 |
| `params` | 否 | 透传给类构造函数的参数；未知参数会触发 `TypeError`，该配置被跳过并记录 error |

### 2.4 执行流程与退出条件

| 场景 | 行为 |
| --- | --- |
| `--data-dir` 不存在 | error + `sys.exit(1)` |
| 股票池为空 | error + `sys.exit(1)` |
| 所有 CSV 均加载失败 | error + `sys.exit(1)` |
| 某个 Selector 实例化失败 | error 记录后 `continue`，不影响其他战法 |
| 未指定 `--date` | 取所有 DataFrame `date.max()` 的最大值 |
| 命中结果为空 | 输出“无符合条件股票” |

结果格式示例（同时写入日志）：

```
============== 选股结果 [少妇战法] ==============
交易日: 2025-09-10
符合条件股票数: 3
000001, 600000, 300750
```

### 2.5 CLI

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--data-dir` | `./data` | 行情目录；`all` 模式只扫描一级 `*.csv` |
| `--config` | `./configs.json` | Selector 配置 |
| `--date` | 数据最新日期 | 交易日，交给 `pd.to_datetime` 解析 |
| `--tickers` | `all` | `all` 或逗号分隔代码；**README 未记录该参数** |

---

## 3. `Selector.py` — 指标与战法库

**职责**：提供公共指标函数与 6 个 Selector 类。全部为无副作用的纯计算（不读写文件、不发网络请求）。

### 3.1 符号总览

| 类别 | 符号 | 位置 |
| --- | --- | --- |
| 指标 | `compute_kdj` | [L9](../Selector.py#L9) |
| 指标 | `compute_bbi` | [L29](../Selector.py#L29) |
| 指标 | `compute_rsv` | [L37](../Selector.py#L37) |
| 指标 | `compute_dif` | [L52](../Selector.py#L52) |
| 判定 | `bbi_deriv_uptrend` | [L59](../Selector.py#L59) |
| 工具 | `_find_peaks` | [L107](../Selector.py#L107) |
| 工具 | `last_valid_ma_cross_up` | [L144](../Selector.py#L144) |
| 指标 | `compute_zx_lines` | [L171](../Selector.py#L171) |
| 过滤 | `passes_day_constraints_today` | [L190](../Selector.py#L190) |
| 判定 | `zx_condition_at_positions` | [L211](../Selector.py#L211) |
| 战法 | `BBIKDJSelector` | [L247](../Selector.py#L247) |
| 战法 | `SuperB1Selector` | [L348](../Selector.py#L348) |
| 战法 | `PeakKDJSelector` | [L474](../Selector.py#L474) |
| 战法 | `BBIShortLongSelector` | [L591](../Selector.py#L591) |
| 战法 | `MA60CrossVolumeWaveSelector` | [L700](../Selector.py#L700) |
| 战法 | `BigBullishVolumeSelector` | [L836](../Selector.py#L836) |

详细签名与调用契约见 [03-api-reference.md](./03-api-reference.md)，业务规则见 [04-strategies.md](./04-strategies.md)。

### 3.2 统一调用契约

所有 Selector 类均满足：

```python
def select(self, date: pd.Timestamp, data: Dict[str, pd.DataFrame]) -> List[str]
def _passes_filters(self, hist: pd.DataFrame) -> bool
```

`select` 的公共职责：

1. 对每只股票执行 `df[df["date"] <= date]`（“截至目标日”语义，非“等于目标日”）；
2. 按各战法所需历史长度做 `tail(n)` 截断；
3. 调用 `_passes_filters`，命中则收集代码。

> 由于指标在**截断后的窗口**内重新计算，KDJ 的递推初值、EMA/MA 预热均从窗口首行开始，因此同一只股票在不同战法（不同缓冲长度）下的指标值可能不同。详见 [05 · 已知限制](./05-data-config-runtime.md#7-已知限制与坑)。

---

## 4. `SectorShift.py` — J 值行业分布

**职责**：统计“指定交易日（或之前最近一日）日线 J 值低于阈值”的股票，并按行业聚合。

> 文件头注释为 `# backend/j_industry_service.py`，说明它是从一个后端服务中抽出的分析函数；README 未介绍该脚本。

### 4.1 函数清单

| 函数 | 位置 | 说明 |
| --- | --- | --- |
| `_list_codes_from_data_dir(data_dir)` | [L14](../SectorShift.py#L14) | 递归扫描 `*.csv/*.feather/*.parquet/*.pkl`，用正则 `(\d{6})` 从文件名提取代码，返回排序去重列表 |
| `_load_industry_from_stocklist(stocklist_path, codes)` | [L30](../SectorShift.py#L30) | 从清单读取代码与行业，返回 `['代码','行业']`；支持 `symbol/ts_code/code` 代码列与 `industry/行业` 行业列 |
| `compute_j_industry_distribution(...)` | [L73](../SectorShift.py#L73) | 主分析函数，返回汇总 dict |
| `__main__` | [L172](../SectorShift.py#L172) | CLI：只打印 `industry_counts` 的 JSON |

### 4.2 `compute_j_industry_distribution` 流程

签名：

```python
compute_j_industry_distribution(
    *,
    data_dir,                       # 行情目录（关键字参数）
    stocklist_path="stocklist.csv",
    j_threshold=15.0,
    export_excel_path=None,
    trade_date=None,
) -> Dict
```

步骤：

1. 解析 `trade_date`：支持 `datetime`、`YYYYMMDD`、`YYYY-MM-DD`；非法值抛 `ValueError`；
2. `_list_codes_from_data_dir` 扫描代码；
3. `load_data` 读取行情（**只读 `{code}.csv`**）；
4. 对每只股票取 `date <= trade_date` 的最后一行，计算 KDJ 的当日 `J`；
5. 从 `stocklist.csv` 取行业并左连接；
6. 过滤 `J < j_threshold`，按行业 `value_counts`；
7. 可选导出 Excel（工作表名 `行业分布`）；
8. 返回：

```json
{
  "meta": {
    "total_codes": 5429,
    "selected_count": 120,
    "j_threshold": 15.0,
    "trade_date": "2025-09-10"
  },
  "industry_counts": [
    { "行业": "银行", "股票数": 20 }
  ]
}
```

### 4.3 注意事项

- 代码扫描是**递归**且支持多格式，但 `load_data` 只按 `data_dir/{code}.csv` 读取；若目录下只有 `.feather/.parquet` 或文件位于子目录，会被扫描到但加载跳过。
- 缺少行业列时抛 `ValueError`，不会自动推断。
- CLI 未暴露 `export_excel_path`，导出能力只能通过函数调用使用。

### 4.4 CLI

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--data_dir` | `./data` | 行情目录 |
| `--stocklist` | `stocklist.csv` | 股票清单路径 |
| `--j_threshold` | `15.0` | 日线 J 阈值 |
| `--trade_date` | `None` | 交易日；为空时使用每只股票数据的最后一行 |

---

## 5. `find_stock_by_price_concurrent.py` — 历史价格反查

**职责**：给定价格与容差，在全部行情 CSV 中找出满足条件的历史记录。

### 5.1 函数清单

| 函数 | 位置 | 说明 |
| --- | --- | --- |
| `load_single_stock_data(csv_file)` | [L22](../find_stock_by_price_concurrent.py#L22) | 读取单个 CSV，返回 `(代码, DataFrame)`；失败或空表返回 `None` |
| `load_stock_data_concurrent(data_dir, max_workers=None)` | [L33](../find_stock_by_price_concurrent.py#L33) | 递归 `glob("**/*.csv")`，`ProcessPoolExecutor` 并发读取，合并为列表 |
| `find_by_price_single_stock(...)` | [L70](../find_stock_by_price_concurrent.py#L70) | 单只股票按价格区间与日期区间筛选，返回 `(代码, 价格, 日期)` 列表 |
| `find_by_price_concurrent(...)` | [L120](../find_stock_by_price_concurrent.py#L120) | 校验 `price_type`，`partial` 固定参数后进程池并发查询 |
| `print_results(results, price_type)` | [L187](../find_stock_by_price_concurrent.py#L187) | 控制台表格输出 |
| `main()` | [L207](../find_stock_by_price_concurrent.py#L207) | 参数解析、加载、查询、可选性能统计 |

### 5.2 查询语义

- `price_type ∈ {close, high, low}`，其它值抛 `ValueError`；
- 价格区间为 `[target - tolerance, target + tolerance]`，闭区间；
- 日期筛选：
  - 只给 `start_date` → 同时作为 `end_date`；
  - 只给 `end_date` → 同时作为 `start_date`；
  - 都不给 → 全历史；
- 结果按 `(代码, 日期)` 排序。

### 5.3 并发与入口

- 模块入口显式调用 `mp.set_start_method("spawn", force=True)`（[L268](../find_stock_by_price_concurrent.py#L268)），兼容 Windows 与跨平台；
- 加载与查询均使用进程池，默认 `min(cpu_count, 文件数)`；
- 代价：DataFrame 需要在进程间序列化，大股票池时内存与启动开销明显。

### 5.4 CLI

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `price`（位置参数） | 必填 | 目标价格 |
| `--data-dir` | `./data` | 数据目录，递归查找 CSV |
| `--price-type` | `close` | `close` / `high` / `low` |
| `--start-date` | 无 | `YYYY-MM-DD` |
| `--end-date` | 无 | `YYYY-MM-DD` |
| `--tolerance` | `0.00` | 价格容差；help 文本写的是 0.01，与实现默认值不一致 |
| `--max-workers` | CPU 核数 | 并发进程数 |
| `--benchmark` | 关闭 | 输出加载/查询/总耗时与统计 |

---

## 6. 数据与配置资产

| 文件 | schema / 内容 | 使用方 |
| --- | --- | --- |
| [stocklist.csv](../stocklist.csv) | `ts_code,symbol,name,area,industry`，约 5429 行 | `fetch_kline` 取股票池；`SectorShift` 取行业 |
| [configs.json](../configs.json) | `{"selectors": [...]}`，6 个战法配置 | `select_stock` |
| [appendix.json](../appendix.json) | `{"data": ["002527"]}` | 源码中**无任何引用**，可视为遗留示例 |
| `data/{code}.csv` | `date,open,close,high,low,volume` | 所有选股与分析脚本 |
| `fetch.log` | 抓取日志，追加 | 运维排查 |
| `select_results.log` | 选股结果日志，追加 | 结果留档 |

`data/` 目录不在版本库中，首次运行 `fetch_kline.py` 后生成。
