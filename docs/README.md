# Z哥战法 Python 实现 · Code Wiki

> 本 Wiki 面向开发者，基于仓库当前源码逐行梳理生成，覆盖整体架构、模块职责、关键类/函数、依赖关系、策略算法、数据规范与运行方式。

## 1. 项目定位

这是一个 A 股日线级别的**技术面选股工具**：

1. 通过 **Tushare** 拉取全市场日线行情（前复权 qfq），落盘为每只股票一个 CSV；
2. 通过 **配置驱动** 的方式加载若干“战法”（Selector），对历史行情做批量过滤；
3. 输出符合条件的股票代码，并支持行业分布统计、按历史价格反查等辅助分析。

项目是单机脚本集合，不包含 Web 服务、数据库、任务调度或测试框架。

## 2. 技术栈

| 类别 | 选型 | 说明 |
| --- | --- | --- |
| 语言 | Python 3.11 / 3.12 | README 推荐 3.11/3.12，源码使用了 `from __future__ import annotations` 与 `X | None` 类型写法 |
| 数据处理 | pandas 2.3.0 | 行情读取、滚动指标、批量过滤 |
| 数值计算 | numpy（由 pandas/scipy 间接安装）、scipy 1.14.1 | `scipy.signal.find_peaks` 用于填坑战法的峰值识别 |
| 数据源 | tushare 1.4.21 | `ts.pro_bar` 获取日线 |
| 进度展示 | tqdm 4.66.4 | 并发抓取进度条 |
| 并发 | 标准库 `concurrent.futures` | 抓取用线程池，价格反查用进程池 |
| 配置 | JSON | `configs.json` 定义选择器与参数 |

依赖声明见 [requirements.txt](../requirements.txt)。

## 3. 文档地图

| 文档 | 内容 |
| --- | --- |
| [01-architecture.md](./01-architecture.md) | 分层架构、端到端数据流、模块依赖图、设计模式与扩展点 |
| [02-modules.md](./02-modules.md) | 每个文件的职责、公开入口、关键函数、日志与副作用 |
| [03-api-reference.md](./03-api-reference.md) | 指标函数与 6 个 Selector 类的参数、返回值、调用契约 |
| [04-strategies.md](./04-strategies.md) | 六大战法的业务规则、条件流水线、公式与配置映射 |
| [05-data-config-runtime.md](./05-data-config-runtime.md) | 数据/配置规范、全部 CLI 参数、运行步骤、错误处理、已知限制 |

推荐阅读路径：

- **第一次接触项目**：本页 → 01 → 05（跑通流程）→ 04（理解策略）
- **要改策略**：03 → 04 → 01 的“扩展点”
- **要排查选股结果异常**：05 的“已知限制与坑” → 03 的“调用契约”

## 4. 仓库文件清单

| 路径 | 类型 | 职责 |
| --- | --- | --- |
| [fetch_kline.py](../fetch_kline.py) | 入口脚本 | 从 `stocklist.csv` 读取股票池，用 Tushare 并发抓取 qfq 日线，按股票全量覆盖写入 `data/` |
| [select_stock.py](../select_stock.py) | 入口脚本 | 读取 `data/` 行情与 `configs.json`，动态实例化 Selector 并批量选股，结果写控制台与 `select_results.log` |
| [Selector.py](../Selector.py) | 库 | 通用技术指标 + 6 个战法 Selector 类 |
| [SectorShift.py](../SectorShift.py) | 分析脚本/库 | 统计指定交易日“日线 J 值低于阈值”的股票行业分布，可导出 Excel |
| [find_stock_by_price_concurrent.py](../find_stock_by_price_concurrent.py) | 入口脚本 | 在全部行情 CSV 中并发反查“某历史价格”出现过的股票与日期 |
| [configs.json](../configs.json) | 配置 | 6 个 Selector 的类名、别名、启用开关与参数 |
| [stocklist.csv](../stocklist.csv) | 数据 | 全市场股票清单（`ts_code,symbol,name,area,industry`），约 5429 条记录 |
| [appendix.json](../appendix.json) | 数据 | 仅含 `{"data": ["002527"]}`，源码中未被引用，属遗留/示例文件 |
| `data/` | 运行时生成 | 每只股票一个 `XXXXXX.csv`，由 `fetch_kline.py` 创建 |
| `fetch.log` / `select_results.log` | 运行时生成 | 各自脚本追加写入的日志 |
| [README.md](../README.md) | 文档 | 项目原始说明，包含策略介绍与免责声明 |

## 5. 五分钟快速开始

```bash
# 1. 安装依赖
pip install -r requirements.txt

# 2. 配置 Tushare token（必须）
export TUSHARE_TOKEN=你的token

# 3. 抓取行情（示例：排除创业板/科创板/北交所）
python fetch_kline.py \
  --start 20240101 \
  --end today \
  --stocklist ./stocklist.csv \
  --exclude-boards gem star bj \
  --out ./data \
  --workers 6

# 4. 运行选股
python select_stock.py \
  --data-dir ./data \
  --config ./configs.json \
  --date 2025-09-10
```

不带 `--date` 时，使用已加载行情中的最后一个交易日。

## 6. 核心术语

| 术语 | 含义 | 实现位置 |
| --- | --- | --- |
| 战法 / Selector | 一套选股规则，对外统一提供 `select(date, data)` | [Selector.py](../Selector.py) |
| 当日过滤 | 当日涨跌幅绝对值 `< 2%` 且振幅 `< 7%`，用于规避异动日 | `passes_day_constraints_today` |
| 知行约束 | 收盘价与“长期线” `ZXDKX`、短期线 `ZXDQ` 的先后关系 | `compute_zx_lines` / `zx_condition_at_positions` |
| ZXDQ（短期线） | `EMA(EMA(close,10),10)` | `compute_zx_lines` |
| ZXDKX（长期线） | `(MA14 + MA28 + MA57 + MA114) / 4` | `compute_zx_lines` |
| BBI | `(MA3 + MA6 + MA12 + MA24) / 4` | `compute_bbi` |
| 有效上穿 MA60 | 前一日收盘 `< MA60` 且当日收盘 `≥ MA60` | `last_valid_ma_cross_up` |
| 分位过滤 | 当日 J 值同时满足“绝对阈值”或“历史分位阈值” | 各 Selector 的 KDJ 判断 |
| 缓冲窗口 | `select` 在目标日前额外截取的 K 线数量，保证指标有足够历史 | 各 Selector 的 `select` |

## 7. 免责声明

仓库仅用于学习与技术研究，不构成投资建议。行情接口与复权数据可能随时间变化，请合法合规使用。
