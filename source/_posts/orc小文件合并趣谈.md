---
title: orc小文件合并趣谈
date: 2020-04-17 23:09:00
tags: [2020,orc,文件合并]
category: 大数据
---
## 前言
这周做了个事情趁热沉淀一下。问题很明确`治理小文件`。问题由来，要追溯到去年，`集群治理`了。之前做到`存储`和`计算`的管理，后续做了简单`hdfs画像`（其中，就有小文件趋势监控）。最近，集群中namenode压力有所显现。于是，针对小文件多的目录进行了排查和治理。进而，有了今天的这个主题**ORC小文件合并趣谈**。

## 核心问题
这里，首先治理的是实时导入数据的目录。这里增量数据采用`SparkSQL`以动态分区增量写入的方式。众所周知，spark在处理时，每个`task`都会写入一个文件（如果task处理的数据，包含n个分区的数据，就会产生n个文件）。进而，在并行度高的情况下，导致对应增量分区文件很多（存储并不大）。

在`存储治理`中，平台统一要求将hive表的格式向`orc`格式靠拢。orc的表在存储和查询上都有很好的提升。所以，这个问题就间接的转化为**解决orc小文件问题**。

<!--more-->

## 解决问题
解决问题，就先从根源入手，即sparksql小文件产生源头。在spark 2.4 版本中提供了hit的方式（https://issues.apache.org/jira/browse/SPARK-24940）。


处理上，采用程序升级和定时合并的方式。本文，主要介绍如何定时合并orc文件。

## 措施
### 方案对比
经过分析，总结了两种方式。
* 使用ORC原生DDL方式合并小文件功能。
> ALTER TABLE table_name [PARTITION partition_spec] CONCATENATE;

附：[hive-ddl](https://orc.apache.org/docs/hive-ddl.html)

优点：
1. 原生支持，开发量小。
2. 避免了数据的解压、解码过程。

缺点：
1. 不够优雅，无法指定最终合并的文件数，需要多次执行。
2. 产生一个hivesql处理，中间过程分区目录会产生`.staging-hive* `的文件。
3. 比较耗时。

* 重新造轮子，实现文件合并功能。

优点：
1. 省时间，直接操作hdfs，省去了hive处理过程。
2. 可以控制最终文件数和大小。

缺点：
1. 需要一定的开发量。
2. 合并后，hive元数据需要主动去刷新处理（直接操作hdfs文件，无法同步到hive元数据）**这点很重要**。

### 实现
#### 流程图
![流程图](https://github.com/alanzhang211/blog-image/raw/master/2020/04/liucheng.png)

1. 主线程`main`从元数据库`MetaStore`获取需要合并处理（文件数大于1）的分区信息。
2. 根据分工不同，使用两个线程池完成异步处理。
2. `mergeHdfsThreadPool`管理合并orc格式的hdfs线程。
3. `flushMetastoreThreadPool`管理统计合并分区的元数据信息线程，回馈到元数据库`MetaStore`中。


#### 核心代码实现
##### ORC合并
这里，参照官网[Using Core Java](https://orc.apache.org/docs/core-java.html)。方式，实现简单的文件合并处理。

```
    /**
     * 合并orc文件
     * @param fileDir 需要合并的分区目录
     * @throws Exception
     */
    public static void orcFileRollUp(String fileDir) throws Exception {
        if (StringUtils.isBlank(fileDir)) {
            throw new Exception("fileDir is null");
        }

        fileDir = fileDir.replace(HDFS_HOST,"");

        Path srcPath = new Path(fileDir);
        if (!fs.exists(srcPath)) {
            throw new Exception("fileDir is not exists");
        }
        if (!fs.isDirectory(srcPath)) {
            throw new Exception("fileDir is not directory");
        }

        FileStatus[] files = fs.listStatus(srcPath);
        try {
            TypeDescription schema = getSchema(files);
            if (schema != null) {
                //删除merge.working临时目录
                String outFile = fileDir + File.separator + MARGE_FILE_NAME;
                Path outMergeFilePath=null;
                try {
                    outMergeFilePath = new Path(outFile);
                    if (fs.exists(outMergeFilePath)) {
                        fs.delete(outMergeFilePath, false);
                    }
                } catch (FileNotFoundException e) {

                }

                Writer writer = OrcFile.createWriter(outMergeFilePath, OrcFile.writerOptions(fs.getConf()).setSchema(schema));
                List<Path> delSrcPathList = new ArrayList<>();
                for (FileStatus file : files) {
                    String filePath = file.getPath().toString();
                    Path srcPathTmp = new Path(filePath);
                    Reader reader = OrcFile.createReader(srcPathTmp, OrcFile.readerOptions(fs.getConf()));
                    VectorizedRowBatch batch = reader.getSchema().createRowBatch();
                    RecordReader rows = reader.rows();
                    while (rows.nextBatch(batch)) {
                        if (batch != null) {
                            writer.addRowBatch(batch);
                        }
                    }
                    rows.close();
                    delSrcPathList.add(srcPathTmp);
                }
                writer.close();

                //处理合并文件
                outFile = fs.getFileStatus(outMergeFilePath).getPath().getName();
                if (outFile.endsWith(".working")) {
                    int lastIndexOf = outFile.lastIndexOf(".working");
                    outFile = outFile.substring(0, lastIndexOf);
                }
                Path parent = outMergeFilePath.getParent();
                Path newPath = null;
                //移除上一次merge文件
                try {
                    newPath = new Path(parent, outFile);
                    Path oldMergeFile = new Path(fileDir + File.separator + outFile);
                    if (fs.exists(oldMergeFile)) {
                        fs.delete(oldMergeFile,false);
                    }
                    fs.rename(outMergeFilePath, newPath);
                } catch (FileNotFoundException e) {

                }

                //删除srcPath
                for (Path path : delSrcPathList) {
                    if (path.getName().endsWith("merge")) {
                        continue;
                    }
                    fs.delete(path, false);
                }
                LOGGER.info("合并分区{}成功,合并文件={}", fileDir,newPath);
            }
        } catch (Exception e) {
            LOGGER.error("合并分区{}失败={}", fileDir,ExceptionUtils.getFullStackTrace(e));
            throw new Exception(ExceptionUtils.getFullStackTrace(e));
        }
    }
```

文件合并会先写到`.merge.working`文件中，合并完成后，再将`.merge.working`重命名为正式文件`.merge`结尾。最后，将之前的小文件删除。

##### 重新统计分区元数据
采用hive原生的统计方式。[StatsDev](https://cwiki.apache.org/confluence/display/Hive/StatsDev)。
![StatsDev](https://github.com/alanzhang211/blog-image/raw/master/2020/04/hiveStat.png)

##### 其他注意点
1. `flushMetastoreThreadPool`要在`mergeHdfsThreadPool`内的线程结束后执行。如何知道一个线程池内所有线程执行完毕？
线程池的`isTerminated`，当所有线程都关闭时，会返回`true`。

```
while(true){
        if(mergeHdfsThreadPool.isTerminated()){
                //转交flushMetastoreThreadPool执行
                break;
            }
    }
```

## 结束
寥寥数笔，欢迎交流。
