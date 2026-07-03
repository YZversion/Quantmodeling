# Phase 4：回测与验证 — Claude Code 执行 Prompt

> 使用方法：逐条复制发给 Claude Code。Phase 3 完成后再开始。

---

## P4.01 — 回测引擎框架

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 backtest_engine.py：

实现一个完整的回测引擎，处理所有回测中的陷阱问题。

依赖数据：stocks.db + Phase 3 产出的策略信号

1. 类 BacktestEngine：

2. 初始化参数：
   - initial_capital = 100000（10万元起始资金）
   - commission_rate = 0.0003（券商佣金万三）
   - slippage = 0.001（滑点千一）
   - stamp_tax = 0.0005（印花税千五，卖出时收取）
   - min_commission = 5（最低佣金5元）

3. 方法 run_backtest(strategy_func, start_date, end_date, rebalance_freq='M')：
   strategy_func 是一个函数，输入(date, portfolio, data)，输出{ts_code: target_weight}
   
   回测流程：
   for each rebalance_date:
     1. 调用 strategy_func 生成目标仓位
     2. 计算需要买入/卖出的数量
     3. 应用滑点：买入价 = close × (1 + slippage)，卖出价 = close × (1 - slippage)
     4. 计算手续费：买入和卖出分别扣除
     5. 更新持仓和现金
     6. 记录每日净值
   
   返回 BacktestResult 对象

4. 方法 handle_survivorship_bias()：
   - 从 stocks 表读取所有曾经上市的股票（包括已退市的）
   - 在回测中包含退市股票（如果有的话）
   - [MISSING: A股退市公司数据需要补充]

5. 方法 handle_lookahead_bias(factor_values)：
   - 确保因子值使用的是发布时可用数据，而非未来数据
   - 财报因子：使用上一期财报数据（而非本期，本期可能还没发布）
   - 例：2025Q1财报通常4月底才发布，3月份只能用2024Q4的数据
   - 实现方式：factor_date = max(available_report_date before trade_date)

6. 方法 handle_min_trade_unit()：
   - A股最小交易单位 = 100股（1手）
   - 计算目标股数时向下取整到100的倍数
   - 记录因零股限制导致的仓位偏差

7. 类 BacktestResult：
   属性：
   - daily_nav: 每日净值序列
   - trade_log: 所有交易记录
   - positions_log: 每日持仓记录
   
   方法：
   - calc_annual_return()
   - calc_max_drawdown()
   - calc_sharpe_ratio(risk_free_rate=0.02)
   - calc_calmar_ratio()
   - calc_win_rate()（盈利交易占比）
   - calc_profit_loss_ratio()（平均盈利/平均亏损）
   - compare_benchmark(benchmark_code='000300.SH')（与沪深300对比）
   - print_summary()（打印回测摘要）
   - save_report(path)（保存详细报告到CSV）

8. main() 函数：
   - 用 Phase 3 的板块轮动策略做回测
   - 用 Phase 3 的多因子选股策略做回测
   - 用 Phase 3 的供应链因果策略做回测
   - 对比三个策略的表现
   - 输出综合报告到 量化/data/csv/backtest_summary.csv

中文注释，详细记录每笔交易。
```

---

## P4.02 — 样本外验证

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 out_of_sample_test.py：

实现样本外验证和参数稳定性检验。

1. 类 OutOfSampleTest：

2. 方法 split_data(train_ratio=0.6)：
   - 将数据分为：
     训练集：2020-01-01 到 2023-06-30（约60%）
     验证集：2023-07-01 到 2024-12-31（约25%）
     测试集：2025-01-01 到 2026-06-30（约15%）
   - 返回三个日期区间

3. 方法 train_on_period(strategy, train_start, train_end)：
   - 在训练集上训练策略参数（因子权重、阈值等）
   - 返回优化后的策略参数

4. 方法 test_on_period(strategy, params, test_start, test_end)：
   - 用训练好的参数在测试集上运行回测
   - 不做任何参数调整
   - 返回 BacktestResult

5. 方法 compare_train_test(train_result, test_result)：
   - 计算训练集和测试集的关键指标差异：
     | 指标 | 训练集 | 测试集 | 差异 | 过拟合风险 |
     | 年化收益 | 15% | 8% | -7% | 高 |
   - 过拟合判定标准：
     测试集夏普 < 训练集夏普 × 0.5 → 可能过拟合
     测试集最大回撤 > 训练集 × 1.5 → 可能过拟合

6. 方法 walk_forward_test(strategy, n_splits=5, window=12)：
   - 滚动前推测试：
     第1轮：训练2020-2021 → 测试2022
     第2轮：训练2020-2022 → 测试2023
     第3轮：训练2020-2023 → 测试2024
     第4轮：训练2020-2024 → 测试2025
     第5轮：训练2020-2025 → 测试2026
   - 每轮独立训练参数，在测试集上评估
   - 计算5轮的平均测试集表现 → 这是策略的真实alpha估计

7. 方法 sensitivity_analysis(strategy, param_name, param_range)：
   - 对关键参数做敏感性分析
   - 例：板块轮动中的RS lookback，从3到12逐一测试
   - 绘制参数敏感性图
   - 如果参数微小变化导致回测结果剧烈变化 → 过拟合风险高

8. main() 函数：
   - 对三个策略分别做：
     1. 训练集/测试集分割验证
     2. 滚动前推测试
     3. 关键参数敏感性分析
   - 输出过拟合风险评估报告到 量化/data/csv/oos_report.csv
   - 格式：
     strategy | train_sharpe | test_sharpe | wf_avg_sharpe | overfitting_risk | verdict

9. 判定标准总结：
   | verdict | 条件 | 含义 |
   | 通过 | test_sharpe > train × 0.6，wf稳定 | 真实alpha |
   | 疑似过拟合 | test_sharpe < train × 0.5 | 需要简化 |
   | 过拟合 | test_sharpe < 0.3 | 丢弃策略 |

中文注释。
```

---

## P4.03 — 因子衰减监控

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 factor_decay_monitor.py：

实现因子和传导关系的衰减检测与监控。

1. 类 FactorDecayMonitor：

2. 方法 calc_rolling_ic(factor_id, window=12)：
   - 计算因子IC的12个月滚动均值
   - 如果滚动IC从正变为负 → 因子已反转（从有效变为反向）
   - 如果滚动IC从显著变为不显著 → 因子已衰减

3. 方法 detect_ic_decay(factor_id)：
   - 对比：
     近12个月IC均值 vs 全历史IC均值
     近6个月IC均值 vs 近12个月IC均值
   - 如果近6月 < 近12月 × 0.7 → 因子正在衰减
   - 输出衰减报告

4. 方法 detect_causal_decay(link_id)：
   - 从 supply_chain_links 读取传导关系
   - 用 Phase 2 的 Granger 检验方法重新检验
   - 对比全样本和近2年的检验结果
   - 如果近2年不再显著 → 传导关系已衰减

5. 方法 calc_model_performance_decay(strategy_name)：
   - 对比策略在近6个月 vs 前6个月的回测表现
   - 如果近6月表现显著下降 → 策略可能失效

6. 方法 generate_decay_report()：
   - 对所有因子做衰减检测
   - 对所有传导关系做衰减检测
   - 对所有策略做表现衰减检测
   - 输出到 量化/data/csv/decay_report.csv
   - 格式：
     type | id | full_ic/was_significant | recent_ic/is_significant | decay_pct | status
     factor | F_PE_TTM | 0.06 | 0.02 | 67% | 衰减
     causal | NVDA→688041 | significant | not_significant | - | 已失效
     strategy | rotation | sharpe_1.2 | sharpe_0.4 | 67% | 衰减

7. 方法 recommend_action(decay_report)：
   - 根据衰减报告推荐行动：
     因子衰减 → 从组合中剔除或降低权重
     传导失效 → 不再使用该传导信号
     策略失效 → 暂停实盘，重新研究
     全部正常 → 继续执行

8. 方法 setup_monitoring_schedule()：
   - 创建一个监控配置文件 量化/config/monitor_schedule.json：
     {
       "ic_check_freq": "monthly",
       "causal_check_freq": "quarterly",
       "model_check_freq": "monthly",
       "alert_threshold": {
         "ic_decay_pct": 0.7,
         "causal_significance_level": 0.05,
         "model_sharpe_drop_pct": 0.5
       }
     }

9. main() 函数：
   - 运行全量衰减检测
   - 生成衰减报告
   - 推荐行动
   - 创建监控配置
   - 打印摘要

中文注释。
```

---

## 完成标志

Phase 4 完成后你应该有以下产出：

| 产出 | 验证方法 |
|------|---------|
| backtest_summary.csv | 三策略回测结果 + 基准对比 |
| oos_report.csv | 样本外验证 + 过拟合风险评估 |
| decay_report.csv | 因子/传导/策略衰减检测 |
| monitor_schedule.json | 定期监控配置 |
| 参数敏感性图 | 关键参数变化对结果的影响 |

> 📌 完成后进入 Phase 5 实盘与迭代
