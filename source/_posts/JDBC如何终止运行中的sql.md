---
title: JDBC如何终止运行中的sql
date: 2016-12-15 21:34:54
tags: [2016,java,JDBC]
category: [数据库]
---
# 背景
数据开发平台继续“挖坑中”，某个需求中需要实现终止sql的功能。

那么，这个cancel发生在哪里呢（那个核心对象上，参见:[JDBC必知必会之核心对象](http://alanzhang.me/2016/12/14/JDBC必知必会之核心对象/#more)）？

<!--more-->

# 思路
既然要终止查询，那么连接是必须存在的。终止发生在执行sql运行的时刻。Connection在运行之前，Connection排除；另外，ResultSet已经有反回了，也不会终止。最后，责任落在了Statement身上。最后查看源码，果真寻觅到。

```
/**
     * Cancels this <code>Statement</code> object if both the DBMS and
     * driver support aborting an SQL statement.
     * This method can be used by one thread to cancel a statement that
     * is being executed by another thread.
     *
     * @exception SQLException if a database access error occurs or
     * this method is called on a closed <code>Statement</code>
     * @exception SQLFeatureNotSupportedException if the JDBC driver does not support
     * this method
     */
    void cancel() throws SQLException;
```

接口已经申明，具体实现要看不同驱动的实现类。**if the JDBC driver does not support
this method**

# 分析
以mysql为例：
StatementImpl实现类中
```
/**
     * Cancels this Statement object if both the DBMS and driver support
     * aborting an SQL statement. This method can be used by one thread to
     * cancel a statement that is being executed by another thread.
     */
    public void cancel() throws SQLException {
        if (!this.statementExecuting.get()) {
            return;
        }

        if (!this.isClosed && this.connection != null && this.connection.versionMeetsMinimum(5, 0, 0)) {
            Connection cancelConn = null;
            java.sql.Statement cancelStmt = null;

            try {
                cancelConn = this.connection.duplicate();
                cancelStmt = cancelConn.createStatement();
                cancelStmt.execute("KILL QUERY " + this.connection.getIO().getThreadId());
                this.wasCancelled = true;
            } finally {
                if (cancelStmt != null) {
                    cancelStmt.close();
                }

                if (cancelConn != null) {
                    cancelConn.close();
                }
            }

        }
    }
```

同时，看到一个内部Task类（CancelTask）

```
/**
     * Thread used to implement query timeouts...Eventually we could be more
     * efficient and have one thread with timers, but this is a straightforward
     * and simple way to implement a feature that isn't used all that often.
     */
    class CancelTask extends TimerTask {

        long connectionId = 0;
        String origHost = "";
        SQLException caughtWhileCancelling = null;
        StatementImpl toCancel;
        Properties origConnProps = null;
        String origConnURL = "";

        CancelTask(StatementImpl cancellee) throws SQLException {
            this.connectionId = cancellee.connectionId;
            this.origHost = StatementImpl.this.connection.getHost();
            this.toCancel = cancellee;
            this.origConnProps = new Properties();

            Properties props = StatementImpl.this.connection.getProperties();

            Enumeration<?> keys = props.propertyNames();

            while (keys.hasMoreElements()) {
                String key = keys.nextElement().toString();
                this.origConnProps.setProperty(key, props.getProperty(key));
            }

            this.origConnURL = StatementImpl.this.connection.getURL();
        }

        @Override
        public void run() {

            Thread cancelThread = new Thread() {

                @Override
                public void run() {

                    Connection cancelConn = null;
                    java.sql.Statement cancelStmt = null;

                    try {
                        if (StatementImpl.this.connection.getQueryTimeoutKillsConnection()) {
                            CancelTask.this.toCancel.wasCancelled = true;
                            CancelTask.this.toCancel.wasCancelledByTimeout = true;
                            StatementImpl.this.connection.realClose(false, false, true,
                                    new MySQLStatementCancelledException(Messages.getString("Statement.ConnectionKilledDueToTimeout")));
                        } else {
                            synchronized (StatementImpl.this.cancelTimeoutMutex) {
                                if (CancelTask.this.origConnURL.equals(StatementImpl.this.connection.getURL())) {
                                    //All's fine
                                    cancelConn = StatementImpl.this.connection.duplicate();
                                    cancelStmt = cancelConn.createStatement();
                                    cancelStmt.execute("KILL QUERY " + CancelTask.this.connectionId);
                                } else {
                                    try {
                                        cancelConn = (Connection) DriverManager.getConnection(CancelTask.this.origConnURL, CancelTask.this.origConnProps);
                                        cancelStmt = cancelConn.createStatement();
                                        cancelStmt.execute("KILL QUERY " + CancelTask.this.connectionId);
                                    } catch (NullPointerException npe) {
                                        //Log this? "Failed to connect to " + origConnURL + " and KILL query"
                                    }
                                }
                                CancelTask.this.toCancel.wasCancelled = true;
                                CancelTask.this.toCancel.wasCancelledByTimeout = true;
                            }
                        }
                    } catch (SQLException sqlEx) {
                        CancelTask.this.caughtWhileCancelling = sqlEx;
                    } catch (NullPointerException npe) {
                        // Case when connection closed while starting to cancel
                        // We can't easily synchronize this, because then one thread can't cancel() a running query

                        // ignore, we shouldn't re-throw this, because the connection's already closed, so the statement has been timed out.
                    } finally {
                        if (cancelStmt != null) {
                            try {
                                cancelStmt.close();
                            } catch (SQLException sqlEx) {
                                throw new RuntimeException(sqlEx.toString());
                            }
                        }

                        if (cancelConn != null) {
                            try {
                                cancelConn.close();
                            } catch (SQLException sqlEx) {
                                throw new RuntimeException(sqlEx.toString());
                            }
                        }

                        CancelTask.this.toCancel = null;
                        CancelTask.this.origConnProps = null;
                        CancelTask.this.origConnURL = null;
                    }
                }
            };

            cancelThread.start();
        }
    }
 ```

此时联想到，配置数据源连接时，一般都会有timeout的配置项。系统在执行一个 sql 查询时，jdbc 会给你一个超时时间。为了保证超时时间后，能够关闭statement，会打开一个保护关闭的定时任务。如果超时情况下，sql 还没响应执行，cancel task 就会执行关闭任务。

# 实现方式
获取statement对象，然后调用cancel方法。

问题：如何记录要关闭的statement？

方式：使用缓存维持statement对象池。

---
*2016-12-21更新*
---
# 实战

> statement是通过connection建立的，相同的sql，不同的连接下。产生的pid不同。

然后，看到mysql中cancle方法
```
cancelConn = this.connection.duplicate();
cancelStmt = cancelConn.createStatement();
cancelStmt.execute("KILL QUERY " + this.connection.getIO().getThreadId());
this.wasCancelled = true;
 ```

## 思路一
### 实现
将connection.getIO().getThreadId()保存起来，然后执行execute方法。感觉不错。

### 问题

不同的数据库事项，处理的cancle逻辑不同。所以，此方案否决。

参见postgresql实现
```
public void sendQueryCancel() throws SQLException {
    if (cancelPid <= 0)
        return ;

    PGStream cancelStream = null;

    // Now we need to construct and send a cancel packet
    try
    {
        if (logger.logDebug())
            logger.debug(" FE=> CancelRequest(pid=" + cancelPid + ",ckey=" + cancelKey + ")");

        cancelStream = new PGStream(pgStream.getHostSpec());
        cancelStream.SendInteger4(16);
        cancelStream.SendInteger2(1234);
        cancelStream.SendInteger2(5678);
        cancelStream.SendInteger4(cancelPid);
        cancelStream.SendInteger4(cancelKey);
        cancelStream.flush();
        cancelStream.ReceiveEOF();
        cancelStream.close();
        cancelStream = null;
    }
    catch (IOException e)
    {
        // Safe to ignore.
        if (logger.logDebug())
            logger.debug("Ignoring exception on cancel request:", e);
    }
    finally
    {
        if (cancelStream != null)
        {
            try
            {
                cancelStream.close();
            }
            catch (IOException e)
            {
                // Ignored.
            }
        }
    }
}
```
## 思路二
### 实现
使用ThreadLocal缓存statement对象，实现不同线程之间的数据隔离。但是针对同一个查询的执行线程和取消线程，其实是对同一个statement操作。所以，使用ThreadLocal存储就不合理了。

针对以上，使用ConcurrentHashMap（多线程下考虑并发数据安全）成员变量存储，实现数据共享。key为查询sql唯一标识，如id主键等。

### 问题
如何区别不同客户端用户的查询？

如：用户A操作sql查询，用户B操作同样的sql查询。用户A终止操作，这里用户B的查询还需要继续执行。所以，使用sql唯一标识不太合理。

### 目的
实现不同线程（执行线程和取消线程）之间的statement共享的，同时，又要区分不同客户端请求的statement不同处理。

### 实现
map的key需要增加用户属性，如用户ID等，用意区分不同用户端执行相同sql的操作。


参考：
1. [MySQL Statement CancelTask淤积的那些事儿](http://chuansong.me/n/405421851047)
2. [mysql查询超时JDBC源码浅析](http://www.itdadao.com/articles/c15a432639p0.html)
