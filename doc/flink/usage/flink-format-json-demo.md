<!-- 
使用Flink format json 的一个例子
-->
### Flink Format Json Usage
社区 [JSON Format](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/connectors/formats/json.html) 介绍
#### 数据
```
{
  "id" : 1238123899121,
  "name" : "asdlkjasjkdla998y1122",
  "bytes" : "r9HYcv34SHBC7uf4ZQYSrJkLK5Fm1ACHYlK1kOVZC/a+P1I76DRvOBJgEJhe/KECI0hiHFH7OZoRBm1eFGymdycfJu37gSfFfZP3Jl/O5aLKevTbQynhLD7q9a8CkSSRGWR6lQA==",
  "date1" : "1990-10-14",
  "date2" : "1990-10-14",
  "time" : {
    "time1" : "12:12:43Z"
  },
  "time2" : "12:12:43Z",
  "timestamp1" : "1990-10-14T12:12:43Z",
  "timestamp2" : "1990-10-14T12:12:43Z",
  "map" : {
    "flink" : 123
  },
  "map2map" : {
    "inner_map" : {
      "key" : 234
    }
  }
}
```

#### DDL
```
CREATE TABLE json_source (
    id            BIGINT,
    name          STRING,
    date1         DATE,
    date2         DATE,
    time1         MAP<STRING,TIME>,
    time2         TIME,
    timestamp1    TIMESTAMP(3),
    timestamp2    TIMESTAMP(3),
    `map`         MAP<STRING,BIGINT>,
    map2map       MAP<STRING,MAP<STRING,INTEGER>>,
    proctime      as PROCTIME()
 ) WITH (
    'connector.type'                            = 'kafka',              // 数据源的类型,比如kafka
    'connector.topic'                           = 'json1',              // topic的名字
    'connector.properties.bootstrap.servers'    = 'localhost:9092',     // kafka的bootstrap.servers地址
    'connector.properties.group.id'             = 'testGroup',          // 消费使用的group.id
    'connector.version'                         = 'universal',          // kafka的版本 默认都是 "universal"
    'format.type'                               = 'json',               // kafka中传输的消息格式，json
    'connector.startup-mode'                    = 'latest-offset'        // 从kafka中开始消费的位置
);
```
*Notes：字段名需要和json中的第一层名字一一对应。关键字需要加(``)。例如： `map`  MAP<STRING,BIGINT>。*
