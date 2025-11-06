# SQL组实施指南 - PostgreSQL方案



## 📋 任务清单

- [ ] Task 1: 安装PostgreSQL（30分钟）
- [ ] Task 2: 创建简化的数据库Schema（1小时）
- [ ] Task 3: 创建索引和视图（30分钟）
- [ ] Task 4: 编写数据加载脚本（2小时）
- [ ] Task 5: 实现5个查询（2小时）
- [ ] Task 6: 性能优化（1小时）
- [ ] Task 7: 测试和文档（1小时）

---

## 🛠️ Task 1: 安装PostgreSQL

### 方法1：Docker（推荐）

```bash
# 拉取PostgreSQL镜像
docker pull postgres:15

# 启动容器
docker run -d \
  --name token-analyzer-postgres \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=token_analyzer \
  -p 5432:5432 \
  postgres:15

# 验证安装
docker exec -it token-analyzer-postgres psql -U postgres -d token_analyzer -c "SELECT version();"
```

### 方法2：本地安装

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib

# macOS
brew install postgresql@15
brew services start postgresql@15

# 创建数据库
createdb token_analyzer
```

---

## 📊 Task 2: 创建简化Schema

保存为 `sql_schema_simplified.sql`:

```sql
-- ============================================================================
-- 区块链代币分析系统 - 简化Schema (课程项目版本)
-- 数据库: PostgreSQL 14+
-- 实体数量: 7个核心表
-- ============================================================================

DROP TABLE IF EXISTS ALERT CASCADE;
DROP TABLE IF EXISTS PROJECT CASCADE;
DROP TABLE IF EXISTS TOKEN_SCORE CASCADE;
DROP TABLE IF EXISTS TOKEN_HOLDER CASCADE;
DROP TABLE IF EXISTS CEX_PRICE CASCADE;
DROP TABLE IF EXISTS DEX_PRICE CASCADE;
DROP TABLE IF EXISTS TOKEN CASCADE;

-- ============================================================================
-- 1. TOKEN（代币基本信息）
-- ============================================================================
CREATE TABLE TOKEN (
    token_address VARCHAR(42) PRIMARY KEY,
    symbol VARCHAR(20) NOT NULL,
    name VARCHAR(100) NOT NULL,
    chain VARCHAR(20) NOT NULL DEFAULT 'ETH',
    is_proxy BOOLEAN DEFAULT false,
    is_upgradeable BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_token_symbol ON TOKEN(symbol);
CREATE INDEX idx_token_risk ON TOKEN(is_proxy, is_upgradeable)
WHERE is_proxy = true OR is_upgradeable = true;

COMMENT ON TABLE TOKEN IS '代币基本信息表';
COMMENT ON COLUMN TOKEN.is_proxy IS '代理合约（高风险）';
COMMENT ON COLUMN TOKEN.is_upgradeable IS '可升级合约（高风险）';

-- ============================================================================
-- 2. DEX_PRICE（DEX价格快照 - 时序数据）
-- ============================================================================
CREATE TABLE DEX_PRICE (
    price_id BIGSERIAL,
    token_address VARCHAR(42) NOT NULL,
    price DECIMAL(36, 18) NOT NULL,
    tvl DECIMAL(36, 2),
    liquidity_depth DECIMAL(36, 2),
    volume_24h DECIMAL(36, 2),
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (price_id, timestamp),
    CONSTRAINT fk_dex_token FOREIGN KEY (token_address) REFERENCES TOKEN(token_address)
) PARTITION BY RANGE (timestamp);

-- 创建分区（近3个月）
CREATE TABLE DEX_PRICE_2025_01 PARTITION OF DEX_PRICE
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE DEX_PRICE_2025_02 PARTITION OF DEX_PRICE
FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE TABLE DEX_PRICE_2025_03 PARTITION OF DEX_PRICE
FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

-- 索引
CREATE INDEX idx_dex_token_time ON DEX_PRICE(token_address, timestamp DESC);
CREATE INDEX idx_dex_time ON DEX_PRICE(timestamp DESC);

COMMENT ON TABLE DEX_PRICE IS 'DEX价格时序数据（按月分区）';

-- ============================================================================
-- 3. CEX_PRICE（CEX市场数据 - 时序数据）
-- ============================================================================
CREATE TABLE CEX_PRICE (
    price_id BIGSERIAL,
    token_symbol VARCHAR(20) NOT NULL,
    exchange VARCHAR(50) NOT NULL,
    spot_price DECIMAL(36, 18),
    funding_rate DECIMAL(10, 8),
    volume_24h DECIMAL(36, 2),
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (price_id, timestamp)
) PARTITION BY RANGE (timestamp);

-- 创建分区
CREATE TABLE CEX_PRICE_2025_01 PARTITION OF CEX_PRICE
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE CEX_PRICE_2025_02 PARTITION OF CEX_PRICE
FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE TABLE CEX_PRICE_2025_03 PARTITION OF CEX_PRICE
FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

-- 索引
CREATE INDEX idx_cex_symbol_time ON CEX_PRICE(token_symbol, timestamp DESC);
CREATE INDEX idx_cex_exchange ON CEX_PRICE(exchange);

COMMENT ON TABLE CEX_PRICE IS 'CEX市场数据（按月分区）';

-- ============================================================================
-- 4. TOKEN_HOLDER（持仓分布）
-- ============================================================================
CREATE TABLE TOKEN_HOLDER (
    holder_id BIGSERIAL PRIMARY KEY,
    token_address VARCHAR(42) NOT NULL,
    holder_address VARCHAR(42) NOT NULL,
    balance DECIMAL(78, 0) NOT NULL,
    percentage DECIMAL(10, 6) NOT NULL,
    rank INT NOT NULL,
    snapshot_date DATE NOT NULL DEFAULT CURRENT_DATE,

    CONSTRAINT fk_holder_token FOREIGN KEY (token_address) REFERENCES TOKEN(token_address),
    CONSTRAINT uq_holder_snapshot UNIQUE (token_address, holder_address, snapshot_date)
);

CREATE INDEX idx_holder_token_date ON TOKEN_HOLDER(token_address, snapshot_date DESC);
CREATE INDEX idx_holder_rank ON TOKEN_HOLDER(token_address, rank) WHERE rank <= 10;

COMMENT ON TABLE TOKEN_HOLDER IS '代币持仓分布快照';

-- ============================================================================
-- 5. TOKEN_SCORE（代币评分）
-- ============================================================================
CREATE TABLE TOKEN_SCORE (
    score_id BIGSERIAL PRIMARY KEY,
    token_address VARCHAR(42) NOT NULL,
    score INT NOT NULL CHECK (score >= 0 AND score <= 100),
    score_factors JSONB,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_score_token FOREIGN KEY (token_address) REFERENCES TOKEN(token_address)
);

CREATE INDEX idx_score_token_time ON TOKEN_SCORE(token_address, timestamp DESC);
CREATE INDEX idx_score_value ON TOKEN_SCORE(score DESC);

COMMENT ON TABLE TOKEN_SCORE IS '代币综合评分';

-- ============================================================================
-- 6. PROJECT（项目信息）
-- ============================================================================
CREATE TABLE PROJECT (
    project_id SERIAL PRIMARY KEY,
    token_address VARCHAR(42) NOT NULL UNIQUE,
    project_name VARCHAR(100) NOT NULL,
    github_repo VARCHAR(200),
    github_stars INT DEFAULT 0,
    commit_count_7d INT DEFAULT 0,
    last_commit_at TIMESTAMP,

    CONSTRAINT fk_project_token FOREIGN KEY (token_address) REFERENCES TOKEN(token_address)
);

CREATE INDEX idx_project_token ON PROJECT(token_address);
CREATE INDEX idx_project_stars ON PROJECT(github_stars DESC);

COMMENT ON TABLE PROJECT IS '项目基本信息和GitHub活跃度';

-- ============================================================================
-- 7. ALERT（预警记录）
-- ============================================================================
CREATE TABLE ALERT (
    alert_id BIGSERIAL PRIMARY KEY,
    token_address VARCHAR(42) NOT NULL,
    alert_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    message TEXT,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_alert_token FOREIGN KEY (token_address) REFERENCES TOKEN(token_address)
);

CREATE INDEX idx_alert_token_time ON ALERT(token_address, timestamp DESC);
CREATE INDEX idx_alert_severity ON ALERT(severity, timestamp DESC) WHERE severity IN ('HIGH', 'CRITICAL');

COMMENT ON TABLE ALERT IS '风险预警记录';

-- ============================================================================
-- 视图: 代币仪表盘
-- ============================================================================
CREATE OR REPLACE VIEW v_token_dashboard AS
SELECT
    t.token_address,
    t.symbol,
    t.name,
    t.is_proxy,
    t.is_upgradeable,
    d.price as latest_dex_price,
    d.tvl,
    d.volume_24h,
    s.score as latest_score,
    p.github_stars,
    p.commit_count_7d
FROM TOKEN t
LEFT JOIN LATERAL (
    SELECT price, tvl, volume_24h
    FROM DEX_PRICE
    WHERE token_address = t.token_address
    ORDER BY timestamp DESC
    LIMIT 1
) d ON true
LEFT JOIN LATERAL (
    SELECT score
    FROM TOKEN_SCORE
    WHERE token_address = t.token_address
    ORDER BY timestamp DESC
    LIMIT 1
) s ON true
LEFT JOIN PROJECT p ON t.token_address = p.token_address;

COMMENT ON VIEW v_token_dashboard IS '代币综合仪表盘视图';

-- ============================================================================
-- 初始化示例数据
-- ============================================================================
INSERT INTO TOKEN (token_address, symbol, name, is_proxy, is_upgradeable)
VALUES
    ('0x0000000000000000000000000000000000000001', 'ETH', 'Ethereum', false, false),
    ('0xdAC17F958D2ee523a2206206994597C13D831ec7', 'USDT', 'Tether USD', true, false),
    ('0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', 'USDC', 'USD Coin', true, true)
ON CONFLICT (token_address) DO NOTHING;

-- ============================================================================
-- 完成
-- ============================================================================
SELECT 'Schema创建完成！共7个表' as message;
```

执行Schema:

```bash
# 使用Docker
docker exec -i token-analyzer-postgres psql -U postgres -d token_analyzer < sql_schema_simplified.sql

# 本地安装
psql -U postgres -d token_analyzer -f sql_schema_simplified.sql
```

---

## 🐍 Task 4: 数据加载脚本

创建 `sql_data_loader.py`:

```python
import psycopg2
from psycopg2.extras import execute_batch
from datetime import datetime, timedelta
import random
from faker import Faker

fake = Faker()

# 数据库连接
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    database="token_analyzer",
    user="postgres",
    password="yourpassword"
)
cur = conn.cursor()

print("开始生成数据...")

# 1. 生成100个代币
print("生成TOKEN数据...")
tokens = []
for i in range(100):
    tokens.append((
        f"0x{fake.hexify(text='^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^')}",
        fake.cryptocurrency_code(),
        fake.cryptocurrency_name(),
        'ETH',
        random.choice([True, False]),
        random.choice([True, False])
    ))

execute_batch(cur, """
    INSERT INTO TOKEN (token_address, symbol, name, chain, is_proxy, is_upgradeable)
    VALUES (%s, %s, %s, %s, %s, %s)
    ON CONFLICT (token_address) DO NOTHING
""", tokens)

print(f"✓ 插入了 {len(tokens)} 个代币")

# 获取所有token_address
cur.execute("SELECT token_address, symbol FROM TOKEN")
token_list = cur.fetchall()

# 2. 生成DEX价格数据（每个代币1000条记录 = 10万条）
print("生成DEX_PRICE数据（这会花几分钟）...")
dex_prices = []
for token_address, symbol in token_list:
    base_price = random.uniform(0.1, 10000)
    for i in range(1000):
        timestamp = datetime.now() - timedelta(minutes=i)
        price = base_price * (1 + random.uniform(-0.05, 0.05))
        dex_prices.append((
            token_address,
            price,
            random.uniform(100000, 10000000),  # TVL
            random.uniform(50000, 5000000),    # liquidity_depth
            random.uniform(10000, 1000000),    # volume_24h
            timestamp
        ))

# 批量插入（每次1000条）
batch_size = 1000
for i in range(0, len(dex_prices), batch_size):
    batch = dex_prices[i:i+batch_size]
    execute_batch(cur, """
        INSERT INTO DEX_PRICE (token_address, price, tvl, liquidity_depth, volume_24h, timestamp)
        VALUES (%s, %s, %s, %s, %s, %s)
    """, batch)
    if (i // batch_size) % 10 == 0:
        print(f"  已插入 {i + len(batch)} / {len(dex_prices)} 条DEX价格...")

print(f"✓ 插入了 {len(dex_prices)} 条DEX价格数据")

# 3. 生成CEX价格数据
print("生成CEX_PRICE数据...")
exchanges = ['Binance', 'OKX', 'Coinbase', 'Bybit']
cex_prices = []
for token_address, symbol in token_list[:50]:  # 只为前50个代币生成CEX数据
    for i in range(500):
        timestamp = datetime.now() - timedelta(minutes=i*2)
        cex_prices.append((
            symbol,
            random.choice(exchanges),
            random.uniform(0.1, 10000),
            random.uniform(-0.0001, 0.0001),
            random.uniform(10000, 1000000),
            timestamp
        ))

execute_batch(cur, """
    INSERT INTO CEX_PRICE (token_symbol, exchange, spot_price, funding_rate, volume_24h, timestamp)
    VALUES (%s, %s, %s, %s, %s, %s)
""", cex_prices, page_size=1000)

print(f"✓ 插入了 {len(cex_prices)} 条CEX价格数据")

# 4. 生成持仓数据
print("生成TOKEN_HOLDER数据...")
holders = []
for token_address, symbol in token_list:
    # 每个代币生成20个持仓地址
    for rank in range(1, 21):
        holders.append((
            token_address,
            f"0x{fake.hexify(text='^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^')}",
            random.randint(1000000, 100000000),
            random.uniform(0.5, 20.0),
            rank,
            datetime.now().date()
        ))

execute_batch(cur, """
    INSERT INTO TOKEN_HOLDER (token_address, holder_address, balance, percentage, rank, snapshot_date)
    VALUES (%s, %s, %s, %s, %s, %s)
    ON CONFLICT DO NOTHING
""", holders, page_size=1000)

print(f"✓ 插入了 {len(holders)} 条持仓数据")

# 5. 生成评分数据
print("生成TOKEN_SCORE数据...")
scores = []
for token_address, symbol in token_list:
    for i in range(10):
        timestamp = datetime.now() - timedelta(hours=i*24)
        scores.append((
            token_address,
            random.randint(30, 95),
            {
                "liquidity_score": random.randint(50, 100),
                "holder_score": random.randint(40, 90),
                "dev_score": random.randint(30, 80)
            },
            timestamp
        ))

execute_batch(cur, """
    INSERT INTO TOKEN_SCORE (token_address, score, score_factors, timestamp)
    VALUES (%s, %s, %s::jsonb, %s)
""", [(t, s, str(f).replace("'", '"'), ts) for t, s, f, ts in scores])

print(f"✓ 插入了 {len(scores)} 条评分数据")

# 6. 生成项目数据
print("生成PROJECT数据...")
projects = []
for token_address, symbol in token_list[:80]:  # 80%的代币有项目信息
    projects.append((
        token_address,
        f"{symbol} Project",
        f"https://github.com/{fake.user_name()}/{symbol.lower()}",
        random.randint(10, 10000),
        random.randint(0, 100),
        datetime.now() - timedelta(days=random.randint(0, 30))
    ))

execute_batch(cur, """
    INSERT INTO PROJECT (token_address, project_name, github_repo, github_stars, commit_count_7d, last_commit_at)
    VALUES (%s, %s, %s, %s, %s, %s)
    ON CONFLICT (token_address) DO NOTHING
""", projects)

print(f"✓ 插入了 {len(projects)} 个项目")

# 7. 生成预警数据
print("生成ALERT数据...")
alert_types = ['LIQUIDITY_DROP', 'PRICE_SPIKE', 'WHALE_MOVEMENT', 'RUG_PULL_RISK']
severities = ['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']
alerts = []
for token_address, symbol in random.sample(token_list, 30):
    for i in range(5):
        alerts.append((
            token_address,
            random.choice(alert_types),
            random.choice(severities),
            f"Alert for {symbol}: {fake.sentence()}",
            datetime.now() - timedelta(hours=i*6)
        ))

execute_batch(cur, """
    INSERT INTO ALERT (token_address, alert_type, severity, message, timestamp)
    VALUES (%s, %s, %s, %s, %s)
""", alerts)

print(f"✓ 插入了 {len(alerts)} 条预警")

conn.commit()
cur.close()
conn.close()

print("\n✅ 数据加载完成！")
print(f"总计:")
print(f"  - {len(tokens)} 个代币")
print(f"  - {len(dex_prices)} 条DEX价格记录")
print(f"  - {len(cex_prices)} 条CEX价格记录")
print(f"  - {len(holders)} 条持仓记录")
print(f"  - {len(scores)} 条评分记录")
print(f"  - {len(projects)} 个项目")
print(f"  - {len(alerts)} 条预警")
```

安装依赖并运行:

```bash
pip install psycopg2-binary faker

python sql_data_loader.py
```

---

## 📝 Task 5: 实现5个查询

创建 `sql_queries.py`:

```python
import psycopg2
import time

conn = psycopg2.connect(
    host="localhost",
    port=5432,
    database="token_analyzer",
    user="postgres",
    password="yourpassword"
)

def benchmark_query(name, query, params=None, iterations=100):
    """执行查询并测量性能"""
    cur = conn.cursor()

    # 预热
    cur.execute(query, params)
    cur.fetchall()

    # 正式测试
    latencies = []
    for _ in range(iterations):
        start = time.time()
        cur.execute(query, params)
        results = cur.fetchall()
        latency = (time.time() - start) * 1000  # 转换为毫秒
        latencies.append(latency)

    cur.close()

    avg_latency = sum(latencies) / len(latencies)
    p95_latency = sorted(latencies)[int(len(latencies) * 0.95)]

    print(f"\n{name}")
    print(f"  平均延迟: {avg_latency:.2f}ms")
    print(f"  P95延迟: {p95_latency:.2f}ms")
    print(f"  返回行数: {len(results)}")

    return avg_latency, p95_latency

# ============================================================================
# Q1: 点查询 - 获取单个代币最新信息
# ============================================================================
q1_query = """
SELECT
    t.symbol,
    t.name,
    d.price as latest_price,
    d.tvl,
    s.score
FROM TOKEN t
LEFT JOIN LATERAL (
    SELECT price, tvl
    FROM DEX_PRICE
    WHERE token_address = t.token_address
    ORDER BY timestamp DESC
    LIMIT 1
) d ON true
LEFT JOIN LATERAL (
    SELECT score
    FROM TOKEN_SCORE
    WHERE token_address = t.token_address
    ORDER BY timestamp DESC
    LIMIT 1
) s ON true
WHERE t.token_address = %s;
"""

# 获取一个示例token地址
cur = conn.cursor()
cur.execute("SELECT token_address FROM TOKEN LIMIT 1")
sample_token = cur.fetchone()[0]
cur.close()

benchmark_query("Q1: 点查询 (单个代币信息)", q1_query, (sample_token,))

# ============================================================================
# Q2: 范围查询 - 价格历史走势
# ============================================================================
q2_query = """
SELECT timestamp, price, tvl
FROM DEX_PRICE
WHERE token_address = %s
  AND timestamp > NOW() - INTERVAL '24 hours'
ORDER BY timestamp DESC;
"""

benchmark_query("Q2: 范围查询 (24小时价格走势)", q2_query, (sample_token,))

# ============================================================================
# Q3: 聚合查询 - Top 10高评分代币
# ============================================================================
q3_query = """
SELECT
    t.symbol,
    s.score,
    d.tvl
FROM TOKEN t
JOIN LATERAL (
    SELECT score
    FROM TOKEN_SCORE
    WHERE token_address = t.token_address
    ORDER BY timestamp DESC
    LIMIT 1
) s ON true
LEFT JOIN LATERAL (
    SELECT tvl
    FROM DEX_PRICE
    WHERE token_address = t.token_address
    ORDER BY timestamp DESC
    LIMIT 1
) d ON true
ORDER BY s.score DESC
LIMIT 10;
"""

benchmark_query("Q3: 聚合查询 (Top 10高评分代币)", q3_query)

# ============================================================================
# Q4: 复杂JOIN - 持仓集中度分析
# ============================================================================
q4_query = """
SELECT
    t.symbol,
    SUM(CASE WHEN h.rank <= 10 THEN h.percentage ELSE 0 END) as top10_concentration
FROM TOKEN t
JOIN TOKEN_HOLDER h ON t.token_address = h.token_address
WHERE h.snapshot_date = CURRENT_DATE
GROUP BY t.symbol, t.token_address
HAVING SUM(CASE WHEN h.rank <= 10 THEN h.percentage ELSE 0 END) > 30
ORDER BY top10_concentration DESC;
"""

benchmark_query("Q4: 复杂JOIN (持仓集中度分析)", q4_query, iterations=50)

# ============================================================================
# Q5: 写入测试 - 批量插入价格数据
# ============================================================================
print("\nQ5: 批量写入测试 (插入1000条价格记录)")

cur = conn.cursor()

# 准备数据
from datetime import datetime
test_data = [(sample_token, 100.0 + i*0.1, 1000000.0, 500000.0, 100000.0, datetime.now())
             for i in range(1000)]

start = time.time()
cur.executemany("""
    INSERT INTO DEX_PRICE (token_address, price, tvl, liquidity_depth, volume_24h, timestamp)
    VALUES (%s, %s, %s, %s, %s, %s)
""", test_data)
conn.commit()
write_time = (time.time() - start) * 1000

cur.close()

print(f"  批量插入1000条: {write_time:.2f}ms")
print(f"  平均每条: {write_time/1000:.2f}ms")

conn.close()

print("\n✅ SQL查询测试完成！")
```

运行测试:

```bash
python sql_queries.py
```

---

## 📊 Task 6: 性能优化

### 检查和创建缺失的索引

```sql
-- 检查慢查询
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 添加额外的复合索引（如果需要）
CREATE INDEX idx_dex_price_token_ts_price ON DEX_PRICE(token_address, timestamp DESC, price);

-- 分析表统计信息
ANALYZE TOKEN;
ANALYZE DEX_PRICE;
ANALYZE TOKEN_SCORE;
```

### EXPLAIN分析

```sql
-- 分析查询计划
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...（你的查询）;
```

---

## 📄 Task 7: 文档和交付

创建 `SQL_README.md`:

```markdown
# SQL方案实现说明

## 数据库配置
- **数据库**: PostgreSQL 15
- **数据规模**:
  - 100个代币
  - 10万条价格记录
  - 2000条持仓记录
- **分区策略**: 按月分区（DEX_PRICE, CEX_PRICE）

## Schema设计要点
1. **时序数据分区**: 价格表按月分区，提高查询性能
2. **索引优化**: 为常用查询创建复合索引
3. **LATERAL JOIN**: 使用PostgreSQL特性优化子查询

## 性能测试结果

| 查询 | 平均延迟 | P95延迟 |
|------|---------|---------|
| Q1 点查询 | XX ms | XX ms |
| Q2 范围查询 | XX ms | XX ms |
| Q3 聚合查询 | XX ms | XX ms |
| Q4 复杂JOIN | XX ms | XX ms |
| Q5 批量写入 | XX ms/1000条 | - |

## 文件清单
- `sql_schema_simplified.sql` - 建表脚本
- `sql_data_loader.py` - 数据加载脚本
- `sql_queries.py` - 查询测试脚本
```

---

## ✅ 交付检查清单

完成后确认：

- [ ] PostgreSQL已安装并运行
- [ ] 7个表创建成功
- [ ] 数据加载成功（100个代币，10万+条记录）
- [ ] 5个查询都能正常执行
- [ ] 记录了性能数据（延迟、吞吐量）
- [ ] 文档完整

---

## 🆘 常见问题

### Q: psycopg2安装失败？
A: 使用二进制版本：`pip install psycopg2-binary`

### Q: 分区表插入失败？
A: 确保timestamp在分区范围内，或创建更多分区

### Q: 查询太慢？
A: 运行 `ANALYZE` 更新统计信息，检查索引是否生效

### Q: 连接数据库失败？
A: 检查端口、密码，确认Docker容器正在运行

---

**完成时间**: Day 1-2，共8小时
**下一步**: 将性能数据交给实验组用于对比分析
