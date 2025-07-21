---
title: MyBatis总结
tags: 
  - MyBatis
date: 2022-02-14 20:08:54
categories:
  - MyBatis
---

## 介绍

Mybatis是一款ORM（）的框架

## MyBatis使用方法（以SpringBoot项目为例）

### 新建一个SpringBoot工程，以sca-system为例

![mybatis](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202203312034558.png)

sca-system工程下分为5个子工程

- sca-system-api

- sca-system-main

- sca-system-model

- sca-system-rest

- sca-system-service

mapper文件一般建在model工程的resources目录下

并且要在springboot启动类（SystemApplication.java）中配置

![启动类](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202203312036739.png)

@MapperScan()

这个注解表明扫描的是特定包下的mapper文件，一定要配置

然后在application.yml中配置mybatis的依赖

```yaml
mybatis:
  mapper-locations: classpath:mapper/*Mapper.xml
```

**配置文件到此告一段落**

### 开始编写xml

先建立一个mybaitis格式的xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.yim.mapper.SysUserMapper">

</mapper>
```

mapper标签表示这个xml文件对应哪个dao（持久层）

### 总结对应的标签

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.yim.mapper.SysUserMapper">
  <resultMap id="BaseResultMap" type="com.yim.dto.User.SysUserDTO">
    <id column="id" jdbcType="VARCHAR" property="id" />
    <result column="username" jdbcType="VARCHAR" property="username" />
    <result column="password" jdbcType="VARCHAR" property="password" />
  </resultMap>
  <sql id="Base_Column_List">
    id, username, password
  </sql>
  <select id="selectByPrimaryKey" parameterType="java.lang.String" resultMap="BaseResultMap">
    select 
    <include refid="Base_Column_List" />
    from sys_user
    where id = #{id,jdbcType=VARCHAR}
  </select>
  <select id="selectSysUser" resultMap="BaseResultMap">
    select
    <include refid="Base_Column_List" />
    from sys_user
  </select>
  <delete id="deleteByPrimaryKey" parameterType="java.lang.String">
    delete from sys_user
    where id = #{id,jdbcType=VARCHAR}
  </delete>
  <insert id="insert" keyColumn="id" keyProperty="id" parameterType="com.yim.dto.User.SysUserDTO" useGeneratedKeys="true">
    insert into sys_user (username, password)
    values (#{username,jdbcType=VARCHAR}, #{password,jdbcType=VARCHAR})
  </insert>
  <insert id="insertSelective" keyColumn="id" keyProperty="id" parameterType="com.yim.dto.User.SysUserDTO" useGeneratedKeys="true">
    insert into sys_user
    <trim prefix="(" suffix=")" suffixOverrides=",">
      <if test="username != null">
        username,
      </if>
      <if test="password != null">
        `password`,
      </if>
    </trim>
    <trim prefix="values (" suffix=")" suffixOverrides=",">
      <if test="username != null">
        #{username,jdbcType=VARCHAR},
      </if>
      <if test="password != null">
        #{password,jdbcType=VARCHAR},
      </if>
    </trim>
  </insert>
  <update id="updateByPrimaryKeySelective" parameterType="com.yim.dto.User.SysUserDTO">
    update sys_user
    <set>
      <if test="username != null">
        username = #{username,jdbcType=VARCHAR},
      </if>
      <if test="password != null">
        `password` = #{password,jdbcType=VARCHAR},
      </if>
    </set>
    where id = #{id,jdbcType=VARCHAR}
  </update>
  <update id="updateByPrimaryKey" parameterType="com.yim.dto.User.SysUserDTO">
    update sys_user
    set username = #{username,jdbcType=VARCHAR},
      password = #{password,jdbcType=VARCHAR}
    where id = #{id,jdbcType=VARCHAR}
  </update>
</mapper>
```

### 各标签的作用

#### 1.< resultMap>< /resultMap>

定义返回的参数列表

参数：

id表示一个表的主键id，**column**表示对应**实体类**的字段名称，**property**表示对应数据库的字段名称

#### 2.< select>< /select>

表示查询语句的标签

参数：

- id

表示这个语句的唯一标识，与dao中的方法名称对应

- parameterType

表示这个语句的参数类型，与dao中的方法的参数类型对应，也可以是对象，如java.lang.String；com.yim.pojo.SysUser

- parameterMap

表示这个语句的参数类型，service中传进一个map，map里面塞有对应名称的参数，xml中配置parameterMap

- resultType

表示返回的参数类型，用于查询一个参数时，如java.lang.String

- resultMap

表示返回的参数类型，用于查询多个参数，如BaseResultMap

我们在写sql的时候，要注意传入的参数写法是#{}，而不是${}，是因为jdbc在翻译的时候，$会直接把参数翻译出来，有可能会出现sql注入的情况

```sql
// This example creates a prepared statement, something like select * from teacher where name = ?; 
@Select("Select * from teacher where name = #{name}") 
Teacher selectTeachForGivenName(@Param("name") String name); 
// This example creates n inlined statement, something like select * from teacher where name = 'someName'; 
@Select("Select * from teacher where name = '${name}'") 
Teacher selectTeachForGivenName(@Param("name") String name);   
```

#### 3.< insert>< /insert>

表示插入语句的标签

参数：

- id

表示方法的唯一标识

- parameterType

表示返回参数的类型，如com.yim.dto.user.SysUserDTO

我们也可以在标签内写一些动态sql，这个在后面进行描述

#### 4.< update>< /update>

表示更新语句的标签

参数：

- id

表示方法的唯一标识

- parameterType

表示返回参数的类型，如com.yim.dto.user.SysUserDTO

#### 5.< delete>< /delete>

表示删除语句的标签

参数：

- id

表示方法的唯一标识

- parameterType

表示返回参数的类型，如com.yim.dto.user.SysUserDTO

**动态sql**

我们在更新数据或者插入数据的时候，经常会用到不知道要更新哪个字段的尴尬场面，mybatis提供了动态sql的标签，我们可以通过这种标签去操作我们需要修改的字段

**JDBCType对应数据库一览表**

| JdbcType    | Oracle         | MySql              |
| ----------- | -------------- | ------------------ |
| ARRAY       |                |                    |
| BIGINT      |                | BIGINT             |
| BINARY      |                |                    |
| BIT         |                | BIT                |
| BLOB        | BLOB           | BLOB               |
| BOOLEAN     |                |                    |
| CHAR        | CHAR           | CHAR               |
| CLOB        | CLOB           | 修改为TEXT         |
| DATE        | DATE           | DATE               |
| DECIMAL     | DECIMAL        | DECIMAL            |
| DOUBLE      | NUMBER         | DOUBLE             |
| FLOAT       | FLOAT          | FLOAT              |
| INTEGER     | INTEGER        | INTEGER            |
| LONGVARCHAR | LONG VARCHAR   |                    |
| NCHAR       | NCHAR          |                    |
| NCLOB       | NCLOB          |                    |
| NULL        |                |                    |
| NUMERIC     | NUMERIC/NUMBER | NUMERIC/           |
| SMALLINT    | SMALLINT       | SMALLINT           |
| TIME        |                | TIME               |
| TIMESTAMP   | TIMESTAMP      | TIMESTAMP/DATETIME |
| TINYINT     |                | TINYINT            |
| VARCHAR     | VARCHAR        | VARCHAR            |

## MyBatis 工作原理

- 加载mybatis全局配置文件（数据源、mapper映射文件等），解析配置文件，MyBatis基于XML配置文件生成Configuration，和一个个MappedStatement（包括了参数映射配置、动态SQL语句、结果映射配置），其对应着标签项。
- SqlSessionFactoryBuilder通过Configuration对象生成SqlSessionFactory，用来开启SqlSession。
- SqlSession对象完成和数据库的交互：a、用户程序调用mybatis接口层api（即Mapper接口中的方法）b、SqlSession通过调用api的Statement ID找到对应的MappedStatement对象c、通过Executor（负责动态SQL的生成和查询缓存的维护）将MappedStatement对象进行解析，sql参数转化、动态sql拼接，生成jdbc Statement对象d、JDBC执行sql。e、借助MappedStatement中的结果映射关系，将返回结果转化成HashMap、JavaBean等存储结构并返回。
