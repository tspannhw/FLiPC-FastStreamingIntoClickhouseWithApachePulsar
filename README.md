# FLiPC-FastStreamingIntoClickhouseWithApachePulsar

Fast Streaming into Clickhouse with Apache Pulsar - Meetup 2022

StreamNative - Apache Pulsar - Stream to Altinity Cloud - Clickhouse

## Meetup

https://www.meetup.com/San-Francisco-Bay-Area-ClickHouse-Meetup/events/285271332/

![Meetup](https://www.meetup.com/_next/image/?url=https%3A%2F%2Fsecure-content.meetupstatic.com%2Fimages%2Fclassic-events%2F501649071%2F676x380.webp&w=3840&q=75)

## altinity cloud setup


![Clickhouse](https://github.com/tspannhw/FLiPC-FastStreamingIntoClickhouseWithApachePulsar/blob/main/altinityClickhouse.jpg?raw=true)

* Login
* Launch Cluster
* Explore
* Build table

```

drop table iotjetsonjson ON CLUSTER '{cluster}';
drop table iotjetsonjson_local ON CLUSTER '{cluster}';

CREATE TABLE iotjetsonjson_local
(
	uuid String, 
	camera String,
	ipaddress String,  
	networktime String, 
        top1pct String, 
	top1 String, 
	cputemp String, 
	gputemp String,
        gputempf String,
	cputempf String, 
	runtime String,
	host String,
	filename String,  
	host_name String, 
        macaddress String, 
	te String, 
	systemtime String,
	cpu String,
        diskusage String,
	memory String, 
	imageinput String
)
ENGINE = MergeTree()
  PARTITION BY uuid
  ORDER BY (uuid);
  
  
CREATE TABLE iotjetsonjson ON CLUSTER '{cluster}' AS iotjetsonjson_local
ENGINE = Distributed('{cluster}', default, iotjetsonjson_local, rand());

```

## Queries

```
select uuid, top1pct, top1, gputempf, cputempf
from iotjetsonjson
where toFloat32OrZero(top1pct) > 40
order by toFloat32OrZero(top1pct) desc, systemtime desc


select uuid, systemtime, networktime, te, top1pct, top1, cputempf, gputempf, cpu, diskusage, memory,filename
from iotjetsonjson 
order by systemtime desc

select top1, max(toFloat32OrZero(top1pct)), max(gputempf), max(cputempf)
from iotjetsonjson
group by top1

select top1, max(toFloat32OrZero(top1pct)) as maxTop1, max(gputempf), max(cputempf)
from iotjetsonjson
group by top1
order by maxTop1
```

## Altinity Cloud / Clickhouse / JDBC Sink Configuration

```

tenant: "public"
namespace: "default"
name: "jdbc-clickhouse-sink-iot"
topicName: "persistent://public/default/iotjetsonjson"
sinkType: "jdbc-clickhouse"
configs:
    userName: "youradminname"
    password: "somepasswordthatiscool"
    jdbcUrl: "jdbc:clickhouse://mydomainiscool.cloud:8443/default?ssl=true"
    tableName: "iotjetsonjson_local"

```

## Build the Pulsar environment (Or Just click create topic in StreamNative Cloud)

```

bin/pulsar-admin sinks stop --tenant public --namespace default --name jdbc-clickhouse-sink-iot
bin/pulsar-admin sinks delete --tenant public --namespace default --name jdbc-clickhouse-sink-iot
bin/pulsar-admin sinks restart --tenant public --namespace default --name jdbc-clickhouse-sink-iot

bin/pulsar-admin sinks create --archive ./connectors/pulsar-io-jdbc-clickhouse-2.10.0.nar --inputs iotjetsonjson --name jdbc-clickhouse-sink-iot --sink-config-file conf/clickhouseiot.yml --parallelism 1
bin/pulsar-admin sinks list --tenant public --namespace default
bin/pulsar-admin sinks get --tenant public --namespace default --name jdbc-clickhouse-sink-iot
bin/pulsar-admin sinks status --tenant public --namespace default --name jdbc-clickhouse-sink-iot

bin/pulsar-client consume "persistent://public/default/iotjetsonjson" -s iotjetsonjson-reader

```

### Pulsar SQL / Trino / Presto SQL

```
use pulsar."public/default";

describe iotjetsonjson;

select max(gputempf) as maxgputempf, max(cputemp) as maxcputemp, max(memory) as maxmemory from pulsar."public/default".iotjetsonjson;

select * from pulsar."public/default".iotjetsonjson order by systemtime desc;

```

### Spark SQL

```
val dfPulsar = spark.readStream.format("pulsar").option("service.url", "pulsar://localhost:6650").option("admin.url", "http://localhost:8080").option("topic", "persistent://public/default/iotjetsonjson").load()

dfPulsar.printSchema()

val pQuery = dfPulsar.selectExpr("*").writeStream.format("console").option("truncate", "false").start()

```


### Flink SQL

```
CREATE CATALOG pulsar WITH (
   'type' = 'pulsar',
   'service-url' = 'pulsar://pulsar1:6650',
   'admin-url' = 'http://pulsar1:8080',
   'format' = 'json'
);

USE CATALOG pulsar;

SHOW TABLES;

describe `iotjetsonjson`;

SELECT * FROM `iotjetsonjson`;

SELECT top1, top1pct, cputempf, gputempf, memory, te, systemtime from `iotjetsonjson`;

SELECT top1, top1pct, SUBSTRING(imageinput from 48) as imgnm, cputempf, gputempf, memory, te, systemtime from `iotjetsonjson`;
```


## References

* https://github.com/tspannhw/FLiPS-Xavier-Sensor
* https://github.com/tspannhw/FLiP-CloudIngest
* https://github.com/tspannhw/FLiP-SQL/
* https://github.com/tspannhw/FLiP-EdgeAI
* https://docs.altinity.com/altinitycloud/quickstartguide/yourfirstqueries/
* https://clickhouse.com/docs/en/sql-reference/functions/date-time-functions/
* https://clickhouse.com/docs/en/sql-reference/functions/type-conversion-functions/
* https://github.com/tspannhw/FLiP-CloudIngest
* https://github.com/tspannhw/StreamingAnalyticsUsingFlinkSQL
* https://github.com/tspannhw/FLiP-Stream2Clickhouse
