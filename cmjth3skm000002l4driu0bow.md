---
title: "Monitoring - Giám sát Database"
datePublished: Wed Dec 31 2025 03:45:24 GMT+0000 (Coordinated Universal Time)
cuid: cmjth3skm000002l4driu0bow
slug: monitoring-giam-sat-database
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1767152650359/3bcd44ae-1082-4b75-bc24-6c9325bf326e.png
tags: devops

---

## **Mục đích của Database Test Environment**

### **Tại sao cần môi trường test riêng cho database monitoring?**

Trong production, việc enable logging chi tiết và profiling có thể ảnh hưởng performance. Môi trường test này được thiết kế để:

1. **Thử nghiệm cấu hình logging** trước khi áp dụng lên production
    
2. **Validate log parsing** trong Promtail pipeline
    
3. **Test alert rules** cho database metrics
    
4. **Học cách analyze** database logs và metrics
    
5. **Benchmark performance impact** của logging/profiling
    

### **Điểm khác biệt với Production**

| **Aspect** | **Test Environment** | **Production** |
| --- | --- | --- |
| **Logging Level** | ALL statements | Chỉ slow queries (&gt;100ms) |
| **Profiling** | 100% operations | Sampling (1-10%) |
| **Log Retention** | 7 days | 30+ days |
| **Ports** | Internal only | Có thể expose |
| **Resources** | Minimal | Optimized |

## **Kiến trúc Database Monitoring**

```mermaid
graph TD
    %% Databases
    subgraph DBENV["Database Test Environment (10.99.3.67)"]
        subgraph PG["PostgreSQL 16<br/>:5432 (internal)"]
            PGLOG[Logging<br/>• All statements<br/>• Connections<br/>• Slow queries<br/>• Checkpoints<br/>• Lock waits<br/>Logs: /pg_log/*.log]
        end

        subgraph MG["MongoDB 7<br/>:27017 (internal)"]
            MGLOG[Logging<br/>• Verbose<br/>• All operations<br/>• Profiling 100%<br/>• Auth events<br/>• Slow queries<br/>Logs: /mongod.log]
        end

        %% Exporters
        PG -->|SQL queries| PGEXP[Postgres Exporter<br/>:9187<br/>• Connections<br/>• Query stats<br/>• Table sizes<br/>• Replication lag<br/>• Deadlocks]

        MG -->|MongoDB commands| MGEXP[MongoDB Exporter<br/>:9216<br/>• Connections<br/>• Operations<br/>• Latency<br/>• Cache hit ratio<br/>• Replication lag]

        %% Promtail
        subgraph PROMTAIL["Promtail (Log Collector)"]
            PTJOB1[PostgreSQL logs<br/>• Multiline SQL<br/>• user, db, app, client, level]
            PTJOB2[MongoDB logs<br/>• JSON parse<br/>• severity, component, message]
        end

        PGLOG --> PROMTAIL
        MGLOG --> PROMTAIL
    end

    %% External systems
    PROM[(Prometheus<br/>External<br/>:9090)]
    LOKI[(Loki<br/>External<br/>:3100)]

    PGEXP -->|HTTP /metrics| PROM
    MGEXP -->|HTTP /metrics| PROM

    PROMTAIL -->|HTTP Push<br/>/loki/api/v1/push| LOKI

```

## **PostgreSQL Test Database**

### **Cấu hình Logging Chi tiết**

PostgreSQL trong môi trường này được cấu hình để log **TẤT CẢ** hoạt động. Đây là điều bạn **KHÔNG NÊN** làm trong production.

#### **Tại sao log ALL statements?**

**Use cases:**

* **Audit**: Biết chính xác ai đã chạy query gì, khi nào
    
* **Debug**: Reproduce lỗi bằng cách replay queries
    
* **Performance analysis**: Tìm N+1 queries, missing indexes
    
* **Security**: Phát hiện SQL injection attempts
    

**Trade-offs:**

```bash
Pros:
 - Visibility hoàn toàn
 - Không miss bất kỳ query nào
 - Dễ debug

Cons:
 - Log volume cực lớn (GB/day)
 - I/O overhead (~5-10% performance)
 - Disk space
 - Noise (quá nhiều thông tin)
```

#### **Log Configuration Breakdown**

```yaml
command: |
  postgres
  -c logging_collector=on
  -c log_directory=/var/lib/postgresql/data/pg_log
  -c log_filename=postgresql-%Y-%m-%d_%H%M%S.log
  -c log_min_duration_statement=0
  -c log_statement=all
  -c log_connections=on
  -c log_disconnections=on
  -c log_duration=on
  -c log_line_prefix='%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

**Giải thích từng tham số:**

**1\.** `logging_collector=on`

* Enable background process thu thập logs
    
* Logs được ghi vào files thay vì stderr
    
* Cho phép log rotation
    

**2\.** `log_directory` và `log_filename`

```bash
/var/lib/postgresql/data/pg_log/postgresql-2025-12-30_153045.log
│                                           │
│                                           └─ Timestamp: YYYY-MM-DD_HHMMSS
└─ Log directory
```

**Tại sao timestamp trong filename?**

* Mỗi PostgreSQL restart = file mới
    
* Dễ identify logs theo thời gian
    
* Log rotation tự động
    

**3\.** `log_min_duration_statement=0`

**Ý nghĩa:** Log tất cả statements, kể cả nhanh nhất.

```sql
-- Query 1ms → Logged
SELECT * FROM users WHERE id = 1;

-- Query 0.1ms → Cũng logged
SELECT 1;
```

**Production setting:**

```bash
log_min_duration_statement=100  # Chỉ log queries > 100ms
```

**4\.** `log_statement=all`

**Options:**

* `none`: Không log statements
    
* `ddl`: Chỉ log DDL (CREATE, ALTER, DROP)
    
* `mod`: Log DDL + DML (INSERT, UPDATE, DELETE)
    
* `all`: Log tất cả (bao gồm SELECT)
    

**Test environment:** `all` để thấy mọi thứ  
**Production:** `ddl` hoặc `mod` (SELECT quá nhiều)

**5\.** `log_connections=on` và `log_disconnections=on`

**Output:**

```bash
2025-12-30 15:30:45 UTC [1234]: [1-1] user=testuser,db=testdb,app=psql,client=10.0.0.1 LOG: connection received: host=10.0.0.1 port=54321
2025-12-30 15:35:00 UTC [1234]: [2-1] user=testuser,db=testdb,app=psql,client=10.0.0.1 LOG: disconnection: session time: 0:04:15.123
```

**Use cases:**

* Track connection pool behavior
    
* Identify connection leaks
    
* Monitor concurrent connections
    
* Audit user access
    

**6\.** `log_duration=on`

**Output:**

```bash
2025-12-30 15:30:45 UTC [1234]: [3-1] user=testuser,db=testdb LOG: duration: 123.456 ms  statement: SELECT * FROM users;
```

**Kết hợp với** `log_min_duration_statement`:

* `log_min_duration_statement=0` + `log_duration=on`: Log tất cả với duration
    
* `log_min_duration_statement=100`: Chỉ log slow queries với duration
    

**7\.** `log_line_prefix`

**Format:** `'%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '`

**Placeholders:**

* `%t`: Timestamp (2025-12-30 15:30:45 UTC)
    
* `%p`: Process ID (1234)
    
* `%l`: Log line number trong session
    
* `%u`: Username (testuser)
    
* `%d`: Database name (testdb)
    
* `%a`: Application name (psql, pgAdmin, myapp)
    
* `%h`: Client hostname/IP (10.0.0.1)
    

**Tại sao structured prefix?**

```bash
2025-12-30 15:30:45 UTC [1234]: [3-1] user=testuser,db=testdb,app=myapp,client=10.0.0.1 LOG: duration: 123.456 ms
│                               │         │                                                  │
│                               │         │                                                  └─ Log message
│                               │         └─ Context (dễ parse)
│                               └─ Process ID
└─ Timestamp
```

**Promtail có thể parse:**

```yaml
regex:
  expression: '^(?P<timestamp>...) \[(?P<pid>\d+)\]: ... user=(?P<user>[^,]*),db=(?P<database>[^,]*)...'
```

→ Extract thành labels để filter trong Grafana:

```bash
{service="postgresql", user="admin", database="production"}
```

#### **Additional Logging Features**

**Trong** `postgresql.conf`:

```ini
log_checkpoints = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
```

**1\.** `log_checkpoints=on`

**Checkpoint** = PostgreSQL flush dirty pages từ memory → disk.

**Log output:**

```bash
LOG: checkpoint starting: time
LOG: checkpoint complete: wrote 1234 buffers (7.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=0.123 s, sync=0.045 s, total=0.234 s
```

**Tại sao quan trọng?**

* Checkpoints quá thường xuyên → I/O spike → slow queries
    
* Tune `checkpoint_timeout`, `max_wal_size` dựa vào logs
    

**2\.** `log_lock_waits=on`

**Log khi query phải chờ lock &gt;** `deadlock_timeout` (default 1s):

```bash
LOG: process 1234 still waiting for ShareLock on transaction 5678 after 1000.123 ms
DETAIL: Process holding the lock: 5678. Wait queue: 1234, 1235.
```

**Use cases:**

* Identify lock contention
    
* Find queries causing blocking
    
* Optimize transaction isolation levels
    

**3\.** `log_temp_files=0`

**Log khi query tạo temporary files (work\_mem không đủ):**

```bash
LOG: temporary file: path "base/pgsql_tmp/pgsql_tmp1234.0", size 104857600
STATEMENT: SELECT * FROM large_table ORDER BY created_at;
```

**Ý nghĩa:**

* Query dùng disk thay vì RAM → chậm
    
* Cần tăng `work_mem` hoặc optimize query
    

**4\.** `log_autovacuum_min_duration=0`

**Log tất cả autovacuum operations:**

```bash
LOG: automatic vacuum of table "testdb.public.users": index scans: 1
    pages: 0 removed, 1234 remain, 0 skipped due to pins
    tuples: 567 removed, 12345 remain, 0 are dead but not yet removable
    buffer usage: 234 hits, 56 misses, 12 dirtied
    avg read rate: 1.234 MB/s, avg write rate: 0.567 MB/s
    system usage: CPU: user: 0.12 s, system: 0.03 s, elapsed: 1.23 s
```

**Tại sao quan trọng?**

* Autovacuum chậm → table bloat → slow queries
    
* Tune autovacuum parameters dựa vào logs
    

### **PostgreSQL Exporter**

**Vai trò:** Chuyển đổi PostgreSQL internal stats thành Prometheus metrics.

#### **Cách hoạt động**

```bash
1. Exporter kết nối PostgreSQL:
   postgresql://testuser:testpass@postgres-test:5432/testdb

2. Chạy SQL queries:
   SELECT * FROM pg_stat_database;
   SELECT * FROM pg_stat_activity;
   SELECT * FROM pg_stat_user_tables;
   ...

3. Parse kết quả → Prometheus format:
   pg_stat_database_xact_commit{datname="testdb"} 12345
   pg_stat_activity_count{state="active"} 5

4. Expose qua HTTP:
   http://postgres-exporter:9187/metrics
```

#### **Key Metrics**

**Database-level:**

```bash
# Transaction rate
rate(pg_stat_database_xact_commit[5m])
rate(pg_stat_database_xact_rollback[5m])

# Rollback ratio (should be low)
rate(pg_stat_database_xact_rollback[5m]) 
/ 
rate(pg_stat_database_xact_commit[5m])

# Database size
pg_database_size_bytes{datname="testdb"}
```

**Connection stats:**

```bash
# Active connections
pg_stat_activity_count{state="active"}

# Idle connections
pg_stat_activity_count{state="idle"}

# Idle in transaction (bad!)
pg_stat_activity_count{state="idle in transaction"}
```

**Tại sao "idle in transaction" xấu?**

* Giữ locks
    
* Block autovacuum
    
* Waste connection slots
    

**Table stats:**

```bash
# Sequential scans (should be low for large tables)
pg_stat_user_tables_seq_scan{relname="users"}

# Index scans (should be high)
pg_stat_user_tables_idx_scan{relname="users"}

# Seq scan ratio
pg_stat_user_tables_seq_scan 
/ 
(pg_stat_user_tables_seq_scan + pg_stat_user_tables_idx_scan)
```

**High seq scan ratio → Missing indexes**

#### **Exporter Configuration**

```yaml
environment:
  - DATA_SOURCE_NAME=postgresql://testuser:testpass@postgres-test:5432/testdb?sslmode=disable
  - PG_EXPORTER_AUTO_DISCOVER_DATABASES=true
  - PG_EXPORTER_EXCLUDE_DATABASES=template0,template1
```

`AUTO_DISCOVER_DATABASES=true`:

* Tự động monitor tất cả databases
    
* Không cần config mỗi database riêng
    

`EXCLUDE_DATABASES`:

* `template0`, `template1`: System databases
    
* Không có user data → không cần monitor
    

## **MongoDB Test Database**

### **Logging và Profiling**

MongoDB sử dụng **structured JSON logs** và **operation profiling**.

#### **System Log Configuration**

```yaml
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  verbosity: 1
  component:
    accessControl:
      verbosity: 1
    command:
      verbosity: 2
    query:
      verbosity: 2
```

**Verbosity levels:**

* `0`: Info (default)
    
* `1`: Debug
    
* `2`: Fine-grained debug
    
* `3-5`: Trace (cực kỳ chi tiết)
    

**Component-specific verbosity:**

**1\.** `command: verbosity: 2`

* Log tất cả commands (insert, update, delete, find)
    
* Include command details và execution time
    

**2\.** `query: verbosity: 2`

* Log query planning
    
* Index selection
    
* Query optimization
    

**Log format (JSON):**

```json
{
  "t": {"$date": "2025-12-30T15:30:45.123Z"},
  "s": "I",
  "c": "COMMAND",
  "ctx": "conn123",
  "msg": "command executed",
  "attr": {
    "command": "find",
    "collection": "users",
    "filter": {"age": {"$gt": 18}},
    "planSummary": "IXSCAN { age: 1 }",
    "durationMillis": 45
  }
}
```

**Fields:**

* `t.$date`: Timestamp (ISO 8601)
    
* `s`: Severity (I=Info, W=Warning, E=Error)
    
* `c`: Component (COMMAND, QUERY, STORAGE, etc.)
    
* `ctx`: Context (connection ID)
    
* `msg`: Message
    
* `attr`: Attributes (structured data)
    

#### **Operation Profiling**

```yaml
operationProfiling:
  mode: all
  slowOpThresholdMs: 0
  slowOpSampleRate: 1.0
```

**Profiling modes:**

* `off`: Disabled
    
* `slowOp`: Chỉ log slow operations
    
* `all`: Log tất cả operations (test environment)
    

`slowOpThresholdMs: 0`:

* Threshold = 0ms → tất cả operations đều "slow"
    
* Kết hợp `mode: all` → profile 100%
    

`slowOpSampleRate: 1.0`:

* 1.0 = 100% sampling
    
* 0.1 = 10% sampling (production)
    

**Profiler output (trong system log):**

```json
{
  "s": "I",
  "c": "COMMAND",
  "msg": "Slow query",
  "attr": {
    "type": "command",
    "ns": "testdb.users",
    "command": {
      "find": "users",
      "filter": {"age": {"$gt": 18}}
    },
    "planSummary": "COLLSCAN",
    "durationMillis": 234,
    "numYields": 5,
    "nreturned": 1000,
    "docsExamined": 10000,
    "keysExamined": 0
  }
}
```

**Key metrics:**

* `durationMillis`: Execution time
    
* `planSummary`: IXSCAN (good) vs COLLSCAN (bad)
    
* `docsExamined`: Documents scanned
    
* `nreturned`: Documents returned
    
* `keysExamined`: Index entries scanned
    

**Efficiency ratio:**

```bash
docsExamined / nreturned = 10000 / 1000 = 10

Ideal: Ratio = 1 (mỗi doc scan = 1 doc return)
Bad: Ratio > 10 (scan nhiều, return ít)
```

### **MongoDB Exporter**

#### **Metrics Collection**

**Exporter chạy MongoDB commands:**

```javascript
db.serverStatus()
db.stats()
db.getReplicationInfo()
```

**Key metrics:**

**Operations:**

```bash
# Operation counters
rate(mongodb_op_counters_total{type="query"}[5m])
rate(mongodb_op_counters_total{type="insert"}[5m])
rate(mongodb_op_counters_total{type="update"}[5m])
rate(mongodb_op_counters_total{type="delete"}[5m])
```

**Latency:**

```bash
# Average read latency (microseconds)
rate(mongodb_ss_opLatencies_latency{type="reads"}[5m]) 
/ 
rate(mongodb_ss_opLatencies_ops{type="reads"}[5m])

# Convert to milliseconds
(...) / 1000
```

**Connections:**

```bash
# Current connections
mongodb_connections{conn_type="current"}

# Available connections
mongodb_connections{conn_type="available"}

# Connection usage %
mongodb_connections{conn_type="current"} 
/ 
(mongodb_connections{conn_type="current"} + mongodb_connections{conn_type="available"}) 
* 100
```

**WiredTiger Cache:**

```bash
# Cache hit ratio
rate(mongodb_mongod_wiredtiger_cache_pages_requested_from_cache_total[5m])
/
(
  rate(mongodb_mongod_wiredtiger_cache_pages_requested_from_cache_total[5m])
  +
  rate(mongodb_mongod_wiredtiger_cache_pages_read_total[5m])
)

# Should be > 0.95 (95%)
```

#### **Exporter Flags**

```yaml
command:
  - "--mongodb.uri=mongodb://admin:pass@mongodb-test:27017"
  - "--collect-all"
  - "--compatible-mode"
  - "--collector.replicasetstatus"
  - "--discovering-mode"
```

`--collect-all`:

* Default: Chỉ collect basic metrics
    
* Flag này: Collect tất cả metrics available
    

`--compatible-mode`:

* Hỗ trợ MongoDB 3.x, 4.x, 5.x, 6.x, 7.x
    
* Tự động detect version và adjust queries
    

`--collector.replicasetstatus`:

* Enable replica set metrics
    
* Replication lag, member health, etc.
    

`--discovering-mode`:

* Tự động discover databases và collections
    
* Monitor tất cả databases, không cần config riêng
    

## **Promtail - Database Log Collector**

### **PostgreSQL Log Pipeline**

```yaml
- job_name: postgresql
  static_configs:
    - targets: [localhost]
      labels:
        job: postgresql
        service_name: postgresql-log-test
        db_type: postgres
        host: 10.99.3.67
        instance: test
        __path__: /var/log/postgres/**/*.log
```

`__path__: /var/log/postgres/**/*.log`:

* `**`: Recursive glob
    
* Match: `/var/log/postgres/pg_log/postgresql-2025-12-30_153045.log`
    

#### **Pipeline Stages**

**1\. Multiline Stage**

```yaml
- multiline:
    firstline: '^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}'
    max_wait_time: 3s
```

**Vấn đề:** SQL queries có thể span nhiều lines:

```bash
2025-12-30 15:30:45 UTC [1234]: [3-1] user=testuser,db=testdb LOG: duration: 123.456 ms  statement: SELECT *
FROM users
WHERE age > 18
  AND city = 'Hanoi'
ORDER BY created_at DESC;
```

**Giải pháp:**

* `firstline`: Regex match dòng đầu (có timestamp)
    
* Các dòng sau không match → append vào log entry trước
    
* `max_wait_time: 3s`: Chờ tối đa 3s cho dòng tiếp theo
    

**2\. Regex Parsing**

```yaml
- regex:
    expression: '^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}(?:\.\d+)? \w+) \[(?P<pid>\d+)\]: \[(?P<line_num>\d+-\d+)\] user=(?P<user>[^,]*),db=(?P<database>[^,]*),app=(?P<application>[^,]*),client=(?P<client>[^ ]*) (?P<level>\w+):\s+(?P<message>.*)'
```

**Parse:**

```bash
Input:
2025-12-30 15:30:45 UTC [1234]: [3-1] user=testuser,db=testdb,app=myapp,client=10.0.0.1 LOG: duration: 123.456 ms

Extracted:
timestamp: "2025-12-30 15:30:45 UTC"
pid: "1234"
line_num: "3-1"
user: "testuser"
database: "testdb"
application: "myapp"
client: "10.0.0.1"
level: "LOG"
message: "duration: 123.456 ms"
```

**3\. Labels Stage**

```yaml
- labels:
    user:
    database:
    application:
    level:
```

**Promote extracted fields → Loki labels:**

```bash
{
  job="postgresql",
  service_name="postgresql-log-test",
  user="testuser",
  database="testdb",
  application="myapp",
  level="LOG"
}
```

**Query trong Grafana:**

```bash
{service_name="postgresql-log-test", database="production", level="ERROR"}
```

**4\. Template Stage**

```yaml
- template:
    source: application
    template: '{{ if eq .Value "[unknown]" }}undefined{{ else }}{{ .Value }}{{ end }}'
```

**Xử lý edge case:**

* PostgreSQL log `app=[unknown]` khi không set application\_name
    
* Template replace `[unknown]` → `undefined`
    

**5\. Timestamp Stage**

```yaml
- timestamp:
    source: timestamp
    format: '2006-01-02 15:04:05 MST'
```

**Parse timestamp từ log → Loki timestamp:**

* `source: timestamp`: Field đã extract
    
* `format`: Go time format (2006-01-02 = reference date)
    

**Tại sao quan trọng?**

* Loki index theo timestamp
    
* Query by time range chính xác
    

**6\. Output Stage**

```yaml
- output:
    source: message
```

**Final log line = chỉ message:**

```bash
Before: 2025-12-30 15:30:45 UTC [1234]: [3-1] user=testuser,db=testdb,app=myapp,client=10.0.0.1 LOG: duration: 123.456 ms

After: duration: 123.456 ms
```

**Tại sao?**

* Timestamp, user, database, etc. đã là labels
    
* Log line chỉ cần message
    
* Giảm storage, dễ đọc
    

### **MongoDB Log Pipeline**

```yaml
- job_name: mongodb
  static_configs:
    - targets: [localhost]
      labels:
        job: mongodb
        service_name: mongodb-log-test
        db_type: mongodb
        __path__: /var/log/mongodb/*.log
```

#### **Pipeline Stages**

**1\. JSON Parsing**

```yaml
- json:
    expressions:
      timestamp: t."$date"
      severity: s
      component: c
      context: ctx
      message: msg
      attr: attr
```

**MongoDB log (JSON):**

```json
{
  "t": {"$date": "2025-12-30T15:30:45.123Z"},
  "s": "I",
  "c": "COMMAND",
  "ctx": "conn123",
  "msg": "Slow query",
  "attr": {"durationMillis": 234}
}
```

**Extracted:**

* `timestamp`: `"2025-12-30T15:30:45.123Z"` (nested field `t.$date`)
    
* `severity`: `"I"`
    
* `component`: `"COMMAND"`
    
* `context`: `"conn123"`
    
* `message`: `"Slow query"`
    
* `attr`: `{"durationMillis": 234}` (entire object)
    

**2\. Labels**

```yaml
- labels:
    severity:
    component:
```

**Loki labels:**

```bash
{
  job="mongodb",
  service_name="mongodb-log-test",
  severity="I",
  component="COMMAND"
}
```

**Query:**

```bash
# Errors and warnings
{service_name="mongodb-log-test", severity=~"E|W"}

# Query-related logs
{service_name="mongodb-log-test", component="QUERY"}
```

**3\. Timestamp**

```yaml
- timestamp:
    source: timestamp
    format: RFC3339
```

**RFC3339:** `2025-12-30T15:30:45.123Z`

**4\. Output**

```yaml
- output:
    source: message
```

**Final log:**

```bash
Slow query
```

**Attributes (attr) đi đâu?**

* Không extract thành labels (high cardinality)
    
* Vẫn trong log line (JSON string)
    
* Query với `| json | attr_durationMillis > 100`
    

## **Sử dụng Logs và Metrics**

### **PostgreSQL Use Cases**

**1\. Tìm slow queries:**

**Logs:**

```bash
{service_name="postgresql-log-test"} 
  |= "duration:" 
  | regexp "duration: (?P<duration>\\d+\\.\\d+) ms" 
  | duration > 100
```

**Metrics:**

```bash
# Long running queries
pg_stat_activity_max_tx_duration{state="active"} > 300
```

**2\. Monitor connections:**

**Metrics:**

```bash
# Connection count
pg_stat_activity_count

# Connection usage %
pg_stat_activity_count 
/ 
pg_settings_max_connections 
* 100
```

**Logs:**

```bash
# Connection events
{service_name="postgresql-log-test"} |~ "connection (received|authorized|disconnection)"
```

**3\. Identify missing indexes:**

**Metrics:**

```bash
# Sequential scan ratio
pg_stat_user_tables_seq_scan 
/ 
(pg_stat_user_tables_seq_scan + pg_stat_user_tables_idx_scan)
```

**Logs:**

```bash
# Queries creating temp files (work_mem exceeded)
{service_name="postgresql-log-test"} |= "temporary file"
```

### **MongoDB Use Cases**

**1\. Slow queries:**

**Logs:**

```bash
{service_name="mongodb-log-test", component="COMMAND"} 
  | json 
  | attr_durationMillis > 100
```

**Metrics:**

```bash
# Average query latency
rate(mongodb_ss_opLatencies_latency{type="commands"}[5m]) 
/ 
rate(mongodb_ss_opLatencies_ops{type="commands"}[5m]) 
/ 1000  # Convert to ms
```

**2\. Collection scans (missing indexes):**

**Logs:**

```bash
{service_name="mongodb-log-test"} 
  | json 
  | attr_planSummary = "COLLSCAN"
```

**3\. Connection issues:**

**Metrics:**

```bash
# Connection pool exhaustion
mongodb_connections{conn_type="available"} < 10
```

**Logs:**

```bash
{service_name="mongodb-log-test", component="NETWORK"}
```

## **Tổng kết**

### **Database Monitoring Flow**

```bash
1. Databases log operations
   ├─ PostgreSQL: Text logs với structured prefix
   └─ MongoDB: JSON logs

2. Exporters collect metrics
   ├─ Query database stats
   └─ Expose Prometheus metrics

3. Promtail collects logs
   ├─ Parse logs (regex/JSON)
   ├─ Extract labels
   └─ Push to Loki

4. Prometheus scrapes metrics
   └─ Store time series

5. Grafana visualizes
   ├─ Metrics dashboards
   └─ Log exploration
```

### **Key Takeaways**

**Test environment** = Full logging để học và experiment  
**PostgreSQL** = Text logs, structured prefix, ALL statements  
**MongoDB** = JSON logs, operation profiling 100%  
**Exporters** = Metrics từ database internal stats  
**Promtail** = Parse logs, extract labels, multiline support  
**Logs + Metrics** = Complete visibility

### **Production Recommendations**

Khi áp dụng lên production, điều chỉnh:

**PostgreSQL:**

```ini
log_min_duration_statement = 100  # Chỉ slow queries
log_statement = 'ddl'             # Chỉ DDL
```

**MongoDB:**

```yaml
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 0.1  # 10% sampling
systemLog:
  verbosity: 0  # Info only
```

**Trade-off:**

* Test: 100% visibility, high overhead
    
* Production: Balanced visibility, low overhead