# NoSQL组实施指南 - MongoDB + Redis方案


## 📋 任务清单

- [ ] Task 1: 安装MongoDB和Redis（30分钟）
- [ ] Task 2: 设计文档结构（1小时）
- [ ] Task 3: 创建索引（30分钟）
- [ ] Task 4: 编写数据加载脚本（2小时）
- [ ] Task 5: 实现5个查询（2小时）
- [ ] Task 6: 配置Redis缓存（1小时）
- [ ] Task 7: 测试和文档（1小时）

---

## 🛠️ Task 1: 安装MongoDB和Redis

### 方法1：Docker Compose（推荐）

创建 `docker-compose.yml`:

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: token-analyzer-mongo
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: yourpassword
      MONGO_INITDB_DATABASE: token_analyzer
    volumes:
      - mongodb_data:/data/db

  redis:
    image: redis:7-alpine
    container_name: token-analyzer-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  mongodb_data:
  redis_data:
```

启动:

```bash
docker-compose up -d

# 验证
docker ps
docker exec -it token-analyzer-mongo mongosh -u admin -p yourpassword
docker exec -it token-analyzer-redis redis-cli ping
```

### 方法2：本地安装

```bash
# Ubuntu/Debian
sudo apt install mongodb redis-server

# macOS
brew install mongodb-community redis
brew services start mongodb-community
brew services start redis
```

---

## 📊 Task 2: 设计文档结构

NoSQL的核心优势是**内嵌文档**和**灵活schema**。

### 集合结构设计

#### 1. tokens（代币集合）

```javascript
{
  _id: ObjectId("..."),
  address: "0x...",  // 索引
  symbol: "ETH",
  name: "Ethereum",
  chain: "ETH",
  is_proxy: false,
  is_upgradeable: false,

  // 内嵌最新价格（提高查询速度）
  latest_price: {
    dex_price: 1800.50,
    cex_price: 1801.20,
    tvl: 1500000000,
    timestamp: ISODate("2025-01-09T...")
  },

  // 内嵌最新评分
  latest_score: {
    score: 85,
    factors: {
      liquidity_score: 90,
      holder_score: 75,
      dev_score: 88
    },
    timestamp: ISODate("2025-01-09T...")
  },

  // 内嵌项目信息
  project: {
    name: "Ethereum",
    github_repo: "https://github.com/ethereum/go-ethereum",
    stars: 45000,
    commit_count_7d: 50,
    last_commit_at: ISODate("2025-01-09T...")
  },

  created_at: ISODate("2025-01-01T..."),
  updated_at: ISODate("2025-01-09T...")
}
```

**设计理由**：
- 将常用的最新数据**内嵌**到主文档，避免JOIN
- 一次查询就能获取代币的全部关键信息

#### 2. dex_prices（DEX价格时序数据）

```javascript
{
  _id: ObjectId("..."),
  token_address: "0x...",  // 索引
  price: 1800.50,
  tvl: 1500000000,
  liquidity_depth: 5000000,
  volume_24h: 2000000,
  timestamp: ISODate("2025-01-09T..."),  // 索引

  // 可选：小时聚合数据（预聚合优化）
  hour: "2025-01-09T10:00:00Z"
}
```

#### 3. cex_prices（CEX价格数据）

```javascript
{
  _id: ObjectId("..."),
  token_symbol: "ETH",  // 索引
  exchange: "Binance",
  spot_price: 1801.20,
  funding_rate: 0.0001,
  volume_24h: 5000000,
  timestamp: ISODate("2025-01-09T...")  // 索引
}
```

#### 4. token_holders（持仓数据）

```javascript
{
  _id: ObjectId("..."),
  token_address: "0x...",  // 索引
  holders: [  // 内嵌数组（所有持仓者）
    {
      address: "0x...",
      balance: "1000000000000000000",
      percentage: 15.5,
      rank: 1
    },
    {
      address: "0x...",
      balance: "500000000000000000",
      percentage: 8.2,
      rank: 2
    }
    // ... Top 20
  ],
  snapshot_date: ISODate("2025-01-09"),
  top10_concentration: 65.8  // 预计算
}
```

**设计理由**：
- 将所有持仓者存为**数组**，一次查询获取全部数据
- 预计算 `top10_concentration`，避免实时聚合

#### 5. token_scores（评分历史）

```javascript
{
  _id: ObjectId("..."),
  token_address: "0x...",  // 索引
  score: 85,
  score_factors: {
    liquidity_score: 90,
    holder_score: 75,
    dev_score: 88
  },
  timestamp: ISODate("2025-01-09T...")  // 索引
}
```

#### 6. alerts（预警记录）

```javascript
{
  _id: ObjectId("..."),
  token_address: "0x...",  // 索引
  alert_type: "LIQUIDITY_DROP",
  severity: "HIGH",
  message: "TVL dropped by 25% in the last hour",
  timestamp: ISODate("2025-01-09T..."),  // 索引
  acknowledged: false
}
```

---

## 🔧 Task 3: 创建索引

创建 `nosql_indexes.js`:

```javascript
// 连接到数据库
use token_analyzer;

// 1. tokens集合索引
db.tokens.createIndex({ "address": 1 }, { unique: true });
db.tokens.createIndex({ "symbol": 1 });
db.tokens.createIndex({ "latest_score.score": -1 });
db.tokens.createIndex({ "is_proxy": 1, "is_upgradeable": 1 });

// 2. dex_prices索引（时序数据）
db.dex_prices.createIndex({ "token_address": 1, "timestamp": -1 });
db.dex_prices.createIndex({ "timestamp": -1 });
db.dex_prices.createIndex({ "hour": 1 });  // 用于小时聚合

// 3. cex_prices索引
db.cex_prices.createIndex({ "token_symbol": 1, "timestamp": -1 });
db.cex_prices.createIndex({ "exchange": 1, "timestamp": -1 });

// 4. token_holders索引
db.token_holders.createIndex({ "token_address": 1, "snapshot_date": -1 }, { unique: true });
db.token_holders.createIndex({ "snapshot_date": -1 });
db.token_holders.createIndex({ "holders.address": 1 });  // 多键索引

// 5. token_scores索引
db.token_scores.createIndex({ "token_address": 1, "timestamp": -1 });
db.token_scores.createIndex({ "score": -1, "timestamp": -1 });

// 6. alerts索引
db.alerts.createIndex({ "token_address": 1, "timestamp": -1 });
db.alerts.createIndex({ "severity": 1, "timestamp": -1 });
db.alerts.createIndex({ "acknowledged": 1, "timestamp": -1 });

print("✓ 所有索引创建完成");
```

执行索引创建:

```bash
docker exec -i token-analyzer-mongo mongosh -u admin -p yourpassword token_analyzer < nosql_indexes.js

# 验证索引
docker exec -it token-analyzer-mongo mongosh -u admin -p yourpassword
use token_analyzer
db.tokens.getIndexes()
```

---

## 🐍 Task 4: 数据加载脚本

创建 `nosql_data_loader.py`:

```python
from pymongo import MongoClient
import redis
from datetime import datetime, timedelta
import random
from faker import Faker
import json

fake = Faker()

# 连接MongoDB
mongo_client = MongoClient(
    "mongodb://admin:yourpassword@localhost:27017/",
    serverSelectionTimeoutMS=5000
)
db = mongo_client["token_analyzer"]

# 连接Redis
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

print("开始生成数据...")

# 1. 生成100个代币
print("生成tokens数据...")
tokens = []
token_addresses = []

for i in range(100):
    address = f"0x{fake.hexify(text='^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^')}"
    token_addresses.append(address)

    token = {
        "address": address,
        "symbol": fake.cryptocurrency_code(),
        "name": fake.cryptocurrency_name(),
        "chain": "ETH",
        "is_proxy": random.choice([True, False]),
        "is_upgradeable": random.choice([True, False]),
        "latest_price": {
            "dex_price": random.uniform(0.1, 10000),
            "tvl": random.uniform(100000, 10000000),
            "timestamp": datetime.now()
        },
        "latest_score": {
            "score": random.randint(30, 95),
            "factors": {
                "liquidity_score": random.randint(50, 100),
                "holder_score": random.randint(40, 90),
                "dev_score": random.randint(30, 80)
            },
            "timestamp": datetime.now()
        },
        "project": {
            "name": f"{fake.cryptocurrency_code()} Project",
            "github_repo": f"https://github.com/{fake.user_name()}/{fake.word()}",
            "stars": random.randint(10, 10000),
            "commit_count_7d": random.randint(0, 100),
            "last_commit_at": datetime.now() - timedelta(days=random.randint(0, 30))
        },
        "created_at": datetime.now(),
        "updated_at": datetime.now()
    }
    tokens.append(token)

result = db.tokens.insert_many(tokens)
print(f"✓ 插入了 {len(result.inserted_ids)} 个代币")

# 缓存到Redis（点查询优化）
print("缓存代币数据到Redis...")
for token in tokens:
    cache_key = f"token:{token['address']}:latest"
    cache_data = {
        "symbol": token["symbol"],
        "name": token["name"],
        "price": token["latest_price"]["dex_price"],
        "tvl": token["latest_price"]["tvl"],
        "score": token["latest_score"]["score"]
    }
    redis_client.setex(cache_key, 300, json.dumps(cache_data))  # 5分钟TTL

print("✓ Redis缓存已更新")

# 2. 生成DEX价格数据（每个代币1000条 = 10万条）
print("生成dex_prices数据（这会花几分钟）...")
batch_size = 1000
dex_prices = []

for address in token_addresses:
    base_price = random.uniform(0.1, 10000)
    for i in range(1000):
        timestamp = datetime.now() - timedelta(minutes=i)
        price = base_price * (1 + random.uniform(-0.05, 0.05))
        dex_prices.append({
            "token_address": address,
            "price": price,
            "tvl": random.uniform(100000, 10000000),
            "liquidity_depth": random.uniform(50000, 5000000),
            "volume_24h": random.uniform(10000, 1000000),
            "timestamp": timestamp,
            "hour": timestamp.replace(minute=0, second=0, microsecond=0)
        })

        # 批量插入
        if len(dex_prices) >= batch_size:
            db.dex_prices.insert_many(dex_prices)
            print(f"  已插入 {len(dex_prices)} 条DEX价格...")
            dex_prices = []

# 插入剩余数据
if dex_prices:
    db.dex_prices.insert_many(dex_prices)

print(f"✓ 插入了约 100,000 条DEX价格数据")

# 3. 生成CEX价格数据
print("生成cex_prices数据...")
exchanges = ['Binance', 'OKX', 'Coinbase', 'Bybit']
cex_prices = []

for token in tokens[:50]:  # 前50个代币
    symbol = token["symbol"]
    for i in range(500):
        timestamp = datetime.now() - timedelta(minutes=i*2)
        cex_prices.append({
            "token_symbol": symbol,
            "exchange": random.choice(exchanges),
            "spot_price": random.uniform(0.1, 10000),
            "funding_rate": random.uniform(-0.0001, 0.0001),
            "volume_24h": random.uniform(10000, 1000000),
            "timestamp": timestamp
        })

db.cex_prices.insert_many(cex_prices)
print(f"✓ 插入了 {len(cex_prices)} 条CEX价格数据")

# 4. 生成持仓数据
print("生成token_holders数据...")
holders_docs = []

for address in token_addresses:
    # 每个代币生成20个持仓者
    holders = []
    for rank in range(1, 21):
        holders.append({
            "address": f"0x{fake.hexify(text='^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^')}",
            "balance": str(random.randint(1000000, 100000000)),
            "percentage": random.uniform(0.5, 20.0),
            "rank": rank
        })

    # 计算Top 10集中度
    top10_concentration = sum(h["percentage"] for h in holders[:10])

    holders_docs.append({
        "token_address": address,
        "holders": holders,
        "top10_concentration": top10_concentration,
        "snapshot_date": datetime.now()
    })

db.token_holders.insert_many(holders_docs)
print(f"✓ 插入了 {len(holders_docs)} 个代币的持仓数据")

# 5. 生成评分数据
print("生成token_scores数据...")
scores = []

for address in token_addresses:
    for i in range(10):  # 每个代币10条历史评分
        timestamp = datetime.now() - timedelta(hours=i*24)
        scores.append({
            "token_address": address,
            "score": random.randint(30, 95),
            "score_factors": {
                "liquidity_score": random.randint(50, 100),
                "holder_score": random.randint(40, 90),
                "dev_score": random.randint(30, 80)
            },
            "timestamp": timestamp
        })

db.token_scores.insert_many(scores)
print(f"✓ 插入了 {len(scores)} 条评分数据")

# 6. 生成预警数据
print("生成alerts数据...")
alert_types = ['LIQUIDITY_DROP', 'PRICE_SPIKE', 'WHALE_MOVEMENT', 'RUG_PULL_RISK']
severities = ['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']
alerts = []

for address in random.sample(token_addresses, 30):
    for i in range(5):
        alerts.append({
            "token_address": address,
            "alert_type": random.choice(alert_types),
            "severity": random.choice(severities),
            "message": f"Alert: {fake.sentence()}",
            "timestamp": datetime.now() - timedelta(hours=i*6),
            "acknowledged": False
        })

db.alerts.insert_many(alerts)
print(f"✓ 插入了 {len(alerts)} 条预警")

print("\n✅ NoSQL数据加载完成！")
print(f"总计:")
print(f"  - {len(tokens)} 个代币 (MongoDB + Redis缓存)")
print(f"  - ~100,000 条DEX价格记录")
print(f"  - {len(cex_prices)} 条CEX价格记录")
print(f"  - {len(holders_docs)} 个代币的持仓记录")
print(f"  - {len(scores)} 条评分记录")
print(f"  - {len(alerts)} 条预警")

mongo_client.close()
```

安装依赖并运行:

```bash
pip install pymongo redis faker

python nosql_data_loader.py
```

---

## 📝 Task 5: 实现5个查询

创建 `nosql_queries.py`:

```python
from pymongo import MongoClient
import redis
import time
import json
from datetime import datetime, timedelta

# 连接
mongo_client = MongoClient("mongodb://admin:yourpassword@localhost:27017/")
db = mongo_client["token_analyzer"]
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def benchmark_query(name, query_func, iterations=100):
    """执行查询并测量性能"""
    # 预热
    query_func()

    # 正式测试
    latencies = []
    for _ in range(iterations):
        start = time.time()
        results = query_func()
        latency = (time.time() - start) * 1000  # 毫秒
        latencies.append(latency)

    avg_latency = sum(latencies) / len(latencies)
    p95_latency = sorted(latencies)[int(len(latencies) * 0.95)]

    print(f"\n{name}")
    print(f"  平均延迟: {avg_latency:.2f}ms")
    print(f"  P95延迟: {p95_latency:.2f}ms")
    print(f"  返回结果: {len(results) if isinstance(results, list) else 'N/A'}")

    return avg_latency, p95_latency

# 获取示例token地址
sample_token = db.tokens.find_one()["address"]

# ============================================================================
# Q1: 点查询 - 获取单个代币最新信息（使用Redis缓存）
# ============================================================================
def q1_query():
    # 先查Redis
    cache_key = f"token:{sample_token}:latest"
    cached = redis_client.get(cache_key)

    if cached:
        return json.loads(cached)

    # Redis未命中，查MongoDB
    result = db.tokens.find_one({"address": sample_token})

    # 更新缓存
    cache_data = {
        "symbol": result["symbol"],
        "name": result["name"],
        "price": result["latest_price"]["dex_price"],
        "tvl": result["latest_price"]["tvl"],
        "score": result["latest_score"]["score"]
    }
    redis_client.setex(cache_key, 300, json.dumps(cache_data))

    return cache_data

benchmark_query("Q1: 点查询 (单个代币信息 - Redis缓存)", q1_query)

# ============================================================================
# Q2: 范围查询 - 价格历史走势
# ============================================================================
def q2_query():
    start_time = datetime.now() - timedelta(hours=24)
    results = list(db.dex_prices.find({
        "token_address": sample_token,
        "timestamp": {"$gte": start_time}
    }).sort("timestamp", -1))
    return results

benchmark_query("Q2: 范围查询 (24小时价格走势)", q2_query)

# ============================================================================
# Q3: 聚合查询 - Top 10高评分代币
# ============================================================================
def q3_query():
    results = list(db.tokens.find().sort("latest_score.score", -1).limit(10))
    return results

benchmark_query("Q3: 聚合查询 (Top 10高评分代币)", q3_query)

# ============================================================================
# Q4: 复杂查询 - 持仓集中度分析（利用预计算字段）
# ============================================================================
def q4_query():
    # 利用内嵌的top10_concentration字段
    pipeline = [
        {"$match": {"top10_concentration": {"$gt": 30}}},
        {"$lookup": {
            "from": "tokens",
            "localField": "token_address",
            "foreignField": "address",
            "as": "token_info"
        }},
        {"$unwind": "$token_info"},
        {"$project": {
            "symbol": "$token_info.symbol",
            "top10_concentration": 1
        }},
        {"$sort": {"top10_concentration": -1}}
    ]
    results = list(db.token_holders.aggregate(pipeline))
    return results

benchmark_query("Q4: 复杂查询 (持仓集中度分析)", q4_query, iterations=50)

# ============================================================================
# Q5: 写入测试 - 批量插入价格数据
# ============================================================================
print("\nQ5: 批量写入测试 (插入1000条价格记录)")

test_data = [{
    "token_address": sample_token,
    "price": 100.0 + i*0.1,
    "tvl": 1000000.0,
    "liquidity_depth": 500000.0,
    "volume_24h": 100000.0,
    "timestamp": datetime.now(),
    "hour": datetime.now().replace(minute=0, second=0, microsecond=0)
} for i in range(1000)]

start = time.time()
db.dex_prices.insert_many(test_data)
write_time = (time.time() - start) * 1000

print(f"  批量插入1000条: {write_time:.2f}ms")
print(f"  平均每条: {write_time/1000:.2f}ms")

mongo_client.close()

print("\n✅ NoSQL查询测试完成！")
```

运行测试:

```bash
python nosql_queries.py
```

---

## 🚀 Task 6: Redis缓存策略

### 缓存热点数据

```python
# 缓存策略示例
import redis
import json

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# 1. 缓存最新价格（5分钟TTL）
def cache_latest_price(token_address, price_data):
    key = f"token:{token_address}:latest_price"
    redis_client.setex(key, 300, json.dumps(price_data))

# 2. 缓存Top 10代币（10分钟TTL）
def cache_top_tokens(top_tokens):
    key = "tokens:top10"
    redis_client.setex(key, 600, json.dumps(top_tokens))

# 3. 缓存代币评分（1小时TTL）
def cache_token_score(token_address, score_data):
    key = f"token:{token_address}:score"
    redis_client.setex(key, 3600, json.dumps(score_data))
```

### 缓存失效策略

```python
# 当数据更新时，清除相关缓存
def invalidate_cache(token_address):
    redis_client.delete(f"token:{token_address}:latest_price")
    redis_client.delete(f"token:{token_address}:score")
    redis_client.delete("tokens:top10")
```

---

## 📄 Task 7: 文档和交付

创建 `NOSQL_README.md`:

```markdown
# NoSQL方案实现说明

## 数据库配置
- **文档数据库**: MongoDB 7.0
- **缓存**: Redis 7.0
- **数据规模**:
  - 100个代币
  - 10万条价格记录
  - 2000条持仓记录（内嵌文档）

## Schema设计要点
1. **内嵌文档**: 将最新价格和评分内嵌到代币主文档
2. **数组存储**: 持仓者存为数组，避免JOIN
3. **预计算字段**: top10_concentration预先计算
4. **Redis缓存**: 热点数据5分钟TTL

## 性能优势
- 点查询通过Redis缓存，延迟<5ms
- 内嵌文档避免JOIN，减少网络往返
- 批量写入性能优于关系型数据库

## 性能测试结果

| 查询 | 平均延迟 | P95延迟 |
|------|---------|---------|
| Q1 点查询 (Redis) | XX ms | XX ms |
| Q2 范围查询 | XX ms | XX ms |
| Q3 聚合查询 | XX ms | XX ms |
| Q4 复杂查询 | XX ms | XX ms |
| Q5 批量写入 | XX ms/1000条 | - |

## 文件清单
- `nosql_indexes.js` - 索引创建脚本
- `nosql_data_loader.py` - 数据加载脚本
- `nosql_queries.py` - 查询测试脚本
- `docker-compose.yml` - Docker部署配置
```

---

## ✅ 交付检查清单

完成后确认：

- [ ] MongoDB和Redis已安装并运行
- [ ] 6个集合创建成功并创建索引
- [ ] 数据加载成功（100个代币，10万+条记录）
- [ ] Redis缓存正常工作
- [ ] 5个查询都能正常执行
- [ ] 记录了性能数据（延迟、吞吐量）
- [ ] 文档完整

---

## 🆘 常见问题

### Q: Docker容器启动失败？
A: 检查端口占用，使用 `docker-compose logs` 查看日志

### Q: MongoDB连接超时？
A: 检查认证信息，确认容器正在运行

### Q: Redis缓存不生效？
A: 检查TTL设置，使用 `redis-cli` 验证数据

### Q: 查询太慢？
A: 检查索引是否创建成功：`db.collection.getIndexes()`

---

**完成时间**: Day 1-2，共8小时
**下一步**: 将性能数据交给实验组用于对比分析
