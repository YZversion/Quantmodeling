# Phase 5：实盘与迭代 — Claude Code 执行 Prompt

> 使用方法：逐条复制发给 Claude Code。Phase 4 回测验证通过后再开始。
> ⚠️ **Phase 5 意味着真金白银。只有 Phase 4 的过拟合风险评估为"通过"的策略才可以实盘。**

---

## P5.01 — 策略执行自动化

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 strategy_executor.py：

实现策略信号 → 交易执行的自动化流程。

1. 类 StrategyExecutor：

2. 方法 load_latest_signals()：
   - 从 量化/data/csv/ 下读取最新的信号文件：
     rotation_signals.csv（板块轮动信号）
     mf_picks.csv（多因子选股）
     causal_signals.csv（供应链因果信号）
     sentiment_signals.csv（情绪动量信号）
   - 合并为统一的交易信号表

3. 方法 generate_trade_plan(date, total_capital)：
   - 根据信号生成交易计划：
     板块配置：从 rotation_signals 读取 OW/N/UW
     个股选择：从 mf_picks + causal_signals 读取
     仓位大小：
       单股 ≤ 15% 总资金
       单板块 ≤ 30% 总资金
       总仓位 ≤ 60%（留40%现金）
   - 输出到 量化/data/csv/trade_plan.csv
   - 格式：
     ts_code | name | sector | signal_source | target_weight | current_weight | action | shares | estimated_cost

4. 方法 calc_actual_positions(current_portfolio, trade_plan)：
   - 对比目标仓位和当前仓位
   - 生成具体买卖指令：
     需要买入的：计算股数（向下取整到100股）
     需要卖出的：计算股数
   - 计算预估交易成本（佣金+印花税+滑点）

5. 方法 check_risk_limits(trade_plan)：
   - 检查以下硬性限制：
     单股仓位 ≤ 15%？ → 如果超出，削减该股仓位
     单板块仓位 ≤ 30%？ → 如果超出，削减板块内仓位
     总仓位 ≤ 60%？ → 如果超出，削减所有仓位
     单日换仓 ≤ 50%？ → 如果超出，分2天执行
   - 如果任何限制被触发，调整 trade_plan 并打印警告

6. 方法 generate_execution_checklist(trade_plan)：
   - 生成执行前的检查清单（打印到终端）：
     ✅ 今日信号已加载
     ✅ 板块配置: OW=[芯片,GPU] / N=[PCB,光模块] / UW=[数据库,工业]
     ✅ 单股最大仓位: 12%（海光信息）→ 合规
     ✅ 总仓位: 55% → 合规
     ✅ 止损线已设定: 所有持仓8%止损
     ✅ 预估交易成本: 325元
     ⚠️ 注意: 供应链因果信号今天新增海光信息买入建议
     
   - 保存到 量化/data/csv/execution_log/YYYY-MM-DD_checklist.md

7. main() 函数：
   - 加载最新信号
   - 生成交易计划
   - 检查风险限制
   - 生成执行清单
   - 打印清单供人工确认

注意：此脚本只生成交易建议，不自动下单。
实盘下单需要人工确认后手动操作（或后续接入券商API）。
中文注释。
```

---

## P5.02 — 风控系统

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 risk_controller.py：

实现实盘风控的硬性规则和监控。

1. 类 RiskController：

2. 方法 set_stop_loss(ts_code, entry_price, stop_pct=0.08)：
   - 止损价 = entry_price × (1 - stop_pct)
   - 存入 risk_params 表（在 stocks.db 中创建）：
     ts_code | entry_price | stop_price | stop_pct | set_date | is_active

3. 方法 check_stop_loss(date)：
   - 读取所有活跃持仓的止损设置
   - 对比当日收盘价与止损价
   - 如果 close < stop_price → 触发止损
   - 输出止损清单：
     ts_code | name | entry_price | stop_price | current_price | loss_pct | action
   - 保存到 量化/data/csv/execution_log/YYYY-MM-DD_stops.md

4. 方法 set_position_limits()：
   - 创建风控参数配置 量化/config/risk_params.json：
     {
       "max_single_stock_pct": 0.15,
       "max_sector_pct": 0.30,
       "max_total_position_pct": 0.60,
       "stop_loss_pct": 0.08,
       "trailing_stop_pct": 0.12,
       "max_daily_turnover_pct": 0.50,
       "model_signal_error_threshold": 3,
       "model_signal_error_action": "reduce_position_50pct"
     }

5. 方法 check_model_signal_accuracy(last_n_signals=3)：
   - 检查最近N个信号是否连续错误
   - 连续3次信号错误 → 降低仓位50%
   - 连续5次信号错误 → 暂停模型，转手动管理
   - 输出到 量化/data/csv/model_accuracy_log.csv

6. 方法 calc_portfolio_risk_metrics(date)：
   - 计算当前组合的风险指标：
     组合波动率（基于持仓权重和个股波动率）
     组合最大可能损失（VaR，95%置信度）
     单板块集中度风险
   - 输出到 量化/data/csv/risk_metrics/YYYY-MM-DD.csv

7. 方法 generate_daily_risk_report(date)：
   - 每日风控报告：
     1. 持仓清单 + 盈亏状态
     2. 止损检查结果
     3. 仓位合规检查
     4. 模型信号准确率
     5. 组合风险指标
     6. 行动建议（如有触发风控规则）
   - 保存到 量化/data/csv/daily_risk_report/YYYY-MM-DD.md

8. main() 函数：
   - 加载风控参数
   - 检查止损
   - 检查仓位限制
   - 检查模型信号准确率
   - 生成每日风控报告
   - 打印摘要

中文注释。这是实盘最重要的脚本。
```

---

## P5.03 — 月度复盘 + 策略迭代

```
在 D:/NOTETAKING/STOCKS/量化/scripts/ 下创建 monthly_review.py：

实现月度复盘和策略迭代的标准化流程。

1. 类 MonthlyReviewer：

2. 方法 calc_monthly_performance(month)：
   - 计算本月组合表现：
     月收益率、最大回撤、夏普
     每只持仓的盈亏
     vs 沪深300基准
   - 存入 monthly_performance 表（stocks.db）

3. 方法 analyze_trade_log(month)：
   - 从 execution_log 读取本月所有交易
   - 统计：
     交易次数、胜率、平均盈利、平均亏损
     盈亏比 = 平均盈利 / 平均亏损
     最大单笔盈利和亏损
     按板块分布
   - 存入 monthly_trade_stats 表

4. 方法 review_signals_accuracy(month)：
   - 回顾本月所有信号（rotation/mf/causal/sentiment）
   - 对比信号方向与实际股价走势
   - 计算各信号源的准确率
   - 输出：
     signal_source | total_signals | correct | accuracy | avg_return_if_follow

5. 方法 review_factor_ic_trend()：
   - 运行 factor_decay_monitor 的 IC 衰减检测
   - 标记本月衰减的因子

6. 方法 review_causal_links()：
   - 检查供应链传导关系是否在本月有效
   - 例：如果NVIDIA本月超预期 → 国内GPU公司是否如期上涨？

7. 方法 generate_review_report(month)：
   - 整合所有分析为月度复盘报告
   - 格式（markdown）：
     # 2026年X月复盘报告
     
     ## 一、组合表现
     - 月收益率: X%
     - vs 沪深300: Y%
     - 超额收益: Z%
     
     ## 二、交易统计
     - 交易次数: N
     - 胜率: W%
     - 盈亏比: R
     
     ## 三、信号准确率
     - 板块轮动: A%
     - 多因子: B%
     - 供应链因果: C%
     - 情绪动量: D%
     
     ## 四、因子衰减
     - 衰减因子: [列表]
     - 新发现: [列表]
     
     ## 五、供应链传导验证
     - 有效传导: [列表]
     - 失效传导: [列表]
     
     ## 六、行动建议
     - [具体建议]
     
   - 保存到 量化/data/csv/monthly_reviews/2026-XX.md

8. 方法 suggest_strategy_updates(review_report)：
   - 基于复盘结果，建议策略调整：
     因子衰减 → 建议剔除/降权
     传导失效 → 建议不再使用该信号
     信号准确率下降 → 建议调整权重或暂停
     新发现 → 建议纳入新因子
   - 输出到 量化/data/csv/strategy_updates.csv

9. 方法 update_model_weights(strategy_name, new_weights)：
   - 用新权重更新策略参数
   - 保存到 量化/config/ 目录下的策略配置文件
   - 记录更新历史

10. main() 函数：
   - 计算上月表现
   - 分析交易日志
   - 复盘信号准确率
   - 检查因子衰减
   - 生成复盘报告
   - 推荐策略调整
   - 打印摘要

中文注释。月度复盘是持续改进的核心。
```

---

## P5.04 — Docker 部署（自动化 pipeline）

```
在 D:/NOTETAKING/STOCKS/量化/ 下创建 Docker 自动化部署：

1. 创建 Dockerfile：
   FROM python:3.11-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY scripts/ ./scripts/
   COPY config/ ./config/
   
   # 定时任务入口
   CMD ["python", "scripts/daily_update.py"]

2. 创建 requirements.txt：
   tushare>=1.2
   akshare>=1.10
   yfinance>=0.2
   pandas>=2.0
   numpy>=1.24
   scipy>=1.10
   scikit-learn>=1.3
   statsmodels>=0.14
   matplotlib>=3.7
   networkx>=3.1
   sqlalchemy>=2.0

3. 创建 docker-compose.yml：
   services:
     quant-daily:
       build: .
       volumes:
         - ./data:/app/data
         - ./config:/app/config
         - ./scripts:/app/scripts
       environment:
         - TUSHARE_TOKEN=${TUSHARE_TOKEN}
       # 每日18:00运行数据更新
       # (用 cron 或 Windows Task Scheduler 外部调度)

4. 创建 scripts/run_daily_pipeline.py：
   每日执行的完整 pipeline：
   1. daily_update.py → 更新数据
   2. factor_engine.py → 计算因子
   3. tech_factors.py → 计算技术因子
   4. sector_rotation.py → 生成轮动信号
   5. multi_factor_model.py → 生成选股信号
   6. causal_model.py → 生成传导信号
   7. sentiment_momentum.py → 生成情绪信号
   8. strategy_executor.py → 生成交易计划
   9. risk_controller.py → 风控检查
   10. 打印今日完整报告

   每步执行后检查是否有错误，如果有则打印警告并继续下一步。

5. 创建 scripts/run_monthly_review.py：
   每月1日执行的复盘 pipeline：
   1. monthly_review.py → 生成复盘报告
   2. factor_decay_monitor.py → 因子衰减检测
   3. 根据复盘结果决定是否更新策略权重

6. 创建 .env.example：
   TUSHARE_TOKEN=your_token_here
   QWEN_ENDPOINT=http://localhost:8080/v1/chat/completions

7. 创建 README.md（简短的运行指南，不是详细文档）：
   - 如何设置环境变量
   - 如何运行每日 pipeline
   - 如何运行月度复盘

请确保所有脚本可以独立运行也可以组合运行。
中文注释。
```

---

## 完成标志

Phase 5 完成后你应该有以下产出：

| 产出 | 验证方法 |
|------|---------|
| strategy_executor.py | 能生成每日交易计划 + 执行清单 |
| risk_controller.py | 能检查止损 + 仓位限制 + 模型准确率 |
| monthly_review.py | 能生成月度复盘报告 |
| Dockerfile + docker-compose.yml | 能用 docker-compose up 运行每日pipeline |
| run_daily_pipeline.py | 一键运行全流程 |
| risk_params.json | 风控参数配置 |
| config/ 目录 | 策略配置文件 |

---

## 最终产出总览

从 Phase 1 到 Phase 5，你将拥有一个完整的量化研究系统：

```
量化/
├── 00_量化研究规划.md
├── config/
│   ├── config.ini (Tushare token)
│   ├── risk_params.json (风控参数)
│   ├── monitor_schedule.json (监控配置)
│   ├── strategy_weights/ (各策略权重配置)
│   └── .env.example
├── data/
│   ├── db/
│   │   ├── stocks.db (核心数据库)
│   │   └── macro.db (宏观数据)
│   ├── csv/ (所有输出报告)
│   ├── raw/ (原始数据缓存)
│   └── chroma/ (NLP向量)
├── scripts/ (所有Python脚本)
│   ├── init_db.py
│   ├── seed_stocks.py
│   ├── fetch_tushare.py
│   ├── fetch_supplementary.py
│   ├── daily_update.py
│   ├── check_data_quality.py
│   ├── factor_engine.py
│   ├── tech_factors.py
│   ├── factor_ic_test.py
│   ├── supply_chain_granger.py
│   ├── factor_orthogonal.py
│   ├── sector_rotation.py
│   ├── multi_factor_model.py
│   ├── causal_model.py
│   ├── sentiment_momentum.py
│   ├── backtest_engine.py
│   ├── out_of_sample_test.py
│   ├── factor_decay_monitor.py
│   ├── strategy_executor.py
│   ├── risk_controller.py
│   ├── monthly_review.py
│   ├── run_daily_pipeline.py
│   └── run_monthly_review.py
├── prompts/ (Claude Code执行prompt)
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

> 🎯 **整个系统从数据拉取到实盘执行全部自动化，月度复盘驱动持续迭代。**

*本材料仅供学习研究参考，不构成投资建议。实盘前务必完成 Phase 4 的样本外验证，确认策略无过拟合风险。*
