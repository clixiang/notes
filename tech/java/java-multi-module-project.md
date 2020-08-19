## java多module项目实践

工具： Intellij IDEA 2020.1

### 1、新建父项目

新建一个基于SpringBoot的项目 `msg_center`。

![](https://raw.githubusercontent.com/clixiang/notes/master/images/2020/idea-create-project-msgcenter.jpg)

创建完成后，目录结构如下：

```
msgcenter/
    pom.xml
    src/
        main/
            java/
            resource/
        test/
```

除了pom.xml文件以外，其他文件都删除。

### 2、修改父项目pom.xml

```xml
    
    <!-- 修改打包的方式 -->
    <packaging>pom</packaging>
    
    <!-- 加上下面这项，后面要添加3个模块，先加上配置 -->
    <!-- 网上有的说下面这段会在创建子模块的时候自动加上，我实际操作的时候发现不行 -->
    <modules>
        <module>common</module>
        <module>server</module>
        <module>admin</module>
    </modules>
    
    <dependencies>
        <!-- dependencies加上所有子模块通用的依赖即可，其他子模块依赖的放到子模块的pom.xml -->
    </dependencies>

    <!-- 把下面这段配置给去掉 -->
    <!--
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <fork>true</fork>
                </configuration>
            </plugin>
        </plugins>
    </build>
    -->
```

### 3、新建module

在Idea项目视窗下，右键点击项目根目录，选择菜单 New > Module 创建模块。创建模块的操作和创建项目差不多。同样的方式创建3个子模块：

- common：通用模块，被以下2个模块调用
- admin： 后台模块
- server： 服务端模块

创建完成后，项目结构如下：

```
msgcenter/
    pom.xml
    common/
    server/
    admin/
        pom.xml
        src/
            main/
                java/
                resource/
            test/
```

### 4、配置子模块pom.xml

新建完子模块之后，需要修改子模块的pom.xml。

例如，修改admin的pom.xml：

```xml
    <!-- 1、修改parent部分为如下内容 -->
    <parent>
        <groupId>com.xxli</groupId>
        <artifactId>msgcenter</artifactId>
        <version>1.0.0</version>
        <relativePath>../pom.xml</relativePath> <!-- lookup parent from repository -->
    </parent>

    <!-- 2、去掉groupId的配置，它会继承父项目的，保留artifactId -->
    <!-- <groupId>com.xxli</groupId> -->
    <artifactId>admin</artifactId>
    <packaging>jar<packaging>
    
    <!-- 3、子模块admin依赖common模块，需加上配置 -->
    <dependencies>
        <dependency>
            <groupId>com.xxli</groupId>
            <artifactId>common</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>

    <!-- 4、子模块admin可构建运行，需加上build配置，否则不用，如common模块就不需要加 -->
    <build>
        <plugins>
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

其他2个子模块的配置根据上面的修改即可。

### 5、配置profile

这步是指定项目不同的环境配置。如有需要就配置。

修改父项目pom.xml，添加以下配置：

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

然后再**在子模块配置** `application.properties`：

```ini
spring.profiles.active=@profileActive@
```

### 6、运行模块

比如，在开发的时候，要运行admin模块开发调试：

```bash
$ mvn spring-boot:run -pl admin
```

比如，要打包某个模块：

```bash
$ mvn clean package -P prod -pl admin
```

