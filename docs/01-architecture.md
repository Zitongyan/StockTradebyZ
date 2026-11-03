# 01 · 整体架构

## 1. 架构总览

项目采用“**离线数据 + 配置驱动策略**”的批处理架构。没有服务端，也没有持久化数据库，所有环节都围绕本地 CSV 文件展开。

```
                    ┌────────────────────────────────────────────┐
                    │              外部数据源                     │
                    │         Tushare Pro (日线, qfq)             │
                    └──────────────────────┬─────────────────────┘
                                           │ ts.pro_bar
┌──────────────────────────────────────────▼──────────────────────────────────────────┐
│ 数据采集层  fetch_kline.py                                                           │
│  · 读取 stocklist.csv 股票池，按板块过滤                                               │
│  · ThreadPoolExecutor 并发抓取（默认 6 线程），3 次重试 + 限流冷却                        │
│  · 单只股票全量覆盖写 data/{6位代码}.csv                                                │
└──────────────────────────────────────────┬──────────────────────────────────────────┘
                                           │
                                     data/*.csv
                    ┌──────────────────────┴──────────────────────┐
                    │                                             │
┌───────────────────▼───────────────────┐      ┌────────────────▼─────────────────────┐
│ 策略计算层  Selector.py                │      │ 辅助分析                              │
│  · 指标函数库（KDJ/BBI/RSV/DIF/知行线）│      │  SectorShift.py                      │
│  · 6 个 Selector 类                    │      │   按行业统计 J<阈值 的股票             │
│  · 统一当日过滤 + 知行约束              │      │  find_stock_by_price_concurrent.py    │
└───────────────────┬───────────────────┘      │   并发反查历史价格                      │
                    │ importlib 动态加载        └────────────────┬─────────────────────┘
┌───────────────────▼───────────────────┐                       │
│ 编排层  select_stock.py                │                       │
│  · 扫描 data 目录 / --tickers          │                       │
│  · 解析 configs.json（3 种结构兼容）    │                       │
│  · 反射实例化 Selector 并逐个执行       │                       │
│  · 控制台 + select_results.log 输出     │                       │
└───────────────────────────────────────┘                       │
                    │                                             │
                    └─────────────── data/*.csv ──────────────────┘
                 stocklist.csv（行业信息）   configs.json（策略参数）
```

## 2. 分层职责

| 层 | 组成 | 输入 | 输出 | 是否有网络访问 |
| --- | --- | --- | --- | --- |
| 数据采集层 | `fetch_kline.py` | `stocklist.csv`、环境变量 `TUSHARE_TOKEN`、日期参数 | `data/{code}.csv`、`fetch.log` | 是（Tushare HTTP API） |
| 策略计算层 | `Selector.py` | 单只股票的行情 `DataFrame` | `bool`（是否命中） | 否 |
| 编排层 | `select_stock.py` | `data/*.csv`、`configs.json` | 控制台结果、`select_results.log` | 否 |
| 辅助分析层 | `SectorShift.py`、`find_stock_by_price_concurrent.py` | `data/*.csv`、`stocklist.csv` | JSON/Excel、控制台表格 | 否 |

这种分层决定了：

- 选股阶段可以**完全离线**、可重复执行，适合反复调参；
- 数据抓取与策略计算是解耦的，替换数据源只需产出相同 schema 的 CSV；
- Selector 是无状态的纯函数式对象（只读自身参数，不写文件、不发请求）。

## 3. 端到端数据流

### 3.1 抓取阶段

1. `main()` 读取 `--stocklist` 指向的 CSV（[fetch_kline.py](../fetch_kline.py#L134-L141)）。
   - 必须包含 `symbol` 与 `ts_code` 两列；
   - 依据 `--exclude-boards` 去掉创业板（`300/301`）、科创板（`688`）、北交所（`.BJ` / `4`、`8` 开头）；
   - `symbol` 补足 6 位并去重，得到股票代码列表。
2. 设置 `TUSHARE_TOKEN`，创建模块级 `pro` 会话；同时把 `api.waditu.com` 等域名写入 `NO_PROXY`，避免代理拦截（[fetch_kline.py](../fetch_kline.py#L192-L200)）。
3. 对每个代码提交 `fetch_one` 到 `ThreadPoolExecutor`（[fetch_kline.py](../fetch_kline.py#L223-L235)）。
4. `fetch_one` 内部最多尝试 3 次（[fetch_kline.py](../fetch_kline.py#L144-L170)）：
   - 调 `ts.pro_bar(ts_code=..., adj="qfq", freq="D")`；
   - 列重命名 `trade_date → date`、`vol → volume`，只保留 6 列并按日期升序；
   - `validate` 去重、检查缺失日期与未来日期；
   - **直接覆盖** `data/{code}.csv`。
5. 全部完成后打印数据目录。

### 3.2 选股阶段

1. `main()` 解析参数，确认 `data` 目录存在（[select_stock.py](../select_stock.py#L80-L107)）。
2. 股票池来源二选一：
   - `--tickers all`（默认）：`data_dir.glob("*.csv")`，**只扫描一级目录**；
   - `--tickers 000001,600000`：逗号分隔。
3. `load_data` 逐个读取 CSV，`parse_dates=["date"]` 并升序排序；缺失文件仅告警跳过（[select_stock.py](../select_stock.py#L28-L37)）。
4. 确定交易日：`--date` 指定值，否则取所有已加载行情 `date.max()` 的最大值（[select_stock.py](../select_stock.py#L108-L114)）。
5. `load_config` 解析配置，兼容三种顶层结构（[select_stock.py](../select_stock.py#L40-L59)）：
   - 数组 `[ {...}, {...} ]`；
   - 对象且含 `selectors` 键；
   - 单个对象。
6. 遍历配置：`activate: false` 直接跳过；否则 `instantiate_selector` 通过 `importlib.import_module("Selector")` + `getattr` 反射创建实例（[select_stock.py](../select_stock.py#L62-L75)）。
7. 调用 `selector.select(trade_date, data)`，输出每个战法的命中数量与代码列表（[select_stock.py](../select_stock.py#L120-L136)）。

### 3.3 单只股票的判定链路

所有 Selector 遵循同一套接口约定：

```
df（该股票全历史行情）
   │  select():  df[df.date <= date].tail(缓冲长度)
   ▼
hist（截至目标日的窗口）
   │  _passes_filters():
   │    1) 数据长度检查
   │    2) passes_day_constraints_today()   ← 除暴力K外均启用
   │    3) 战法特有指标（KDJ / BBI / MA60 / RSV / 峰值 / 量能）
   │    4) zx_condition_at_positions()      ← 知行约束
   ▼
bool → picks.append(code)
```

## 4. 模块依赖关系

```
fetch_kline.py
   └── tushare, pandas, tqdm, concurrent.futures

select_stock.py
   ├── pandas, json, argparse
   ├── configs.json（数据依赖）
   └── Selector.py（运行时 importlib 动态导入，非文件顶部静态导入）

Selector.py
   ├── numpy, pandas
   └── scipy.signal.find_peaks

SectorShift.py
   ├── Selector.compute_kdj
   ├── select_stock.load_data
   └── stocklist.csv（行业列）

find_stock_by_price_concurrent.py
   ├── pandas
   └── data/**/*.csv（多进程读取）
```

要点：

- **不存在导入环**：`select_stock` 只在函数内部动态导入 `Selector`；`SectorShift` 虽然同时导入两者，但方向是单向的。
- `Selector.py` 不依赖任何项目内其他模块，可独立复用。
- `fetch_kline.py` 与其余模块完全解耦，可单独替换数据源。
- 没有 `__init__.py`、`setup.py`、`pyproject.toml`，模块以脚本所在目录为导入根。

## 5. 关键设计决策

| 决策 | 实现 | 影响 |
| --- | --- | --- |
| 配置驱动 + 反射加载 | `class` 字段 → `importlib` → `getattr` | 新增战法只需加类与配置，无需改 `select_stock.py` |
| 统一的策略接口 | 所有类实现 `select(date, data)` 与 `_passes_filters(hist)` | 批量编排逻辑完全复用 |
| 每只股票 CSV 落盘 | `data/{code}.csv` | 简单、可人工检查；但无增量更新、无索引 |
| 每次全量覆盖 | `to_csv` 无合并逻辑 | 规避前复权历史值变动导致的拼接错误 |
| 抓取用线程池 | IO 密集，`ThreadPoolExecutor` | 6 线程默认；限流命中时整体冷却 |
| 反查用进程池 | CPU/解析密集，`ProcessPoolExecutor` | 注意 DataFrame 跨进程序列化开销 |
| 当日统一过滤 | `passes_day_constraints_today` | 过滤掉当日大涨大跌/高振幅标的，降低追高风险 |
| 指标在截断窗口内计算 | 各 `select` 按缓冲长度 `tail` | 结果依赖缓冲长度与指标预热，详见 05 的已知限制 |

## 6. 时序概览

### 6.1 抓取

```
CLI → main → load_codes_from_stocklist → ThreadPoolExecutor
                                         ├─ fetch_one(code1) → pro_bar → validate → csv
                                         ├─ fetch_one(code2) → ...
                                         └─ tqdm 汇总
```

### 6.2 选股

```
CLI → main → load_data → load_config
     → for cfg in configs:
          activate? → instantiate_selector → selector.select(date, data)
                                     └─ for code, df: _passes_filters(hist)
          → logger 输出
```

## 7. 扩展点

### 7.1 新增一个战法

1. 在 [Selector.py](../Selector.py) 中新增类，构造函数接收配置参数；
2. 实现 `_passes_filters(self, hist: pd.DataFrame) -> bool`；
3. 实现 `select(self, date, data) -> List[str]`（可复用现有类的 `tail` 缓冲写法）；
4. 在 `configs.json` 的 `selectors` 数组追加：

```json
{
  "class": "YourSelector",
  "alias": "你的战法",
  "activate": true,
  "params": { "your_param": 1 }
}
```

5. 无需修改 `select_stock.py`。若构造函数含嵌套参数字典（如 `B1_params`），直接在 `params` 中嵌套即可。

### 7.2 替换数据源

保持输出 CSV 的列契约：`date, open, close, high, low, volume`，日期升序且可被 `pd.to_datetime` 解析，即可直接接入现有选股流程。

### 7.3 增加分析维度

`SectorShift.py` 展示了复用方式：从 `Selector` 取指标、从 `select_stock` 取数据加载，组合成新的统计函数。新增同类分析脚本时沿用此模式，避免复制 CSV 读取逻辑。
