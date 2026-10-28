# 1. Spring Security概念

Spring Security 是 Spring 采用 AOP 思想，基于 Servlet 过滤器实现的安全框架。它提供了完善的认证机制和方法级的授权功能。是一款非常优秀的权限管理框架。
Spring Security 是一个功能强大且高度可定制的身份验证和访问控制框架。它是用于保护基于 Spring 的应用程序的事实上的标准。
Spring Security 是一个框架，致力于为 Java 应用程序提供身份验证和授权。像所有 Spring 项目一样，Spring Security 的真正强大之处在于它可以轻松扩展以满足定制需求的能力。

特征：
- 对身份验证和授权的全面且可扩展的支持
* 保护免受会话固定，点击劫持，跨站点请求伪造等攻击
* Servlet API 集成
* 与 Spring Web MVC 的可选集成

# 2. 快速入门

## 2.1 环境准备
准备一个 SpringMVC+Spring+JSP 的 Web 环境，然后在这个基础上整合SpringSecurity。

项目结构：
```
── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com
│   │   │       └── arc
│   │   │           ├── controller
│   │   │           └── service
│   │   ├── resources
│   │   │   ├── applicationContext.xml
│   │   │   ├── log4j.properties
│   │   │   └── springmvc.xml
│   │   └── webapp
│   │       ├── index.jsp
│   │       └── WEB-INF
│   │           └── web.xml
```
### pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">  
  <modelVersion>4.0.0</modelVersion>  
  
  <groupId>com.arc</groupId>  
  <artifactId>SpringSecurity</artifactId>  
  <version>1.0-SNAPSHOT</version>  
  
  <repositories>    <repository>      <id>aliyunmaven</id>  
      <name>阿里云公共仓库</name>  
      <url>https://maven.aliyun.com/repository/public</url>  
      <releases>        <enabled>true</enabled>  
      </releases>      <snapshots>        <enabled>false</enabled>  
      </snapshots>    </repository>  </repositories>  
  <properties>    <jdk.version>1.8</jdk.version>  
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>  
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>  
  </properties>  
  <dependencies>    <dependency>      <groupId>org.springframework</groupId>  
      <artifactId>spring-webmvc</artifactId>  
      <version>5.3.39</version>  
    </dependency>    <dependency>      <groupId>org.springframework.security</groupId>  
      <artifactId>spring-security-config</artifactId>  
      <version>5.8.16</version>  
    </dependency>    <dependency>      <groupId>org.springframework.security</groupId>  
      <artifactId>spring-security-taglibs</artifactId>  
      <version>5.8.16</version>  
    </dependency>    <dependency>      <groupId>javax.servlet</groupId>  
      <artifactId>javax.servlet-api</artifactId>  
      <version>4.0.1</version>  
      <scope>provided</scope>  
    </dependency>    <dependency>      <groupId>javax.servlet.jsp</groupId>  
      <artifactId>javax.servlet.jsp-api</artifactId>  
      <version>2.3.3</version>  
      <scope>provided</scope>  
    </dependency>    <dependency>      <groupId>org.slf4j</groupId>  
      <artifactId>slf4j-log4j12</artifactId>  
      <version>1.7.36</version>  
    </dependency>    <dependency>      <groupId>junit</groupId>  
      <artifactId>junit</artifactId>  
      <version>4.13.2</version>  
      <scope>test</scope>  
    </dependency>  </dependencies>  
  <build>    <resources>      <resource>        <directory>src/main/java</directory>  
        <includes>          <include>**/*.xml</include>  
          <include>**/*.properties</include>  
        </includes>        <filtering>true</filtering>  
      </resource>      <resource>        <directory>src/main/resources</directory>  
        <filtering>true</filtering>  
      </resource>    </resources>    <plugins>      <plugin>        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-compiler-plugin</artifactId>  
        <version>3.13.0</version>  
        <executions>          <execution>            <phase>compile</phase>  
            <goals>              <goal>compile</goal>  
            </goals>          </execution>        </executions>        <configuration>          <skip>true</skip>  
          <source>${jdk.version}</source>  
          <target>${jdk.version}</target>  
          <encoding>${project.build.sourceEncoding}</encoding>  
        </configuration>      </plugin>      <plugin>        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-resources-plugin</artifactId>  
        <version>3.3.1</version>  
        <configuration>          <encoding>${project.build.sourceEncoding}</encoding>  
        </configuration>      </plugin>      <plugin>        <groupId>org.apache.tomcat.maven</groupId>  
        <artifactId>tomcat7-maven-plugin</artifactId>  
        <version>2.2</version>  
        <configuration>          <port>8082</port>  
          <path>/</path>  
        </configuration>      </plugin>    </plugins>  </build>  
</project>
```