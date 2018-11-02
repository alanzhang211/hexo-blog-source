---
title: JSqlParser概述
date: 2016-12-26 21:32:58
tags: [2016,JSqlParser,sql]
category: [数据库]
---
# 概述

**一切不从需求出发的开发工作，都是“耍流氓”。**

近期项目需求，需要对标准sql进行解析，比如：将一个select语句的查询字段解析出来。

需求明确，开始造轮子。造轮子之前，往往要去网上“爬”一遍，前车之鉴总是有必要的。

发现了sql解析利器[jsqlparser](http://jsqlparser.sourceforge.net/)。看了下github，还是有活跃的[github地址](https://github.com/JSQLParser/JSqlParser)(截止2016-12-26)。

![](https://github.com/alanzhang211/blog-image/raw/master//2016/12/%E6%95%B0%E6%8D%AE%E5%BA%93/jsqlparser.JPG)

<!---more-->

# 实践

## 简述
JSqlParser解析SQL语句，并转化为Java类的层次结构，生成的层次结构可使用访问者模式。

## 案例
以select 查询语句为例。

```
SELECT 1,a.*,TRIM(b.task_name) AS task_name1 FROM task a JOIN task_running_sts b ON a.id=b.task_id;
```

测试程序

```
public class JSqlParserDemo {

	public static void main(String[] args) {
		CCJSqlParserManager pm = new CCJSqlParserManager();

		String sql = "SELECT 1,a.*,TRIM(b.task_name) AS task_name1 FROM task a JOIN task_running_sts b ON a.id=b.task_id;" ;

		Statement statement;
		try {
			statement = pm.parse(new StringReader(sql));
			if (statement instanceof Select) {
				Select selectStatement = (Select) statement;
				TablesNamesFinder tablesNamesFinder = new TablesNamesFinder();
				List tableList = tablesNamesFinder.getTableList(selectStatement);
				for (Iterator iter = tableList.iterator(); iter.hasNext();) {
					System.out.println(iter.next());
				}

			}
		} catch (JSQLParserException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

	}

}
```
debug,看到selectItem对象。

![](https://github.com/alanzhang211/blog-image/raw/master//2016/12/%E6%95%B0%E6%8D%AE%E5%BA%93/select.JPG)

select结果为，别名，表达式，table等，情况。

再看源码
![](https://github.com/alanzhang211/blog-image/raw/master//2016/12/%E6%95%B0%E6%8D%AE%E5%BA%93/selectItem.jpg)

分三种情况处理，JSqlParser封装的结构简单明了。


## 封装实现

```
    @Service
public class SqlParserHandler {

    private static final Logger logger = LoggerFactory.getLogger(SqlParserHandler.class);
    /**
     * sql 获取列值
     * @param singleSql
     * @return
     */
    public List<String> getSelectColumns(String singleSql) throws Exception{
        if (singleSql == null) {
            throw new Exception("params is null!");
        }
        CCJSqlParserManager ccjSqlParserManager = new CCJSqlParserManager();
        Statement statement;
        List<String> columns = new ArrayList<String>();
        try {
            statement = ccjSqlParserManager.parse(new StringReader(singleSql));
            if (statement instanceof Select) {
                Select selectStatement = (Select) statement;
                SelectBody selectBody = selectStatement.getSelectBody();
                List<SelectItem> selectItems =  ((PlainSelect) selectBody).getSelectItems();
                if (selectItems != null) {
                    for (SelectItem item : selectItems ) {
                        if (item instanceof  AllColumns) {
                            String column = item.toString();
                            columns.add(column);
                        }
                        if (item instanceof AllTableColumns) {
                            columns.add(item.toString());
                        }
                        if (item instanceof SelectExpressionItem) {
                            Alias alias = ((SelectExpressionItem) item).getAlias();
                            Expression expression = ((SelectExpressionItem) item).getExpression();
                            if (alias != null) {
                                String column = alias.getName();
                                columns.add(column);
                            } else if (expression != null) {
                                columns.add(expression.toString());
                            }
                        }
                    }
                }
            }
        } catch (JSQLParserException e) {
            logger.error(e.getMessage());
            throw new JSQLParserException(e.getMessage());
        }
        return columns;
    }
}
```

# 结束语
> 授人以鱼，不如授人以渔 ---《淮南子·说林训》
