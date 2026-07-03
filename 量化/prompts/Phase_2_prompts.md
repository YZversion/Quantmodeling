# Phase 2：因子研究 — Claude Code 执行 Prompt

> 使用方法：逐条复制发给 Claude Code。Phase 1 完成后再开始。

---

## P2.01 — 基本面因子计算引擎

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 factor_engine.py：

实现基本面因子的批量计算和存储。

依赖数据：量化/data/db/stocks.db（Phase 1 产出）

1. 类 FactorEngine：

2. 方法 calc_valuation_factors(ts_code, date)：
   从 daily_price + financials 表计算：
   - F_PE_TTM = 总市值 / 最近12月净利润
     （总市值 = close_price × 总股本，总股本从 financials 或 stocks 获取）
   - F_PB_MRQ = 总市值 / 最新报告期净资产
   - F_PS_TTM = 总市值 / 最近12月营收
   - 注意：PE为负的公司标记为 NaN，不参与排序

3. 方法 calc_growth_factors(ts_code, report_date)：
   从 financials 表计算：
   - F_GROWTH_REV = (本期营收 - 上期营收) / 上期营收（同比）
   - F_GROWTH_PROFIT = (本期净利润 - 上期净利润) / 上期净利润（同比）
   - F_RD_RATIO = 研发费用 / 营收
   - 处理基数过小的情况（上期营收 < 1亿时增速可能扭曲）

4. 方法 calc_quality_factors(ts_code, report_date)：
   - F_ROE_TTM = 近12月净利润 / 股东权益
   - F_GROSS_MARGIN = 毛利率（直接从 financials 读取或计算）
   - F_INV_TURN = 营收 / 平均库存（库存周转率）
   - F_INV_DAYS_CHANGE = 本期库存周转天数 - 上期库存周转天数（变化量）

5. 方法 calc_all_fundamental_factors(date=None)：
   - 如果 date=None，对所有有数据的日期计算
   - 如果 date='YYYYMMDD'，只计算该日期
   - 批量写入 factor_values 表
   - 打印进度

6. 方法 calc_sector_relative_factors(ts_code, date)：
   计算因子在同板块内的排名百分位：
   - F_PE_RANK = PE在同板块内的排名（低PE排名高=1，高PE排名低=0）
   - F_GROWTH_RANK = 营收增速在同板块内的排名
   - F_ROE_RANK = ROE在同板块内的排名
   - 从 stocks 表读取 sector，从 factor_values 读取原始值，计算排名

7. main() 函数：
   - 实例化 FactorEngine
   - 计算所有基本面因子
   - 计算板块相对排名因子
   - 打印：已计算 XX 个因子，XX 条记录

中文注释，处理 NaN 和极端值，可独立运行。
```

---

## P2.02 — 技术面因子计算

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 tech_factors.py：

实现技术面因子计算，从 daily_price 表读取数据。

1. 类 TechFactorCalculator：

2. 方法 calc_momentum(ts_code, lookback=20)：
   - T_MOM_20 = (close[t] - close[t-20]) / close[t-20]
   - T_MOM_60 = (close[t] - close[t-60]) / close[t-60]
   - 处理上市不满60天的情况（返回NaN）

3. 方法 calc_reversal(ts_code, lookback=5)：
   - T_REV_5 = (close[t] - close[t-5]) / close[t-5]
   - 注意：反转因子方向与动量相反（跌了之后可能反弹）

4. 方法 calc_volatility(ts_code, window=20)：
   - T_VOL_20 = std(daily_return, 20天) × sqrt(250) （年化波动率）
   - daily_return = pct_chg / 100

5. 方法 calc_volume_factors(ts_code, window=20)：
   - T_TURN_20 = mean(换手率, 20天)
   - T_VOL_CHANGE = (vol[t] - mean(vol[t-5:t])) / mean(vol[t-5:t])

6. 方法 calc_all_tech_factors(end_date=None)：
   - 对所有A股标的计算所有技术因子
   - 写入 factor_values 表
   - 打印进度

7. 注意事项：
   - 港股和美股的换手率可能字段名不同，需要适配
   - 涨跌停日的数据需要正常处理（不要排除）
   - 新股上市不满20天的因子标记NaN

中文注释，可独立运行。
```

---

## P2.03 — 因子 IC 检验

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 factor_ic_test.py：

实现因子有效性检验（IC = Information Coefficient）。

1. 类 FactorICTest：

2. 方法 calc_forward_return(ts_code, trade_date, horizon=20)：
   - 从 daily_price 计算未来20日收益率
   - forward_return = (close[t+20] - close[t]) / close[t]
   - 如果 t+20 超出数据范围，返回NaN

3. 方法 calc_ic(factor_id, date, horizon=20)：
   - 在 date 这天，对所有有数据的股票：
     1. 读取 factor_values 中该因子的值
     2. 计算对应的前瞻收益率 forward_return
     3. 计算 Spearman 秩相关系数（scipy.stats.spearmanr）
   - 返回 IC值 和 p值

4. 方法 calc_ic_series(factor_id, start_date, end_date, horizon=20)：
   - 对每个交易日计算 IC
   - 返回 IC时间序列
   - 计算：IC均值、IC标准差、ICIR（IC均值/IC标准差）、|IC|>0.03的比例

5. 方法 calc_all_factors_ic_report(start_date='20210101', end_date='20260630')：
   - 对所有因子计算 IC 报告
   - 输出为 DataFrame，包含：
     factor_id | IC_mean | IC_std | ICIR | pct_positive | pct_abs_003 | rank_corr_pvalue
   - 保存到 量化/data/csv/factor_ic_report.csv

6. 方法 calc_ic_by_sector(factor_id, sector, start_date, end_date)：
   - 在指定板块内计算 IC（只比较同板块的股票）
   - 返回分板块的 IC 报告

7. 方法 visualize_ic(factor_id)：
   - 用 matplotlib 绘制 IC 时间序列图
   - 保存到 量化/data/csv/ic_charts/ 目录
   - 包含 IC 滚动均值线（12个月滚动窗口）

8. main() 函数：
   - 计算所有基本面因子 IC
   - 计算所有技术面因子 IC
   - 计算分板块 IC
   - 打印 IC > 0.05 的有效因子清单

9. 输出格式参考：
   factor_id  | IC_mean | IC_std | ICIR  | pct_abs_003 | verdict
   F_PE_TTM   | -0.06  | 0.15  | -0.40 | 45%         | 有效
   F_ROE_TTM  |  0.04  | 0.12  |  0.33 | 38%         | 弱有效
   T_MOM_20   |  0.02  | 0.18  |  0.11 | 25%         | 无效

中文注释，需要 scipy 和 matplotlib。
```

---

## P2.04 — 供应链传导 Granger 因果检验

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 supply_chain_granger.py：

这是你的核心 alpha——验证供应链传导假设。

依赖数据：量化/data/db/stocks.db + 量化/data/db/macro.db

1. 类 SupplyChainGranger：

2. 方法 prepare_series(upstream_code, downstream_code, upstream_metric='revenue_growth', downstream_metric='monthly_return')：
   - 从 daily_price 和 financials 准备月度数据序列
   - upstream_metric 选项：
     'revenue_growth'（营收增速，季度→需要转月度）
     'nvidia_revenue'（NVIDIA季报营收，NVDA.US）
     'capex'（云资本开支，从 macro.db 读取 [MISSING: 需补充数据]）
   - downstream_metric 选项：
     'monthly_return'（月度股价收益率）
     'revenue_growth'（下游营收增速）
   - 如果上游数据是季度，用线性插值转为月度
   - 处理日期对齐（确保两个序列的月份完全匹配）

3. 方法 granger_test(upstream_series, downstream_series, max_lag=6)：
   - 用 statsmodels.tsa.stattools.grangercausalitytests 检验
   - 返回每个lag的 F检验p值
   - 标记显著lag（p < 0.05）

4. 方法 test_all_links()：
   - 从 supply_chain_links 表读取所有传导关系
   - 对每条关系：
     1. prepare_series
     2. granger_test
     3. 更新 supply_chain_links 表：
        - verified = True
        - last_verify_date = 今天
        - lag_months = 最显著的lag值
        - strength = 最显著lag的F统计量
   - 打印每条结果

5. 方法 rolling_regression(upstream_series, downstream_series, window=12, lag=2)：
   - 用 sklearn LinearRegression 做滚动回归
   - 窗口=12个月，上游领先lag个月
   - 返回传导系数的时间序列

6. 方法 visualize_transmission(upstream_code, downstream_code)：
   - 绘制两列对比图：上游指标 vs 下游股价
   - 标记显著Granger因果的lag
   - 保存到 量化/data/csv/granger_charts/

7. main() 函数：
   - 对所有传导关系做Granger检验
   - 生成传导验证报告：量化/data/csv/supply_chain_report.csv
   - 报告格式：
     upstream | downstream | significant_lags | best_lag | F_stat | strength | verdict

8. 特别处理 NVIDIA → 国内传导链：
   - NVIDIA季报数据需要从 financials 表读取（ts_code='NVDA.US'）
   - 季度→月度插值方法：线性插值（假设季度内均匀分布）
   - 国内股价数据从 daily_price 转为月度收益率

中文注释，中文变量名和函数文档。
```

---

## P2.05 — 因子正交化 + 因子组合初版

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 factor_orthogonal.py：

实现因子正交化和组合权重计算。

1. 类 FactorOrthogonalizer：

2. 方法 calc_factor_correlation(start_date, end_date)：
   - 计算所有因子之间的相关系数矩阵
   - 输出 DataFrame：factor1 × factor2 的 Spearman 相关系数
   - 保存到 量化/data/csv/factor_corr_matrix.csv

3. 方法 orthogonalize_rank(factor_df)：
   - 对因子做排名正交化（Madoff方法）：
     1. 所有因子转为排名（cross-section rank）
     2. 对排名做PCA正交化
     3. 返回正交化后的因子值
   - 使用 sklearn.decomposition.PCA

4. 方法 calc_ic_weighted_combination(ic_report, factor_values_df)：
   - 根据 IC 报告计算因子权重：
     weight_i = IC_mean_i / IC_std_i （ICIR加权）
   - 对 IC < 0 的因子（如PE），取负权重
   - 计算每只股票的综合因子得分 = Σ(weight_i × factor_value_i)
   - 返回综合得分 DataFrame

5. 方法 calc_sector_factor_weights(sector, ic_by_sector_report)：
   - 用分板块IC报告计算各板块的因子权重
   - 不同板块可能用不同的因子组合
   - 返回 dict: {sector: {factor_id: weight}}

6. 方法 generate_top_picks(composite_score_df, sector, top_n=3)：
   - 在每个板块内，按综合得分排序
   - 取 top_n 只股票
   - 输出到 量化/data/csv/sector_top_picks.csv

7. main() 函数：
   - 计算因子相关性矩阵
   - 正交化所有有效因子
   - 计算ICIR加权组合得分
   - 分板块计算因子权重
   - 生成各板块 Top 3 选股结果
   - 打印每个板块的推荐标的

中文注释，需要 sklearn。
```

---

## 完成标志

Phase 2 完成后你应该有以下产出：

| 产出 | 验证方法 |
|------|---------|
| factor_values 表 | 有30+个因子 × 5年 × 30+只股票的数据 |
| factor_ic_report.csv | IC均值、ICIR、|IC|>0.03比例 |
| supply_chain_report.csv | 每条传导关系的Granger检验结果 |
| factor_corr_matrix.csv | 因子间相关性矩阵 |
| sector_top_picks.csv | 各板块 Top 3 选股 |
| ic_charts/ | IC时间序列可视化 |
| granger_charts/ | 传导链可视化 |

> 📌 完成后进入 Phase 3 模型构建
