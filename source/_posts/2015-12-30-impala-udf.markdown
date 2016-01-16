---
layout: post
title: "impala udf"
date: 2015-12-30 21:25:20 +0800
comments: true
categories: impala
tags: impala
description: impala udf
toc: true
---

Impala 从1.2开始就支持 udf 了，本文介绍 在 impala 里如何使用udf

<!--more-->

impala 使用已有的 hive java udf 及 编写 impala c++ udf

---

## hive udf

我们先编写一个 hive 的 java udf ，然后让impala调用，来模拟这是一个 已有的 hive udf 是否能被 impala调用

1.新建maven工程，添加pom依赖

``` xml
<dependencies>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-mapreduce-client-core</artifactId>
        <version>2.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-common</artifactId>
        <version>2.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-mapreduce-client-common</artifactId>
        <version>2.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-mapreduce-client-jobclient</artifactId>
        <version>2.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hive</groupId>
        <artifactId>hive-exec</artifactId>
        <version>1.1.0</version>
    </dependency>
</dependencies>
```

2.编写udf代码

``` java
package com.yxl;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.Text;

public class Main extends UDF {

    private static final String PRE = "hello_";

    public Text evaluate(String str){

        if (StringUtils.isBlank(str)){
            return new Text("unknow");
        }
        return new Text(PRE + str);
    }

}
```

3.导出jar包，可以用 IDE 导出不含依赖的 jar 。或者用 `mvn clean install` 直接安装到本地仓库，拿本地仓库的jar即可。

4.把此jar包放到 hdfs 上

5.进入impala-shell ，并进入对应的数据库上，执行类似如下命令（根据自己情况修改hdfs路径）

注：由于 Impala 和 hive 共享元数据，所以我们需要把新创建的 function 取跟hive function 里不一样的名字，尽管它们是同一个jar

``` bash
create function my_foo(string) returns string location '/share/Udf4Hive.jar' symbol='com.yxl.Main'
```

6.使用 hive udf

``` bash
[node007012:21000] > desc pokes;
Query: describe pokes
+------+--------+---------+
| name | type   | comment |
+------+--------+---------+
| foo  | int    |         |
| bar  | string |         |
+------+--------+---------+

[node007012:21000] > select foo,test.my_foo(bar) from pokes;
Query: select foo,test.my_foo(bar) from pokes
+-----+------------------+
| foo | test.my_foo(bar) |
+-----+------------------+
| 1   | hello_hello      |
| 2   | hello_pear       |
| 3   | hello_world      |
+-----+------------------+
Fetched 3 row(s) in 0.12s

```

---

## C++ udf

[cloudrea文档](http://www.cloudera.com/content/cloudera/zh-CN/documentation/core/v5-3-x/topics/impala_udf.html)

概念

* UDF 一次处理一行的函数，0到多个入参，1个返回值
* UDAF 一次处理多行的函数，类似聚集函数 sum 这样的，一个返回值 （未试验）

官网示例  [<font color="#296ead">Github</font>](https://github.com/cloudera/impala-udf-samples)

## 环境准备

1.安装impala udf 安装包及编译器，可参见上文 Impala rpm 安装。

``` bash
# Use the appropriate package installation command for your Linux distribution.
sudo yum install gcc-c++ cmake boost-devel
sudo yum install impala-udf-devel
```

2.编写 c++ 文件 ，包含头文件、逻辑文件、测试文件、和CMAKE文件

头文件 `udf-helloworld.h`

``` cpp
#ifndef HELLOWORLD_UDF_H
#define HELLOWORLD_UDF_H

#include <impala_udf/udf.h>

using namespace impala_udf;

StringVal Hello(FunctionContext* context, const StringVal& arg1);

#endif
```

逻辑文件 `udf-helloworld.cc` ，实现添加一个固定前缀

``` cpp
#include "udf-helloworld.h"

#include <cctype>
#include <cmath>
#include <string>
#include <sstream>

StringVal Hello(FunctionContext* context, const StringVal& arg1){
    if (arg1.is_null) return StringVal::null();

    int index;
    std::string original((const char *)arg1.ptr,arg1.len);
    std::string shorter("hello_");

    int length;
    length = original.length();
    for (index = 0; index < length; index++){
        uint8_t c = original[index];

        shorter.append(1, (char)c);
    }

    StringVal result(context, shorter.size());
    memcpy(result.ptr, shorter.c_str(), shorter.size());

    return result;
}
```

测试文件 `udf-helloworld-test.cc`

```
#include <iostream>

#include <impala_udf/udf-test-harness.h>
#include "udf-helloworld.h"

using namespace impala;
using namespace impala_udf;
using namespace std;

int main(int argc, char** argv) {
    bool passed = true;

    passed &= UdfTestHarness::ValidateUdf<StringVal, StringVal>(
          Hello, StringVal("Tom"), StringVal("hello_Tom"));
    passed &= UdfTestHarness::ValidateUdf<StringVal, StringVal>(
          Hello, StringVal::null(), StringVal::null());

    cout << "Tests " << (passed ? "Passed." : "Failed.") << endl;
    return !passed;
}
```

CMAKE文件 固定名字 `CMakeLists.txt`

``` cmake
# Copyright 2012 Cloudera Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 2.6)

# where to put generated libraries
set(LIBRARY_OUTPUT_PATH "build")
# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "build")

find_program(CLANG_EXECUTABLE clang++)

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -ggdb")

# Function to generate rule to cross compile a source file to an IR module.
# This should be called with the .cc src file and it will generate a
# src-file-ir target that can be built.
# e.g. COMPILE_TO_IR(test.cc) generates the "test-ir" make target.
# Disable compiler optimizations because generated IR is optimized at runtime
set(IR_COMPILE_FLAGS "-emit-llvm" "-c")
function(COMPILE_TO_IR SRC_FILE)
  get_filename_component(BASE_NAME ${SRC_FILE} NAME_WE)
  set(OUTPUT_FILE "build/${BASE_NAME}.ll")
  add_custom_command(
    OUTPUT ${OUTPUT_FILE}
    COMMAND ${CLANG_EXECUTABLE} ${IR_COMPILE_FLAGS} ${SRC_FILE} -o ${OUTPUT_FILE}
    DEPENDS ${SRC_FILE})
  add_custom_target(${BASE_NAME}-ir ALL DEPENDS ${OUTPUT_FILE})
endfunction(COMPILE_TO_IR)

add_library(udfhello SHARED udf-helloworld.cc)

if (CLANG_EXECUTABLE)
  COMPILE_TO_IR(udf-helloworld.cc )
endif(CLANG_EXECUTABLE)

target_link_libraries(udfhello ImpalaUdf)
add_executable(udf-helloworld-test udf-helloworld-test.cc)
target_link_libraries(udf-helloworld-test udfhello)

```

>C++ 定义的数据类型（位于 /usr/include/impala_udf/udf.h）是：
>
>IntVal 代表 INT 列。
>
>BigIntVal 代表 BIGINT 列。即使你不需要 BIGINT 值的全部范围，将函数参数设置为 BigIntVal 也会非常有用，以方便调用以不同种类的整数列和表达式作为参数的函数。Impala 会在适当时将较小的整数类型自动转换为较大的整数类型，但不会隐式将较大的整型转换为教小的。
>
>SmallIntVal 代表SMALLINT 列。
>
>TinyIntVal 代表 TINYINT 列。
>
>StringVal 代表 STRING 列。其中 len 字段表示字符串长度，而 ptr 字段指向该字符串数据。基于 null 结尾的 C 风格字符串，或者一个指针加上长度，构造函数可以创建一个新的 StringVal 结构；这些新结构仍然参照原来的字符串数据，而不是为数据分配一个新的缓冲区。同时还包括一个构造函数，带有指向FunctionContext 结构和长度的指针，并且会为新复制的字符串数据分配空间，UDF 将使用该空间返回字符串值。
>
>BooleanVal 代表 BOOLEAN 列。
>
>FloatVal 代表 FLOAT 列。
>
>DoubleVal 代表 DOUBLE 列。
>
>TimestampVal 代表TIMESTAMP 列。具有一个 32 位整数的 date 字段，用以表示公历日期，历元后的天数。还具有一个 64 位整数的 time_of_day 字段，以纳秒表示当前时间。

3.进入文件夹，执行 `cmake . && make`

``` bash
[root@node007012 udf]# make
[ 50%] Built target udfhello
Scanning dependencies of target udf-helloworld-test
[100%] Building CXX object CMakeFiles/udf-helloworld-test.dir/udf-helloworld-test.cc.o
Linking CXX executable build/udf-helloworld-test
[100%] Built target udf-helloworld-test
```

4.进入build子文件夹，将里面的动态库文件，上传到hdfs上

5.进入impala-shell，到对应的数据库，并创建 function

``` bash
[node007012:21000] >  create function hello (string) returns string location '/share/libudfhello.so' symbol='Hello';
Query: create function hello (string) returns string location '/share/libudfhello.so' symbol='Hello'

Fetched 0 row(s) in 0.15s
```

6.使用impala c++ function 查询，可见比java确实快很多，虽然只有3条数据，但能看见差异，反复测试现象一样

``` bash
[node007012:21000] > select foo,hello(bar) from pokes;
Query: select foo,hello(bar) from pokes
+-----+-----------------+
| foo | test.hello(bar) |
+-----+-----------------+
| 1   | hello_hello     |
| 2   | hello_pear      |
| 3   | hello_world     |
+-----+-----------------+
Fetched 3 row(s) in 0.01s
```

需要注意的是：

>目前，Metastore 数据库中不保留以 C++ 语言写入的 Impala UDF 和 UDA。这些函数的相关信息保留在 catalogd 守护程序的内存中。
>每次重新启动 catalogd 守护程序时，必须再次运行 CREATE FUNCTION 语句才能加载它们。该限制不适用于以 Java 语言写入的 Impala UDF 和 UDA。

---

## 重新注册函数

1.拿刚才的为例，前缀改成 hello123_

```
std::string shorter("hello123_");
```

2.重新cmake 当前目录，并且make

3.删除已有的 hdfs so 文件

4.进入impala-shell，查看已有函数

``` bash
[node007012:21000] > show functions;
Query: show functions
+-------------+----------------+
| return type | signature      |
+-------------+----------------+
| STRING      | hello(STRING)  |
| STRING      | my_foo(STRING) |
+-------------+----------------+
Fetched 2 row(s) in 0.02s
```

5.由于函数可能会重载，所以删除函数的时候，需要指定参数，才能对应上要删除的函数

``` bash
[node007012:21000] > drop function hello(string);
Query: drop function hello(string)
```

6.重新创建并查询

``` bash
[node007012:21000] > create function hello (string) returns string location '/share/libudfhello.so' symbol='Hello';
Query: create function hello (string) returns string location '/share/libudfhello.so' symbol='Hello'

Fetched 0 row(s) in 0.04s

[node007012:21000] > select foo,hello(bar) from pokes;
Query: select foo,hello(bar) from pokes
+-----+-----------------+
| foo | test.hello(bar) |
+-----+-----------------+
| 1   | hello123_hello  |
| 2   | hello123_pear   |
| 3   | hello123_world  |
+-----+-----------------+
Fetched 3 row(s) in 0.02s

```
