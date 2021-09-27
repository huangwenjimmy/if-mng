# 接口管理

## 使用场景

* sql书写较多的应用，即sql驱动型服务应用，承认以SQL为中心

* 服务频繁更新，业务逻辑、算法等频繁调整的服务

* 数据模型支持Pojo，也支持Map/List 这种快速模型，更建议后者（支持快速开发，但开发规范需要在可控范围）

## 项目引入说明

1、引入sml-all-web.jar ，jdk1.6及以上，调整配置文件sml.properties(放在classpath目录下即可)

``` properties
sml.ioc.scan=default
sml.source.defJt=oracle://username:password@192.168.0.2:1521/default/dbid.sid?ctl:keepalive=true&maxActive=30
sml.redisUrl=redis://password@192.168.1.2:6379/6?maxTotal=200&maxActive=10

sml.support.init.model=update
```
* 数据源配置(必选项):sml.source.{dbId}={dbType}://{username}:{password}@{ip}:{port}/{poolType}/{serverName[.sid]}?uri-params
``` descr
dbId:数据源id，默认需要`defJt` 框架数据源
dbType:数据库类型【oracle|mysql|gbase|mariadb|sqlserver|db2|sybase|postgresql|sqlite|h2|hive】关系型数据库可做为配置-管理源
poolType:连接池类型【dbcp,dbcp2,druid,c3p0,hikari,default】，配置`default` 默认从系统classpath读取存在的连接池
uri-params:连接池控制参数，带`ctl:`为控制参数 `ctl:jdbcUrl`:原始的jdbcurl,`ctl:keepalive`:保持连接校验等，其它默认拼在jdbcUrl后面
serverName[.sid]：为数据库名，oracle如果为sid则，加后.sid后缀
```
* redis配置(可选)用于接口缓存：redis://[{password}]@{ip}:{port}/{dbindex}?uri-params
> 支持单节点、集群、主备、Sharded四类配置
> 集群增加：?hostAndPorts=ip1:port1,ip2:port2
> 主备模式：?master=mymaster&hostAndPorts=ip1:port1,ip2:port2

* 配置项
> sml.ioc.scan  扫描包路径，`default`默认即可
> sml.support.init.model  `update`:创建接口表并初始化管理接口

2.1、 传统war应用 web.xml，需要配置load-on-startup初始化加载sml.properties

```xml
        <!-- sml-Servlet -->
        <servlet> 
                <servlet-name>sml-servlet</servlet-name>
                <servlet-class>org.hw.sml.manager.SmlServlet</servlet-class>
                <load-on-startup>0</load-on-startup>
        </servlet>
        <servlet-mapping>
                <servlet-name>sml-servlet</servlet-name>
                <url-pattern>/sml/*</url-pattern>
        </servlet-mapping>
```

2.2、spring-boot系引入

``` java
	@Bean
	public ServletRegistrationBean<SmlServlet> smlServletRegistrationBean(){
		ServletRegistrationBean<SmlServlet> srb = new ServletRegistrationBean<SmlServlet>(new SmlServlet(),"/sml/*");
		srb.setLoadOnStartup(0);
		return srb;
	}
```

## 接口配置说明

### 表结构
* `id`：主键，唯一即可，建议采集     [项目]-[模块]-[描述]-[类型:查询|更新]。方便统一管理

* `mainsql`: sql编写区域 引擎语法标记（指令） `isEmpty`,`isNotEmpty`,`select`,`jdbcType`,`if`,`smlParam`
```sql

<isEmpty property="参数名"> xxx </isEmpty>
<if test="['a','b'].contains('@param1')"> xxxx </if>
<select id="sql1">xxx</select>`  代码段复用  <included id="select"/>
and age=#age#     //##变量绑定 
and classes in( #{classes.split((',')} )  //#{}变量绑定支持表达示语言
order by $sidx$ $desc$  //$$ 替换
nulls ${nullslast.equals('true').elseif('last','first')}   //${}表达示结果替换

```
> 变量处理：
>
> #name#    ：预处理绑定，支持对象，及数组
>
> $name$     :  替换，仅支持对象
>
> #{name.split('-').get(0)}   : 预处理绑定变量，支持表达示语言
>
> ${name.split('-').get(0)}: 替换，支持表达示语言

* `condition_info`：参数信息，默认采用{参数名},{参数类型},{默认值},{描述}一行一参数配置，第一行可以以**#**打头紧跟列分隔符，如`#|`替换`,`为`|`

>参数类型：[`char`,`number`,`date`,`array-char`,`array-date`]
>
>默认值：支持表达示语言，如当前时间:`#{smlDateHelper.date()}`

* `rebuild_info`: 返回结果包装器，支持常见结构返回，集合|对象|分页对象|树等

* `cache_enabled`：缓存策略，0：无，1：缓存，2：异步缓存(适合做大屏类接口，对数据刷新要求较高场景)

* `cache_minutes`：缓存时长，正数单位分钟，负数单位秒

* `db_id`：数据源，参考sml.properties   sml.source.{dbId}=配置

* `describle`：接口描述

* `update_time`：更新时间

>使用样例参考系统初始化接口：id like 'if-%'

## api接口

### 查询 `{contextPath}/sml/query/{ifId}`

> 支持协议

* GET：适用于参数较少的场景

* POST :post分两类，form表单提交，body类提交。可按需要选择

> 查询参数约定

* 已配置参数类型支持[`char`,`number`,`date`,`array-char`,`array-date`],后两者一般用于 sql  in   中

* 管理类参数：`FLUSHCACHE`=true|false 用于清除接口配置及数据缓存，`RECACHE`=true|false 用于刷新数据缓存

* body放参数特殊类：`_bodyTypeSeparate_`|`_bodyType_` 三类[map,list,char]  默认map,list时requstBody为集合值，char时requestBody可为任意字符串。list和char是会自动生成一个查询参数`listIds`,list默认以逗号拼接

* 特殊类参数：`condition_{operator}_{fieldName}[@fieldType]`

1、`condition_`  样例  condition_eq_create_time@date，其中

operator: 支持`eq`(=),  `ge`(>=),`gt`(>),`lt`(<),`le`(<=),`ne`(!=),`like`(like),`notlike`(not like),`in`(in),`notin`(not in) |对于in类，value值默认用`,`号分隔. or条件拼接    可用`condition_or1eq_field1`

value特殊类：  sql://打头支持

> sql://ori:select id from xxxx

> sql://base64:c2VsZWN0IGlkIGZyb20geHh4eA==


### 更新 `{contextPath}/sml/update/{ifId}`

> 仅支持POST协议，form表单提交或body提交

> 参数跟查询类同理

## 设计理念

* 简单配置，统一配置，采集URI方式来统一管理配置信息，如redis|datasource，达到配置项大量减少
* 表达示语言，通过反射来解释表达示语言，并借用js ProtoType模式来给表达示内对象新增方法及属性，更多的灵活性，开放性



