## 从0开始SpringBoot项目实践 - 搭建项目

工具： Intellij IDEA 2020.1
SpringBoot版本： 2.3.2.RELEASE

### 1、创建项目

直接使用idea的New Project的Spring Initializr向导创建项目。

![](https://raw.githubusercontent.com/clixiang/notes/master/images/2020/idea-new-project-springboot.jpg)

选择依赖：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
```


### 2、环境配置

1）配置pom.xml

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

2）配置application.properties

```ini
spring.profiles.active=@profileActive@
```

### 3、配置热部署

上面已经选择了依赖`spring-boot-devtools`，还需要以下配置:

1）配置pom.xml

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.flywaydb</groupId>
                <artifactId>flyway-maven-plugin</artifactId>
                <version>6.5.3</version>
            </plugin>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <fork>true</fork>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

2）idea开启自动build

![](https://raw.githubusercontent.com/clixiang/notes/master/images/2020/idea-build-auto-completed.jpg)

3）配置registry

我用的是macos系统，在idea连续按两下shift键，弹出界面再搜索Registry，勾选 `compiler.automake.allow.when.app.running`。

![](https://raw.githubusercontent.com/clixiang/notes/master/images/2020/idea-registry-compiler-automake.jpg)







