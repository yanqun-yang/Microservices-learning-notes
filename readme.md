# 微服务学习笔记

&emsp;微服务是一种软件架构风格，它是以专注于单一职责的很多小型项目为基础，组合出复杂的大型应用。

## MyBatisPlus

&emsp;MyBatisPlus官方提供了starter，其中集成了MyBatis和MyBatisPlus的所有功能，并且实现了自动装配效果。

### 使用MyBatisPlus基本步骤：

1. 引入MyBatisPlus依赖，代替MyBatis依赖
2. 定义Mapper接口并继承BaseMapper（extends BaseMapper\<User>指定泛型）

### 常见注解

&emsp;MyBatisPlus通过扫描实体类，并基于反射获取实体类信息作为数据库信息。

### Q&A

#### MybatisPlus是如何获取实现CRUD的数据库表信息的？

- 默认以类名驼峰转下划线作为表名
- 默认把名为id的字段作为主键
- 默认把变量名驼峰转下划线作为表的字段名

#### MyBatisPlus的常用注解有哪些？

- @TableName：指定表名称及全局配置
- @TableId：指定id字段及相关配置
- @TableField：指定普通字段及相关配置

#### IdType的常见类型有哪些？

&emsp;AUTO、ASSIGN_ID、INPUT

#### 使用@TableField的常见场景是？

- 成员变量名与数据库字段名不一致
- 成员变量名以is开头，且是布尔值（因为扫描会自动去除is，@TableField（“is_married”））
- 成员变量名与数据库关键字冲突（@TableField("\`order`")）
- 成员变量不是数据库字段（@TableField(exist = false)）

#### MybatisPlus使用的基本流程是什么？

1. 引入起步依赖
2. 自定义Mapper基础BaseMapper
3. 在实体类上添加注解声明 表信息
4. 在application.yml中根据需要添加配置

### 核心功能

#### 条件构造器

&emsp;MyBatisPlus支持各种复杂的where条件，可以满足日常开发的所有需求。

##### 用法：

- QueryWrapper和LambdaQueryWrapper通常用来构建select、delete、update的where条件部分
- UpdateWrapper和LambdaUpdateWrapper通常只有在set语句比较特殊才使用
- 尽量使用LambdaQueryWrapper和LambdaUpdateWrapper，避免硬编码 （即写死）。

#### 自定义SQL

&emsp;我们可以利用MyBatisPlus的Wrapper来构建复杂的Where条件，然后自己定义SQL语句中剩下的部分。

&emsp;因为用MyBatisPlus来写Where条件很简单，不想动态SQL那么繁琐，而且企业不让把数据库操作写在Server层，应该在Mapper层，所以一般就是复杂的Where条件先用MyBatisPlus构造好，再去调用Mapper层接口拼接完SQL语句进行数据库操作。

&emsp;在Mapper方法参数中用Param注解声明Wrapper变量名称，必须是ew（返回值 方法名(@Param("ew") LambdaQueryWrapper\<User> wrapper, @Param...)）

&emsp;自定义SQL语句中，用${ew.customSqlSegment}进行拼接

#### IService接口

&emsp;MP（MyBatisPlus）的Service接口使用流程：

- 自定义Service接口继承IService接口
- 自定义Service实现类，实现自定义接口并继承ServiceImpl类（继承时泛型需执行Mapper跟实体类）

#### IService批量新增

- 普通for循环插入：速度极差，每条操作都是一次网络请求跟一条SQL插入语句（n次网络请求+m次SQL语句插入）
- IService的批量插入：基于预编译批处理，性能不错，只需要一次网络请求就可以发送多条SQL语句（1次网络请求+m次SQL语句插入）
- 配置jdbc参数，开启rewriteBatchedStatements=true参数，多条插入SQL语句会重写成一条语句，只需一次网络请求跟一条SQL语句，性能最好（1次网络请求+1次SQL语句插入）

### 扩展功能

#### 代码生成

&emsp;代码生成插件（MyBatisPlus（二次元图片那个）），可根据数据库自动生成Controller、Server、Mapper，entry各种代码

#### DB静态工具

&emsp;有时会出现多个Service之间会相互调用，如果采用AutoWire自动注入的方式，相互注入就会出现循环依赖，静态工具的提供的方法跟Service差不多，不过因为是静态的，在调用的时候需要指定字节码（.class）

&emsp;一个用户有多个地址，先对用户进行分组，分组完得到Map\<userId, list\<address>>，这样的话插入数据的时候，插入用户，再去设置用户的地址。比插入用户，再从address中遍历所有属于该用户的地址要高效的多，不用每个用户都去遍历地址，而是在一开始就先把地址跟用户Map好。

### 逻辑删除

&emsp;逻辑删除就是基于代码逻辑模拟删除效果，但并不会真正删除数据。思路如下：

- 在表中添加一个字段标记数据是否被删除
- 当删除数据是把标记置为1/true
- 查询时只查询标记为0/false的数据

&emsp;MybatisPlus提供了逻辑删除功能，无需改变方法的调用方式，而是在底层帮我们自动修改CRUD的语句。我们要做的就是在application.yaml文件中配置逻辑删除的字段名称和值即可：

```yaml
mybatis-plus:
	global-config:
		db-config:
			logic-delete-field: flag #全局逻辑删除的实体字段名，字段类型可以是boolean、integer
			logic-delete-value: 1 # 逻辑已删除值（默认为1）
			logic-not-delete-value: 0 #逻辑未删除值（默认为0）
```

#### 优点&缺点

- 优点：数据保留了，对后续的用户数据分析、画像、留证有重要意义
- 缺点：
- - 会导致数据库表垃圾数据越来越多，影响查询效率
  - SQL中全都需要对逻辑删除字段做判断，影响查询效率

因此，不太推荐采用逻辑删除功能，如果数据不能删除，可以采用**把数据迁移到其他表**的办法。

### 枚举处理器

如何实现PO类中的枚举类型变量与数据库字段的转换？

1. 给枚举中的数据库对应value值添加@EnumValue注解
2. 在配置文件中配置统一的枚举处理器，实现类型转换

```yaml
mybatis-plus:
	configuration:
		default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
```

3. 在返回前端数据时，给枚举变量的某个属性加上@JsonValue，这样返回给前端的就不再是枚举对象的名称，而是加了注解的值

### Json处理器

&emsp;数据库中有Json类型的字段，在实体类中，需要定义一个对象来接收Json，在该对象上面添加注解`@TableField(typeHandler = JacksonTypeHandler.class)`并且在实体类上面添加注解`@TableName(autoResultMap = true)`，就可以实现数据库的Json字段自动转换为实体类中的对象各种数据。

