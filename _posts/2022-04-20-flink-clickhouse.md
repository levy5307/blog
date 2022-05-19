---
layout: post
title: Flink导入ClickHouse
date: 2022-04-20
Author: levy5307
tags: []
comments: true
toc: true
---

## 说明

当前使用的flink-connector-jdbc仅支持Flink DataStreamAPI的方式向ClickHouse导入数据，TableAPI和FlinkSQL尚不支持。

## 依赖

需要在pom.xml中添加如下依赖，分别为flink connector和clickhouse jdbc驱动

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-jdbc_2.11</artifactId>
  <version>1.12.7</version>
</dependency>
<dependency>
  <groupId>com.clickhouse</groupId>
  <artifactId>clickhouse-jdbc</artifactId>
  <version>0.3.2-patch8</version>
  <classifier>http</classifier>
  <exclusions>
    <exclusion>
      <groupId>*</groupId>
      <artifactId>*</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

## 代码示例

在如下代码中，主要介绍了如何通过flink向clickhouse sink中写入数据，source部分可以稍加修改变成kafka等其他服务

```java
public class SinkClickHouse {
    private static final String CLICKHOUSE_URL = "jdbc:ch://{host}:{port}/{database}";
    private static final String CLICKHOUSE_JDBC_DRIVER = "com.clickhouse.jdbc.ClickHouseDriver";
    private static final String CLICKHOUSE_USER_NAME = "";
    private static final String CLICKHOUSE_PASSWD = "";

    public static void main(String [] args) {
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        /**
         * 这里可以修改kafka或者其他服务作为source
         **/
        // sql语句，用问号做占位符
        String sql = "insert into test_jdbc(name, user_id) values(?, ?)";
        // 伪造数据
        Tuple2<String, Integer> bjTp = Tuple2.of("zhaoliwei", 10086);

        JdbcConnectionOptions jdbcOptions = new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                .withUrl(CLICKHOUSE_URL)                  // ck服务地址
                .withDriverName(CLICKHOUSE_JDBC_DRIVER)   // ck jdbc driver
                .withUsername(CLICKHOUSE_USER_NAME)       // ck user name
                .withPassword(CLICKHOUSE_PASSWD)          // ck password
                .build();
        env.fromElements(bjTp)
           .returns(Types.TUPLE(Types.STRING, Types.INT))
           // 添加JDBCSink
           .addSink(JdbcSink.sink(
                   sql,
                   (ps, tp) -> {
                       ps.setString(1, tp.f0);
                       ps.setInt(2, tp.f1);
                   },
                   jdbcOptions)
                );

        // 执行
        try {
            env.execute();
        } catch (Exception e) {
        }
    }
}
```

## 部署运行

将上述代码打包成jar包，部署开启flink任务即可。

## Reference

[从JDBC connector导入ClickHouse](https://help.aliyun.com/document_detail/175749.html)

[详述Flink SQL Connector写入clickhouse的问题与方法](https://blog.csdn.net/weixin_44056920/article/details/116457512)

