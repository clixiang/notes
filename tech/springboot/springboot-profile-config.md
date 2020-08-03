## SpringBoot环境配置

在一个项目中，开发环境、测试环境、生产环境等的配置一般是不同的，如何能方便地在不同的环境中，切换不同的配置文件？

Spring提供了profile机制，可以实现这一点。

```
Spring Profiles provide a way to segregate parts of your application configuration and make it be available only in certain environments.
```

### 1、Spring配置文件加载

在了解profile机制之前，先了解下Spring是怎样加载配置文件的。

- Spring参考文档： https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config

#### 1）application.properties

Spring默认会从以下路径去加载一个叫 `application.properties` 或 `application.yml` 的配置文件，并加入到Spring的 `Environment`：

-  当前目录下的 `config/` 子目录
-  当前目录
-  classpath:/config/
-  chasspath:/

以上位置从上到下，优先级从高到低。所有位置的文件都会被加载，高优先级配置内容会覆盖低优先级配置内容。如果高优先级配置文件属性与低优先级配置文件存在不冲突的属性，则会共同存在，互补配置。

#### 2）application-{profile}.properties

除了 application.properties 文件以外，还可以按以下命名格式定义配置文件：

```
application-{profile}.properties
```

指定 profile 的配置优先于默认配置。

对于一个项目，开发、测试、生产环境的配置参数不一样。针对这些不同的环境，分别定义不同的配置文件，在执行的时候就可以指定对应的配置文件：

- 开发环境： application-dev.properties
- 测试环境： application-test.properties
- 生产环境： application-prod.properties

#### 3）配置加载流程

Spring 默认加载配置文件的位置是 `classpath:/, classpath:/config/, file:./, file:./config/`，文件名为 application。其主要流程是：

- 从环境中获取指定 profile 集合，包括 active 和 include
- 加载 application.yaml(或 application.properties) 文件
- 读取配置文件转为 PropertySource 对象，并获取 profiles.active 和 profiles.include 配置项，加入到 profile 集合
- 遍历 profile 集合，加载 application-${profile}.properties, 重复上一步骤，直至配置完所有 profile

### 2、Active profile配置

指定 profile 的配置优先于默认配置，故可以把与环境无关的配置放在 application. properties 中，环境相关的配置写在 `application-${profile}.properties` 中。其中激活 profile 指定有三种方式：

- 命令行启动参数设置 `--spring.profiles.active={profile}`
- Java 环境或系统环境变量 `spring.profiles.active={profile}`
- application.yaml 中 `spring.profiles.active` 配置项

优先级从前到后，命令行指定的 profile 优先级最高，其次是环境变量，最后才是配置文件。前两种一般是本地开发测试时使用，大部分项目要求在打包时就需要生成特定环境的构件，所以需要在 application.properties 中指定 profile。

下面主要讲述第三种方式，如何在 application.properties 中配置使用动态profile，可以按以下步骤操作。

#### 1）配置pom.xml

可根据不同的环境，自行定义不同的profile。

```xml
  <profiles>
    <profile>
        <id>dev</id>
        <properties>
            <profileActive>dev</profileActive>
        </properties>
        <activation>
            <!-- 默认生效dev -->
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
    <profile>
        <id>test</id>
        <properties>
            <profileActive>test</profileActive>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <profileActive>prod</profileActive>
        </properties>
    </profile>
  </profiles>
```

#### 2）配置application.properties

```ini
spring.profiles.active=@profileActive@
```

注：SpringBoot默认的占位符是`@@`，而不是`${}`。

#### 3）指定profile

```bash
# 本地开发测试时，执行
# 默认会启用dev的profile
$ mvn spring-boot:run

# 打包，可用-P指定profile
# 查看target/classes/application.properties可以看到@profileActive@会被替换为prod
$ mvn clean package -P prod -Dmaven.test.skip=true

# 如已经指定环境打包，在执行的时候，又想指定调用的环境，可以在命令行指定
# 以下2种方式都可以
$ java -jar app.jar --spring.profiles.active=dev
$ java -Dspring.profiles.active=dev -jar app.jar
```

### 3、Profile include配置

在实际的项目中，会出现不同的环境，会有部分相同的配置。例如开发环境与测试环境的数据库连接相同。这时，应该把不同环境相同配置的部分抽出来成为一个profile，再通过include的方式引用。

例如：

```ini
# application-dev.properties
spring.profiles.include=database-test,redis-test

# application-test.properties
spring.profiles.include=database-test,redis-test

# application-database-test.properties
database.url=xxxxxx

# application-redis-test.properties
redis.host=127.0.0.1
```

### 4、读取外部配置文件

如果把application-*.properties打进包里，那每次修改配置，都要重新打包。

springboot读取外部配置文件的方法，如下优先级：

- 在执行命令的目录下建config文件夹。（在jar包的同一目录下建config文件夹，执行命令需要在jar包目录下才行），然后把配置文件放到这个文件夹下
- 直接把配置文件放到jar包的同级目录
- 在classpath下建一个config文件夹，然后把配置文件放进去
- 在classpath下直接放配置文件

springboot默认是优先读取它本身同级目录下的一个 config/application.properties 文件的。在src/main/resources 文件夹下创建的application.properties 文件的优先级是最低的

**所以springboot启动读取外部配置文件，只需要在外面加一层配置文件覆盖默认的即可，不用修改代码。**

#### 按profile读取外部文件

根据profile的指定，不同的环境读取不同的配置文件，如：

- 开发环境读取application-dev.properties
- 生产环境读取application-prod.properties

具体参照上面profile的配置说明。

#### 读取其他外部文件

如果不想使用上面的application-*.properties文件，可以有以下2种方式。

1）命令行指定

```bash
$ java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties

# 或者
$ java -jar -Dspring.config.location=D:\config\config.properties springbootrestdemo-0.0.1-SNAPSHOT.jar 
```

2）在代码指定

```java
@SpringBootApplication
@PropertySource(value={"file:config.properties"})
public class SpringbootrestdemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootrestdemoApplication.class, args);
    }
}
```

## 参考

- 官方参考文档：https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html




