@[toc]
# 背景
PostgreSQL数据库中的基础数据类型一共是41种，实际应用中，数据库使用类型、数据库实现类型、Go语言承载类型，如何做一一对应，这下子就清楚了。

# 映射表
|数据库基本类型|数据库实现类型|Go语言承载类型|
|----|----|----|
|bigint|INT8|int64|
|bigserial|INT8|int64|
|bit(4)|BIT|interface{}|
|bit varying(4)|VARBIT|interface{}|
|boolean|BOOL|bool|
|box|BOX|interface{}|
|bytea|BYTEA|[]uint8|
|character(4)|BPCHAR|interface{}|
|character varying(4)|VARCHAR|interface{}|
|cidr|CIDR|interface{}|
|circle|CIRCLE|interface{}|
|date|DATE|time.Time|
|double precision|FLOAT8|interface{}|
|inet|INET|interface{}|
|integer|INT4|int32|
|interval|INTERVAL|interface{}|
|json|JSON|interface{}|
|jsonb|JSONB|interface{}|
|line|LINE|interface{}|
|lseg|LSEG|interface{}|
|macaddr|MACADDR|interface{}|
|money|MONEY|interface{}|
|numeric|NUMERIC|interface{}|
|path|PATH|interface{}|
|pg_lsn|PG_LSN|interface{}|
|point|POINT|interface{}|
|polygon|POLYGON|interface{}|
|real|FLOAT4|interface{}|
|smallint|INT2|int16|
|smallserial|INT2|int16|
|serial|INT4|int32|
|text|TEXT|string|
|time without time zone|TIME|time.Time|
|time with time zone|TIMETZ|time.Time|
|timestamp without time zone|TIMESTAMP|time.Time|
|timestamp with time zone|TIMESTAMPTZ|time.Time|
|tsquery|TSQUERY|interface{}|
|tsvector|TSVECTOR|interface{}|
|txid_snapshot|TXID_SNAPSHOT|interface{}|
|uuid|UUID|interface{}|
|xml|XML|interface{}|