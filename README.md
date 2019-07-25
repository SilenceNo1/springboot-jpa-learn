# springboot-jpa-learn
前言
第一次使用 Spring JPA 的时候，感觉这东西简直就是神器，几乎不需要写什么关于数据库访问的代码一个基本的 CURD 的功能就出来了。下面我们就用一个例子来讲述以下 JPA 使用的基本操作。
新建项目，增加依赖
在 Intellij IDEA 里面新建一个空的 SpringBoot 项目。具体步骤参考
SpringBoot 的第一次邂逅。根据本样例的需求，我们要添加下面三个依赖
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>6.0.6</version>
</dependency>

准备数据库环境
为这个项目，我们专门新建一个 springboot_jpa 的数据库，并且给 springboot 用户授权
create database springboot_jpa;

grant all privileges on springboot_jpa.* to 'springboot'@'%' identified by 'springboot';

flush privileges;

项目配置
#通用数据源配置
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://10.110.2.56:3306/springboot_jpa?charset=utf8mb4&useSSL=false
spring.datasource.username=springboot
spring.datasource.password=springboot
# Hikari 数据源专用配置
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
# JPA 相关配置
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=create

这李前面的数据源配置和前文《SpringBoot 中使用 JDBC Templet》中的一样。后面的几个配置需要解释一下

spring.jpa.show-sql=true 配置在日志中打印出执行的 SQL 语句信息。
spring.jpa.hibernate.ddl-auto=create 配置指明在程序启动的时候要删除并且创建实体类对应的表。这个参数很危险，因为他会把对应的表删除掉然后重建。所以千万不要在生成环境中使用。只有在测试环境中，一开始初始化数据库结构的时候才能使用一次。
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect 。在 SrpingBoot 2.0 版本中，Hibernate 创建数据表的时候，默认的数据库存储引擎选择的是 MyISAM （之前好像是 InnoDB，这点比较诡异）。这个参数是在建表的时候，将默认的存储引擎切换为 InnoDB 用的。

建立第一个数据实体类
数据库实体类是一个 POJO Bean 对象。这里我们先建立一个 UserDO 的数据库实体。数据库实体的源码如下
package com.yanggaochao.springboot.learn.springbootjpalearn.security.domain.dao;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;

/**
 * 用户实体类
 *
 * @author 杨高超
 * @since 2018-03-12
 */
@Entity
@Table(name = "AUTH_USER")
public class UserDO {
    @Id
    private Long id;
    @Column(length = 32)
    private String name;
    @Column(length = 32)
    private String account;
    @Column(length = 64)
    private String pwd;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAccount() {
        return account;
    }

    public void setAccount(String account) {
        this.account = account;
    }

    public String getPwd() {
        return pwd;
    }

    public void setPwd(String pwd) {
        this.pwd = pwd;
    }
}

其中：

@Entity 是一个必选的注解，声明这个类对应了一个数据库表。
@Table(name = "AUTH_USER") 是一个可选的注解。声明了数据库实体对应的表信息。包括表名称、索引信息等。这里声明这个实体类对应的表名是 AUTH_USER。如果没有指定，则表名和实体的名称保持一致。
@Id 注解声明了实体唯一标识对应的属性。
@Column(length = 32) 用来声明实体属性的表字段的定义。默认的实体每个属性都对应了表的一个字段。字段的名称默认和属性名称保持一致（并不一定相等）。字段的类型根据实体属性类型自动推断。这里主要是声明了字符字段的长度。如果不这么声明，则系统会采用 255 作为该字段的长度

以上配置全部正确，则这个时候运行这个项目，我们就可以看到日志中如下的内容：
Hibernate: drop table if exists auth_user
Hibernate: create table auth_user (id bigint not null, account varchar(32), name varchar(32), pwd varchar(64), primary key (id)) engine=InnoDB

系统自动将数据表给我们建好了。在数据库中查看表及表结构






springboot_jpa 表及表结构

以上过程和我们前使用 Hibernate 的过程基本类似，无论是数据库实体的声明还是表的自动创建。下面我们才正式进入 Spring Data JPA 的世界，来看一看他有什么惊艳的表现
实现一个持久层服务
在 Spring Data JPA 的世界里，实现一个持久层的服务是一个非常简单的事情。以上面的 UserDO 实体对象为例，我们要实现一个增加、删除、修改、查询功能的持久层服务，那么我只需要声明一个接口，这个接口继承
org.springframework.data.repository.Repository<T, ID>  接口或者他的子接口就行。这里为了功能的完备，我们继承了 org.springframework.data.jpa.repository.JpaRepository<T, ID> 接口。其中 T 是数据库实体类，ID 是数据库实体类的主键。
然后再简单的在这个接口上增加一个 @Repository 注解就结束了。
package com.yanggaochao.springboot.learn.springbootjpalearn.security.dao;


import com.yanggaochao.springboot.learn.springbootjpalearn.security.domain.dao.UserDO;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

/**
 * 用户服务数据接口类
 *
 * @author 杨高超
 * @since 2018-03-12
 */

@Repository
package com.yanggaochao.springboot.learn.springbootjpalearn.security.dao;


import com.yanggaochao.springboot.learn.springbootjpalearn.security.domain.dao.UserDO;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

/**
 * 用户服务数据接口类
 *
 * @author 杨高超
 * @since 2018-03-12
 */

@Repository
public interface UserDao extends JpaRepository<UserDO, Long> {
}

一行代码也不用写。那么针对 UserDO 这个实体类，我们已经拥有下下面的功能






UserDao 保存实体功能






UserDao 保存实体删除功能






UserDao 查询实体删除功能

例如，我们用下面的代码就将一些用户实体保存到数据库中了。
package com.yanggaochao.springboot.learn.springbootjpalearn;

import com.yanggaochao.springboot.learn.springbootjpalearn.security.dao.UserDao;
import com.yanggaochao.springboot.learn.springbootjpalearn.security.domain.dao.UserDO;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.Optional;

@RunWith(SpringRunner.class)
@SpringBootTest
public class UserDOTest {

    @Autowired
    private UserDao userDao;

    @Before
    public void before() {
        UserDO userDO = new UserDO();
        userDO.setId(1L);
        userDO.setName("风清扬");
        userDO.setAccount("fengqy");
        userDO.setPwd("123456");
        userDao.save(userDO);
        userDO = new UserDO();
        userDO.setId(3L);
        userDO.setName("东方不败");
        userDO.setAccount("bubai");
        userDO.setPwd("123456");
        userDao.save(userDO);
        userDO.setId(5L);
        userDO.setName("向问天");
        userDO.setAccount("wentian");
        userDO.setPwd("123456");
        userDao.save(userDO);
    }
    @Test
    public void testAdd() {
        UserDO userDO = new UserDO();
        userDO.setId(2L);
        userDO.setName("任我行");
        userDO.setAccount("renwox");
        userDO.setPwd("123456");
        userDao.save(userDO);
        userDO = new UserDO();
        userDO.setId(4L);
        userDO.setName("令狐冲");
        userDO.setAccount("linghuc");
        userDO.setPwd("123456");
        userDao.save(userDO);
    }

    @After
    public void after() {
        userDao.deleteById(1L);
        userDao.deleteById(3L);
        userDao.deleteById(5L);
    }

}

这个是采用 Junit 来执行测试用例。@Before 注解在测试用例之前执行准备的代码。这里先插入三个用户信息。 执行执行这个测试，完成后，查看数据库就可以看到数据库中有了 5 条记录：





数据库记录

我们还可以通过测试用例验证通过标识查找对象功能，查询所有数据功能的正确性，查询功能甚至可以进行排序和分页
    @Test
    public void testLocate() {
        Optional<UserDO> userDOOptional = userDao.findById(1L);
        if (userDOOptional.isPresent()) {
            UserDO userDO = userDOOptional.get();
            System.out.println("name = " + userDO.getName());
            System.out.println("account = " + userDO.getAccount());
        }
    }

    @Test
    public void testFindAll() {
        List<UserDO> userDOList = userDao.findAll(new Sort(Sort.Direction.DESC,"account"));
        for (UserDO userDO : userDOList) {
            System.out.println("name = " + userDO.getName());
            System.out.println("account = " + userDO.getAccount());
        }
    }

可以看到，我们所做的全部事情仅仅是在 SpingBoot 工程里面增加数据库配置信息，声明一个 UserDO 的数据库实体对象，然后声明了一个持久层的接口，改接口继承自 org.springframework.data.jpa.repository.JpaRepository<T, ID> 接口。然后，系统就自动拥有了丰富的增加、删除、修改、查询功能。查询功能甚至还拥有了排序和分页的功能。
这就是 JPA 的强大之处。除了这些接口外，用户还会有其他的一些需求， JPA 也一样可以满足你的需求。
扩展查询
从上面的截图 “UserDao 查询实体删除功能” 中，我们可以看到，查询功能是不尽人意的，很多我们想要的查询功能还没有。不过放心。JPA 有非常方便和优雅的方式来解决
根据属性来查询
如果想要根据实体的某个属性来进行查询我们可以在 UserDao 接口中进行接口声明。例如，如果我们想根据实体的 account 这个属性来进行查询（在登录功能的时候可能会用到）。我们在 com.yanggaochao.springboot.learn.springbootjpalearn.security.dao.UserDao 中增加一个接口声明就可以了
  UserDO findByAccount(String account);

然后增加一个测试用例
@Test
    public void testFindByAccount() {
        UserDO userDO = userDao.findByAccount("wentian");
        if (userDO != null) {
            System.out.println("name = " + userDO.getName());
            System.out.println("account = " + userDO.getAccount());
        }
    }

运行之后，会在日志中打印出
name = 向问天
account = wentian

这种方式非常强大，不经能够支持单个属性，还能支持多个属性组合。例如如果我们想查找账号和密码同时满足查询条件的接口。那么我们在 UserDao 接口中声明
UserDO findByAccountAndPwd(String account, String pwd);

再例如，我们要查询 id 大于某个条件的用户列表，则可以声明如下的接口
List<UserDO> findAllByIdGreaterThan(Long id);

这个语句结构可以用下面的表来说明





JPA 关键字说明

自定义查询
如果上述的情况还无法满足需要。那么我们就可以通过通过 import org.springframework.data.jpa.repository.Query 注解来解决这个问题。例如我们想查询名称等于某两个名字的所有用户列表，则声明如下的接口即可
@Query("SELECT O FROM UserDO O WHERE O.name = :name1  OR O.name = :name2 ")
List<UserDO> findTwoName(@Param("name1") String name1, @Param("name2") String name2);

这里是用 PQL 的语法来定义一个查询。其中两个参数名字有语句中的 : 后面的支付来决定
如果你习惯编写 SQL 语句来完成查询，还可以在用下面的方式实现
@Query(nativeQuery = true, value = "SELECT * FROM AUTH_USER WHERE name = :name1  OR name = :name2 ")
List<UserDO> findSQL(@Param("name1") String name1, @Param("name2") String name2);

这里在 @Query 注解中增加一个 nativeQuery = true 的属性，就可以采用原生 SQL 语句的方式来编写查询。
联合主键
从 org.springframework.data.jpa.repository.JpaRepository<T, ID> 接口定义来看，数据实体的主键是一个单独的对象，那么如果一个数据库的表的主键是两个或者两个以上字段联合组成的怎么解决呢。
我们扩充一下前面的场景。假如我们有一个角色 Role 对象，有两个属性 一个 id ，一个 name ，对应了 auth_role 数据表，同时有一个角色用户关系对象 RoleUser，说明角色和用户对应关系，有两个属性 roleId,userId 对应 auth_role_user 表。那么我们需要声明一个 RoleDO 对象如下
package com.yanggaochao.springboot.learn.springbootjpalearn.security.domain.dao;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;

/**
 * 角色实体类
 *
 * @author 杨高超
 * @since 2018-03-12
 */
@Entity
@Table(name = "AUTH_ROLE")
public class RoleDO {
    @Id
    private Long id;
    @Column(length = 32)
    private String name;
    @Column(length = 64)
    private String note;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getNote() {
        return note;
    }

    public void setNote(String note) {
        this.note = note;
    }
}

对于有多个属性作为联合主键的情况，我们一般要新建一个单独的主键类，他的属性和数据库实体主键的字段一样，要实现 java.io.Serializable 接口，类声明如下
package com.yanggaochao.springboot.learn.springbootjpalearn.security.domain.dao;

import java.io.Serializable;

/**
 * 联合主键对象
 *
 * @author 杨高超
 * @since 2018-03-12
 */
public class RoleUserId implements Serializable {
    private Long roleId;
    private Long userId;
}


同样的，我们声明一个 RoleUserDO 对象如下
package com.yanggaochao.springboot.learn.springbootjpalearn.security.domain.dao;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.IdClass;
import javax.persistence.Table;
import java.io.Serializable;

/**
 * 角色用户关系实体类
 *
 * @author 杨高超
 * @since 2018-03-12
 */
@Entity
@IdClass(RoleUserId.class)
@Table(name = "AUTH_ROLE_USER")
public class RoleUserDO  {
    @Id
    private Long roleId;
    @Id
    private Long userId;

    public Long getRoleId() {
        return roleId;
    }

    public void setRoleId(Long roleId) {
        this.roleId = roleId;
    }

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }
}


这里因为数据实体类和数据实体主键类的属性一样，所以我们可以删除掉这个数据实体主键类，然后将数据实体类的主键类声明为自己即可。当然，自己也要实现 java.io.Serializable 接口。
这样，我们如果要查询某个角色下的所有用户列表，就可以声明如下的接口
@Query("SELECT U FROM UserDO U ,RoleUserDO RU WHERE U.id = RU.userId AND RU.roleId = :roleId")
List<UserDO> findUsersByRole(@Param("roleId") Long roleId);

当然了，这种情况下，我们会看到系统自动建立了 AUTH_ROLE 和 AUTH_ROLE_USER 表。他们的表结构如下所示





auth_role 和 auth_role_user 表结构

注意这里 auth_role_user 表中，属性名 userId 转换为了 user_id， roleId 转换为了 role_id.
如果我们要用 SQL 语句的方式实现上面的功能，那么我们就把这个接口声明修改为下面的形式。
@Query("SELECT U.* FROM AUTH_USER U ,AUTH_ROLE_USER RU WHERE U.id = RU.user_id AND RU.role_id = :roleId")
List<UserDO> findUsersByRole(@Param("roleId") Long roleId);

后记
这个样例基本上讲述了 JPA 使用过程中的一些细节。我们可以看出。使用 JPA 来完成关于关系数据库增删改查的功能是非常的方便快捷的。所有代码已经上传到 github 的仓库  springboot-jpa-learn 上了
 
