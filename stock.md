我想做一个交易系统
分两个层，一个数据源层，一个策略层
数据源层
1、将最新的沪深300数据写入监控股票池，每半年更新一次股票池里面股票
2、将加密货币市值前10的货币写入监控池，每月更新一次需要监控的十个加密货币

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
