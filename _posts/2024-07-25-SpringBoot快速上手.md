# SpringBoot快速上手

### 什么是SpringBoot
Spring Boot 是一个基于 Spring 的框架，旨在简化 Spring 应用的配置和开发过程，通过自动配置和约定大于配置的原则，使开发者能够快速搭建独立、生产级别的应用程序。

### SpringBoot快速创建项目
构建工具插件：Maven | Gradle 这里选择Maven
- 1.新建一个maven项目，并导入依赖
- 2.新建项目选择Spring Initializr
- 3.在下一步之后，可以选择Springboot版本并添加Spring web依赖，最后点击完成就可以了

### SpringBoot起步依赖
Maven 的 pom.xml 文件是用于构建项目的配方。

Spring Boot 提供了多个 “Starter”，可以让您方便地将 jar 添加到 classpath下。spring-boot-starter-parent 是一个特殊 Starter，它提供了有用的 Maven 默认配置。此外它还提供了依赖管理功能，可以轻松地声明依赖关系。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.11</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>demo</description>
    <url/>
    <licenses>
        <license/>
    </licenses>
    <developers>
        <developer/>
    </developers>
    <scm>
        <connection/>
        <developerConnection/>
        <tag/>
        <url/>
    </scm>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```
使用Spring Boot时，强烈推荐从spring-boot-starter-parent继承，因为这样就可以引入Spring Boot的预置配置。然后引入了依赖spring-boot-starter-web和spring-boot-starter-jdbc，它们分别引入了Spring MVC相关依赖和Spring JDBC相关依赖，无需指定版本号，因为在父依赖中已经指定了版本，只有我们自己引入的某些第三方jar包需要指定版本号。
在这里要注意的是spring boot的版本，我这里选择的是2.7.11。java.version指定的是jdk版本，这里选择的是1.8。

如果选择不对，那么在运行的时候就会报错java:错误：无效的源发行版。此时需要修改idea的设置，在File->Settings->Build,Execution,Deployment->Java Compiler中修改为1.8。然后在pox.xml中修改为1.8。我这里遇到问题是修改了还不行，然后没管它了。第二天重新打开项目，就正常了。

### 编写demo程序
在src/main/java目录下自动创建com.example.demo包，并在该包下自动生成DemoApplication启动类：
```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```
@SpringBootApplication 注解是 Spring Boot 的核心注解，它是一个组合注解：
- @SpringBootConfiguration：组合了@Configuration注解，实现配置文件的功能。
- @EnableAutoConfiguration：组合了@Import({AutoConfigurationImportSelector.class})注解，导入选择器。
- @ComponentScan：组合了@Import({Registrar.class})注解，导入选择器。

Autoconfiguration是Spring Boot框架的核心特性之一，它使得开发者能够更加便捷地创建和配置Spring应用程序。通过Autoconfiguration，Spring Boot可以根据项目中的类路径、属性设置和其他条件，自动配置和管理Bean的定义和依赖关系。使用Autoconfiguration非常简单，在pom.xml或build.gradle中添加Spring Boot Starter依赖，然后使用@SpringBootApplication注解来启用自动配置。

main 方法是一个标准方法，其遵循 Java 规范中定义的应用程序入口点。我们的 main 方法通过调用 run 来委托 Spring Boot 的 SpringApplication 类，SpringApplication 类将引导我们的应用，启动 Spring，然后启动自动配置的 Tomcat web 服务器。

在src/main/java目录下自动创建com.example.demo.controller包，并在该包下创建User类：
```java
package com.example.demo.controller;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.web.bind.annotation.*;

@RestController
@EnableAutoConfiguration
public class User {
    @RequestMapping("/")
    String home() {
        return "Hello World!";
    }

    @GetMapping("welcome")
    public String world() {
        return "welcome to java world!";
    }

    public static void main(String[] args) throws Exception {
        SpringApplication.run(User.class, args);
    }
}
```
@RestController 与 @RequestMapping  是 Spring MVC 注解，它们不是 Spring Boot 特有的。注解的作用是：
- @RestController 注解是 Spring3.0 版本之后引入的，它相当于 @Controller + @ResponseBody 的合集，表示该方法返回的结果直接写入 HTTP response body 中。该注解被称作 stereotype 注解。
- @RequestMapping 注解是 Spring3.0 版本之后引入的，它的作用就是将 HTTP 请求映射到特定的方法上。


### SpringBoot基础配置
SpringBoot提供了多种属性配置方式，包括：
 - 默认是application.properties，在src/main/resources目录下自动创建。
```properties
server.port=8081
```
- application.yml
```yaml
server:
  port: 8081
  servlet:
```
- application.yaml
```yaml
server:
  port: 8081
  servlet:
  context-path: /demo
```

### SpringBoot配置文件加载顺序
SpringBoot配置文件加载顺序：
application.properties > application.yml > application.yaml


### SpringBoot程序启动
SpringBoot的引导类是项目的入口，运行main方法就可以启动项目，在控制台可以看到启动日志：
```shell
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::               (v2.7.11)

2024-07-25 16:43:13.294  INFO 20012 --- [           main] com.example.demo.DemoApplication         : Starting DemoApplication using Java 1.8.0_271 on 
……
```
Spring Boot自动启动了嵌入式Tomcat，当看到Started Application in xxx seconds时，Spring Boot应用启动成功。在浏览器输入localhost:8080就可以直接访问页面。

### SpringBoot热部署
SpringBoot热部署是开发过程中经常使用的功能，它能够实时的将代码修改后的内容重新加载到内存中，不需要重启应用。
在pom.xml中添加依赖：
```xml
    <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
    </dependency>
    <dependency>
```
默认配置下，针对/static、/public和/templates目录中的文件修改，不会自动重启，因为禁用缓存后，这些文件的修改可以实时更新。

### SpringBoot打包部署
SpringBoot项目打包部署非常简单，只需要在pom.xml中添加如下配置：
```xml
    <build>
    <plugins>
    <plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
    </plugins>
    </build>
```
在项目根目录下执行命令：
```shell
mvn clean package -Dmaven.test.skip=true
```
打包成功后，在target目录下生成jar包：
```shell
demo-0.0.1-SNAPSHOT.jar
```
### 运行Java JAR包
在项目根目录下执行命令：
```shell
java -jar demo-0.0.1-SNAPSHOT.jar
```
如果需要指定端口号，使用如下命令：
```shell
java -jar demo-0.0.1-SNAPSHOT.jar --server.port=8082
```
如果JAR需要更多的内存，可以使用-Xms和-Xmx参数指定最小和最大内存：
```shell
java -Xms512m -Xmx1024m -jar demo-0.0.1-SNAPSHOT.jar
```

### SpringBoot打包部署到Docker容器中
在pom.xml中添加如下配置：
```xml
    <build>
    <plugins>
    <plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
    <excludes>
    <exclude>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    </exclude>
    </excludes>
    </configuration>
    </plugin>
    <plugin>
    <groupId>com.spotify</groupId>
    <artifactId>dockerfile-maven-plugin</artifactId>
    <version>1.4.13</version>
    <executions>
    <execution>
    <id>default</id>
    <goals>
    <goal>build</goal>
    </goals>
    </execution>
    </executions>
    <configuration>
    <repository>${docker.image.prefix}/${project.artifactId}</repository>
    <buildArgs>
    <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
    </buildArgs>
    </configuration>
    </plugin>
    </plugins>
    </build>
    <properties>
    <docker.image.prefix>registry.cn-hangzhou.aliyuncs.com/demo</docker.image.prefix>
    </properties>
```
在项目根目录下执行命令：
```shell
mvn clean package dockerfile:build -Dmaven.test.skip=true
```
打包成功后，在target目录下生成Dockerfile文件：
```shell
Dockerfile
```
在项目根目录下执行命令：
```shell
docker build -t demo .
```
执行成功后，在本地镜像中生成demo镜像：
```shell
REPOSITORY                    TAG       IMAGE ID       CREATED          SIZE
registry.cn-hangzhou.aliyuncs.com/demo   demo          7623548019b9    About a minute ago   107MB
```
在项目根目录下执行命令：
```shell
docker run -d -p 8081:8080 demo
```
执行成功后，在本地容器中生成demo容器：
```shell
CONTAINER ID   IMAGE                    COMMAND       CREATED          STATUS         PORTS                               NAMES
1325689740b3   demo                    "java -jar /demo-0.0.1-SNAPSHOT.jar"   1 second ago      Up 
```
在浏览器输入localhost:8081就可以直接访问页面。