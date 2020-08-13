## 数据库迁移最佳实践 - 使用flyway

> 意译自：https://dbabulletin.com/index.php/2018/03/29/best-practices-using-flyway-for-database-migrations/

Flyway 和 Liquibase 都是 Java 项目中常用的 DB migration 工具，这里只讨论flyway。

### 1、flyway

- 官方地址：https://flywaydb.org/
- 官方文档：https://flywaydb.org/documentation
- 配置参数：https://flywaydb.org/documentation/configfiles

flyway 需要在 DB 中先创建一个 metdata 表 (缺省表名为 `flyway_schema_history`), 在该表中保存着每次 migration 的记录, 记录包含 migration 脚本的版本号和 SQL 脚本的 checksum 值. 当一个新的 SQL 脚本被扫描到后, Flyway 解析该 SQL 脚本的版本号, 并和 metadata 表已 apply 的的 migration 对比, 如果该 SQL 脚本版本更新的话, 将在指定的 DB 上执行该 SQL 文件, 否则跳过该 SQL 文件.

### 2、根据团队与项目分支策略使用flyway

根据团队情况的不同，以及项目分支使用的情况不同，flyway的使用方式会不一样。

#### 2.1、如果团队共同使用一个数据库并且有独立的DBA

如果团队所有的开发者使用一个开发数据库，并且有独立的DBA来处理数据库的变更，并且不使用分支开发，那么可以很好地使用flyway官网所提供的默认配置和版本编号方案，对delta文件使用整数的版本号，如：

- `V1__Update_1.sql`
- `V2__Update_2.sql`
- 等等

![](https://raw.githubusercontent.com/clixiang/notes/master/images/2020/Branching-Flyway-Trunk.png)

使用这种方式，最好将所有的migration都提交到主干，DBA要处理冲突，并且确保在应用新的delta文件时数据库操作不会失败，同时还要负责迁移回滚，并在出现问题时清理数据库。

也即，这种方式适用于比较简单的环境以及团队情况。

#### 2.2、如果允许多人可以修改数据库

如果团队中允许多人可以同时修改数据库，并且他们分别独立开发不同的功能，那么需要有一个严格的流程来处理数据库migration。当团队成员有权限去修改数据库，那么他们应该要很小心地去维护delta文件，要遵循一下的原则。

**1）每个开发人员应该有自己的数据库**

这一点至关重要。由于数据库的更改可能会破坏其他人的工作，因此每个开发人员都应该有一个数据库的个人副本，这可以在本地的开发环境下搭建。

**2）不同的功能在不同的分支下开发**

开发人员在开发某个功能的时候，应该在一个独立的分支上进行代码开发以及数据库变更的维护。并且要注意的是如果多人同时开发一个功能，那么应该在同一个分支中工作。

开发人员在开发的分支下，要负责数据库更改、回滚、失败清理的操作。这会使用到flyway的相关命令。

**3）使用时间作为版本号，而不使用简单的整数**

如果所有的数据库迁移文件使用简单的整数递增版本号（正如flyway官网建议的那样），那么分支合并到主干的时候，有可能会产生冲突，这样的话开发人员在创建迁移文件的新版本的时候，就需要经常相互检查。另外一种方法是对迁移文件的版本号使用时间戳，而不是简单递增的整数，这样就大大减少了冲突的可能性。

flyway只识别整数的版本号，因此时间要转换为整数。如“12/30/2016 12:30:55.282”可转换为20161230123055282。

![](https://raw.githubusercontent.com/clixiang/notes/master/images/2020/Branching-Flyway-Branch.png)

**4）允许flyway不按顺序执行迁移**

默认情况下，flyway执行迁移的时候，会忽略比已经应用的迁移版本老的迁移文件。如果较大版本号的迁移文件已经执行，那么较旧的版本号将不会执行，这会带来问题。因为我们采用不同的分支开发，不同的分支的版本号会存在版本号不按顺序的情况。

为了解决这种问题，在flyway中可以配置outOfOrder选项。在SpringBoot中，可以配置 `flyway.out-of-order=true`。参考flyway的文档：https://flywaydb.org/documentation/commandline/migrate#outOfOrder

**5）使用持续集成的环境合成所有的数据库更改**

一个好的做法是有一个专门的环境，用来执行所有的数据库更改操作。比如可以使用开发环境、测试环境等。通常一个环境会跟一个分支相关联。如果有一个专门执行CI构建的专用环境，那么就使用该环境来执行flyway更新操作。

**6）在合并数据库修改时执行review**

在合并分支前，开发人员需要检查其他人的数据库迁移文件，以防破坏别人的修改。这是很重要的一点。 

**7）如果可行，将不同分支的多个更改合并到一个增量文件**

这步可选，它有助于维护更干净的部署代码。假设在发布之前，已经累积了100个迁移文件，这些迁移文件将会按顺序应用到生产环境，那么在一个迁移文件中维护所有的更改将会更容易，为此，在将各个分支合并到主干之前，应该将多个分支中的迁移文件手动合并到一个将被推送到主干的迁移文件中。

这个方法并不总是可行的。因为它不能很好地处理CI和CD过程。不过如果在发布前合并所有分支的迁移文件一次，或者有一个专门的DBA团队不介意进行这种操作，那么从长远来看，当迁移文件的数量在几年后变得很多的时候，这种方法将会节省时间和精力。

### 3、保证迁移文件幂等

每个迁移文件应该要做到可以重复执行，并且每次执行的结果都是一样的（幂等操作）。

这意味着每个迁移脚本文件中的每个操作之前，都应该有一个检查是否已存在的操作，或者反转更改的操作。

检查是否存在是可取的操作，但并不是能很容易做这种检查。例如MySQL不允许在过程之外使用if-else语句，尽管有一些变通的办法，但是他们不能很好地处理H2单元测试。

MySQL常见的幂等方法是创建一个执行迁移的存储过程，然后删除该存储过程。存储过程将使用IF-ELSE检查现有对象。示例如下：

```sql
DELIMITER $$

DROP PROCEDURE IF EXISTS upgrade_database_1_0_to_2_0 $$

CREATE PROCEDURE upgrade_database_1_0_to_2_0()

BEGIN

-- rename a table safely

IF NOT EXISTS( (SELECT * FROM information_schema.COLUMNS WHERE TABLE_SCHEMA=DATABASE()

AND TABLE_NAME='my_old_table_name') ) THEN

RENAME TABLE

my_old_table_name TO my_new_table_name,

END IF;

-- add a column safely

IF NOT EXISTS( (SELECT * FROM information_schema.COLUMNS WHERE TABLE_SCHEMA=DATABASE()

AND COLUMN_NAME='my_additional_column' AND TABLE_NAME='my_table_name') ) THEN

ALTER TABLE my_table_name ADD my_additional_column varchar(2048) NOT NULL DEFAULT '';

END IF;

END $$

CALL upgrade_database_1_0_to_2_0() $$

DELIMITER ;
```

### 4、基准版本

在开始的时候，可以把现有数据库设置为基准版本，然后flyway再基于此版本应用所有的迁移。基准版本可以从生产环境的数据库的DDL开始，这样可以使用flyway从头开始创建与生产数据库一致的环境，从而实现DB的持续集成。

为了不对生产环境的数据库产生破坏，当在开发环境/测试环境建立基准版本并测试所有的迁移后，在预发布环境再进行一次迁移测试。

即使在项目开始使用DB的时候就使用了flyway，但是在以后也会将现有的数据库建立基准版本，这样就可以避免检查以及应用多年来建立的数百次迁移版本，这可以参考flyway官方的流程：https://flywaydb.org/documentation/existing.html

### 5、在SpringBoot中使用flyway

SpringBoot提供了对flyway的支持，开箱即用。

`pom.xml` 配置：

```xml
        <plugins>
            <plugin>
                <groupId>org.flywaydb</groupId>
                <artifactId>flyway-maven-plugin</artifactId>
                <version>6.5.4</version>
                <configuration>
                    <url>jdbc:mysql://10.2.0.3:3306/jops</url>
                    <user>jops</user>
                    <password>jops</password>
                </configuration>
            </plugin>
        </plugins>
```

执行：

```bash
$ mvn flyway:migrate
```

在应用启动的时候，flyway将会被自动调用，如果使用了H2进行单元测试，也将会调用flyway进行H2初始化。

flyway将会查找classpath下的迁移文件（delta 脚本），然后按照顺序执行。

flyway可以在 `application.properteis` 或者 `application.yml` 中配置，可以参考： https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html#data-migration-properties。

```ini
## 设定 db source 属性
spring.datasource.url=jdbc:mysql://localhost:3306/world
spring.datasource.username=root
spring.datasource.password=root

## 设定 flyway 属性 
# flyway 的 clean 命令会删除指定 schema 下的所有 table, 杀伤力太大了, 应该禁掉.
spring.flyway.cleanDisabled = true 
# 启用或禁用 flyway
spring.flyway.enabled = true
# 设定 SQL 脚本的目录,多个路径使用逗号分隔, 比如取值为 classpath:db/migration,filesystem:/sql-migrations
spring.flyway.locations =classpath:db/migration
# 如果指定 schema 包含了其他表,但没有 flyway schema history 表的话, 在执行 flyway migrate 命令之前, 必须先执行 flyway baseline 命令.
# 设置 spring.flyway.baseline-on-migrate 为 true 后, flyway 将在需要 baseline 的时候, 自动执行一次 baseline. 
spring.flyway.baselineOnMigrate=true
# 指定 baseline 的版本号,缺省值为 1, 低于该版本号的 SQL 文件, migrate 的时候被忽略. 
spring.flyway.baselineVersion=1 
# Encoding of SQL migrations (default: UTF-8) 
spring.flyway.encoding=
# 设定 flyway 的 metadata 表名, 缺省为 flyway_schema_history
spring.flyway.table=flyway_schema_history_myapp
spring.flyway.outOfOrder=true 
# 需要 flyway 管控的 schema list, 缺省的话, 使用的时 dbsource.connection直连上的那个 schema, 可以指定多个schema, 但仅会在第一个schema下建立 metadata 表, 也仅在第一个schema应用migration sql 脚本. 但flyway Clean 命令会依次在这些schema下都执行一遍.
#spring.flyway.schemas=
```

### 6、版本化以及可重复的迁移

flyway的迁移文件分为两类：

- Versioned migration：版本化的迁移
- Repeatable migration： 可重复加载的迁移

版本化迁移文件默认前缀是大写字母 `V`，有唯一的版本号，并只会应用一次。

可重复加载的迁移，是指一旦版本迁移文件的checksum有变动，flyway就会重新应用该迁移文件，它并不用于版本更新。这类迁移总是在版本化迁移文件执行后，才会被执行。

默认情况下，迁移文件的命名规则如下图：

![](https://raw.githubusercontent.com/clixiang/notes/master/images/2020/flyway-version-file.png)

一般在以下情况会使用可重复的迁移：

- 重建索引、视图或者存储过程
- 添加权限
- 其他一些维护的情况

可重复加载的迁移，在处理多人编辑一个版本迁移文件引起的冲突问题尤其有用。

另外，如果想在每次应用flyway之后，执行相同的迁移后脚本，需要使用回调（callback）来替代执行可重复加载的迁移： https://flywaydb.org/documentation/callbacks.html。

flyway执行过的脚本，不要再去更改它，以免checksum检查不通过。

### 7、多实例执行flyway

flyway是“线程安全”的，如果生产环境中有很多个实例同时执行 flyway migrate（比如多个服务器节点都同时执行flyway），其中一个实例的migration会锁定 `flyway_schema_history` 表，使得其他的实例需要等待该实例执行完毕才会执行，所以不会发生冲突，为了避免重复迁移，请保证迁移文件是幂等的。

由于Flyway一次只锁定其中一个delta脚本的 `flyway_schema_history` 表，因此多个Flyway实例可以获取不同的迁移并按照迁移版本号定义的顺序实现它们。

### 8、生产环境中使用flyway

在生产环境中使用flyway之前，记得在预发布环境先测试一次，尤其是第一次在项目中引入flyway的时候。

- 要建立生产环境数据库的基准版本
- 禁止flyway的clean命令
- 配置允许 `out-of-order`
- 考虑为flyway设置一个专门的数据库连接
- 需要确定应用程序在生产环境中触发flyway迁移，还是DBA手动从命令行执行

一些企业会禁止在生产配置文件中存储密码，因此应用程序不能在这样的情况下执行flyway，不过DBA可以从命令行调用flyway，手动输入密码。


























