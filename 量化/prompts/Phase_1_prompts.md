# Phase 1：数据基建 — Claude Code 执行 Prompt

> 使用方法：逐条复制下面的 prompt 发给 Claude Code，按编号顺序执行。每条 prompt 是一个独立的子任务，完成后再发下一个。

---

## P1.01 — 项目初始化 + SQLite 数据库建表

```
在 D:/NOTETAKING/STOCKS/量化/ 目录下创建 Python 量化研究项目的数据基础设施：

1. 创建目录结构：
   - 量化/data/db/        （SQLite数据库文件）
   - 量化/data/csv/        （CSV缓存）
   - 量化/data/raw/        （原始数据缓存）
   - 量化/scripts/         （Python脚本）
   - 量化/notebooks/       （Jupyter notebook，如果需要）

2. 在 scripts/ 下创建 init_db.py，用 SQLite 建立以下表结构：

   stocks 表:
   - ts_code TEXT PRIMARY KEY (如 '000001.SZ')
   - name TEXT
   - sector TEXT (8个领域之一: 芯片/PCB/GPU显卡/电脑设备/工业设备/光纤光模块/数据库/算力服务器)
   - sub_sector TEXT
   - supply_chain_pos TEXT (upstream/midstream/downstream)
   - cross_sector TEXT (跨领域映射，逗号分隔)
   - market TEXT (A股/港股/美股)
   - listing_date TEXT
   - is_active BOOLEAN DEFAULT 1

   daily_price 表:
   - ts_code TEXT
   - trade_date TEXT (YYYYMMDD)
   - open REAL, high REAL, low REAL, close REAL
   - vol REAL, amount REAL
   - turn REAL (换手率)
   - pct_chg REAL (涨跌幅)
   - PRIMARY KEY (ts_code, trade_date)

   financials 表:
   - ts_code TEXT
   - report_date TEXT (如 '20250331')
   - revenue REAL (营业收入)
   - net_profit REAL (净利润)
   - rd_expense REAL (研发费用)
   - gross_margin REAL (毛利率)
   - roe REAL
   - eps REAL
   - inventory_days REAL (库存周转天数)
   - PRIMARY KEY (ts_code, report_date)

   factor_values 表:
   - factor_id TEXT
   - ts_code TEXT
   - trade_date TEXT
   - factor_value REAL
   - PRIMARY KEY (factor_id, ts_code, trade_date)

   supply_chain_links 表:
   - id INTEGER PRIMARY KEY AUTOINCREMENT
   - upstream_code TEXT
   - downstream_code TEXT
   - lag_months INTEGER
   - strength REAL
   - verified BOOLEAN DEFAULT 0
   - last_verify_date TEXT

3. 运行 init_db.py 创建 量化/data/db/stocks.db

4. 在 scripts/ 下创建 seed_stocks.py，按以下映射表插入 stocks 表数据：

   芯片: 中芯国际(688981.SH), 寒武纪(688256.SH), 海光信息(688041.SH), 北方华创(002371.SZ), 韦尔股份(603501.SH)
   PCB: 鹏鼎控股(002938.SZ), 深南电路(002916.SZ), 沪电股份(002463.SZ), 生益科技(600183.SH)
   GPU显卡: NVIDIA(NVDA,美股), AMD(AMD,美股), 海光信息(688041.SH,跨领域GPU)
   电脑设备: 联想集团(00992.HK,港股), 戴尔(DELL,美股), 惠普(HPQ,美股)
   工业设备: 大族激光(002008.SZ), 华工科技(000988.SZ), 精测电子(300567.SZ), 先导智能(300450.SZ)
   光纤光模块: 中际旭创(300308.SZ), 新易盛(300502.SZ), 光迅科技(002281.SZ), 长飞光纤(601869.SH)
   数据库: 达梦数据(688692.SH), 中国软件(600536.SH)
   算力服务器: 浪潮信息(000977.SZ), 中科曙光(603019.SH)

   注意: 港股和美股的 ts_code 格式分别是 '00992.HK' 和 'NVDA.US'

5. 在 supply_chain_links 表中插入以下传导关系（全部 verified=0）：

   NVIDIA(NVDA.US) → 海光信息(688041.SH), lag=2, strength=0.4
   NVIDIA(NVDA.US) → 中际旭创(300308.SZ), lag=4, strength=0.5
   云厂商CapEx → 中际旭创(300308.SZ), lag=4, strength=0.6 (upstream_code='CLOUD_CAPEX')
   联想集团(00992.HK) → 鹏鼎控股(002938.SZ), lag=2, strength=0.4
   中芯国际(688981.SH) → 深南电路(002916.SZ), lag=1, strength=0.3
   中芯国际(688981.SH) → 北方华创(002371.SZ), lag=4, strength=0.4

6. 运行 seed_stocks.py

请确保所有代码有清晰的中文注释，脚本可以独立运行，并打印执行结果确认。
```

---

## P1.02 — Tushare 数据拉取 Pipeline

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 fetch_tushare.py：

实现一个数据拉取类 TushareFetcher，功能如下：

1. 初始化时读取 Tushare token（从环境变量 TUSHARE_TOKEN 或从 量化/config.ini 文件读取）
   - 如果 config.ini 不存在，先创建它，包含 tushare_token=YOUR_TOKEN_HERE 占位符

2. 方法 fetch_daily(ts_code, start_date='20200101', end_date='20260630')：
   - 调用 ts.pro_api().daily() 获取日线数据
   - 写入 SQLite 量化/data/db/stocks.db 的 daily_price 表
   - 如果数据已存在则跳过（按 ts_code + trade_date 检查）
   - 打印进度：已拉取 X/Y 只股票

3. 方法 fetch_financials(ts_code, start_period='20201', end_period='20261')：
   - 调用 ts.pro_api().income() 获取利润表数据
   - 计算毛利率 = (total_revenue - total_cogs) / total_revenue（如果字段存在）
   - 写入 financials 表

4. 方法 fetch_all_stocks_daily()：
   - 从 stocks 表读取所有 ts_code
   - 对 A股代码调用 fetch_daily
   - 港股和美股跳过（用其他数据源）

5. 方法 fetch_daily_basic(ts_code, start_date, end_date)：
   - 调用 ts.pro_api().daily_basic() 获取 PE_TTM、PB、turnover_rate 等
   - 写入 factor_values 表，factor_id 分别为 'F_PE_TTM'、'F_PB_MRQ'、'F_TURN_20'

6. 创建 main() 函数：
   - 实例化 TushareFetcher
   - 依次调用 fetch_all_stocks_daily()、fetch_financials()、fetch_daily_basic()
   - 最后打印：已拉取 XX 只股票，XX 条日线记录，XX 条财报记录

7. 异常处理：
   - Tushare API 限频：每次调用后 sleep 0.5s
   - 网络超时：重试3次
   - 数据缺失：打印警告但继续

8. 代码注释用中文，函数文档用中文

运行测试时如果 TUSHARE_TOKEN 还没设置，请打印提示："请先在 量化/config.ini 中设置你的 Tushare token"。
```

---

## P1.03 — AKShare + yfinance 补充数据拉取

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 fetch_supplementary.py：

实现港股和美股的数据补充拉取：

1. 类 SupplementaryFetcher：

2. 方法 fetch_hk_daily(code, start='2020-01-01', end='2026-06-30')：
   - 用 akshare 的 ak.stock_hk_daily(symbol=code) 拉取港股日线
   - code 格式：对于联想集团用 '00992'
   - 转换字段名匹配 daily_price 表结构
   - ts_code 格式转为 '00992.HK'
   - 写入 SQLite

3. 方法 fetch_us_daily(symbol, start='2020-01-01', end='2026-06-30')：
   - 用 yfinance 的 yf.Ticker(symbol).history(start=start, end=end) 拉取美股日线
   - symbol: 'NVDA', 'AMD', 'DELL', 'HPQ'
   - 转换字段名匹配 daily_price 表结构
   - ts_code 格式转为 'NVDA.US' 等
   - 写入 SQLite

4. 方法 fetch_macro_data()：
   - 用 AKShare 拉取以下宏观数据，存入 量化/data/db/macro.db（新建）：
     - GDP：ak.macro_china_gdp()
     - CPI：ak.macro_china_cpi()
     - PMI：ak.macro_china_pmi()
     - 北向资金：ak.stock_em_hsgt_north_net_flow_in_em()
   - 各数据存入对应表

5. 方法 fetch_nvidia_financials()：
   - 用 yfinance 拉取 NVIDIA 季度财报数据
   - yf.Ticker('NVDA').quarterly_financials
   - 提取 Total Revenue、Net Income、Gross Profit
   - 计算营收增速、毛利率、净利率
   - 写入 financials 表，ts_code='NVDA.US'

6. main() 函数：
   - 拉取港股（联想集团）日线
   - 拉取美股（NVDA, AMD, DELL, HPQ）日线
   - 拉取 NVIDIA 财报
   - 拉取宏观数据
   - 打印统计信息

7. 异常处理：
   - AKShare 可能因版本变化API不同，需要做兼容性检查
   - yfinance 限频问题处理
   - 网络超时重试

中文注释，可独立运行。
```

---

## P1.04 — 数据质量检查 + 每日更新脚本

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建两个脚本：

### 1. check_data_quality.py

数据质量检查脚本，功能：

1. 检查 daily_price 表：
   - 每只股票应有约 1300 条记录（2020-2026，约5.5年×250交易日）
   - 检查是否有日期缺失（连续交易日中有gap）
   - 检查是否有异常值（涨跌幅 > 30% 且不是涨跌停、成交量为0等）
   - 生成报告：缺失天数、异常值数量

2. 检查 financials 表：
   - 每只股票应有约 20 条记录（5年×4季度）
   - 检查毛利率是否在合理范围（0-100%）
   - 检查是否有负营收（可能数据错误）

3. 检查 stocks 表：
   - 所有 ts_code 在 daily_price 中是否有数据
   - 哪些股票数据缺失

4. 输出报告到 量化/data/csv/data_quality_report.csv

### 2. daily_update.py

每日数据更新脚本：

1. 默认拉取昨天的日线数据（自动计算昨天的日期）
2. 更新所有A股标的的日线数据
3. 如果是财报季末月份（3/6/9/12），额外拉取财报数据
4. 拉取北向资金数据
5. 打印更新结果统计

6. 设计为可以用 cron 或 Windows Task Scheduler 定时运行
   - 脚本接受参数 --date YYYYMMDD 指定日期
   - 默认为 yesterday

中文注释，可独立运行。
```

---

## 完成标志

Phase 1 完成后你应该有以下产出：

| 产出 | 验证方法 |
|------|---------|
| stocks.db | SQLite中有30+只股票、5年日线数据、20+条财报 |
| macro.db | GDP/CPI/PMI/北向资金数据 |
| 数据质量报告 | 无严重缺失，无异常值 |
| 每日更新脚本 | 手动运行一次，确认能拉取最近交易日数据 |

> 📌 完成后进入 Phase 2 因子研究
