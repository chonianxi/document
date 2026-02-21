二买
在一个明确的上升趋势中，价格完成过一次上涨后出现回调。
该回调没有破坏趋势结构（未跌破关键均线或前低），
同时回调阶段的下跌动能明显弱于上一段上涨动能。

当回调结束、下跌动能衰竭，并出现新的上涨动能（如 MACD 由负转正或 DIF 上穿 DEA）时，
认为趋势继续的概率较高，该位置称为「二次买入点（Second Buy）」。

关键特征：

趋势方向仍然向上

回调是“弱回调”，而非趋势反转

动能由弱转强发生在趋势内部
Rule Name: SECOND_BUY

Context:
- Market: CN or CRYPTO
- Timeframe: 30m / 1d

Preconditions (Trend):
1. trend_flag == 1
2. close_price > MA20
3. MA20 is rising

Pullback Conditions (Weak Correction):
4. previous_red_macd_area > 0
5. current_green_macd_area > 0
6. current_green_macd_area < previous_red_macd_area * 0.5

Structure Protection:
7. lowest_price_during_pullback > MA20
   OR
   lowest_price_during_pullback > previous_swing_low

Trigger Conditions (Re-Acceleration):
8. MACD_histogram crosses from negative to positive
   OR
   DIF crosses above DEA

Decision:
IF all above conditions are satisfied
THEN
    signal_type = "SECOND_BUY"
    signal_confidence = "HIGH"

Explanation:
- Second Buy occurs inside an existing uptrend.
- The pullback must show weakening downside momentum.
- The trend structure must remain intact.
- The signal is triggered by the first sign of renewed upside momentum.
- This signal favors trend continuation rather than bottom picking.


防止假二买
Invalid Conditions (False Second Buy):
- trend_flag != 1
- current_green_macd_area >= previous_red_macd_area
- price breaks below MA20
- signal occurs inside a sideways or range-bound market

一买
Rule Name: FIRST_BUY

Context:
- Market: CN or CRYPTO
- Timeframe: 5m / 30m

Downtrend Preconditions:
1. trend_flag == -1
2. close_price < MA20
3. MA20 is falling

Exhaustion Conditions (Momentum Weakening):
4. previous_green_macd_area > 0
5. current_green_macd_area > 0
6. current_green_macd_area < previous_green_macd_area
   (downside momentum decreasing)

Optional Strong Condition:
7. Price makes lower low
   AND
   MACD histogram does NOT make new low
   (bullish divergence)

Structure Stabilization:
8. Current low >= previous low
   OR
   Price forms a higher low on smaller timeframe

Trigger Condition:
9. MACD histogram crosses from negative to positive
   OR
   DIF crosses above DEA

Decision:
IF conditions (1-6) AND (8) AND (9)
THEN
    signal_type = "FIRST_BUY"
    signal_confidence = "MEDIUM"

更准确的一买增加这个指标
IF
    FIRST_BUY conditions
AND boll_squeeze_level >= 1
THEN
    signal_confidence += 10

    

Explanation:
- First Buy happens inside a downtrend.
- It identifies momentum exhaustion rather than confirmed reversal.
- The signal is weaker than Second Buy.
- Risk is higher because trend reversal is not yet confirmed.
- Suitable for early entry with strict stop-loss.

二买和一买的区别
| 维度   | 一买   | 二买   |
| ---- | ---- | ---- |
| 所处趋势 | 下跌中  | 上升中  |
| 性质   | 抄底尝试 | 顺势跟进 |
| 成功率  | 中等偏低 | 高    |
| 风险   | 高    | 低    |
| 回撤   | 大    | 小    |




我想做一个交易系统
分两个层，一个数据源层，一个策略层
数据源层
1、将最新的沪深300数据写入监控股票池，每半年更新一次股票池里面股票
2、将加密货币市值前10的货币写入监控池，每月更新一次需要监控的十个加密货币
3、本地测试验证的时候，我只监控沪深市值前20的股票，具体监控哪些可以通过配置来调整

监控池表结构
mysql
CREATE TABLE watch_symbol (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '主键ID',

    market    VARCHAR(10)  NOT NULL COMMENT '市场标识：CN(A股)、CRYPTO(加密货币)',
    symbol    VARCHAR(20)  NOT NULL COMMENT '证券代码 / 交易对，如 600519、BTC/USDT',
    name      VARCHAR(50)  NOT NULL COMMENT '名称，如 贵州茅台、Bitcoin',

    category  VARCHAR(20)  NOT NULL COMMENT '分类：HS300 / CRYPTO_TOP10 / CUSTOM',
    rank_no   INT          COMMENT '市值或指数内排名（如加密Top10排名）',

    enabled   TINYINT      NOT NULL DEFAULT 1 COMMENT '是否启用监控：1启用 0停用',

    remark    VARCHAR(255) COMMENT '备注说明',
    last_fetch_ts DATETIME COMMENT '上一次成功抓取时间',
    fetch_status  TINYINT DEFAULT 1 COMMENT '抓取状态：1成功 0失败',
    source_conf   JSON COMMENT '数据源特定配置，用于保存特定数据源参数（如 akshare token、ccxt 交易所）'


    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
                 ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

    UNIQUE KEY uk_market_symbol (market, symbol),
    INDEX idx_market_category (market, category),
    INDEX idx_enabled (enabled)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
COMMENT='监控标的配置表（沪深300 / 加密货币Top市值）';

pg
CREATE TABLE watch_symbol (
    id            BIGSERIAL PRIMARY KEY,
    market        VARCHAR(10) NOT NULL,  -- 市场标识：CN(A股)、CRYPTO(加密货币)
    symbol        VARCHAR(20) NOT NULL,  -- 证券代码 / 交易对，如 600519、BTC/USDT
    name          VARCHAR(50) NOT NULL,  -- 名称，如 贵州茅台、Bitcoin
    category      VARCHAR(20) NOT NULL,  -- 分类：HS300 / CRYPTO_TOP10 / CUSTOM
    rank_no       INT,                   -- 市值或指数内排名（如加密Top10排名）
    enabled       SMALLINT NOT NULL DEFAULT 1, -- 是否启用监控：1启用 0停用
    remark        VARCHAR(255),          -- 备注说明
    last_fetch_ts TIMESTAMPTZ,           -- 上一次成功抓取时间
    fetch_status  SMALLINT DEFAULT 1,    -- 抓取状态：1成功 0失败
    source_conf   JSON,                  -- 数据源特定配置，用于保存特定数据源参数（如 akshare token、ccxt 交易所）
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 唯一约束
ALTER TABLE watch_symbol
ADD CONSTRAINT uk_market_symbol UNIQUE (market, symbol);

-- 索引
CREATE INDEX idx_market_category ON watch_symbol(market, category);
CREATE INDEX idx_enabled ON watch_symbol(enabled);

COMMENT ON TABLE watch_symbol IS '监控标的配置表（沪深300 / 加密货币Top市值）';



2、运行某个数据源的实现能离线获取对应指标的5分钟k线并入库作为基础K线，获取数据的时候都是根据数据库本地数据的最近时间去对应数据源获取增量数据，也支持通过某个配置时间段，强行把该时间段内的5分钟数据进行入库
拉回保存到数据库里面的数据结构


mysql保存5分钟K线的表结构
CREATE TABLE kline_base (
    market   VARCHAR(10)   NOT NULL COMMENT '市场标识，如 CN(中国A股)、CRYPTO(加密货币)',
    symbol   VARCHAR(20)   NOT NULL COMMENT '证券代码 / 交易对，如 600519、BTC/USDT',
    ts       DATETIME      NOT NULL COMMENT 'K线时间（K线开始时间）',
    open     DECIMAL(18,6) NOT NULL COMMENT '开盘价',
    high     DECIMAL(18,6) NOT NULL COMMENT '最高价',
    low      DECIMAL(18,6) NOT NULL COMMENT '最低价',
    close    DECIMAL(18,6) NOT NULL COMMENT '收盘价',
    volume   DECIMAL(20,6) NOT NULL COMMENT '成交量',

    PRIMARY KEY (market, symbol, `interval`, ts),
    INDEX idx_kline_ts (ts),
    INDEX idx_kline_symbol (symbol),
    INDEX idx_kline_symbol_interval_ts
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
COMMENT='统一K线数据表（支持A股/加密货币/5分钟基础数据）';

CREATE TABLE kline (
    market   VARCHAR(10)   NOT NULL COMMENT '市场标识，如 CN / CRYPTO',
    symbol   VARCHAR(20)   NOT NULL COMMENT '证券代码 / 交易对',
    `interval` VARCHAR(10) NOT NULL COMMENT 'K线周期，如 5m / 30m / 240m',
    ts       DATETIME      NOT NULL COMMENT 'K线开始时间',

    open     DECIMAL(18,6) NOT NULL COMMENT '开盘价',
    high     DECIMAL(18,6) NOT NULL COMMENT '最高价',
    low      DECIMAL(18,6) NOT NULL COMMENT '最低价',
    close    DECIMAL(18,6) NOT NULL COMMENT '收盘价',
    volume   DECIMAL(20,6) NOT NULL COMMENT '成交量',

    
    macd_dif        DECIMAL(18,8) COMMENT 'MACD DIF（黄线）',
    macd_dea        DECIMAL(18,8) COMMENT 'MACD DEA（白线）',
    macd_hist       DECIMAL(18,8) COMMENT 'MACD 红绿柱（DIF-DEA）',
    macd_hist_area  DECIMAL(20,8) COMMENT 'MACD 红绿柱累计面积',

    
    boll_mid        DECIMAL(18,6) COMMENT '布林中轨',
    boll_upper      DECIMAL(18,6) COMMENT '布林上轨',
    boll_lower      DECIMAL(18,6) COMMENT '布林下轨',
    boll_width      DECIMAL(18,6) COMMENT '布林带宽度（upper-lower）',

    
    close_ma20      DECIMAL(18,6) COMMENT 'MA20（用于趋势判断）',

    
    trend_flag      TINYINT COMMENT '趋势标记：-1下跌 0震荡 1上涨',
    calc_version    VARCHAR(20) COMMENT '指标计算版本号',
    calc_time       DATETIME COMMENT '指标计算时间',

    PRIMARY KEY (market, symbol, `interval`, ts),

    INDEX idx_kline_ts (ts),
    INDEX idx_kline_symbol_interval_ts (symbol, `interval`, ts),
    INDEX idx_kline_trend (market, `interval`, trend_flag)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
COMMENT='多周期K线派生表（含MACD/BOLL指标与面积，用于规则计算）';

pg
CREATE TABLE kline_base (
    market   VARCHAR(10) NOT NULL,  -- 市场标识，如 CN(中国A股)、CRYPTO(加密货币)
    symbol   VARCHAR(20) NOT NULL,  -- 证券代码 / 交易对，如 600519、BTC/USDT
    ts       TIMESTAMPTZ NOT NULL,  -- K线时间（K线开始时间）
    open     NUMERIC(18,6) NOT NULL, 
    high     NUMERIC(18,6) NOT NULL,
    low      NUMERIC(18,6) NOT NULL,
    close    NUMERIC(18,6) NOT NULL,
    volume   NUMERIC(20,6) NOT NULL,
    interval VARCHAR(10) NOT NULL,   -- MySQL 主键里用的 interval

    PRIMARY KEY (market, symbol, interval, ts)
);

-- 索引
CREATE INDEX idx_kline_ts ON kline_base(ts);
CREATE INDEX idx_kline_symbol ON kline_base(symbol);
CREATE INDEX idx_kline_symbol_interval_ts ON kline_base(symbol, interval, ts);

COMMENT ON TABLE kline_base IS '统一K线数据表（支持A股/加密货币/5分钟基础数据）';

CREATE TABLE kline (
    market   VARCHAR(10) NOT NULL,   -- 市场标识，如 CN / CRYPTO
    symbol   VARCHAR(20) NOT NULL,   -- 证券代码 / 交易对
    interval VARCHAR(10) NOT NULL,   -- K线周期，如 5m / 30m / 240m
    ts       TIMESTAMPTZ NOT NULL,   -- K线开始时间

    open     NUMERIC(18,6) NOT NULL,
    high     NUMERIC(18,6) NOT NULL,
    low      NUMERIC(18,6) NOT NULL,
    close    NUMERIC(18,6) NOT NULL,
    volume   NUMERIC(20,6) NOT NULL,

    -- MACD
    macd_dif       NUMERIC(18,8),
    macd_dea       NUMERIC(18,8),
    macd_hist      NUMERIC(18,8),
    macd_hist_area NUMERIC(20,8),

    -- BOLL
    boll_mid   NUMERIC(18,6),
    boll_upper NUMERIC(18,6),
    boll_lower NUMERIC(18,6),
    boll_width NUMERIC(18,6),

    -- 辅助
    close_ma20   NUMERIC(18,6),
    trend_flag   SMALLINT,
    calc_version VARCHAR(20),
    calc_time    TIMESTAMPTZ,

    PRIMARY KEY (market, symbol, interval, ts)
);

-- 索引
CREATE INDEX idx_kline_ts ON kline(ts);
CREATE INDEX idx_kline_symbol_interval_ts ON kline(symbol, interval, ts);
CREATE INDEX idx_kline_trend ON kline(market, interval, trend_flag);

COMMENT ON TABLE kline IS '多周期K线派生表（含MACD/BOLL指标与面积，用于规则计算）';


CREATE TABLE signal (
    id BIGSERIAL PRIMARY KEY,

    -- ========== 基础定位 ==========
    market      VARCHAR(10)  NOT NULL COMMENT '市场：CN / CRYPTO',
    symbol      VARCHAR(20)  NOT NULL COMMENT '标的代码 / 交易对',
    interval    VARCHAR(10)  NOT NULL COMMENT '周期：5m / 30m / 240m',
    ts          TIMESTAMP    NOT NULL COMMENT '信号对应K线时间（K线开始时间）',

    -- ========== MACD 状态 ==========
    macd_above_zero   BOOLEAN COMMENT 'MACD 是否在 0 轴上方',
    macd_dif          DECIMAL(18,8) COMMENT 'DIF 当前值',
    macd_dea          DECIMAL(18,8) COMMENT 'DEA 当前值',
    macd_hist         DECIMAL(18,8) COMMENT '当前柱子值',
    macd_color        SMALLINT COMMENT '柱子颜色：1红 -1绿',

    -- ========== MACD 连续面积统计 ==========
    curr_red_area     DECIMAL(20,8) COMMENT '当前连续红柱累计面积',
    prev_green_area   DECIMAL(20,8) COMMENT '上一段连续绿柱累计面积',
    curr_green_area   DECIMAL(20,8) COMMENT '当前连续绿柱累计面积',
    prev_red_area     DECIMAL(20,8) COMMENT '上一段连续红柱累计面积',

    red_green_area_ratio DECIMAL(10,4) COMMENT '红/绿面积比（强弱指标）',

    -- ========== 趋势判断 ==========
    trend_flag        SMALLINT COMMENT '趋势：1上升 -1下降 0震荡',
    in_trend          BOOLEAN COMMENT '是否在趋势中产生信号',
    ma20              DECIMAL(18,6) COMMENT 'MA20',
    ma_trend_up       BOOLEAN COMMENT 'MA 是否向上',

    -- ========== 信号结果 ==========
    signal_flag       BOOLEAN NOT NULL COMMENT '是否触发信号',
    signal_type       VARCHAR(20) COMMENT '信号类型：BUY / SELL / WATCH',
    signal_score      DECIMAL(6,2) COMMENT '信号强度评分（用于排序）',

    -- ========== 规则 & 追溯 ==========
    rule_code         VARCHAR(50) COMMENT '规则编号',
    rule_version      VARCHAR(20) COMMENT '规则版本',
    calc_time         TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '计算时间',

    -- ========== BOLL 收缩（盘变前夕判断） ==========
    boll_width_pct        DECIMAL(10,6) COMMENT '布林带相对宽度 (upper-lower)/mid',
    boll_squeeze_flag     BOOLEAN COMMENT '是否处于布林收缩区（P20以内）',
    boll_extreme_flag     BOOLEAN COMMENT '是否极限收缩（P10以内）',
    boll_squeeze_level    SMALLINT COMMENT '收缩等级：0正常 1收缩 2极限收缩',
    boll_width_down_cnt   INT COMMENT '布林宽度连续下降K线数量',
    boll_in_trend         BOOLEAN COMMENT '是否趋势内收缩（高质量盘变）',


    -- ========== 唯一性 & 索引 ==========
    UNIQUE (market, symbol, interval, ts, rule_code)
);

signal检查出的信号放到新表里面，表怎么设计，里面包含某个标的，某个时间，某个周期，是否在MACD0轴以上，上升趋势中红柱子连续面积和上一个绿柱子连续面积相比，下降趋势中绿柱子连续面积和上一个红柱子连续面积相比等指标

趋势给方向，MACD 给动力，结构给确认，布林给时机
boll_width = upper - lower
boll_width_pct = (upper - lower) / mid
| 状态   | 条件                   |
| ---- | -------------------- |
| 正常   | boll_width_pct > P50 |
| 收缩   | boll_width_pct ≤ P20 |
| 极限收缩 | boll_width_pct ≤ P10 |
boll_width_pct 连续 N 根下降
防止假突破
✅ 「收缩 + 动能 + 趋势 + 结构确认」= 真突破概率大幅提升

只做「趋势内收缩」
✅ 只做
上升趋势中的回踩收缩
下降趋势中的反弹收缩

真突破前的 MACD 特征
多头
MACD > 0
AND curr_red_area > prev_green_area * 1.2
AND DIF 上行

空头
MACD < 0
AND curr_green_area > prev_red_area * 1.2
AND DIF 下行





datasource_base 定义抽象K线数据基类
a_stock_baostock 定义A股的K线数据源实现
crypto_ccxt 定义加密货币K线数据源的实现


strategy策略层，能从5分钟K线库里面，读取数据生成30分钟和4小时的K线数据，并计算对应的MACD和BOLL，按5分钟，30分钟，4小时分别保存的在数据库
周期和策略都是通过配置
indicators 负责计算MACD，BOLL等
整个数据流向 watch_symbol  →  kline_base  →  kline
signal 使用indicators的计算结果进行规则判断



notify 对符合规则的买卖点进行推送，telegram实现推送到telegram，email实现推送到email

执行engine的runner的时候，可以通过配置现在进行某个数据源的拉取，然后执行分析


strategy/
    indicators.py
    signal.py

data/
    datasource_base.py
    a_stock_akshare.py
    crypto_ccxt.py
notify/
    telegram.py
    email.py
engine/
    runner.py

watch_symbol
     │  (定期更新/启用)
     ▼
kline_base  (5m 原始K线抓取)
     │
     ▼
kline       (多周期派生 + MACD/BOLL计算)
     │
     ▼
strategy    (indicators.py → signal.py)
     │
     ▼
notify      (telegram / email)
