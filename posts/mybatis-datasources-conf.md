---
title: MyBatis多数据源配置
date: 2017-02-17 19:02:25
toc: true
category:
- tech
- mybatis
tags:
- MyBatis
- 多数据源
---
![mybatis][bg]
没想到这么顺利，在项目经理指导下，没有困难的就解决了配置多数据源的问题；完成后却一致没抽出时间给大家分享，现在直接甩出干货吧：

---

## 项目背景

先简单的交代一下项目背景：

* MyBatis 持久层框架
* C3P0数据库连接池
* MySQL & Oracle 数据库

当然，数据库有多种，按照对应的数据库的使用去配置数据源参数，持久层框架和连接池也有多种，基本都是一个套路；

---

## 思路

这个思路嘛，先想想单个数据源的配置：
0. 配置jdbc需要的`driver`、`url`、`user`、`password`参数
1. 配置sessionFactory，使用C3P0中间配置dataSource
2. 使用spring整合注入

简单点：
`db -> dataSource -> sessionFactory -> spring注入`

多数据源重复前两个步骤，然后整合注入，so easy 不是么！

> 复行数十步，豁然开朗

之前对spring的配置文件也是一头雾水，真的是只有经过一番实战，才会豁然开朗。

---
## 具体实现

### 数据库参数

便于配置数据库，将参数配置到`properties`文件（准备多套）
db-mysql.properties
```
mysql.jdbc.driver=com.mysql.jdbc.Driver
mysql.jdbc.url=jdbc:mysql://127.0.0.1:3306/mysql
mysql.jdbc.user=user
mysql.jdbc.password=password
```

db-orcl.properties
```
orcl.jdbc.driver=oracle.jdbc.driver.OracleDriver
orcl.jdbc.url=jdbc:oracle:thin:@192.168.1.2:1521:orcl
orcl.jdbc.user=user
orcl.jdbc.password=passwd
```
注意：键值对的`key`不要重复

### C3P0连接池

接下来都是配置`spring`的配置文件（篇幅有限，仅贴部分关键配置）

```
    <!-- 引入数据库参数properties的参数 -->
    <context:property-placeholder location="classpath:db-mysql.properties" ignore-unresolvable="true"/>

    <bean id="mysqlDataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <property name="driverClass" value="${mysql.jdbc.driver}"/>
        <property name="jdbcUrl" value="${mysql.jdbc.url}"/>
        <property name="user" value="${mysql.jdbc.user}"/>
        <property name="password" value="${mysql.jdbc.password}
        <!-- c3p0连接池的私有属性 -->
        ···
    </bean>
```
> 4·2·3·4·再 来 一 次！

引入`db-orcl.properties`，重复工作配置`orclDataSource`

### sessionFactory

`session`是jdbc与数据库通话的连接会话，不同的数据源配置对应的`sessionFactory`工厂;
注意：entity根据包的路径来区分使用不同的session;

```
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!-- 注入数据库连接池 -->
        <property name="dataSource" ref="mysqlDataSource"/>
        <!-- 配置MyBatis全局配置文件:mybatis-config.xml -->
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <!-- 扫描entity包 使用别名 -->
        <property name="typeAliasesPackage" value="com.asiainfo.dao.entity"/>
        <!-- 扫描sql配置文件:mapper需要的xml文件 -->
        <property name="mapperLocations" value="classpath:path/to/mapper/*.xml"/>
    </bean>
```

```
    <bean id="orclSessionFactory" ...
        <property name="dataSource" ref="orclDataSource"/>
        ...
    </bean>
```

### 注入

注入到哪里呢？当然是用的地方——DAO层；
观察下面的配置：bean不需要指定id了，但是要指明不同的mapper接口包，所以它是根据哪个包下的接口（实现类根据xml字段注入）使用指定的`sessionFactory`;
```
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!-- 注入sqlSessionFactory -->
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
        <!-- 给出需要扫描Dao接口包 -->
        <property name="basePackage" value="com.asiainfo.dao"/>
    </bean>
```

```
    <bean ...>
        <property name="sqlSessionFactoryBeanName" value="orclSessionFactory"/>
    </bean>
```

---

好的，到这里已经配置完了多数据源。

[bg]: mybatis-superbird-small.png
