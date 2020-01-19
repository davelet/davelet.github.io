---
layout: post
title: 使用Mybatis-plus代替原生Mybatis
categories: [dev]
tags: [java, database]
---

mybatis-plus是国人折腾出来的一个mybatis扩展（关于它的介绍可以看[开源中国](https://www.oschina.net/p/mybatis-plus)或官网，用的人越来越多了）。从大的角度看，原生的mybatis对比其他流行orm框架（当然是Hibernate，你可能以为它死多年了吧）较大的差别是需要编写sql。而mybatis-plus把这一点又抹平了：模仿JPA干掉了sql和xml文件！

这里假设项目原来使用的mybatis，我们将其替换为mybatis-plus。

# 一、替换mybatis依赖
将依赖
```xml
<groupId>org.mybatis.spring.boot</groupId>
<artifactId>mybatis-spring-boot-starter</artifactId>
```
替换为
```xml
<groupId>com.baomidou</groupId>
<artifactId>mybatis-plus-boot-starter</artifactId>
```
版本目前最新的是3.3，当然你可以自己确定。

> 如果使用了mybatis-generator（见[使用Mybatis生成Java的模型和Mapper](/mybatis-generator/)），这里也可以移除掉了。mybatis-plus有自己的代码生成器

然后将所有的Mapper接口继承com.baomidou.mybatisplus.core.mapper.BaseMapper接口，泛型参数使用对应的PO类。比如：
```java
public interface CssContractPOMapper extends BaseMapper<CssContractPO> {
}
```
> 注意：PO类的名称不能以PO结尾，mp默认要求模型类必须和表名一样，只是转成驼峰而不能增加后缀，否则要使用注解@TableName指定

# 二、单元测试
接下来对上面的mapper进行单元测试

如果报错：
```java
java.lang.NoSuchMethodError: com.baomidou.mybatisplus.core.MybatisConfiguration.getLanguageDriver(Ljava/lang/Class;)Lorg/apache/ibatis/scripting/LanguageDriver;
```
是因为版本冲突，看看是否有其他地方引入了mybatis，尤其是mybatis-pagehelper。如果使用了pagehelper可以移除掉，mp 也有自己的分页插件（[https://mp.baomidou.com/guide/page.html](https://mp.baomidou.com/guide/page.html)）。

```java 
@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {Bootstrap.class})
public class CssMapperTest {
    @Autowired
    private CssAgreementMapper mapper;

    @Test
    public void test() {
        System.out.println(mapper.selectById(1L));
    }
}
```

# 三、总结
这么快就总结了吗？

关于mp更多的介绍大家可以去其官网学习。我这里主要是说明一下使用mp和mybatis的差别：

1. 依赖不同（当然了）
2. 不使用xml和sql了（主要差别）
3. 代码生成能力不同
4. 分页不同
5. ID生成方式不同，mp可以替你生成ID（重要差别）

mp是mybatis的扩展，所以mybatis本来就有的能力依然支持。想尝试的可以试试，用过的人基本都留了下来。