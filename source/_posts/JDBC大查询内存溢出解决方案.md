---
title: JDBC大查询内存溢出解决方案
date: 2018-01-28 14:53:10
tags: [2018,java,JDBC]
category: [数据库]
---
# 前言
问题还是来源于实际项目。即席查询中，一次查询，返回大量数据会导致内存溢出问题。

# 问题
## 原始方案
直接通过jdbc链接数据库，通过遍历ResultSet获取结果。示例代码：

<!--more-->


```
try {
        Class.forName("org.apache.hadoop.hive.jdbc.HiveDriver");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
            System.exit(1);
        }

        //Hive JDBC URL
        String jdbcURL = "jdbc:hive2://192.168.1.102:10000/stage";
        Connection conn = DriverManager.getConnection(jdbcURL,"test","123456");
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery(sql);
        while(rs.next()) {
            System.out.println(rs.getString(1));
        }
        rs.close();
        stmt.close();
        conn.close();
    }
```

> 问题：这样在大查询的情况下，会导致内存溢出。

## 优化方案1
减少应用服务器内存压力。将查询结果存储为hdfs文件，然后通过文件流式读取数据。
### 步骤
1. 导出数据到hdfs中。

```
insert overwrite [local] directory 'directory' select_statement
```
2. 从hdfs获取查询

```
if (fileSystem.isDirectory(fsPath)) {
        //检查hdfs文件
        FileStatus[] status = fileSystem.globStatus(new Path(path + "*"));
        for (FileStatus fileStatus : status) {
            if (fileSystem.exists(fileStatus.getPath())) {
                //写入文件(hdfs操作)
                ...
            }
        }
    }
```

> 优点：减少应用服务器内存压力，避免了内存溢出OOM的风险。
> 缺点：需要增加hdfs登录过程（不方便对接第三方的数据源：需要处理不同安全认证登陆（ldap、kerberos等）；hdfs的多结点备份，浪费很多存储；不建议小文件存储在hdfs中。

## 优化方案2
使用数据库内置的流式查询。示例代码：
```
private void sendFile(Connection connection, String fileName, String sql) throws Exception {
        PreparedStatement stmt = null;
        ResultSet rs = null;
        FileOutputStream fos = null;
        BufferedWriter writer = null;
        File file = null;
        try {
            //写入本地文件
            file = new File(fileName);
            fos = new FileOutputStream(file);
            writer = new BufferedWriter(new OutputStreamWriter(fos, "UTF-8"),WeMetaConstant.MAX_DATA_LENGTH);
            stmt = connection.prepareStatement(sql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
            stmt.setFetchSize(Integer.MIN_VALUE);
            rs = stmt.executeQuery();
            //获得列集
            ResultSetMetaData rsm = rs.getMetaData();
            int colNum = rsm.getColumnCount();
            while(rs.next()) {
                for (int j = 1; j <= colNum; j++) {
                    writer.write(rs.getObject(j) == null ? "" : rs.getObject(j).toString());
                    if (j<colNum) {
                        writer.write(",");
                    }
                }
                writer.write("\r\n");
            }
        } catch (Exception e) {
            logger.error("error={}",e);
            throw e;
        }  finally {
            if (writer != null) {
                try {
                    writer.flush();
                    writer.close();
                } catch (Exception e) {
                    logger.error("关闭BufferedWriter异常！");
                }
                if (fos != null) {
                    try {
                        fos.close();
                    } catch (Exception e) {
                        logger.error("关闭FileOutputStream异常！");
                    }
                }
            }
        }
    }
```
**核心代码分析**

```
stmt = connection.prepareStatement(sql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
stmt.setFetchSize(Integer.MIN_VALUE);
```
> 驱动中 api中指出，只有同时开启ResultSet.TYPE_FORWARD_ONLY,ResultSet.CONCUR_READ_ONLY，Integer.MIN_VALUE 三个条件才能实现流失处理。

```
/**
 * We only stream result sets when they are forward-only, read-only, and the
 * fetch size has been set to Integer.MIN_VALUE
 *
 * @return true if this result set should be streamed row at-a-time, rather
 * than read all at once.
 */
protected boolean createStreamingResultSet() {
    try {
        synchronized(checkClosed().getConnectionMutex()) {
            return ((this.resultSetType == java.sql.ResultSet.TYPE_FORWARD_ONLY)
                 && (this.resultSetConcurrency == java.sql.ResultSet.CONCUR_READ_ONLY) && (this.fetchSize == Integer.MIN_VALUE));
        }
    } catch (SQLException e) {
        // we can't break the interface, having this be no-op in case of error is ok
        return false;
    }
}
```
> 优点：减少系统复杂度（相比第一种优化，减少了与hdfs系统交互过程）；更加通用，对接其他数据库时，不需要关系上层系统的差异（如：集群认证方式等）；流式方式是每次返回一个记录到内存，所以占用内存开销比较小，并且调用后会马上可以访问数据集的数据。

# 结语
性能问题，往往可以考虑是否可以通过流式编程来优化。

---
动动小手指，关注一下，是给予我最大的动力。

![个人公众号](http://of7369y0i.bkt.clouddn.com/qrcode_for_gh_381787324660_430.jpg)
