# Phase 3：模型构建 — Claude Code 执行 Prompt

> 使用方法：逐条复制发给 Claude Code。Phase 2 完成后再开始。

---

## P3.01 — 板块轮动模型

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 sector_rotation.py：

实现8个领域之间的板块轮动信号模型。

依赖数据：stocks.db 的 daily_price + factor_values + macro.db

1. 类 SectorRotationModel：

2. 方法 calc_sector_return(sector, start_date, end_date)：
   - 从 stocks 表读取该板块所有 ts_code
   - 从 daily_price 计算每只股票的月度收益率
   - 板块收益率 = 板块内所有股票的平均月度收益率（等权）
   - 存入新表 sector_returns（在 stocks.db 中创建）

3. 方法 calc_relative_strength(sector, lookback=6)：
   - RS = 板块近6月累计收益率 / 沪深300近6月累计收益率
   - RS > 1 → 板块相对强势，RS < 1 → 相对弱势
   - 沪深300数据从 daily_price 读取（ts_code='000300.SH'）

4. 方法 calc_sector_momentum(sector, lookback=3)：
   - 板块近3月RS的变化率 = RS[当前] - RS[3月前]
   - RS上升 + RS > 1 → 板块加速走强 = 强买入信号
   - RS下降 + RS < 1 → 板块加速走弱 = 强卖出信号

5. 方法 calc_macro_signal(date)：
   - 从 macro.db 读取：
     PMI > 50 → 制造业扩张 → 工业设备/芯片 利好
     CPI < 2 → 通胀温和 → 成长股 利好
     北向资金净流入 → 市场情绪偏暖 → 整体利好
   - 生成宏观因子向量（每板块一个0-1得分）

6. 方法 generate_rotation_signal(date)：
   - 综合信号 = RS × 0.4 + momentum × 0.3 + macro × 0.3
   - 对8个板块排序，生成配置建议：
     Overweight（前2名）→ 仓位 +5%
     Neutral（中间3名）→ 仓位 0%
     Underweight（后3名）→ 仓位 -5%

7. 方法 backtest_rotation(start_date='20210101', end_date='20260630', rebalance_freq='M')：
   - 每月初按 rotation_signal 调仓
   - Overweight板块配30%，Neutral配20%，Underweight配10%
   - 计算累计收益率、最大回撤、夏普比率
   - 与沪深300基准对比

8. 方法 visualize_rotation()：
   - 绘制板块轮动信号热力图（时间 × 板块，颜色=信号强度）
   - 绘制回测净值曲线 vs 沪深300
   - 保存到 量化/data/csv/rotation_charts/

9. main() 函数：
   - 计算所有板块的RS和momentum
   - 生成最近一期的rotation_signal
   - 输出板块配置建议
   - 运行历史回测
   - 打印回测结果

中文注释，可独立运行。
```

---

## P3.02 — 多因子选股模型

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 multi_factor_model.py：

实现板块内多因子选股模型。

依赖数据：stocks.db 的 factor_values + stocks + Phase 2 的 factor_ic_report.csv

1. 类 MultiFactorModel：

2. 方法 load_factor_weights(sector)：
   - 从 Phase 2 产出的 sector_factor_weights 读取各板块的因子权重
   - 如果没有预计算权重，用 ICIR 方法实时计算：
     weight_i = mean(IC_i) / std(IC_i) （过去12个月滚动IC）
   - 对 IC < 0 的因子（低PE效应），权重取负

3. 方法 calc_composite_score(ts_code, date, sector)：
   - 从 factor_values 读取该股票该日期的所有因子值
   - 用板块因子权重加权
   - score = Σ(weight_i × rank_percentile_i)
   - 返回综合得分

4. 方法 generate_picks(date, sector, top_n=5)：
   - 对板块内所有股票计算 composite_score
   - 按得分排序，取 top_n
   - 输出：ts_code | name | score | top_factor | bottom_factor

5. 方法 backtest_picks(sector, start_date, end_date, rebalance_freq='M', top_n=5)：
   - 每月初选 top_n 只股票，等权持有
   - 月底卖出，下月初重新选
   - 计算累计收益率、最大回撤、夏普
   - 与板块基准对比

6. 方法 backtest_all_sectors(start_date, end_date)：
   - 对8个板块分别回测
   - 汇总结果：
     sector | annual_return | max_drawdown | sharpe | vs_benchmark
   - 保存到 量化/data/csv/mf_backtest_report.csv

7. 方法 optimize_weights(sector, method='icir')：
   - ICIR加权（默认）
   - 可选 'max_sharpe'：用 scipy.optimize 最大化回测夏普比率
   - 可选 'equal'：等权
   - 返回最优权重和对应回测指标

8. main() 函数：
   - 对每个板块生成当前 Top 5 选股
   - 运行历史回测
   - 输出回测报告
   - 打印推荐持仓清单

中文注释。
```

---

## P3.03 — 供应链因果模型（进阶版）

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 causal_model.py：

在 Phase 2 的 Granger 因果检验基础上，构建更完整的供应链因果模型。

1. 类 CausalModel：

2. 方法 build_var_model(variables_list, max_lags=4)：
   - 用 statsmodels VAR 模型
   - variables_list: 供应链相关的时间序列（月度）
   - 自动选择最优 lag（用 AIC）
   - 返回 VARResults 对象

3. 方法 impulse_response(var_results, impulse_var, response_var, periods=10)：
   - 计算脉冲响应函数
   - 例如：NVIDIA营收 +1%冲击 → 海光信息股价如何响应
   - 绘制脉冲响应图
   - 保存到 量化/data/csv/irf_charts/

4. 方法 forecast_upstream_impact(upstream_shock, downstream_code)：
   - 给定上游冲击（如"NVIDIA营收超预期+20%"）
   - 用 VAR 模型预测下游股票1-6个月的响应
   - 返回：{lag_month: expected_response_pct}

5. 方法 build_causal_graph()：
   - 从 supply_chain_links 表读取已验证的传导关系
   - 构建有向因果图（networkx DiGraph）
   - 计算每个节点的因果影响深度（从源头到终端的最长路径）
   - 绘制因果图，保存到 量化/data/csv/causal_graph.png

6. 方法 detect_causal_decay(upstream_code, downstream_code)：
   - 用滚动 Granger 检验检测传导关系是否在衰减：
     - 将数据分成 2020-2022 和 2023-2026 两段
     - 分别做 Granger 检验
     - 如果近段不再显著 → 传导关系已衰减
   - 返回衰减报告

7. 方法 generate_causal_trade_signal(date)：
   - 读取最近的上游数据（NVIDIA财报、云CapEx等）
   - 用因果模型预测下游股票的预期收益
   - 如果预期收益 > 5% → 买入信号
   - 如果预期收益 < -5% → 卖出信号
   - 输出到 量化/data/csv/causal_signals.csv

8. main() 函数：
   - 构建完整供应链 VAR 模型
   - 计算脉冲响应
   - 构建因果图
   - 检测传导衰减
   - 生成当前交易信号

中文注释，需要 statsmodels 和 networkx。
```

---

## P3.04 — 情绪与动量模型（NLP增强版）

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 sentiment_momentum.py：

实现基于 LLM 的情绪因子 + 技术动量的混合模型。

这是你作为AI工程师的独特优势——用NLP做语义分析。

1. 类 SentimentMomentumModel：

2. 方法 fetch_news(ts_code, date_range=7)：
   - 用 AKShare 或 东方财富 爬取近7天新闻标题
   - ak.stock_news_em(symbol=ts_code) 或类似API
   - 如果API不可用，手动从东方财富网页爬取
   - 存入 量化/data/raw/news/ 目录

3. 方法 calc_llm_sentiment(news_titles, model_endpoint='local_qwen')：
   - 把新闻标题列表发给本地 Qwen3.6-27B（你公司部署的）
   - Prompt：
     """
     你是A股市场分析师。请对以下新闻标题逐条评分：
     评分标准：-2(严重利空) / -1(偏利空) / 0(中性) / +1(偏利好) / +2(重大利好)
     关注以下维度：政策影响、业绩预期、行业趋势、风险事件
     
     新闻标题列表：
     {news_titles}
     
     请返回JSON格式：[{"title": "...", "score": X, "reason": "..."}]
     """
   - 计算 sentiment_score = mean(scores)
   - 如果无法访问本地LLM，提供一个备用方案：
     用简单的关键词匹配（预定义利好词和利空词列表）

4. 方法 calc_social_sentiment(ts_code)：
   - 如果能获取社交媒体数据（雪球/东方财富评论）
   - 否则用 news sentiment 作为替代
   - 标记 [MISSING] 需要补充数据源

5. 方法 calc_momentum_signal(ts_code, date)：
   - 从 factor_values 读取动量因子（T_MOM_20, T_MOM_60）
   - 短期动量 > 中期动量 → 加速上涨信号
   - 短期动量 < 0 且中期动量 > 0 → 回调信号（可能买入机会）

6. 方法 calc_hybrid_score(ts_code, date)：
   - hybrid_score = sentiment × 0.4 + momentum × 0.3 + volume_change × 0.3
   - 如果 sentiment 数据缺失，用 momentum × 0.7 + volume × 0.3 替代
   - 返回综合情绪动量得分

7. 方法 generate_short_term_signal(date)：
   - 对所有关注的股票计算 hybrid_score
   - hybrid_score > 1.5 → 短期强势，关注买入
   - hybrid_score < -1.5 → 短期弱势，关注卖出
   - 输出到 量化/data/csv/sentiment_signals.csv

8. 方法 backtest_sentiment(start_date, end_date)：
   - 回测情绪动量策略：每周调仓，只买 hybrid_score > 1 的股票
   - 计算收益、回撤、与纯动量策略对比

9. main() 函数：
   - 拉取近7天新闻
   - 计算LLM情绪得分（如果LLM可访问）
   - 计算动量信号
   - 生成混合信号
   - 输出当前建议

注意：LLM调用是可选功能，如果无法访问Qwen，代码应优雅降级到关键词匹配方案。
中文注释。
```

---

## 完成标志

Phase 3 完成后你应该有以下产出：

| 产出 | 验证方法 |
|------|---------|
| 板块轮动信号 | 8板块每月有配置建议（OW/N/UW） |
| 多因子选股结果 | 每板块 Top 5 选股清单 |
| 供应链因果图 | 传导关系可视化 + 脉冲响应 |
| 情绪动量信号 | 当前股票的情绪-动量综合得分 |
| 回测结果 | 各策略的历史表现报告 |
| causal_graph.png | 供应链因果关系可视化 |

> 📌 完成后进入 Phase 4 回测与验证
