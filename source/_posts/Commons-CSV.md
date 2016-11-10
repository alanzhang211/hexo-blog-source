---
title: Commons CSV
date: 2014-11-27 20:32:19
tags: [2014,Apache,csv]
category: Apache
---

# 概述
本文简单介绍Apache Commons CSV的使用。

官网：http://commons.apache.org/proper/commons-csv

## 结构图
![](http://of7369y0i.bkt.clouddn.com/2014/11/%E6%8D%95%E8%8E%B7.JPG)

<!--more-->

# 测试
## 实现csv文件的写入和读取
```
package com.alan.apache.commons.csv;

import java.io.FileNotFoundException;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;
import java.io.Reader;
import java.util.ArrayList;
import java.util.List;

import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVPrinter;
import org.apache.commons.csv.CSVRecord;
/**
* <p>csv写入、读取</p>
* @author alanzhang211 2014-12-27 下午03:10:00
* @blog http://www.alanzhang.me
* @GitHub https://github.com/alanzhang211
* @version V1.0
* @modificationHistory=============逻辑或功能性重大变更记录
* @modify by user: {修改人} 2014-12-27
* @modify by reason:{方法名}:{原因}
*/
public class CSVTestDemo {

private final static String BASEPATH_STRING = System.getProperty(“user.dir”);
/**
* 写入CSV文件
* @author alanzhang211 2014-12-27 下午03:19:38
* @throws Exception
*/
public void writeCSV() {
FileWriter writer = null;
try {
writer = new FileWriter(BASEPATH_STRING+”/WebRoot/docs/csv/test.csv”);
CSVPrinter printer = new CSVPrinter(writer, CSVFormat.RFC4180.withHeader(“LastName”,”FirstName”).withDelimiter(‘,’));
List<String[]> dataList = new ArrayList<String[]>();
dataList.add(new String[]{“alan”,”zhang”});
printer.printRecords(dataList);
printer.close();
} catch (IOException e) {
e.printStackTrace();
}
}
/**
* 读取CSV文件内容
* @author alanzhang211 2014-12-27 下午07:16:21
* @throws Exception
*/
public void readCSV(){
Reader in = null;
try {
in = new FileReader(BASEPATH_STRING+”/WebRoot/docs/csv/test.csv”);
//不指定文件列名，默认第一行为列名
CSVParser parser = CSVFormat.EXCEL.withHeader().parse(in);
//CSVParser parser = CSVFormat.EXCEL.withHeader(“LastName”,”FirstName”).parse(in);
String firstName = null;
String lastName = null;
for (CSVRecord record : parser) {
lastName = record.get(“LastName”);
firstName = record.get(“FirstName”);
System.out.println(“lastName:”+lastName);
System.out.println(“firstName:”+firstName);
}
parser.close();
} catch (FileNotFoundException e) {
e.printStackTrace();
} catch (IOException e) {
e.printStackTrace();
}
}
}

```

## junit测试
```
package com.alan.apache.commons.csv;
import org.junit.Before;
import org.junit.Test;
/**
* <p>junit测试代码</p>
* @author alanzhang211 2014-12-27 下午04:08:29
* @blog http://www.alanzhang.me
* @GitHub https://github.com/alanzhang211
* @version V1.0
* @modificationHistory==============逻辑或功能性重大变更记录
* @modify by user: {修改人} 2014-12-27
* @modify by reason:{方法名}:{原因}
*/
public class CSVTestDemoTest {
/**CSV实例*/
private CSVTestDemo csvTestDemo;
@Before
public void setUp(){
csvTestDemo = new CSVTestDemo();
}

@Test
public void WriteCSV(){
csvTestDemo.writeCSV();
}

@Test
public void ReadCSV(){
csvTestDemo.readCSV();
}
}
```

# 问题
在依据列名获取内容时报如下错误：

java.lang.IllegalStateException: No header mapping was specified, the record values can’t be accessed by name
at org.apache.commons.csv.CSVRecord.get(CSVRecord.java:99)
at com.alan.apache.commons.csv.CSVTestDemo.readCSV(CSVTestDemo.java:49)
at com.alan.apache.commons.csv.CSVTestDemoTest.ReadCSV(CSVTestDemoTest.java:32)
at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
at java.lang.reflect.Method.invoke(Method.java:597)

原因：在读取时，没有withHeader(…)标识指定列名。
