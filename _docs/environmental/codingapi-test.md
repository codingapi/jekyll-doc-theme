---
title: codingapi-test测试框架
permalink: /docs/codingapi-test/
---


## 单元测试介绍


### 通常单元测试要达到的效果:
* 检验代码是否可以正常工作
* 要达到业务层率覆盖率100%
* 不依赖其他模块与数据可独立运行
* 执行测试以后不允许产生脏数据影响业务
* 要对业务产生的影响做检验确认


## codingapi-test框架

框架基于Springboot2.x研发，可快速准备数据、校验数据、清理数据的测试工具。

使用codingapi-test框架的理由：
* 方便快捷的准备测试数据
* 自动化的校验
* 自动化的清理数据
* 兼容MongoDB、关系型数据库(例如:Mysql、Oracle、SqlServer...)

github:

[https://github.com/codingapi/codingapi-test](https://github.com/codingapi/codingapi-test)

maven:
```
<dependency>
      <groupId>com.codingapi</groupId>
      <artifactId>codingapi-test</artifactId>
      <version>0.0.2</version>
</dependency>
```

## 配置说明

`@TestMethod`提供了对单元测试辅助功能。  
1、导入测试数据。  
2、检查确认数据  
3、数据清理  

```java
@Test
@TestMethod(
        //是否开启导入数据
        enablePrepare = true,
        //导入数据的文件 可以是数组
        prepareData = {"t_demo.xml"},
        //是否开启检查
        enableCheck = true,
        //检查数据项目
        checkMongoData ={
          //MongoDB数据检查
          @CheckMongoData(
                  //关键字质
                  primaryVal = "user:123",
                  //查询关键字
                  primaryKey = "info",
                  //关键值类型
                  type = CheckMongoData.Type.STRING,
                  //错误提示
                  desc = "数据不存在",
                  //加载类对象
                  bean = Logger.class,
                  //校验数据 key:字段，value:值
                  expected = @Expected(key = "id",value = "1"))
        },
        checkMysqlData = {
           //Mysql 数据检查
           @CheckMysqlData(
                   //执行sql
                   sql = "select name from t_demo where name = '123'",
                   //错误提示
                   desc = "数据不存在",
                   //校验数据 key:字段，value:值
                   expected = @Expected(key = "name",value = "123"))
         },
        //开启清理          
        enableClear = true,
        //清理的MongoDB collection
        clearCollectionNames = {"logger"},
        //清理的db table
        clearTableNames = {"t_demo"}
)
public void login_success() {
    try {
        Long id  = demoService.login("123");
        log.info("login - > {}", id);
        Assert.isTrue(id==1 ,"login success .");
    } catch (UserNameNotFoundException exp) {
        exp.printStackTrace();
    }
}
```

t_demo.xml
```
<XmlInfo>
  <!--表名称 -->
  <name>t_demo</name>
  <!--插入语句自动创建的 -->
  <initCmd>insert into t_demo(id,name) values(#{id},#{name})</initCmd>
  <!--数据库类型  MONGODB RELATIONAL-->
  <dbType>RELATIONAL</dbType>
  <!--entity所在类 -->
  <className>com.codingapi.cidemo.domain.Demo</className>
  <list>
    <!--数据 -->
    <list>
      <id>1</id>
      <name>123</name>
    </list>
    <list>
      <id>1</id>
      <name>123</name>
    </list>
  </list>
</XmlInfo>

```

mongo.xml
```
<XmlInfo>
  <!--collection名称 -->
  <name>logger</name>
  <!--暂不需要 -->
  <initCmd></initCmd>
  <!--数据库类型 MONGODB RELATIONAL-->
  <dbType>MONGODB</dbType>
  <!--entity所在类 -->
  <className>com.codingapi.cidemo.collection.Logger</className>
  <list>
    <!--数据 -->
    <list>
      <id>1</id>
      <time>2</time>
      <info>3</info>
    </list>
    <list>
      <id>1</id>
      <time>2</time>
      <info>3</info>
    </list>
  </list>
</XmlInfo>

```

如何创建 数据模块xml

增加`@XmlBuild` 配置表名称和类型即可 `colType`是字段插入语句的类型，分为`UNDERLINE`下划线和`CAMEL`驼峰两种类型
```java
@Data
@XmlBuild(name = "t_demo",dbType= DBType.RELATIONAL,colType = XmlBuild.ColType.UNDERLINE)
public class Demo extends BaseVO {

    private Long id;

    private String name;

}

```

```
#xml创建模式 分为：创建并覆盖、增量、不添加
codingapi.test.mode=addition
#xml数据位置
codingapi.test.outPath=${user.dir}/xml
```



## 如何使用？

1、配置maven

```
<dependency>
      <groupId>com.codingapi</groupId>
      <artifactId>codingapi-test</artifactId>
      <version>0.0.2</version>
</dependency>
```

2、配置Test类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
// 单元测试的profile
@ActiveProfiles("test")
@Slf4j
// 增加监听
@TestExecutionListeners({JunitMethodListener.class,
        DependencyInjectionTestExecutionListener.class})
public class DemoServiceTest {

}
```

3、配置application文件

```
codingapi.test.mode=addition
codingapi.test.outPath=${user.dir}/xml
```


4、demo地址

[https://github.com/1991wangliang/ci-demo](https://github.com/1991wangliang/ci-demo)
