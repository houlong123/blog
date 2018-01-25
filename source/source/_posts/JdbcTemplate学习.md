---
title: JdbcTemplate学习
date: 2017-08-29 14:17:00
tags: Spring
---

> JdbcTemplate是spring对jdbc的封装，提供了操作数据库的模板。

#### 类图
![JdbcTemplate类图](http://thumbsnap.com/i/y3SfJjNH.png)
类图分析：
+ **JdbcOperations接口**：定义了JdbcTemplate可以使用的JDBC操作集合。从查询到更新都进行了声明。
+ **JdbcAccessor抽象类**：主要为子类提供了一些公用属性声明。
  + DataSource：Spring数据访问层对数据库资源的访问，全都建立在`javax.sql.DataSource`标准接口之上。DataSource可以看做JDBC的连接工厂。
  + SQLExceptionTranslator：负责Spring对SQLException的转译。实现了SQLException到其统一的数据访问异常体系的转换。
  
<!-- more -->

JdbcTemplate主要是通过 [模板方法模式](http://blog.csdn.net/zhengzhb/article/details/7405608) 对基于JDBC的数据访问代码进行统一封装。

#### 源码分析
一般我们都是通过xml文件配置spring的JDBC,配置如下：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">

    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <property name="url" value="${mysql.url}" />
        <property name="username" value="${mysql.username}" />
        <property name="password" value="${mysql.password}" />
        <property name="poolPreparedStatements" value="true" />
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20" />
    </bean>
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource"><ref bean="dataSource"/></property>
    </bean>
</beans>
```
由JdbcTemplate的源码，可知上述配置是在初始化数据源：

```java
public JdbcTemplate(DataSource dataSource) {
    //设置抽象类JdbcAccessor中的dataSource对象
    this.setDataSource(dataSource);
    //设置抽象类JdbcAccessor中的exceptionTranslator对象，用于转译SQLException对象，实现统一的数据访问异常处理。
    this.afterPropertiesSet();
}
```
在设置好了数据源后，就可以进行相应的JDBC操作，下面以`query`方法进行源码分析。

##### query源码

```java
public <T> T query(final String sql, final ResultSetExtractor<T> rse) throws DataAccessException {
        Assert.notNull(sql, "SQL must not be null");
        Assert.notNull(rse, "ResultSetExtractor must not be null");
        if(this.logger.isDebugEnabled()) {
            this.logger.debug("Executing SQL query [" + sql + "]");
        }

        //定义了内部类，实现了StatementCallback和SqlProvider接口。
        class QueryStatementCallback implements StatementCallback<T>, SqlProvider {
            QueryStatementCallback() {
            }

            public T doInStatement(Statement stmt) throws SQLException {
                ResultSet rs = null;

                Object var4;
                try {
                    rs = stmt.executeQuery(sql);
                    ResultSet rsToUse = rs;
                    if(JdbcTemplate.this.nativeJdbcExtractor != null) {
                        rsToUse = JdbcTemplate.this.nativeJdbcExtractor.getNativeResultSet(rs);
                    }

                    var4 = rse.extractData(rsToUse);
                } finally {
                    JdbcUtils.closeResultSet(rs);
                }

                return var4;
            }

            public String getSql() {
                return sql;
            }
        }
        
        //调用JdbcTemplate类中的execute方法
        return this.execute((StatementCallback)(new QueryStatementCallback()));
    }
```
功能分析：
在方法中，定义了一个内部类，实现了`StatementCallback接口`和`SqlProvider接口`，然后声明一个匿名类参数传递给`execute()`方法。

##### execute源码

```java
public <T> T execute(StatementCallback<T> action) throws DataAccessException {
        Assert.notNull(action, "Callback object must not be null");
        
        //获取Connection对象。与直接从DataSource取得Connection不同，DataSourceUtils会将取得的Connection绑定到当前线程，以便在使用Spring提供的统一事务抽象层进行事务管理的时候使用。
        Connection con = DataSourceUtils.getConnection(this.getDataSource());
        Statement stmt = null;

        Object var7;
        try {
            Connection ex = con;
            if(this.nativeJdbcExtractor != null && this.nativeJdbcExtractor.isNativeConnectionNecessaryForNativeStatements()) {
                ex = this.nativeJdbcExtractor.getNativeConnection(con);
            }
            
            //JAVA JDBC
            stmt = ex.createStatement();
            
            //设置Statement对象，比如设置每次获取的最大结果集，查询超时时间等。
            this.applyStatementSettings(stmt);
            Statement stmtToUse = stmt;
            if(this.nativeJdbcExtractor != null) {
                stmtToUse = this.nativeJdbcExtractor.getNativeStatement(stmt);
            }

            //回调，将相应的资源通过参数传递给回调函数。回调函数使用提供的资源进行相应的操作。类似于模板方法模式中的抽象方法。
            Object result = action.doInStatement(stmtToUse);
            this.handleWarnings(stmt);
            var7 = result;
        } catch (SQLException var11) {
            //关闭connection
            JdbcUtils.closeStatement(stmt);
            stmt = null;
            DataSourceUtils.releaseConnection(con, this.getDataSource());
            con = null;
            //对SQLException异常进行转换，实现统一异常处理。
            throw this.getExceptionTranslator().translate("StatementCallback", getSql(action), var11);
        } finally {
            JdbcUtils.closeStatement(stmt);
            DataSourceUtils.releaseConnection(con, this.getDataSource());
        }

        return var7;
    }
```
方法分析：

上述方法为[模板方法模式](http://blog.csdn.net/zhengzhb/article/details/7405608) 的典型应用。实现了数据库的操作。与一般的`模板方法模式`不一样的是，JdbcTemplate实现的方式是通过`回调(CallBack)`。这种实现方法避免了每次使用JdbcTemplate的时候都需要进行子类化。

<i><b><font color=#0099ff size=2 face="黑体">备注： 在JDK1.8中，使用匿名类的时候，若接口为[函数式接口](http://www.cnblogs.com/chenpi/p/5890144.html),z则可以转换为使用 [lambda表达式](http://www.importnew.com/16436.html)
</font></b></i>

