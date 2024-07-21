[TOC]

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

## Docker

&emsp;快速构建、运行、管理应用的工具。

### 镜像和容器

&emsp;当我们利用Docker安装应用时，Docker会自动搜索并下载应用镜像（image）。镜像不仅包含应用本身，还包含应用运行所需要的环境、配置、系统函数库。Docker会在运行镜像时创建一个隔离环境，称为容器（container）。

&emsp;**镜像仓库：**存储和管理镜像的平台，Docker官方维护了一个公共仓库：Docker Hub。

### 部署MySQL

&emsp;先停掉虚拟机中的MySQL，确保虚拟机已经安装Docker，且网络开通的情况下，执行下面命令即可安装MySQL：

```shell
docker run -d \
	--name mysql \
	-p 3306:3306 \
	-e TZ=Asia/Shanghai \
	-e MYSQL_ROOT_PASSWORD=123 \
	mysql
```

- **docker run：**创建并运行一个容器，**-d**是让容器在后台运行
- **--name mysql：**给容器起个名字，必须唯一
- **-p 3306:3306：**设置端口映射（宿主机端口映射到容器内端口，因为容器是隔离环境，不同容器可以相同端口，也有自己的ip，而外部无法直接访问容器，需要访问宿主机，由宿主机映射相应的容器）
- **-e KEY=VALUE：**设置环境变量
- **mysql：**指定运行的镜像的名字（一般是Repository:TAG）

#### 镜像命名规范

&emsp;镜像名称一般分两部分组成：**[repository]:[tag]**。

- 其中repository就是镜像名称
- tag是镜像的版本

&emsp;在没有指定tag时，默认是latest，代表最新版本的镜像

### 常见命令

- docker pull：从镜像仓库中拉取到本地镜像
- docker push：将本地镜像推送到镜像仓库
- docker build：编写自定义镜像，通过dockerfile构建本地镜像
- docker save：将本地镜像保存成文件.tar
- docker load：加载文件到本地镜像
- docker images：查看本地所以镜像
- docker rmi：删除本地镜像
- docker run：**创建并**运行本地镜像（本地镜像没有会去镜像仓库拉取）
- docker stop：停止容器内的进程（**容器还在**）
- docker start：开启停止的容器、
- docker ps：查看容器进程状态（ps：process state）
- docker exec：连接容器在容器内部执行命令

### 数据卷

&emsp;**数据卷（volume）**是一个**虚拟**目录，是容器内目录与宿主机目录之间映射的桥梁。方便操作容器内文件，或者迁移容器产生的数据。

#### 如何挂载数据卷？

- **docker run** 创建容器时，利用 **-v 数据卷名:容器内目录** 或 **-v 本地目录:容器内目录** 可以完成本地目录挂载
- 容器创建时，如果发现挂载的数据卷不存在时，会自动创建（匿名数据卷：目录层深，名字太长，不便于后续数据存储与迁移）
- 本地目录必须以“/"或”./“开头，如果直接以名称开头，会被识别为数据卷而非本地目录
- `-v mysql:/var/lib/mysql`会被识别为一个数据卷叫mysql
- `-v ./mysql:/var/lib/mysql`会被识别为当前目录下的mysql目录

#### 数据卷命令

- docker volume create：创建数据卷
- docker volume ls：查看所有数据卷
- docker volume rm：删除指定数据卷
- docker volume inspect：查看某个数据卷的详情
- docker volume prune：删除未使用的数据卷

### 自定义镜像

&emsp;镜像就是包含了应用程序、程序运行的系统函数库、运行配置等文件的文件包。构建镜像的过程其实就是把上述文件打包的过程。

#### 镜像结构

- 基础镜像（BaseImage）：应用依赖的系统函数库、环境、配置、文件等
- 层（Layer）：添加安装包、依赖、配置等，每次操作都形成新的一层
- 入口（EntryPoint）：镜像运行入口，一般是程序启动的脚本和参数

#### Dockerfile

&emsp;Dockerfile就是一个文本文件，其中包含一个个的**指令（Instruction）**，用指令来说明要执行什么操作来构建镜像。将来Docker可以根据Dockerfile帮我们构建镜像。常见指令如下：

|    指令    |                     说明                     |                             示例                             |
| :--------: | :------------------------------------------: | :----------------------------------------------------------: |
|    FROM    |                 指定基础镜像                 |                        FROM centos:6                         |
|    ENV     |        设置环境变量，可在后面指令使用        |                        ENV key value                         |
|    COPY    |         拷贝本地文件到镜像的指定目录         |                   COPY ./jrell.tar.gz /tmp                   |
|    RUN     |  执行Linux的shell命令，一般是安装过程的命令  | RUN tar -zxvf /tmp/jrell.tar.gz <br />&& EXPORTS path=/tmp/jrell:$path |
|   EXPOSE   | 指定容器运行时监听的端口，是给镜像使用者看的 |                         EXPOSE 8080                          |
| ENTRYPOINT |     镜像中应用的启动命令，容器运行时调用     |                 ENTRYPOINT java -jar xx.jar                  |

&emsp;我们可以基于Ubuntu基础镜像，利用Dockerfile描述镜像结构，也可以直接基于JDK为基础镜像，省略前面的步骤。

&emsp;当编写好了Dockerfile，可以利用下面命令来构建镜像：

```shell
docker build -t 镜像名 Dockerfile目录
```

eg：`docker build -t myImage:1.0 .`

- **-t**：是给镜像起名，格式依然是repository:tag的格式，不指定tag时，默认为latest
- **.** ：是指定Dockerfile所在目录，如果就在当前目录，则指定为“.”

### 容器网络互联

&emsp;默认情况下，所有容器都是以bridge方法连接到Docker的一个虚拟网桥上，这可能会导致同一个软件每次创建容器时分配的ip不一致。

&emsp;加入自定义网络的容器才可以通过容器名互相访问，Docker的网络操作命令如下：

|           命令            |           说明           |
| :-----------------------: | :----------------------: |
|   docker network create   |       创建一个网络       |
|     docker network ls     |       查看所有网络       |
|     docker network rm     |       删除指定网络       |
|   docker network prune    |     清楚未使用的网络     |
|  docker network connect   | 使指定容器连接加入某网络 |
| docker network disconnect | 使指定容器连接离开某网络 |
|  docker network inspect   |     查看网络详细信息     |

### 部署项目

#### 部署后端

- 首先把Java应用打包成jar包（通过maven的package），上传到虚拟机
- 上传准备好的Dockerfile
- 执行docker run命令构建并启动

#### 部署前端

- 准备好配置文件（nginx.conf）和静态资源
- 创建容器挂载目录并启动

Java应用要跟mysql互连，Java、mysql、nginx三个容器要在同一个网络，三者通过容器名互连。

### DockerCompose

&emsp;Docker Compose通过一个单独的**docker-compose.yml**模板文件（YAML格式）来定义一组相关联的应用容器，帮助我们实现多个相互关联的Docker容器的快速部署。

&emsp;docker compose的命令格式如下：
`docker compose [OPTIONS] [COMMAND]`

|   类型   | 参数或指令 |             说明             |
| :------: | :--------: | :--------------------------: |
| Options  |     -f     | 指定compose文件的路径和名称  |
| Options  |     -p     |       指定project名称        |
| Commands |     up     |  创建并启动所有service容器   |
| Commands |    down    | 停止并溢出所有容器、**网络** |
| Commands |     ps     |      列出所有启动的容器      |
| Commands |    logs    |      查看指定容器的日志      |
| Commands |    stop    |           停止容器           |
| Commands |   start    |           启动容器           |
| Commands |  restart   |           重启容器           |
| Commands |    top     |        查看运行的进程        |
| Commands |    exec    | 在指定的运行中容器中执行命令 |

