# RQData (米筐 HTTP API) DolphinDB 模块使用说明

## 1. 模块介绍

`RQData` 模块是对米筐 RQData HTTP API 的 DolphinDB 原生封装，用于在 DolphinDB 脚本中直接获取**期货、股票、指数、基金**的行情与特色数据，并自动解析为结构化的 `Table`。

**核心优势**：

- **免 Python 依赖**：不安装 `rqdatac`，仅依赖 DolphinDB 官方 `httpClient` 插件，数据在库内闭环。
- **Token 自动管理**：账号密码换取 token，token 缓存在共享表，90 分钟自动续期；遇到 401/403 自动重登并重试一次。
- **CSV 自动解析**：RQData 返回 CSV 文本，模块自动落临时文件、按 STRING 读入、解析后立即删除临时文件。
- **统一类型口径**：所有列先按 STRING 读入，再由业务层显式转型，避免空串导致 `loadText` 类型推断失败。
- **四类资产覆盖**：行情按资产类型自动匹配字段；另提供期货主力/仓单/持仓排名、股票分红/复权/因子/行业/两融/港股通/资金流等专项接口。
- **期货落库开箱即用**：配套 `syncRqData` 模块，`rqInitDb()` 一键建库建表，可把全市场期货日线/分钟线/合约快照写入 `dfs://RQDATA.futu_*`。

---

## 2. 快速开始

### 2.1 依赖安装与加载

本模块依赖 DolphinDB 官方的 `httpClient` 网络插件。

```
installPlugin("httpClient");
loadPlugin("httpClient");
```

也可以在节点配置文件中设置 `preloadModules=plugins::httpClient`，避免每次手动加载。

### 2.2 安装模块

#### 版本要求

- 支持 DolphinDB Server 2.00.17 及以上版本（开发与验证环境为 3.00.6）。
- 模块文件名必须与模块名一致：`RQData.dos`。

#### 安装步骤

1. 将 `RQData.dos`（以及可选的 `syncRqData.dos`）放入服务器的模块目录（默认为 `<server>/modules`）。
2. 使用 `use` 关键字加载：

```
use RQData
```

3. 校验加载结果：

```
getLoadedModules()
```

### 2.3 凭证获取

RQData HTTP API 使用 `POST /auth` 换取 token，token 有效期 **2 小时**。支持两种凭证：

| 模式 | user | pwd | `/auth` 返回 |
| :-- | :-- | :-- | :-- |
| 许可证（RQSDK） | 固定填 `"license"` | 邮件里的 License Key | 裸 token 文本 |
| 账号密码 | 账号 | 密码 | JSON `{"token":"..."}` |

`rqGetToken` 会自动识别两种返回格式，所以调用方式一致：

```dolphindb
// 许可证模式（RQSDK 购买/试用邮件里的 License Key）
token = RQData::rqGetToken("license", "<License Key>")

// 账号密码模式
token = RQData::rqGetToken("your_user_name", "your_password")
```

> 注意：重复获取 token 会让**旧 token 立即失效**。因此模块内置了 token 缓存，正常情况下每个进程 90 分钟才换一次。
>
> License Key 属敏感信息，不要写进会提交到仓库的文件，建议配合 `syncRqData::rqSetCred` 或环境变量使用。

---

## 3. 基础 API

### rqGetToken

获取（或复用缓存的）访问 token。

**语法**

```dolphindb
RQData::rqGetToken(user, pwd, [force])
```

**参数**

**user** STRING 类型标量。许可证模式固定填 `"license"`，账号密码模式填账号。

**pwd** STRING 类型标量。许可证模式填 License Key，账号密码模式填密码。

**force** BOOL 类型标量，可选，默认 false。为 true 时强制重新登录（用于 token 失效场景）。

**返回值**

STRING 类型标量，token。

**示例**

```dolphindb
token = RQData::rqGetToken("your_user_name", "your_password")
```

---

### rqApi

通用 API 调用，直接透传 rqdatac 的方法名与参数。

**语法**

```dolphindb
RQData::rqApi(token, params, [timeout])
```

**参数**

**token** STRING 类型标量。

**params** DICTIONARY 类型，键值对即 rqdatac 的方法名与参数。**方法名必须放在 `m` 键下**（Python 中为 `method`，HTTP 接口为避免关键字冲突改为 `m`）。

**timeout** INT 类型标量，可选，默认 180000（毫秒）。

**返回值**

DICTIONARY，包含 `responseCode`、`text`（原始 CSV 文本）、`headers`、`elapsed`。

**示例**

```dolphindb
p = dict(STRING, ANY)
p["m"] = "all_instruments"
p["type"] = "Future"
resp = RQData::rqApi(token, p)
resp["responseCode"]   // 200
```

---

### rqRequest

带自动重登的 API 调用：内部先取 token，若返回 401/403 则强制换 token 再重试一次。

**语法**

```dolphindb
RQData::rqRequest(user, pwd, params, [timeout])
```

**参数**

同 `rqApi`，但第一个参数是账号而不是 token。

**返回值**

同 `rqApi`。

---

### rqParseCsv

把 RQData 返回的 CSV 文本解析为 DolphinDB Table。

**语法**

```dolphindb
RQData::rqParseCsv(text)
```

**详情**

- 取第一行为表头；首列名为空时自动命名为 `rq_col_0`（RQData 的索引列通常是匿名的）。
- 列名重复时自动追加序号，避免 `loadText` 报错。
- 全部列按 STRING 读入，不做类型推断（空串不会被误判为数值列）。
- 临时文件名为内容 md5，解析后立即删除。

**返回值**

Table，所有列为 STRING。

---

### rqGetPrice

获取日线 / 分钟线行情。

**语法**

```dolphindb
RQData::rqGetPrice(user, pwd, rqIds, startDate, endDate, [frequency], [fields])
```

**参数**

**user / pwd** STRING 类型标量，凭证（许可证模式为 `"license"` + License Key，详见 2.3）。

**rqIds** STRING 向量，RQData 合约代码，如 `["AU2412","RB2410"]`。

> **期货代码不带交易所后缀**（`AU2412`，交易所看上期所 `SHFE` 等，见 `all_instruments` 的 `exchange` 列）；股票、指数等品种才带后缀（`000001.XSHE`）。传 `AU2412.SHF` 会返回 `at least one valid instrument expected`。**不要硬编码**，请用 `all_instruments` 返回的 `order_book_id` 建立映射。

**startDate / endDate** DATE 类型标量。

**frequency** STRING 类型标量，可选，默认 `"1d"`，支持 `"1m"`、`"5m"`、`"60m"` 等。

**fields** STRING 向量，可选。默认按频率区分：

- 日线：`open, high, low, close, volume, total_turnover, open_interest, settlement, prev_settlement, limit_up, limit_down`
- 分钟线：`open, high, low, close, volume, total_turnover, open_interest, trading_date`

> 分钟线**不支持** `settlement`、`prev_settlement`、`limit_up`、`limit_down`，传了会报 `fields: got invalided value`；日线则没有 `date`、`trading_date` 字段，时间维度统一在 `datetime` 列。

**返回值**

Table（全部列 STRING）。日线含 `order_book_id`、`datetime` 及请求的字段；分钟线额外含 `trading_date`（格式 `yyyy-MM-dd HH:mm:ss`），可直接用于夜盘归属。

**示例**

```dolphindb
tb = RQData::rqGetPrice("license", "<License Key>", ["AU2412"], 2024.10.08, 2024.10.18)
```

**返回样例**

| rq_col_0 | order_book_id | datetime   | open  | high  | low   | close | volume | total_turnover | open_interest | settlement | prev_settlement | limit_up | limit_down |
| :------- | :------------ | :--------- | :---- | :---- | :---- | :---- | :----- | :------------- | :------------ | :--------- | :-------------- | :------- | :--------- |
| 0        | AU2412        | 2024-10-08 | 590.0 | 595.0 | 588.0 | 592.0 | 12345  | 123456789.0    | 5678          | 591.0      | 590.0           | 650.0    | 530.0      |

---

### rqAllInstruments

获取全量合约清单（含已退市合约）。

**语法**

```dolphindb
RQData::rqAllInstruments(user, pwd, [typeName])
```

**参数**

**typeName** STRING 类型标量，可选，默认 `"Future"`。

**返回值**

Table（全部列 STRING）。期货（`type="Future"`）实际返回列：
`order_book_id`、`exchange`、`symbol`、`trading_code`、`underlying_symbol`、`product`、`type`、
`contract_multiplier`、`margin_rate`、`listed_date`、`de_listed_date`、`maturity_date`、
`start_delivery_date`、`end_delivery_date`、`trading_hours`、`round_lot`、`market_tplus`、`industry_name` 等。

> 注意四点：
> 1. `order_book_id` 不带交易所后缀（`AU2412`），交易所取 `exchange` 列（`SHFE`）；
> 2. 品种代码用 `underlying_symbol`（`AU`），`symbol`/`trading_code` 是小写合约代码（`au2412`），老合约的 `symbol` 甚至是中文（`豆一0303`），不要拿它当品种或名称；
> 3. 期货清单**没有 `name` 列**，需要中文名请另调合约详情接口。
> 4. 请求**不带 `date` 参数**，所以返回的是全部历史合约（期货约 1.1 万条）。若带上 `date`，RQData 只返回该日仍在交易的合约，已退市合约（如 `AU2412`）会查不到。

---

### rqTradingDates

获取交易日历。

**语法**

```dolphindb
RQData::rqTradingDates(user, pwd, startDate, endDate)
```

**返回值**

Table，单列日期。

---

## 4. 期货落库模块（syncRqData）

`syncRqData` 是配套的 ETL 模块，把 RQData 数据写入以下表（每张表一个独立 database，表名统一为 `data`）：

| 库表 | 内容 | 引擎 / 分区 |
| :-- | :-- | :-- |
| `dfs://RQDATA.futu_bar.data` | 日线 + 分钟线，17 列 | TSDB，VALUE（`trade_date` 月），`sortColumns=symbol,interval,datetime` |
| `dfs://RQDATA.futu_contract.data` | 每日合约快照，21 列（含 `industry`、`variety_cn`、`delivery_month`、`dominant_id`） | TSDB，VALUE（`trade_date` 月），`sortColumns=symbol,trade_date` |
| `dfs://RQDATA.futu_calendar.data` | 交易日历 | VALUE（`trade_date` 月） |
| `dfs://RQDATA.futu_synclog.data` | 同步日志 | VALUE（`start_date` 月） |

合约静态信息不单独建维表：`contract` 每个交易日都有一行完整快照，取 `max(trade_date)` 那天即“当前合约池”，合约清单每次同步由 `all_instruments` 现取。

日线和分钟线共用一张 `bar` 表，用 `interval` 列（`1d`/`1m`/`5m`/`60m`）区分；分钟线的 `trade_date` 取自 `trading_date`，夜盘自动归属到下一交易日。

### 4.1 RQData 侧字段（实测）

**`all_instruments(type="Future")`** —— 19 列，含已退市合约（当前 11429 条）：

```
contract_multiplier, de_listed_date, end_delivery_date, exchange, industry_name,
listed_date, margin_rate, market_tplus, maturity_date, order_book_id, product,
round_lot, start_delivery_date, symbol, trading_code, trading_hours, type,
underlying_order_book_id, underlying_symbol
```

**`get_price`（期货）** —— 日线 13 列、分钟线 11 列：

| 频率 | 列 |
| :-- | :-- |
| `1d` | `order_book_id, datetime, open, high, low, close, volume, total_turnover, open_interest, settlement, prev_settlement, limit_up, limit_down` |
| `1m` | `order_book_id, datetime, open, high, low, close, volume, total_turnover, open_interest, trading_date` |

**`instruments(["AU2412"])`** —— 19 列，单合约详情，比清单少的只是 `type` 之外无差别，同样**没有 `name` 列**：

```
order_book_id, type, exchange, symbol, underlying_symbol, underlying_order_book_id,
product, maturity_date, de_listed_date, listed_date, round_lot, margin_rate,
contract_multiplier, trading_code, market_tplus, start_delivery_date,
end_delivery_date, trading_hours, industry_name
```

> 期货中文合约名按交易所不统一：大商所/中金所/广期所的 `symbol` 是中文简称（`豆一2701`、`股指2612`、`碳酸锂2701`），上期所/郑商所/能源中心只返回小写代码（`rb2701`、`CF2701`、`sc2612`）。落库时由 `rqVarietyCnNames` 内置字典把这些交易所还原为「品种中文名 + 交割月」（`rb2701` → `螺纹钢2701`），连续合约后缀与 RQData 自带命名保持一致（`主力连续` / `主力连续价差平滑` / `主力连续价差平滑（后复权）` / `次主力连续` / `指数连续`）。

探测方式：任何接口返回的都是 CSV，用 `schema()` 直接看列名与类型。

```dolphindb
RQData::rqGetPrice("license", key, ["AU2412"], 2024.10.08, 2024.10.08, "1d").schema()
RQData::rqAllInstruments("license", key, "Future").schema()
```

**字段映射**

| RQData | FUTU | 说明 |
| :-- | :-- | :-- |
| `date` / `trading_date` | `trade_date` | 交易日期 |
| `datetime` | `datetime` | 日线统一取 15:00 |
| `total_turnover` | `turnover` | 单位：元 |
| `settlement` | `settle` | 结算价 |
| `prev_settlement` | `pre_settle` | 昨结算；分钟线不返回该字段，用同日日线补齐 |
| `volume` | `volume` | 单位：手 |
| `limit_up` / `limit_down` | 同名 | 写入 contract 快照 |
| `contract_multiplier` | `size` | 合约乘数 |
| `listed_date` / `de_listed_date` | `list_date` / `last_date` | 上市/退市日 |
| `margin_rate` | `margin_ratio` | 保证金率，写入 `contract` 快照（原始值为 Python 浮点，落库前保留 6 位小数） |
| `industry_name` | `industry` | 行业分类，写入 `contract` 快照 |
| `get_tick_size` 的 `tick_size` | `pricetick` | 最小变动价位，`all_instruments` 不含，按候选合约单独请求 |
| `get_trading_parameters` 的 `open_commission` | `commission_num` | 开仓手续费 |
| 同上 `commission_type` | `commission_unit` | `by_volume` → `元/手`（原值）；`by_money` → `%`（比例 ×100） |
| 同上 `max_order_quantity` | `max_order_vol` | 单笔最大下单量（手）。交易所只报 `max_order_quantity` 与 `client_limit` 其中之一，故该列会大面积为空，属数据源口径 |
| 同上 `client_limit` | `client_limit` | 客户限仓（手）。与 `max_order_vol` 互补：`RB2701` 只报最大下单量 32000，`A2701` 只报限仓 17779 |

contract 字段的取值优先级：**RQData 当日值 → 当日主力合约（88/888/889 借用）→ 当日次主力合约（88A2 借用）→ 同品种当日主力合约（99 兜底）→ `startDate` 之前最近一份 contract 快照**。
已退市品种（ER / LR / ME / RO / TC / WS / WT）的连续合约在 RQData 侧没有任何交易参数，`commission_num` / `max_order_vol` / `client_limit` 保持 NULL。
2026-11-04 实测覆盖率：`max_order_vol` 496/870、`client_limit` 867/870（仅 `AD2701`、`HC2701`、`RB2701` 为空，前两者分别只报限仓或只报下单量）。

### 4.2 建库建表

首次使用执行一次，幂等，可重复执行：

```dolphindb
use RQData
use syncRqData
syncRqData::rqInitDb()
```

之后的同步函数默认这些库表已存在（`rqLoad` 内部用 `loadTable("dfs://RQDATA.futu_xxx", "data")`）。

### 4.3 首次构建流程

按“建库 → 日历 → 日线 → 分钟线”的顺序执行：

```dolphindb
user = "license"
key  = "<License Key>"

syncRqData::rqInitDb()                                        // 1. 建库建表
syncRqData::rqSetCred(user, key)                              // 2. 凭证写共享表（定时任务用）
syncRqData::rqBackfillAll(user, key, 2010.01.01)              // 3. 日线 + contract 快照 + 交易日历
syncRqData::rqBackfillMinute(user, key, 2026.09.01, today())  // 4. 分钟线（按需，很吃配额）
```

> 两个回补入口都是**基于库内已有数据判断**：已经完整的交易日直接跳过，不会重复拉取，可放心反复执行。底层无条件重拉的 `rqSyncBars` 仅用于调试或强制重刷。

> `rqSyncBars(..., "1d")` 会同时写 `contract` 快照：按“交易日 × 上市合约”生成，**上市期间每个交易日一行**，与该合约当日是否有成交无关。停牌/无成交的行仍保留，只是 `limit_up`/`limit_down` 为 NULL。

分钟线**很吃配额**（全市场单日约 44 万行、约 37 MB），回补前先算好余量：

```dolphindb
syncRqData::rqGetQuota(user, key)   // 查看已用/上限/剩余天数
```

容量参考：单个合约 1 分钟线约 555 行/交易日；全市场活跃合约约 1300 个，一天约 44 万行，一年约 1.1 亿行。`bar` 表按月分区，TSDB 下该量级可直接 append。

### 4.4 分钟线 pre_settle 补齐

RQData 的分钟线不返回 `prev_settlement`，所以分钟行的 `pre_settle` 由**同一交易日的日线** `prev_settlement` 补齐（夜盘归属同一交易日，取值一致）：

- **新写入**：`rqSyncBars(..., "1m")` 落库前自动完成，无需额外步骤。
- **历史数据**（旧版本写入、`pre_settle` 为空）用回填函数，按月 × 合约批处理，幂等可重跑：

```dolphindb
syncRqData::rqFillPreSettle("license", key, 2020.01.01, today(), "1m", 50)
```

- **每日增量**：`rqMinuteJob` 是无参入口，固定回看 5 天，可直接挂定时任务：

```dolphindb
scheduleJob(jobId=`sync_futu_rqdata_minute,
            jobDesc="sync_futu_rqdata_minute",
            jobFunc=syncRqData::rqMinuteJob,
            scheduleTime=[17:20m],
            startDate=today(), endDate=2050.12.31, frequency='D')
```

> 回填按“月 × 合约批”读取单个分区再写回，单次内存占用可控；分钟线单日约 60 万行，建议从最近月份往前推进。

### 4.5 contract 表怎么当维表用

`contract` 每个交易日一行完整快照（静态字段 + 当日涨跌停），所以不再需要单独的 `contract_meta`：

```dolphindb
con = loadTable("dfs://RQDATA.futu_contract", "data")
// 当前合约池（= 最近一个交易日仍在上市的合约）
lastD = exec max(trade_date) from con
cur = select * from con where trade_date == lastD
// 某一天的合约池 / 保证金率 / 涨停价
snap = select * from con where trade_date == 2024.10.08
```

几点说明：

- 静态字段（乘数、pricetick、手续费、上市/退市日、行业）每天重复存一份；按全市场 1.1 万合约、平均存续约 1 年估算，`contract` 总量在千万行以内，TSDB 下可接受。
- RQData 只提供当前保证金率，没有历史；靠每日快照把历史值固化下来，回补历史区间时不会用“今天”的保证金率覆盖过去。
- 合约清单（含已退市）每次同步由 `all_instruments` 现取，不落库；`contract` 的网格就是用这份清单 × 交易日生成的。

### 4.6 连续合约与主力复权

RQData 的 `all_instruments(type="Future")` 已经包含连续合约，`rqSyncBars` 会随正常合约一并拉取行情、一并生成 `contract` 行，无需额外配置：

| 代码 | 含义 | 复权口径 |
| :-- | :-- | :-- |
| `XX88` | 主力连续 | 不复权（换月跳空） |
| `XX888` | 主力连续价差平滑 | 前复权（实测与 `adjust_type="pre"` 完全一致） |
| `XX889` | 主力连续价差平滑（后复权） | 后复权 |
| `XX88A2` | 次主力连续 | 不复权 |
| `XX88A3` | 次次主力连续（仅股指期货） | 不复权 |
| `XX99` | 指数连续（全合约加权） | 无复权概念 |

它们的 `listed_date` / `de_listed_date` 都是 `0000-00-00`，模块按 NULL 处理，因此上市期间每个交易日都会生成 `contract` 行。

`contract.dominant_id` 记录连续合约当日对应的真实合约（普通合约与 `99` 为空）：

| 连续合约 | dominant_id 来源 |
| :-- | :-- |
| `XX88` / `XX888` / `XX889` | `futures.get_dominant_price` 的 `dominant_id`（当日主力） |
| `XX88A2` | `futures.get_dominant(rank=2)`（当日次主力）；股指期货默认 rule 无结果，自动改用 `rule=1` |
| `XX88A3` | `futures.get_dominant(rank=3)`（当日次次主力），同上 |
| `XX99` | 留空（指数连续无单一映射合约） |

`contract` 还提供两个便于筛选的维度列：

| 列 | 说明 |
| :-- | :-- |
| `variety_cn` | 品种中文名（`RB2701` → `螺纹钢`、`CF99` → `棉花`、`IF88A3` → `沪深300股指`） |
| `delivery_month` | 交割月份 MONTH（`RB2701` → `2027.01M`）；连续合约为 NULL，可用 `isNull(delivery_month)` 区分真实合约与连续合约 |

```dolphindb
con = loadTable("dfs://RQDATA.futu_contract", "data")
// AU888 / AU88A2 在某天各自对应哪个真实合约
select trade_date, symbol, dominant_id from con where symbol in ["AU888","AU88A2"] and trade_date == 2024.10.08
// AU888 数据 = AU 品种的主力前复权序列，回测换月时用 dominant_id 对接真实合约
```

后复权不在库里，需要时直接调接口：

```dolphindb
dp = RQData::rqFutDominantPrice(user, pwd, ["AU"], 2024.10.08, 2024.10.09, "post", "1d", ["close"])
```

> `adjust_type` 仅对 `futures.get_dominant_price` 生效；对 `get_price` 的 `88/888/99` 传 `adjust_type` 不会改变结果（实测三种口径完全一致）。

### rqSetCred

把账号密码写入共享表 `rq_config`（节点重启后需重新设置）。定时任务 `rqDailyJob` 从这里读取凭证，避免把密码硬编码进模块。

```dolphindb
use RQData
use syncRqData
// 许可证模式
syncRqData::rqSetCred("license", "<License Key>")
// 账号密码模式
syncRqData::rqSetCred("your_user_name", "your_password")
```

### rqFetchInstruments

把 RQData `all_instruments(type="Future")` 转成带 `rq_id / symbol / exchange / variety / name / size / list_date / last_date / margin_ratio / industry` 的清单，供 `rqSyncBars` 内部使用（含已退市合约，不落库）。

```dolphindb
inst = syncRqData::rqFetchInstruments(user, pwd)
select count(*) from inst
```

### rqSyncBars

按区间批量拉取并落库，日线同时写 `bar` 与 `contract` 快照；`contract` 按“交易日 × 上市合约”生成，行数通常多于 bar。

**语法**

```dolphindb
syncRqData::rqSyncBars(user, pwd, startDate, endDate, [frequency], [batchSize])
```

**参数**

**frequency** STRING 类型标量，`"1d"`（默认）/`"1m"`/`"5m"`/`"60m"`。

**batchSize** INT 类型标量，默认 50，每个 HTTP 请求携带的合约数。

**示例**

```dolphindb
// 日线增量（回看 10 天补漏）
syncRqData::rqSyncBars("user", "pwd", today() - 10, today(), "1d", 50)

// 单品种 1 分钟线回补
syncRqData::rqSyncBars("user", "pwd", 2024.01.01, today(), "1m", 5)
```

### rqSyncCalendar

补齐 `calendar` 表（`exchange` 统一记为 `ALL`）。

### rqBackfillAll

全历史回补入口，等价于依次执行合约刷新 + 日历补齐 + 日线全量。

```dolphindb
syncRqData::rqBackfillAll("user", "pwd", 2010.01.01)
```

### rqRegisterJob

注册每日 17:00 的增量任务（手动执行一次）。

```dolphindb
syncRqData::rqRegisterJob()
getScheduledJobs()
```

---

## 5. 技术特性与校验规则

- **Token 缓存**：共享表 `rq_token_cache` 保存 `user_id/token/fetch_time`，90 分钟内复用；重复登录会使旧 token 失效，因此不要在多进程里各自频繁换 token。
- **双认证模式**：`/auth` 返回值可能是 JSON（账号密码）或裸 token 文本（License Key），`rqGetToken` 自动识别。
- **401 自动重试**：`rqRequest` 捕获 401/403，强制换 token 后重试一次；仍失败则抛异常。
- **列名容错**：CSV 首列匿名、列名重复都已处理；`nan`/`None`/`NaT`/空串统一转为 NULL。
- **类型安全**：先全 STRING 读入再显式转型，避免 DolphinDB 的类型推断把整列判成 STRING。
- **写入幂等**：TSDB 表按 `sortColumns` 去重，同键后写覆盖，增量任务可安全重跑。
- **批次限速**：每批次之间 `sleep(200)` 毫秒，默认 50 合约/批，避免触发服务端限流。

---

## 6. 进阶用法

### 6.1 批量请求与分片

`get_price` 的 `order_book_ids` 支持一次传多个合约。建议按 50 个合约一批，既控制单次 CSV 体积（避免字符串过大），也便于失败重试定位。

### 6.2 分钟线回补

分钟线数据量大，建议按品种、按时间段分片回补，并只写 `bar` 表（不写 contract 快照）：

```dolphindb
ids = ["AU2412"]
tb = RQData::rqGetPrice("license", "<License Key>", ids, 2024.10.01, 2024.10.31, "1m")
```

分钟线默认字段已包含 `trading_date`（格式 `yyyy-MM-dd HH:mm:ss`），落库时取前 10 位日期作为 `trade_date`，即可把夜盘正确归属到下一交易日。

### 6.3 只取部分字段

```dolphindb
tb = RQData::rqGetPrice("license", "<License Key>", ["AU2412"], 2024.10.08, 2024.10.18, "1d",
        ["open","close","volume","open_interest"])
```

> 自定义 `fields` 时模块不会替你补 `trading_date`，分钟线请自己显式加上，否则夜盘归属会落到自然日。

---

## 7. 常见错误

| 现象 | 原因 | 处理 |
| :-- | :-- | :-- |
| `RQData 认证失败(400): invalid username or password` | 账号密码错误 | 检查账号密码 |
| `RQData get_price 失败(401): invalid token` | token 失效（被人重复登录挤掉） | `rqGetToken(..., force=true)` 重新换取 |
| `RQData get_price 失败(400): fields: got invalided value settlement` | 分钟线传了日线专属字段（`settlement`/`prev_settlement`/`limit_up`/`limit_down`） | 分钟线只用 `open/high/low/close/volume/total_turnover/open_interest/trading_date` |
| `RQData get_price 失败(400): at least one valid instrument expected` | 期货代码带了交易所后缀（`AU2412.SHF`） | 期货用不带后缀的 `order_book_id`（`AU2412`） |
| `RQData get_price 失败(400): fields: got invalided value date` | 日线显式请求了 `date`/`trading_date` | 日线不要传这两个字段，用 `datetime`；`trading_date` 只属于分钟线 |
| `RQData CSV 解析失败` | 返回体不是 CSV（多为错误信息） | 先打印 `resp["text"]` 排查 |
| `The add function doesn't support the specified types` | 拼接非字符串对象 | 用 `string()` 显式转型 |

---

## 8. 版本更新说明 (Changelog)

### v1.8.0（syncRqData）

- 新增 `rqPlanBackfill(user, pwd, startDate, endDate, includeMinute)`：**不拉行情**的预算预估，输出每段交易日数、缺失天数与估算 GB，并对比当前剩余额度算出缺口。谈额度前先用它量数。
- 新增 `rqQuotaLeftMb`（剩余配额）、`rqSegByYear` / `rqSegByDays`（分段器）、`rqRunBackfill`（带配额保护的逐段执行）。
- `rqBackfillAll` 改为按自然年分段、`rqBackfillMinute` 改为按 10 天分段（默认合约批 20），避免全历史区间一次性请求超时或内存暴涨；每段前检查剩余配额，低于 2 GB 自动停止。
- 两个回补入口均支持断点续跑：中断后重跑只补缺口。
- 修正：`for(y in a:b+1)` 会被解析成 `(a:b)+1` 导致循环为空，必须写 `a:(b+1)`；`e` 是 DolphinDB 自然常数，不能作变量名；形参为常量，不能原地赋默认值。

### v1.7.0（syncRqData）

- 新增 `rqSyncIncremental(user, pwd, frequency, startDate, endDate, batchSize, refreshDays)`：按库内已有数据判断要拉哪些交易日，已完整的直接跳过，不再盲目重拉。判定标准为“当日 bar 覆盖率 ≥ 当日 contract 行数的 90%”，覆盖率不足视为批次失败、重拉。
- 新增 `rqFilledDates`（判定已完整交易日）与 `rqDateSegments`（把连续交易日合并成区间，减少 `all_instruments` 重复拉取）。
- `rqBackfillAll` / `rqBackfillMinute` 改为走增量判定，可反复执行；新增 `rqBackfillMinute` 分钟线回补入口。
- `rqDailyJob` 回看窗口 10 → 30 天，另强制刷新最近 3 个交易日，用于纠正收盘后才最终确定的结算价。
- `rqMinuteJob` 默认 `batchSize` 5 → 50，`lookback` 5 → 10，且改为只拉缺的交易日。
- 修正：此前分钟线每日固定回看 5 天，等于每天重复拉 5 × 37 MB ≈ 185 MB，配额消耗严重。

### v1.6.0（syncRqData）

- `contract` 新增 `client_limit`（DOUBLE，客户限仓），共 21 列，取 `futures.get_trading_parameters` 的 `client_limit`。
- 原因：交易所只报 `max_order_quantity` 与 `client_limit` 其中之一，此前用 `max_order_quantity` 时 374/870 真实合约为空；两列并列后真实合约覆盖 496 + 867 行，互不混用口径。两列共用同一套兜底链（本合约 → 当日主力 → 同品种 → 上一份快照）。
- 升级需 `dropDatabase("dfs://RQDATA.futu_contract")` 后重新执行 `rqInitDb()`（TSDB 不支持加列）。

### v1.5.9（syncRqData）

- `contract` 新增两列（18 → 20 列）：
  - `variety_cn`（STRING）：品种中文名。优先级为 `rqVarietyCnNames` 字典（上期所/郑商所/能源中心）→ 中金所别名（`IF`/`IC`/`IH`/`IM`/`T`/`TF`/`TS`/`TL`）→ 从 RQData 中文简称去掉交割月与连续后缀（`豆一2701` → `豆一`）。连续合约也带品种名，例如 `RB88` → `螺纹钢`、`CF99` → `棉花`。
  - `delivery_month`（MONTH）：交割月份，取合约代码里的 4 位年月（`RB2701` → `2027.01M`）；连续合约（`88`/`888`/`889`/`88A2`/`88A3`/`99`）无交割月，为 NULL。
- 新增 `88A3`（次次主力连续）支持：`dominant_id` 取 `futures.get_dominant(rank=3)`，`name` 还原为「品种中文名 + 次次主力连续」，交易参数查询按连续合约跳过。
- 修复连续合约品种名提取：`股指次次主力连续` 不再被截成 `股指次`。
- 注意：TSDB 不支持加列，升级需 `dropDatabase("dfs://RQDATA.futu_contract")` 后重新执行 `rqInitDb()`。

### v1.5.8（syncRqData）

- `contract.name` 补齐中文合约名：RQData 只对大商所/中金所/广期所返回中文简称，上期所/郑商所/能源中心返回小写代码。新增 `rqVarietyCnNames`（品种代码 → 中文名）与 `rqFixNames`，按「品种中文名 + 交割月 / 连续后缀」还原，例如 `rb2701` → `螺纹钢2701`、`CF99` → `棉花指数连续`。已有中文名的品种保持原值不改。

### v1.5.7（syncRqData）

- `bar` 表移除两个无用列：`series`（恒为常量 `"C"`）与 `next_symbol`（RQData 不提供、始终为 NULL），列数 19 → 17。换月关系改由 `contract.dominant_id` 表达。
- 注意：TSDB 不支持删列，升级需 `dropDatabase("dfs://RQDATA.futu_bar")` 后重新执行 `rqInitDb()`。

### v1.5.6（syncRqData）

- `dominant_id` 覆盖范围扩展：`88/888/889` 记为当日主力，`88A2` 记为当日次主力（新增 `rqSecondaryIds`，走 `futures.get_dominant(rank=2)`，逐品种查询；股指期货默认 rule 返回空时自动改用 `rule=1`），`99` 保持为空。
- 交易参数的取值链同步调整为“本合约 → 当日主力 → 当日次主力 → 同品种主力 → 上一份快照”。

### v1.3.1（RQData）

- `rqFutDominant` 新增 `rank`（1-主力 / 2-次主力 / 3-次次主力）与 `rule`（主力切换规则）参数；股指期货取次主力须显式传 `rule=1`。

### v1.5.5（syncRqData）

- 新增 `rqTradingParams`，经 `futures.get_trading_parameters` 落库 `commission_num` / `commission_unit` / `max_order_vol`；`by_money` 费率 ×100 转百分数（对齐 HAZQ 口径）。
- 连续合约（88/888/889/99/88A2）接口不返回交易参数，按“同日主力合约 → 同品种”兜底。
- 修复合约 `variety` 提取规则：由“删除所有数字”改为“截到首个数字”，修复 `A88A2`（次主力连续）被解析成 `AA` 的问题。
- 单测：`rqBuildContract` 增加 `tpVar` 入参。

### v1.5.4（syncRqData）

- 新增 `rqTickSizes`，日线同步时经 `get_tick_size` 补齐 `contract.pricetick`（只查候选合约，按 300 个/批请求）。
- `margin_ratio` 落库前保留 6 位小数，修复 RQData 原始浮点值（如 `0.06999999999999999`）直接入库的问题。

### v1.5.3（syncRqData）

- `rqToDate` 改为向量化（`substr` 逐元素截取），修复 `all_instruments` 整列传入时报 “A scalar object is expected”。
- `rqSyncBars` 去掉对 DFS 表句柄的 `isNull` 判断（DFS 表不支持该调用）。
- `rqDominantIds` 的分组聚合改为引用表内列，修复主映射整列取到同一个合约（如 AU88 被写成 BC2612）的问题。

### v1.5.2（syncRqData）

- 库名恢复为 `dfs://RQDATA.futu_bar` / `futu_contract` / `futu_calendar` / `futu_synclog`，表名仍为 `data`。

### v1.5.1（syncRqData）

- 修复 `rqInitDb` 的 `keepDuplicates` 参数：TSDB 要求枚举 `LAST`，传字符串 `"LAST"` 会报错。
- 修复 `rqFillPreSettle` 的月份起始计算：`date(year, month, 1)` 不是合法写法，改用 `d - dayOfMonth(d) + 1`。

### v1.5.0（syncRqData）

- 连续合约（`XX88` 主力连续、`XX888` 价差平滑、`XX99` 指数连续）随正常合约一并同步，`contract` 每个交易日为其生成一行。
- 修复连续合约被过滤：其 `listed_date`/`de_listed_date` 为 `0000-00-00`，`rqToDate` 现按 NULL 处理（原先可能被解析为有效日期导致 `last_date < trade_date` 而被剔除）。
- `contract` 新增 `dominant_id`（STRING，共 18 列）：`88/888` 当日对应的真实主力合约，取自 `futures.get_dominant_price`；`99` 与普通合约为空。
- `name` 优先取 `all_instruments.symbol`（期货中文简称），不再留空。
- 新增 RQData 接口封装：`rqGetTickSize`、`rqFutTradingParams`、`rqFutDominantPrice`。
- 升级需重建 `contract` 表（列数由 17 变 18）。

### v1.4.0（syncRqData）

- 取消 `contract_meta` 表：`contract` 每个交易日一行完整快照（新增 `industry` 列，共 17 列），取 `max(trade_date)` 即当前合约池；合约清单改为每次同步由 `all_instruments` 现取，`rqSyncInstruments` 移除。
- `pricetick` / `commission_num` / `max_order_vol` 改为按“同合约 → 同品种”继承 `startDate` 之前最近一份 `contract` 快照，不再依赖 meta。
- 升级需重建 `contract` 表（列数由 16 变 17），旧库中的 `FUTU_contract_meta` 可手动删除。

### v1.3.1（syncRqData）

- 修正 `contract` 快照口径：原先由 bar 行构造（只有当日有行情才有行），现改为按“交易日 × 上市合约”生成，上市期间每个交易日一行，含停牌/无成交合约；当日无行情时 `limit_up`/`limit_down` 为 NULL。
- 交易日列表优先取库内 `calendar`，不完整时调 `rqTradingDates` 补齐，最后退化为自然日。
- 继承字段（`margin_ratio` / `max_order_vol` / `commission_unit`）改为只参考 `startDate` 之前最近一份快照，避免历史回补时继承到未来的值。

### v1.2.0（syncRqData）

- 新增 `rqInitDb()`：一键创建 5 个库表（`bar` / `contract` / `contract_meta` / `calendar` / `synclog`），幂等可重复执行。
- 库名由 `dfs://FUTU.xxx` 调整为 `dfs://FUTU_xxx`，表名统一为 `data`。
- `bar` 表改为 `keepDuplicates=LAST` 且 `sortColumns=symbol,interval,datetime`：修复重跑产生重复行，以及日线 15:00 与当天最后一根 1 分钟线互相覆盖。
- `contract_meta` 新增 `margin_ratio`、`industry`（取自 `margin_rate` / `industry_name`），共 11 列；`contract` 快照的保证金率优先取该值。
- 修复日线 `datetime` 只有日期（`yyyy-MM-dd`）时解析失败、`trade_date` 落成 NULL 的问题；字典键统一转 STRING，避免 SYMBOL/STRING 不匹配查不到合约。

### v1.3.0（syncRqData）

- 分钟线自动补齐 `pre_settle`：RQData 分钟线无 `prev_settlement`，改用同日日线的 `prev_settlement` 映射填充，写入时完成。
- 新增 `rqFillPreSettle(user, pwd, startDate, endDate, freq, batchSize)`：历史数据的 `pre_settle` 回填，按月 × 合约批处理，幂等。
- 新增 `rqMinuteJob(lookback, frequency, batchSize)`：分钟线每日增量无参入口，可直接用 `scheduleJob` 调度。

### v1.2.1

- `rqAllInstruments` 不再传 `date` 参数：此前传 `date=today` 只返回当日仍在交易的合约，`AU2412` 等已退市合约会查不到；现在返回含已退市合约的全量清单（期货约 1.1 万条）。
- 必填日期传 `NULL` 时由 `rqDateInt` 直接报错：原先会序列化为 JSON `null`，服务端当作“未指定”，静默返回全历史（`get_ex_factor`）或最新快照（`get_margin_stocks`），属于易误用行为。
- 修复 `rqParseCsv` 并发缺陷：临时文件名由“内容 md5”改为随机 UUID，避免相同响应在 `ploop` 并发下互相 `rm`，导致 `The file [...] doesn't exist`。

### v1.1.0

- `rqGetToken` 支持 RQSDK 许可证模式（`user="license"` + License Key），兼容 `/auth` 返回裸 token 文本与 JSON 两种格式。
- 修正期货合约代码口径：`order_book_id` 不带交易所后缀（`AU2412`），交易所取 `exchange` 列；文档与测试同步更新。
- `rqGetPrice` 在分钟线默认字段中自动追加 `trading_date`，修复夜盘归属缺失；`syncRqData` 的日期解析兼容 `yyyy-MM-dd HH:mm:ss`。
- README 补充 `all_instruments` 实际列、日线无 `date` 字段、常见错误排查表。

### v1.0.0

- 首个版本：`RQData` 模块（token 缓存 / 通用调用 / CSV 解析 / 行情、合约、日历接口）。
- 配套 `syncRqData` 模块：日线、分钟线、合约快照、日历落库与定时任务。

<!-- write-test -->
<!-- edit-test -->
