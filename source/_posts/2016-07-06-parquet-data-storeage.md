---
title: parquet data storeage
comments: true
share: true
toc: true
date: 2016-07-06 21:19:30
category: ['hadoop']
tags: hadoop
description: 利用parquet格式来存储数据
---

本文介绍如何将 textfile 转换成为 parquetfile 的过程

<!--more-->

parquet 格式在 impala 中使用效率奇高，本身结合hive使用也十分快。因此转换成 parquet存储格式是十分有必要的。

转换的方式有2种：
1.将原始的 textfile 转换成hive的外部表，再从hive中 insert overwrite into <your-parquet-table> select cols from <your-textfile-table>
2.将原始textfile文件，通过MR程序，先转换成parquet文件，再用hive外部表挂载到此文件上，或其他应用方式。

第一种方法较方便，不做介绍。第二种较复杂，但每次转换的量是可控的，所以也有应用场景。

## 获取parquet schema

1.有textfile，则先将textfile获取几条数据，insert到textfile hive表中，然后再利用第一种方法，转成parquet hive表。因为数据不是很多，所以转换很快。

2.在 git上获取工具 parquet-tools

```
git clone https://github.com/apache/parquet-mr.git
git checkout parquet-1.5.0
```

3.修改顶层pom.xml 的hadoop版本跟自己的版本一致，然后注释掉 Twitter 仓库，加快下载速度

```
<!--
<pluginRepositories>
    <pluginRepository>
      <id>Twitter public Maven repo</id>
      <url>http://maven.twttr.com</url>
    </pluginRepository>
  </pluginRepositories>
—>
```

4.编译子工程（如果添加-Plocal则表示读取本地文件，如果不加，则可以读取hdfs文件，视情况而定）

```
cd ./parquet-tools
mvn clean package [-Plocal]
```

5.成功后 解压并执行文件

```
tar zxf parquet-tools-1.5.0-bin.tar.gz && cd parquet-tools-1.5.0
```

```
./parquet-schema /Users/mfw/Downloads/tmp/front_access_pa2/dt=20160101/000000_0
message hive_schema {
  optional binary remote_addr (UTF8);
  optional binary upstream_addr (UTF8);
  optional binary http_x_forwarded_for (UTF8);
  optional binary visit_time (UTF8);
  optional binary request_uri (UTF8);
  optional binary request_method (UTF8);
  optional binary server_protocol (UTF8);
  optional int32 status;
  optional int32 body_bytes_sent;
  optional float request_time;
  optional int64 uid;
  optional binary uuid (UTF8);
  optional binary user_agent (UTF8);
  optional binary refer (UTF8);
  optional binary request_body (UTF8);
}
```

这里我们就获取到了 parquet schema 的结构 其中 `hive_schema` 可以随意写。
值得注意的是，为了简便于我们后面 mapreduce的编码，建议把这里的 int float 等都换成 binary ，然后对应的hive表的字段都用 string类型

## 编写mr

1.先在pom.xml中添加依赖，将工程打包成包含依赖的 fat jar

```
<dependencies>

        <!-- hadoop -->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.6.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.6.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.6.0</version>
        </dependency>

        <!-- parquet -->
        <dependency>
            <groupId>com.twitter</groupId>
            <artifactId>parquet-hadoop</artifactId>
            <version>1.5.0</version>
        </dependency>
        <dependency>
            <groupId>com.twitter</groupId>
            <artifactId>parquet-column</artifactId>
            <version>1.5.0</version>
        </dependency>
        <dependency>
            <groupId>com.twitter</groupId>
            <artifactId>parquet-common</artifactId>
            <version>1.5.0</version>
        </dependency>
        <dependency>
            <groupId>com.twitter</groupId>
            <artifactId>parquet-format</artifactId>
            <version>2.1.0</version>
        </dependency>

        <!-- Logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.2</version>
        </dependency>

        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.16</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.2</version>
            <exclusions>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

    </dependencies>
```

2.编写主函数

``` java
package com.yxl;

import com.yxl.parquet.WriteParquet;
import org.apache.hadoop.util.ToolRunner;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 入口函数
 *
 * Created by xiaolong.yuanxl on 16-1-28.
 */
public class Main {

    private static final Logger LOG = LoggerFactory.getLogger(Main.class);

    public static void main(String[] args) {
        if (args.length < 2) {
            LOG.warn("Usage: " + " INPUTFILE OUTPUTFILE [compression gzip | snappy]");
            System.out.println("Usage: " + " INPUTFILE OUTPUTFILE [compression gzip | snappy]");
            return;
        }
        String inputPath = args[0];
        String outputPath = args[1];
        String compression = (args.length > 2) ? args[2] : "none";
        try {
            ToolRunner.run(new WriteParquet(), new String[]{inputPath, outputPath, compression});
        } catch (Exception e) {
            LOG.error("run mr JOB convert parquet file happend error: ", e);
        }

    }

}
```

3.编写模块函数

``` java
package com.yxl.parquet;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.*;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
import org.apache.hadoop.util.Tool;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import parquet.example.data.Group;
import parquet.hadoop.ParquetFileReader;
import parquet.hadoop.example.ExampleInputFormat;
import parquet.hadoop.example.ExampleOutputFormat;
import parquet.hadoop.example.GroupWriteSupport;
import parquet.hadoop.metadata.CompressionCodecName;
import parquet.hadoop.metadata.ParquetMetadata;
import parquet.schema.MessageType;

/**
 * 写文件为parquet格式
 *
 * Created by xiaolong.yuanxl on 16-1-28.
 */
public class WriteParquet extends Configured implements Tool {

    private static final Logger LOG = LoggerFactory.getLogger(WriteParquet.class);

    @Override
    public int run(String[] strings) throws Exception {
        String input = strings[0];
        String output = strings[1];
        String compression = strings[2];

        Configuration conf = new Configuration();

        // 删除已有结果集
        FileSystem fs = FileSystem.get(conf);
        Path out = new Path(output);
        if (fs.exists(out)) {
            fs.delete(out, true);
        }

        Job job = Job.getInstance();

        job.setJobName("Convert Text to Parquet");
        job.setJarByClass(getClass());

        job.setMapperClass(WriteParquetMapper.class);
        job.setInputFormatClass(TextInputFormat.class);
        job.setOutputFormatClass(ExampleOutputFormat.class);
        ExampleOutputFormat.setSchema(job, WriteParquetMapper.SCHEMA);
        job.setNumReduceTasks(0);   //不需要reduce
        job.setOutputKeyClass(Void.class);
        job.setOutputValueClass(Group.class);

        //设置压缩
        CompressionCodecName codec = CompressionCodecName.UNCOMPRESSED;
        if (compression.equalsIgnoreCase("snappy")) {
            codec = CompressionCodecName.SNAPPY;
        } else if (compression.equalsIgnoreCase("gzip")) {
            codec = CompressionCodecName.GZIP;
        }
        LOG.info("Output compression: " + codec);
        ExampleOutputFormat.setCompression(job, codec);

        FileInputFormat.setInputPaths(job, new Path(input));
        FileOutputFormat.setOutputPath(job, new Path(output));

        return job.waitForCompletion(true) ? 0 : 1;
    }

}
```

4.编写mapper (Mapper根据自己情况优化代码，这里只实现功能)

```
package com.yxl.parquet;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import parquet.example.data.Group;
import parquet.example.data.GroupFactory;
import parquet.example.data.simple.SimpleGroupFactory;
import parquet.hadoop.ParquetWriter;
import parquet.schema.MessageType;
import parquet.schema.MessageTypeParser;

import java.io.IOException;

/**
 * 写parquet mapper
 *
 * Created by xiaolong.yuanxl on 16-1-28.
 */
public class WriteParquetMapper extends Mapper<LongWritable, Text, Void, Group> {

    public static final MessageType SCHEMA = MessageTypeParser.parseMessageType(
            "message hive_schema {\n" +
                    "  optional binary remote_addr (UTF8);\n" +
                    "  optional binary upstream_addr (UTF8);\n" +
                    "  optional binary http_x_forwarded_for (UTF8);\n" +
                    "  optional binary visit_time (UTF8);\n" +
                    "  optional binary request_uri (UTF8);\n" +
                    "  optional binary request_method (UTF8);\n" +
                    "  optional binary server_protocol (UTF8);\n" +
                    "  optional binary status (UTF8);\n" +
                    "  optional binary body_bytes_sent (UTF8);\n" +
                    "  optional binary request_time (UTF8);\n" +
                    "  optional binary uid (UTF8);\n" +
                    "  optional binary uuid (UTF8);\n" +
                    "  optional binary user_agent (UTF8);\n" +
                    "  optional binary refer (UTF8);\n" +
                    "  optional binary request_body (UTF8);\n" +
                    "}"
    );

    private GroupFactory groupFactory = new SimpleGroupFactory(SCHEMA);

    @Override
    public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = StringUtils.trim(value.toString());
        String[] arr = StringUtils.splitByWholeSeparatorPreserveAllTokens(line, "\t");
        Group group = groupFactory.newGroup();
        try{
            if (arr != null){
                //直接获取下标
                group
                    .append("remote_addr", arr[0])
                    .append("upstream_addr", arr[1])
                    .append("http_x_forwarded_for", arr[2])
                    .append("visit_time", arr[3])
                    .append("request_uri",arr[4])
                    .append("request_method",arr[5])
                    .append("server_protocol", arr[6])
                    .append("status",arr[7])
                    .append("body_bytes_sent",arr[8])
                    .append("request_time", arr[9])
                    .append("uid", arr[10])
                    .append("uuid", arr[11])
                    .append("user_agent", arr[12])
                    .append("refer", arr[13])
                    .append("request_body", arr[14]);

            }
        }catch (Exception e){
            System.out.println("[ERROR]: map happend error " + e.getMessage());
        }
        context.write(null, group);
    }

}

```

5.然后运行即可

```
hadoop jar parquet-0.0.1-SNAPSHOT.jar <input> <output> <压缩格式snappy或gzip>
```

6.验证，可以用刚才我们编译的 parquet-cat 来看一下字段是否都ok了

## 导入hive表（可选，根据自己业务）

```
alter table <your-parquet-table> add partition(dt=20160101,hour=00) location '<output>';
```

附上hive建表语句

```
CREATE EXTERNAL TABLE `nginx_log`(
  `remote_addr` string,
  `upstream_addr` string,
  `http_x_forwarded_for` string,
  `visit_time` string,
  `request_uri` string,
  `request_method` string,
  `server_protocol` string,
  `status` string,
  `body_bytes_sent` string,
  `request_time` string,
  `uid` string,
  `uuid` string,
  `user_agent` string,
  `refer` string,
  `request_body` string)
PARTITIONED BY (
  `dt` string,
  `hour` string)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t'
STORED AS parquetfile
```

可以利用下面脚本每日导入或初始化补数据导入

``` bash
function loadToHive(){
    INPUT_BASE_DIR=/camus/topics/system_nginx
    OUTPUT_BASE_DIR=/user/hive/warehouse/source_log.db/nginx_log

    INPUT_PARTITION=${INPUT_BASE_DIR}/dt=$1/hour=$2/
    OUTPUT_PARTITION=${OUTPUT_BASE_DIR}/dt=$1/hour=$2/

    COMPRESS=snappy

    # 1. delete and mkdir output on hdfs

    /usr/local/datacenter/hadoop/bin/hadoop fs -rm -r -skipTrash ${OUTPUT_PARTITION}
    /usr/local/datacenter/hadoop/bin/hadoop fs -mkdir -p ${OUTPUT_PARTITION}

    # 2. convert textfile to parquetfile

    /usr/local/datacenter/hadoop/bin/hadoop jar /usr/local/datacenter/camus/lib/parquet-0.0.1-SNAPSHOT.jar ${INPUT_PARTITION} ${OUTPUT_PARTITION} ${COMPRESS}

    # 3. after parquet load data into hive external table

    /usr/local/datacenter/hive/bin/hive -e "alter table source_log.nginx_log add partition(dt=$1,hour=$2) location \"${OUTPUT_PARTITION}\";"
}

startdate=20160128
enddate=20160128

curr="$startdate"
while true; do
    echo "$curr"

    #loadToHive $curr 00
    #loadToHive $curr 01
    #loadToHive $curr 02
    #loadToHive $curr 03
    #loadToHive $curr 04
    #loadToHive $curr 05
    #loadToHive $curr 06
    #loadToHive $curr 07
    #loadToHive $curr 08
    #loadToHive $curr 09
    #loadToHive $curr 10
    loadToHive $curr 11
    loadToHive $curr 12
    loadToHive $curr 13
    loadToHive $curr 14
    loadToHive $curr 15
    loadToHive $curr 16
    loadToHive $curr 17
    loadToHive $curr 18
    loadToHive $curr 19
    loadToHive $curr 20
    loadToHive $curr 21
    loadToHive $curr 22
    loadToHive $curr 23

    [ "$curr" \< "$enddate" ] || break
    curr=$( date +%Y%m%d --date "$curr +1 day" )
done

```

PS：你也可以clone 我在 github 上的 demo 工程 https://github.com/yuanxiaolong/ParquetDemo.git
