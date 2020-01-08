---
layout: post
title: 使用Mybatis生成Java的模型和Mapper
categories: [dev]
tags: [java, database]
---

Mybatis是国内Java界广泛使用的持久化框架。除了使用它基本的Mapper调用能力外，通常大家还会使用它的generator生成java模型类以及PageHelper的分页能力。这里介绍一下Mybatis-generator，并说明如何提升使用体验：覆盖更新并生成注释。

# 一、基本用法
Mybatis-generator的用法网上一搜一大把，这里不详细说了，简单说一下搭建过程。

在pom文件的`<build><plugins><plugins></build>`中增加生成插件：

> 这里使用的版本是1.3.5

```xml
<plugin>
 <groupId>org.mybatis.generator</groupId>
 <artifactId>mybatis-generator-maven-plugin</artifactId>
 <version>1.3.5</version>
</plugin>
```

在模块的src/main/resources中增加文件generatorConfig.xml：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
 PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
 "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
 <properties resource="generator.properties"/>
 <classPathEntry location="${jdbc.driverLocation}"/>
 <context id="mysql" targetRuntime="MyBatis3" defaultModelType="conditional">

 <jdbcConnection driverClass="${jdbc.driverClass}"
 connectionURL="${jdbc.connectionURL}" userId="${jdbc.userId}"
 password="${jdbc.password}">
 </jdbcConnection>
 <javaTypeResolver>
 <property name="forceBigDecimals" value="true"/>
 </javaTypeResolver>
 <javaModelGenerator
 targetPackage="com.css.agreement.center.model"
 targetProject="src/main/java">
 <!-- 是否允许子包，即targetPackage.schemaName.tableName -->
 <property name="enableSubPackages" value="false"/>
 <!-- 是否对model添加 构造函数 -->
 <property name="constructorBased" value="false"/>
 <!-- 是否对类CHAR类型的列的数据进行trim操作 -->
 <property name="trimStrings" value="true"/>
 <!-- 建立的Model对象是否 不可改变 即生成的Model对象不会有 setter方法，只有构造方法 -->
 <property name="immutable" value="false"/>
 </javaModelGenerator>
 <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources">
 <property name="enableSubPackages" value="true"/>
 </sqlMapGenerator>
 <javaClientGenerator
 targetPackage="com.css.agreement.center.mapper"
 targetProject="src/main/java" type="XMLMAPPER">
 <property name="enableSubPackages" value="false"/>
 </javaClientGenerator>

 <table tableName="表名" domainObjectName="PO类"
 enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false"
 enableSelectByExample="false" selectByExampleQueryId="false">
 <generatedKey identity="true" sqlStatement="MySql" column="id"/>
 <ignoreColumn column="last_updated_at"/>
 </table>

 </context>
</generatorConfiguration>
```
然后在模块根目录执行mvn mybatis-generator:generate即可在指定目录（包）生成PO类、Mapper接口类和XML文件。

---

上面的方案存在两个问题：一个是如果反复执行，Java文件不会覆盖而是会生成.java.1结尾的文件，xml也是直接在原来文件后面追加节点；第二个问题是PO类的注释信息都很无用。下面分别解决这两个问题。

# 覆盖生成
在pom的插件中增加配置信息：

```xml
<plugin>
  <groupId>org.mybatis.generator</groupId>
  <artifactId>mybatis-generator-maven-plugin</artifactId>
  <version>1.3.5</version>
  <configuration>
    <verbose>true</verbose>
    <overwrite>true</overwrite>
  </configuration>
</plugin>
```
这样的话重复执行生成，Java文件都会被覆盖（如果对生成文件做过变更也会丢失），不会再生成.java.1之类的文件了，但是xml文件还是节点追加，导致语句都重复了。

有两种方法解决。

## 1 使用新版本覆盖
将插件升级到1.3.7+即可。

从1.3.7开始，mybatis-generator增加了一个新插件org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin，在generatorConfig.xml的context节点增加该插件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
 PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
 "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
 <properties resource="generator.properties"/>
 <classPathEntry location="${jdbc.driverLocation}"/>
 <context id="mysql" targetRuntime="MyBatis3" defaultModelType="conditional">
  <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin" />
  <jdbcConnection ...">
  </jdbcConnection>

 ...
```
这样再执行xml就会重新生成。

## 2 自定义插件
上面的插件重新生成xml会把里面的改动都忽略掉，所以如果想要保留自己的改动或者不能升级到1.3.7+，可以自定义插件。

创建一个maven项目，依赖：
```xml
<dependencies>
    <dependency>
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-core</artifactId>
        <version>1.3.5</version>
    </dependency>
</dependencies>
```

新建插件类：
```java 
public class CombineXmlPlugin extends PluginAdapter {
    ShellCallback shellCallback = new DefaultShellCallback(false);
    Document document;

    @Override
    public boolean validate(List<String> warnings) {
        return true;
    }
    
    @Override
    public boolean sqlMapDocumentGenerated(Document document, IntrospectedTable introspectedTable) {
        this.document = document;
        return true;
    }

    @Override
    public boolean sqlMapGenerated(GeneratedXmlFile sqlMap, IntrospectedTable introspectedTable) {
        try {
            File directory = shellCallback.getDirectory(sqlMap.getTargetProject(), sqlMap.getTargetPackage());
            DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
            dbf.setValidating(false);
            DocumentBuilder db = dbf.newDocumentBuilder();
            File xmlFile = new File(directory, sqlMap.getFileName());
            if (!directory.exists() || !xmlFile.exists()) {
                return true;
            }
            org.w3c.dom.Document doc = db.parse(new FileInputStream(xmlFile));
            org.w3c.dom.Element rootElement = doc.getDocumentElement();
            NodeList list = rootElement.getChildNodes();

            List<Element> elements = document.getRootElement().getElements();

            Pattern p = Pattern.compile("<(\\w+)\\s+id=\"(\\w+)\"");
            boolean findSameNode;

            for (Iterator<Element> elementIt = elements.iterator(); elementIt.hasNext(); ) {
                findSameNode = false;
                String newNodeName = "";
                String NewIdValue = "";
                Element element = elementIt.next();
                Matcher m = p.matcher(element.getFormattedContent(0));
                if (m.find()) {
                    newNodeName = m.group(1);
                    NewIdValue = m.group(2);
                }

                for (int i = 0; i < list.getLength(); i++) {
                    Node node = list.item(i);
                    if (node.getNodeType() == Node.ELEMENT_NODE) {
                        if (newNodeName.equals(node.getNodeName())) {
                            NamedNodeMap attr = node.getAttributes();
                            for (int j = 0; j < attr.getLength(); j++) {
                                Node attrNode = attr.item(j);
                                if (attrNode.getNodeName().equals("id") && attrNode.getNodeValue().equals(NewIdValue)) {
                                    elementIt.remove();
                                    findSameNode = true;
                                    break;
                                }
                            }
                            if (findSameNode)
                                break;
                        }
                    }
                }
            }
        } catch (Exception e) {
        }
        return true;
    }
}
```
类的实现并不复杂，就是如果一个节点已经存在就不再生成了。pom中依赖该坐标（假设是a.b.c）:
```xml
			<plugin>
				<groupId>org.mybatis.generator</groupId>
				<artifactId>mybatis-generator-maven-plugin</artifactId>
				<version>1.3.5</version>
				<configuration>
					<verbose>true</verbose>
					<overwrite>true</overwrite>
				</configuration>
				<dependencies>
					<dependency>
						<groupId>a.b.c</groupId>
						<artifactId>c</artifactId>
						<version>1.0</version>
					</dependency>
				</dependencies>
			</plugin>
```

将插件引入：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
 PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
 "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
 <properties resource="generator.properties"/>
 <classPathEntry location="${jdbc.driverLocation}"/>
 <context id="mysql" targetRuntime="MyBatis3" defaultModelType="conditional">
  <plugin type="包名.CombineXmlPlugin" />
  <jdbcConnection ...">
  </jdbcConnection>

 ...
```

# 表注释
mybatis自带的注释生成器是org.mybatis.generator.internal.DefaultCommentGenerator，里面一堆垃圾信息，就算打开addRemarkComments开关也是不忍直视。这里也实现一个自定义注释器。创建一个maven项目，依赖同上，新建类：

```java
public class MybatisCommentGenerator implements CommentGenerator {
    private Properties properties;
    private Properties systemPro;
    private boolean suppressDate;
    private boolean suppressAllComments;
    private String currentDateStr;

    public MybatisCommentGenerator() {
        super();
        properties = new Properties();
        systemPro = System.getProperties();
        suppressDate = false;
        suppressAllComments = false;
        currentDateStr = LocalDate.now().toString();
    }

    public void addConfigurationProperties(Properties properties) {
        this.properties.putAll(properties);
        suppressDate = isTrue(properties.getProperty(PropertyRegistry.COMMENT_GENERATOR_SUPPRESS_DATE));
        suppressAllComments = isTrue(properties.getProperty(PropertyRegistry.COMMENT_GENERATOR_SUPPRESS_ALL_COMMENTS));
    }

    public void addFieldComment(Field field, IntrospectedTable introspectedTable, IntrospectedColumn introspectedColumn) {
        if (suppressAllComments) {
            return;
        }

        StringBuilder sb = new StringBuilder();

        field.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedColumn.getRemarks());
        field.addJavaDocLine(sb.toString());
        field.addJavaDocLine(" */");
    }

    public void addFieldComment(Field field, IntrospectedTable introspectedTable) {
        if (suppressAllComments) {
            return;
        }

        StringBuilder sb = new StringBuilder();

        field.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedTable.getFullyQualifiedTable());
        field.addJavaDocLine(sb.toString());
        field.addJavaDocLine(" */");
    }

    public void addModelClassComment(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        if (suppressAllComments) {
            return;
        }
        StringBuilder sb = new StringBuilder();
        topLevelClass.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedTable.getRemarks());
        topLevelClass.addJavaDocLine(sb.toString());
        topLevelClass.addJavaDocLine(" */");
    }

    public void addClassComment(InnerClass innerClass, IntrospectedTable introspectedTable) {
        if (suppressAllComments) {
            return;
        }
        StringBuilder sb = new StringBuilder();
        innerClass.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedTable.getFullyQualifiedTable());
        sb.append(" ");
        sb.append(getDateString());
        innerClass.addJavaDocLine(sb.toString());
        innerClass.addJavaDocLine(" */");
    }

    public void addClassComment(InnerClass innerClass, IntrospectedTable introspectedTable, boolean markAsDoNotDelete) {
        if (suppressAllComments) {
            return;
        }

        StringBuilder sb = new StringBuilder();
        innerClass.addJavaDocLine("/**");
        sb.append(" * ");
        sb.append(introspectedTable.getFullyQualifiedTable());
        innerClass.addJavaDocLine(sb.toString());
        sb.setLength(0);
        sb.append(" * @author ");
        sb.append(systemPro.getProperty("user.name"));
        sb.append(" ");
        sb.append(currentDateStr);
        addJavadocTag(innerClass, markAsDoNotDelete);
        innerClass.addJavaDocLine(" */");
    }

    public void addEnumComment(InnerEnum innerEnum, IntrospectedTable introspectedTable) {
        if (suppressAllComments) {
            return;
        }

        StringBuilder sb = new StringBuilder();

        innerEnum.addJavaDocLine("/**");
        addJavadocTag(innerEnum, false);
        sb.append(" * ");
        sb.append(introspectedTable.getFullyQualifiedTable());
        innerEnum.addJavaDocLine(sb.toString());
        innerEnum.addJavaDocLine(" */");
    }

    public void addGetterComment(Method method, IntrospectedTable introspectedTable, IntrospectedColumn introspectedColumn) {
        if (suppressAllComments) {
            return;
        }

        method.addJavaDocLine("/**");

        StringBuilder sb = new StringBuilder();
        sb.append(" * 获取");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString());

        sb.setLength(0);
        sb.append(" * @return ");
        sb.append(introspectedColumn.getActualColumnName());
        sb.append(" ");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString());
        method.addJavaDocLine(" */");
    }

    public void addSetterComment(Method method, IntrospectedTable introspectedTable, IntrospectedColumn introspectedColumn) {
        if (suppressAllComments) {
            return;
        }

        method.addJavaDocLine("/**");
        StringBuilder sb = new StringBuilder();
        sb.append(" * 设置");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString());

        Parameter parm = method.getParameters().get(0);
        sb.setLength(0);
        sb.append(" * @param ");
        sb.append(parm.getName());
        sb.append(" ");
        sb.append(introspectedColumn.getRemarks());
        method.addJavaDocLine(sb.toString());
        method.addJavaDocLine(" */");
    }

    public void addGeneralMethodComment(Method method, IntrospectedTable introspectedTable) {
        if (suppressAllComments) {
            return;
        }
        method.addJavaDocLine("/**");
        StringBuilder sb = new StringBuilder();
        sb.append(" * ");
        sb.append(method.getName());
        method.addJavaDocLine(sb.toString());
        method.addJavaDocLine(" */");
    }

    public void addJavaFileComment(CompilationUnit compilationUnit) {

    }

    public void addComment(XmlElement xmlElement) {

    }

    public void addRootComment(XmlElement rootElement) {
    }

    protected void addJavadocTag(JavaElement javaElement, boolean markAsDoNotDelete) {
        javaElement.addJavaDocLine(" *");
        StringBuilder sb = new StringBuilder();
        sb.append(" * ");
        sb.append(MergeConstants.NEW_ELEMENT_TAG);
        if (markAsDoNotDelete) {
            sb.append(" do_not_delete_during_merge");
        }
        String s = getDateString();
        if (s != null) {
            sb.append(' ');
            sb.append(s);
        }
        javaElement.addJavaDocLine(sb.toString());
    }

    protected String getDateString() {
        String result = null;
        if (!suppressDate) {
            result = currentDateStr;
        }
        return result;
    }
}
```

和自定义重写依赖一样引入pom中，并修改generatorConfig.xml为：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <properties resource="generator.properties"/>
    <classPathEntry location="${jdbc.driverLocation}"/>
    <context id="mysql" targetRuntime="MyBatis3" defaultModelType="conditional">
        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin" />
        <commentGenerator type="com.yonghui.mybatis.MybatisCommentGenerator">
            <property name="suppressDate" value="false"/>
            <property name="suppressAllComments" value="false"/>
        </commentGenerator>
```

这样生成后的Java类中属性和方法都会增加自定义注释，PO类的属性注释是表字段的注释，类注释是表注释。