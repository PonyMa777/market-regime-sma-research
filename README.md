# Bull and Bear Regimes in Individual Stocks｜个股牛熊划分研究（震荡待研究）

当前阶段：确定研究目标和划分规则，尚未开始数据分析或回测。

## Quant Strategy Workflow｜量化策略研究流程

先选研究对象，再准备数据、判断市场阶段，最后检验交易规则是否有效。下面是后续路线图，不代表已经开始下载或回测。

~~~mermaid
flowchart TD
    A["1. Stock Selection｜挑选个股<br/>12家主样本 + 4家扩展组<br/>标普500作为参考"]
    B["2. Minute Data｜下载分钟级数据<br/>核对时区、交易时段、缺失和复权"]
    C["3. Market Regimes｜区分牛熊与震荡<br/>分钟数据汇总为日线<br/>用日线中间价划分牛熊"]
    D["4. Strategy Design｜设计各阶段的策略<br/>分别提出牛市、熊市、震荡期假设<br/>不预先认定哪种策略一定有效"]
    E["Optional ML Model｜可选机器学习模型<br/>Multiple Linear Regression｜多元线性回归<br/>Gradient Descent｜梯度下降：一种训练方法"]
    F["Trading Rules｜形成完整交易规则<br/>预测值如何变成买入、卖出或空仓<br/>明确仓位、退出条件和风险限制"]
    G["5. Validation｜回测与样本外验证<br/>计入手续费、滑点及做空成本<br/>与买入持有比较，检查是否使用未来信息"]
    H["6. Paper Trading｜模拟交易<br/>通过验证后才讨论实盘"]
    A --> B --> C --> D
    D --> E --> F
    D -->|"规则型策略，如双均线"| F
    F --> G --> H
~~~

**这里先讲清两件事：**

- **牛熊与震荡不是同一把尺子。** 20%规则只给出牛/熊标签；没有触发20%切换时沿用原状态，不能直接当成震荡。震荡的定义尚未确定，先作为额外维度研究，因此可以出现“牛市中的震荡”或“熊市中的震荡”。
- **模型不是完整策略。** 多元线性回归用多个已知特征预测一个数值，例如未来一段收益；梯度下降是拟合模型参数的一种方法，不是另一种买卖策略。还要规定预测超过什么门槛才交易、买多少、何时退出。模型、标准化和门槛只在训练/验证阶段确定，留出数据用来检查效果。

分钟数据提供细节，牛熊划分仍按日线；状态和信号只能在计算所需的数据全部可知后用于后续交易。若由分钟线汇总日线，必须固定交易时段并处理提前收盘日，否则日线高低价可能不完整。

## Research Objective｜我们想弄清楚什么

我们想研究的是：怎样划分一只股票的牛市和熊市。

未来的交易策略会实施在个股上，所以我们需要先弄清楚个股自身处在什么阶段，不能只凭大盘涨跌来判断。标普500反映的是许多公司的综合表现，不同行业、不同股票的涨跌可能互相抵消。我们担心，大盘信号中有些变化对具体个股并没有解释力，也可能掩盖个股自己的趋势；这是本研究要核对的问题，不能预先认定大盘一定比个股噪音更多。

我们会以标普500为参考，同时观察科技、半导体领域的个股，用同一套规则分别划分它们的牛熊阶段，再把时间线放在一起比较：什么时候个股跟着大盘走？什么时候大盘还在涨，个股已经转弱，或大盘还在跌，个股已经回升？这些差异是短暂波动，还是持续了一段时间的独立走势？

第一步先用20%规则建立一个明确的起点。价格采用日线中间价 M = (High + Low) / 2，即当天最高价与最低价的平均值。从本轮牛市以来最高的 M 下跌至少20%，确认转熊；从本轮熊市以来最低的 M 反弹至少20%，确认转牛。大盘和每只股票都根据自己的价格独立计算，大盘标签不直接套给个股。

我们先把这些划分的依据、转换日期和局限弄清楚，再考虑怎样用于个股交易。双均线是以后可以用来验证的一种策略，不是当前研究的主要目的。

## Research Questions｜具体看哪些问题

1. 按同一套20%规则，大盘与个股的牛熊转换日期有多大差异？
2. 个股与大盘状态一致、背离，各持续多久？个股通常领先还是滞后？
3. 科技和半导体领域的个股有没有相似的表现？同板块个股之间又有哪些差别？
4. 固定20%门槛对不同波动程度的股票是否合适？会不会在某些股票上频繁切换，在另一些股票上确认过晚？
5. 哪些信息能为以后在个股上实施策略提供帮助？需要什么后续实验才能检验这一点？

## Scope and Comparison｜研究对象与对照

主样本确定为12家公司：七姐妹，加上MU、AMD、INTC、TXN、ASML。NVDA虽同时属于七姐妹和半导体，但只算一家。我们按业务分组作对照，不把七姐妹统称为传统互联网企业。

### Main Sample｜12家主样本

| 股票 | 公司与业务 | Listing Date｜上市时间 |
|---|---|---|
| AAPL | Apple 苹果；消费电子与服务 | [1980-12-12](https://investor.apple.com/faq./default.aspx) |
| MSFT | Microsoft 微软；软件与云服务 | [1986-03-13](https://learn.microsoft.com/en-us/shows/history/history-of-microsoft-1986) |
| GOOGL | Alphabet 谷歌；互联网平台、广告与云 | [2004-08-19（Google A类股首次交易；当时代码为GOOG）](https://abc.xyz/assets/investor/static/pdf/2007_google_annual_report.pdf) |
| AMZN | Amazon 亚马逊；电商与云服务 | [1997-05-15](https://ir.aboutamazon.com/faqs/default.aspx) |
| META | Meta；社交平台与广告 | [2012-05-18（当时为Facebook／FB）](https://investor.atmeta.com/investor-news/press-release-details/2012/Facebook-Announces-Pricing-of-Initial-Public-Offering/default.aspx) |
| TSLA | Tesla 特斯拉；电动车与能源 | [2010-06-29](https://ir.nasdaq.com/news-releases/news-release-details/nasdaq-omx-welcomes-tesla-motors-nasdaq-stock-market) |
| NVDA | NVIDIA 英伟达；GPU与AI芯片 | [1999-01-22](https://investor.nvidia.com/investor-resources/faqs/) |
| MU | Micron 美光；存储芯片 | [1984-06（月）](https://archive.computerhistory.org/resources/access/text/2013/04/102723191-05-01-acc.pdf) |
| AMD | AMD；CPU与GPU | [1972-09-27（首次公开发行，场外交易）](https://ir.amd.com/contacts-faq/faq) |
| INTC | Intel 英特尔；处理器与芯片制造 | [1971-10-13](https://www.intc.com/stock-info/faq) |
| TXN | Texas Instruments 德州仪器；模拟与嵌入式芯片 | [1953-10-01（首次公开交易，NYSE）](https://investor.ti.com/resources/frequently-asked-questions) |
| ASML | ASML 阿斯麦；光刻设备 | [1995-03（月）](https://www.chiphistory.org/exhibits/ex_se_asml_20yr_history/chapter4.pdf) |

日期按首次公开发行或交易口径，括号注明特殊情况；MU、ASML目前核实到月份。上市时间不等于数据源可提供的分钟行情起点。

### Extended Sample｜4家扩展组

| 股票 | 公司与业务 | 为什么单列 |
|---|---|---|
| PLTR | Palantir；数据与AI软件 | 保留关注的AI软件案例，但不归为半导体。 |
| CRWV | CoreWeave；AI云基础设施 | 2025年IPO，股票历史短，单独观察，不能据此总结多轮长期牛熊。 |
| SNDK | Sandisk 闪迪；闪存 | 当前独立上市股票始于2025年分拆；不把旧Sandisk历史直接拼接。 |
| WDC | Western Digital 西部数据；存储硬件 | 2025年分拆闪存业务后聚焦硬盘，跨分拆比较须注明业务变化。 |

标普500是大盘参考；如用SPY，注明它是ETF代理。每家公司根据自己的日线中间价独立划分牛熊，再与大盘比较。需要时另加板块参考，具体标的尚未确定。

长期分析和短历史案例分开，不为迁就CRWV、SNDK而把所有样本压缩到2025年以后。这是一组预先选定的公司案例，不能直接代表整个行业；七姐妹本身也带有事后选择成功企业的偏差。

业务及历史核对：[七姐妹名单](https://www.roundhillinvestments.com/etf/mags/)、[TI](https://www.ti.com/about-ti.html)、[ASML](https://www.asml.com/en/company)、[Palantir](https://www.palantir.com/platforms/)、[CoreWeave IPO](https://www.coreweave.com/news/coreweave-announces-pricing-of-initial-public-offering)、[Sandisk分拆](https://www.sandisk.com/sandisk-separation-faqs/)、[WDC年报](https://www.sec.gov/Archives/edgar/data/106040/000010604025000038/wdc-20250627.htm)。

## Regime Definition｜牛熊怎么划分

### Price｜用什么价格

M[t] = (High[t] + Low[t]) / 2，简称HL2。例如当天最高110、最低90，中间价就是100。它不是收盘价，也不是成交均价或买卖报价中点。

High与Low必须使用一致的复权口径；数据源和拆股、分红处理在分析前确定。当天完整的最高最低价只有收盘后才知道，因此当天确认的状态最早供下一交易日使用，中间价不能当作实际成交价。

### Reference Points｜20%从哪里开始算

| 当前状态 | 参考点 | 切换条件 | 新状态的起点 |
|---|---|---|---|
| 牛市 | 本轮牛市以来，截至当天最高的日线中间价H | M ≤ 0.8 × H | 当天确认熊市，低点L从当天M开始记录 |
| 熊市 | 本轮熊市以来，截至当天最低的日线中间价L | M ≥ 1.2 × L | 当天确认牛市，高点H从当天M开始记录 |

参考点随着本阶段的新高或新低更新，未触发时保持原状态。不是与昨天比较，也不是永远与样本第一天比较。

例子：牛市阶段最高M为100，跌到80确认熊市；之后跌到70，再回升到84，较70上涨20%，确认牛市。虽然仍未回到100，规则已经转牛。新牛市的高点从84开始，不沿用上一轮的100。

### Initialization and Timing｜开头怎么处理，什么时候生效

1. **开头不猜牛熊，先看已有历史。** 用评估开始前的数据，从第一条有效中间价记录高低点；首次从高点跌至少20%就确认熊市，从低点涨至少20%就确认牛市。尚未触发或两个条件同时满足时，继续标“未知”，直到只有一个条件满足；之后按已知状态规则运行。
2. **今天收盘确认，明天才能使用。** 当天最高价、最低价收盘后才完整，所以当天确认的牛熊状态最早用于下一交易日，不能事后把标签提前贴到之前的顶部或底部。

### Source｜规则出处

20%例子来自Hamlet Medina、Ernest P. Chan的《Generative AI for Trading for Asset Management》，3.1.4节Hidden Markov Model，本地PDF第167页。

书中举出从高水位下跌20%、从熊市低水位反弹20%的定义。本项目另行约定了HL2、初始化、状态切换后的参考点重置与标签生效时间；这些细节不冒充书中原文。

## Planned Outputs｜希望得到什么

1. **把每只股票的牛熊阶段画清楚。** 一张时间线标出什么时候转牛、什么时候转熊，并展示触发当天的中间价和参考高低点，让每次划分都有据可查。
2. **看清个股什么时候跟大盘走，什么时候走自己的行情。** 把个股与大盘放在同一时间线上，挑出有代表性的同步和背离案例，说明谁先转向、差了多久，以及20%规则有没有确认太晚或频繁切换，为以后在个股上实施策略提供依据。

## Later Strategy Validation｜以后怎样验证策略

等个股牛熊定义和数据确认后，再讨论双均线等策略在不同阶段的表现。策略应在完整时间线上运行，不把零散牛市日期拼接后重新计算指标；跨阶段交易的每日盈亏分别归属，且与同一资产同期买入持有对照。

此前使用的双均线来自[kernc/backtesting.py官方README的Usage示例](https://github.com/kernc/backtesting.py/blob/master/README.md#usage)，类名为SmaCross。它以收盘价计算10/20期简单移动平均线，严格上穿时买入，反向交叉时卖出，并通过exclusive_orders=True实现多空切换。这是入门示例，不是我们原创，也不是上述书中的策略。

如果沿用该基线，牛熊划分使用HL2，均线仍使用收盘价；如需改用HL2均线，应另行注明。首次实现前固定源码版本、仓位、成交时机、手续费、滑点、分红和做空成本，再确定验证与留出区间。

## Next Decisions｜开始分析前要确定

- 确定大盘的数据代理及是否增加板块参考；个股名单已按上文固定。
- 分钟数据来源、共同样本区间、复权口径、交易时段和预热起点。
- 震荡的独立定义，以及后续模型的预测目标、特征和训练区间。
- 如何统计领先滞后，以及如何展示没有一一对应转折的情况。

目前只记录研究目标和规则，不下载新数据、不跑回测、不调参，也不上传原始行情或凭证。
