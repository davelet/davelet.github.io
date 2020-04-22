---
layout: post
title: mybatis plus与mysql分库组件mycat的结合
categories: [dev]
tags: [java, database]
---

之前的文章简单介绍了一下mybatis plus：《[使用Mybatis-plus代替原生Mybatis](/mybatis-plus/)》。截止目前在项目中使用了一段时间的mybatis plus，再也没有写过sql，都用mp的Wrapper封装查询条件了。这里先简单介绍一下mp的用法（抱歉，上一篇里面讲得实在太水了，因为当时自己也不会用），然后再说和mycat的整合。

# 代码生成
先在数据库建好表，假设表名是t_mp_test。然后使用主类：
```java
import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.generator.AutoGenerator;
import com.baomidou.mybatisplus.generator.InjectionConfig;
import com.baomidou.mybatisplus.generator.config.DataSourceConfig;
import com.baomidou.mybatisplus.generator.config.GlobalConfig;
import com.baomidou.mybatisplus.generator.config.PackageConfig;
import com.baomidou.mybatisplus.generator.config.StrategyConfig;
import com.baomidou.mybatisplus.generator.config.rules.DateType;
import com.baomidou.mybatisplus.generator.config.rules.NamingStrategy;

import java.util.HashMap;
import java.util.Map;

public class MpGenerator {
    public static void main(String[] args) {
        AutoGenerator mpg = new AutoGenerator();

        // 全局配置
        GlobalConfig gc = new GlobalConfig();
        String projectPath = System.getProperty("user.dir");
        System.out.println(projectPath);
        gc.setOutputDir(projectPath + "/myplus");
        gc.setAuthor("sheldon");
        gc.setFileOverride(true); //是否覆盖
        gc.setActiveRecord(false);// 不需要ActiveRecord特性的请改为false
        gc.setEnableCache(false);// XML 二级缓存
        gc.setBaseResultMap(true);// XML ResultMap
        gc.setBaseColumnList(true);// XML columList
        gc.setOpen(false);

        gc.setEntityName("%sPO");
        gc.setDateType(DateType.ONLY_DATE);
        mpg.setGlobalConfig(gc);

        // 数据源配置
        DataSourceConfig dsc = new DataSourceConfig();
        dsc.setDbType(DbType.MYSQL);
        dsc.setDriverName("com.mysql.jdbc.Driver");
        dsc.setUsername("root");
        dsc.setPassword("123456");
        dsc.setUrl("jdbc:mysql://127.0.0.1:3306/db?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&transformedBitIsBoolean=true&serverTimezone=GMT%2B8");
        mpg.setDataSource(dsc);

        // 策略配置
        StrategyConfig strategy = new StrategyConfig();
        strategy.setNaming(NamingStrategy.underline_to_camel);// 表名生成策略
        strategy.setInclude(new String[] { "t_mp_test" }); // 需要生成的表
        // 自定义 service 父类
        strategy.setSuperServiceClass("com.baomidou.mybatisplus.extension.service.IService");
        // 自定义 service 实现类父类
        strategy.setSuperServiceImplClass("com.baomidou.mybatisplus.extension.service.impl.ServiceImpl");

        mpg.setStrategy(strategy);

        // 包配置
        PackageConfig pc = new PackageConfig();
        pc.setParent("a.b");
        pc.setModuleName("c");
        pc.setController("controller");
        pc.setEntity("model");
        pc.setMapper("mapper");
        pc.setService("service");
        pc.setServiceImpl("service.impl");
        pc.setXml("mapper.xml");

        mpg.setPackageInfo(pc);
        mpg.setCfg(cfg);

        // 执行生成
        mpg.execute();
    }
}
```
建议把这个文件放到test目录下，执行后会在跟目录下生成代码，分别拷贝到对应的包中即可。

# CRUD
下面分别简单介绍下常用的dml功能。
## 根据ID或其他条件查询
如果是根据ID查询比较简单，使用替你生成的IService的子接口调用
```java 
com.baomidou.mybatisplus.extension.service.IService#getById
```
如果是根据其他条件，返回是单个对象的话使用
```java 
com.baomidou.mybatisplus.extension.service.IService#getOne(com.baomidou.mybatisplus.core.conditions.Wrapper<T>)
```
返回集合的话使用
```java 
com.baomidou.mybatisplus.extension.service.IService#list(com.baomidou.mybatisplus.core.conditions.Wrapper<T>)
```
Wrapper可以传入一个对象用来封装查询条件，也可以直接调用com.baomidou.mybatisplus.core.conditions.query.QueryWrapper的方法封装条件。
QueryWrapper提供了很多方法匹配标准sql的语法，比如eq是相等，ne是不等于，ge是大于等于，lt是小于，等等。还可以分组、排序、模糊查询等等（其实是其抽象父类com.baomidou.mybatisplus.core.conditions.AbstractWrapper提供的）。

> 注意：QueryWrapper的同一个条件多次使用不会被覆盖，所以就相当于无效查询了。比如

```java 
QueryWrapper<?> qw = new QueryWrapper<>();
qw.eq("id", "1209570800020050");
qw.eq("id", "1209570800020051");
qw.in("name", list);
List<?> list = service.list(qw);
```
这段代码期望查询name在集合list中、并且id是第二次传入的结果。但实际上生成的sql条件是where id = "1209570800020050" and id = "1209570800020051" and name in (@list)，所以一定命不中记录。

# 增改
增加可以使用
```java
com.baomidou.mybatisplus.extension.service.IService#save
```
或者批量增加接口
```java 
com.baomidou.mybatisplus.extension.service.IService#saveBatch(java.util.Collection<T>)
```
更新可以使用
```java 
com.baomidou.mybatisplus.extension.service.IService#update(T, com.baomidou.mybatisplus.core.conditions.Wrapper<T>)
```
第一个参数是要更新的字段，第二个参数封装的匹配条件。如果是根据ID更新，则不用传入第二个参数，将ID赋到第一个参数即可。
更新也提供批量接口
```java 
com.baomidou.mybatisplus.extension.service.IService#updateBatchById(java.util.Collection<T>)
```
另外，插入和更新可以使用同样的方法
```java 
com.baomidou.mybatisplus.extension.service.IService#saveOrUpdate(T)
```
或
```java
com.baomidou.mybatisplus.extension.service.IService#saveOrUpdateBatch(java.util.Collection<T>)
```
执行区别是判断是否传入ID，有则更新，无则插入。

# 删除
删除就不说了，现在很少有物理删除功能。如果需要可以查找remove开头的方法，一定能满足你。

# 分库组件mycat
分库分表在国内已经很火爆了，我项目里用的是mycat。

> 其实是基础架构层用的，项目里几乎看不出在使用分库或者使用了mycat。

有一点这里要说一下，就是使用mybatis plus整合mycat 查询的话，sessionFactory必须使用Mp提供的
```java
com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean
```
类，否则可能会找不到mapper语句，因为mp的语句都是自己的代理生成的。