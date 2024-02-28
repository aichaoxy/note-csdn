PostgreSQL 数据表膨胀（Table Bloat）是指数据库表或索引在数据更新（如插入、删除、更新）后，所占用的磁盘空间增加，但实际有效数据量并未显著增长，导致存储空间的浪费。这种现象在 PostgreSQL 中比较常见，尤其是在高并发DML的更新操作下。

# 制造表膨胀场景

## 安装sysbench(以centos7系统为例)
sysbench 是一个开源的数据库性能测试工具，它可以用来评估不同类型的数据库管理系统（DBMS）的性能。sysbench 提供了多种测试场景，包括CPU、内存、磁盘I/O、数据库操作等。对于数据库压测，sysbench 提供了专门的数据库测试模块，可以模拟多种数据库操作，如插入、更新、删除、查询等。

```shell
# 1. 准备基础软件
yum -y install make automake libtool pkgconfig libaio-devel
yum -y install mariadb-devel openssl-devel mysql-devel
yum -y install postgresql-devel
# 2. 编译安装
export SYSBENCH_HOME=/data/software/sysbench-1.0.16
wget https://github.com/akopytov/sysbench/archive/1.0.16.tar.gz
tar zxf 1.0.16.tar.gz
cd sysbench-1.0.16
./autogen.sh
./configure --with-pgsql --prefix=${SYSBENCH_HOME}
make -sj
make install
```

安装完成，验证：
```shell
sysbench --version
sysbench 1.0.15
```
## 使用sysbench制造表膨胀
### 创建测试用户和数据库
```shell
export PG_HOST=1.pg.test_group.com
export PG_PORT=5432

psql -h $PG_HOST -p $PG_PORT -U postgres -d postgres <<EOF
CREATE DATABASE sbtest;
CREATE USER sbtest WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;
EOF
```
### prepare阶段
 下面命令为数据库sbtest中的10个表（sbtest1至sbtest10）每个表生成50万行数据，schema是默认的public。数据是由一个名为`parallel_prepare.lua`的脚本准备的，该脚本可在`${SYSBENCH_HOME}/share/sysbench/tests/include/oltp_legacy`路径下找到。
```shell
export SYSBENCH_HOME=/data/software/sysbench-1.0.16
export PG_HOST=1.pg.test_group.com
export PG_PORT=5432

sysbench  \
--db-driver=pgsql  \
--oltp-table-size=500000 \
--oltp-tables-count=10  \
--threads=1  \
--pgsql-host=${PG_HOST} \
--pgsql-port=${PG_PORT} \
--pgsql-user=sbtest  \
--pgsql-password=password  \
--pgsql-db=sbtest  \
${SYSBENCH_HOME}/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua \
run
```

### workload阶段
 下面命令将从名为`${SYSBENCH_HOME}/share/sysbench/tests/include/oltp_legacy/oltp.lua`的LUA脚本生成OLTP工作负载，针对50万行的10个表，使用64个工作线程在`PG_HOST`（主节点）上运行60秒。每2秒报告一次中间统计数据（–report-interval=2）。
```shell
export SYSBENCH_HOME=/data/software/sysbench-1.0.16
export PG_HOST=1.pg.test_group.com
export PG_PORT=5432

sysbench  \
--db-driver=pgsql  \
--report-interval=2  \
--oltp-table-size=500000  \
--oltp-tables-count=10  \
--threads=64  \
--time=60  \
--pgsql-host=${PG_HOST} \
--pgsql-port=${PG_PORT} \
--pgsql-user=sbtest  \
--pgsql-password=password  \
--pgsql-db=sbtest  \
${SYSBENCH_HOME}/share/sysbench/tests/include/oltp_legacy/oltp.lua \
run
```

# 监控表膨胀
编写 SQL 查询来监控表的膨胀情况
```sql
SELECT relname AS TABLE_NAME,
	pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) AS table_size,
	n_dead_tup,
	n_live_tup,
	(n_dead_tup * 100 / (n_live_tup + n_dead_tup))AS dead_tup_ratio
FROM pg_stat_user_tables 
WHERE n_dead_tup<>0 order by n_dead_tup LIMIT 10;
```

可输出如下结果
```log
 table_name | table_size | n_dead_tup | n_live_tup | dead_tup_ratio 
------------+------------+------------+------------+----------------
 sbtest1    | 123 MB     |      16227 |     500000 |              3
 sbtest10   | 123 MB     |      16316 |     500000 |              3
 sbtest8    | 123 MB     |      16333 |     500000 |              3
 sbtest5    | 123 MB     |      16399 |     500000 |              3
 sbtest4    | 123 MB     |      16426 |     500000 |              3
 sbtest3    | 123 MB     |      16450 |     500000 |              3
 sbtest9    | 123 MB     |      16599 |     500000 |              3
 sbtest7    | 123 MB     |      16601 |     500000 |              3
 sbtest6    | 123 MB     |      16702 |     500000 |              3
 sbtest2    | 123 MB     |      16827 |     500000 |              3
(10 rows)
```

# 处理表膨胀

## 使用 pg_repack 或 pg_reorg 工具
如果你发现表确实存在膨胀问题，可以使用 pg_repack 或 pg_reorg 这样的工具来重新组织表和索引，以回收空间。这些工具可以在不锁定表的情况下工作，对生产环境影响较小。

## 调整 autovacuum 参数
PostgreSQL 的 autovacuum 参数可以帮助自动回收不再需要的空间。合理调整这些参数可以减少表膨胀的发生。例如，可以调整 autovacuum_vacuum_scale_factor 和 autovacuum_vacuum_threshold 参数。

## 定期手动维护
定期执行 VACUUM 或 VACUUM FULL 命令可以帮助回收空间。VACUUM FULL 会创建一个新的表文件来替换旧的表，这在生产环境中可能需要谨慎使用，因为它会锁表。