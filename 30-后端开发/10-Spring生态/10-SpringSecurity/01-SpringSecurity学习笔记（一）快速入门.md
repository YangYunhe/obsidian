# 1. Spring Security概念

Spring Security 是 Spring 采用 AOP 思想，基于 Servlet 过滤器实现的安全框架。它提供了完善的认证机制和方法级的授权功能。是一款非常优秀的权限管理框架。
 Spring Security 是一个功能强大且高度可定制的身份验证和访问控制框架。它是用于保护基于 Spring 的应用程序的事实上的标准。
 Spring Security 是一个框架，致力于为 Java 应用程序提供身份验证和授权。像所有 Spring 项目一样，Spring Security 的真正强大之处在于它可以轻松扩展以满足定制需求的能力。

特征：

- 对身份验证和授权的全面且可扩展的支持
- 保护免受会话固定，点击劫持，跨站点请求伪造等攻击
- Servlet API 集成
- 与 Spring Web MVC 的可选集成

# 2. 快速入门

## 2.1 环境准备

准备一个 SpringMVC+Spring+JSP 的 Web 环境，然后在这个基础上整合SpringSecurity。

项目结构：

```
── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com
│   │   │       └── arc
│   │   │           ├── controller
│   │   │           └── service
│   │   ├── resources
│   │   │   ├── applicationContext.xml
│   │   │   ├── log4j.properties
│   │   │   └── springmvc.xml
│   │   └── webapp
│   │       ├── index.jsp
│   │       └── WEB-INF
│   │           └── web.xml
```

### pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.arc</groupId>
  <artifactId>SpringSecurity</artifactId>
  <version>1.0-SNAPSHOT</version>
  <!-- 打包方式必须是 war,否则 tomcat7 插件运行后，仅仅显示 BUILD SUCCESS，而不运行服务 -->
  <packaging>war</packaging>

  <properties>
    <jdk.version>1.8</jdk.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>5.3.39</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-config</artifactId>
      <version>5.1.13.RELEASE</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-taglibs</artifactId>
      <version>5.1.13.RELEASE</version>
    </dependency>
    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>servlet-api</artifactId>
      <version>2.5</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
      <version>1.7.36</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <resources>
      <resource>
        <directory>src/main/java</directory>
        <includes>
          <include>**/*.xml</include>
          <include>**/*.properties</include>
        </includes>
        <filtering>true</filtering>
      </resource>
      <resource>
        <directory>src/main/resources</directory>
        <filtering>true</filtering>
      </resource>
    </resources>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
        <executions>
          <execution>
            <phase>compile</phase>
            <goals>
              <goal>compile</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <skip>true</skip>
          <source>${jdk.version}</source>
          <target>${jdk.version}</target>
          <encoding>${project.build.sourceEncoding}</encoding>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-resources-plugin</artifactId>
        <version>3.3.1</version>
        <configuration>
          <encoding>${project.build.sourceEncoding}</encoding>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.tomcat.maven</groupId>
        <artifactId>tomcat7-maven-plugin</artifactId>
        <version>2.2</version>
        <configuration>
          <port>8082</port>
          <path>/</path>
        </configuration>
      </plugin>
    </plugins>
  </build>

</project>

```



### Spring 配置文件 applicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 扫描 service 层 -->
    <context:component-scan base-package="com.arc.service"/>

</beans>
```



### SpringMVC 配置文件 springmvc.xml

````xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <!-- 扫描 controller -->
    <context:component-scan base-package="com.arc.controller"/>

    <!-- 配置三大组件：处理器映射器（MappingHandler）、处理器适配器（HandlerAdapter） -->
    <mvc:annotation-driven/>

</beans>
````



### log4j.properties

```properties
log4j.rootCategory=INFO, stdout

log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[QC] %p [%t] %C.%M(%L) | %m%n
```



### web.xml

````xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    
    <!-- Spring 核心配置文件位置 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
    </context-param>

    <!-- 配置 DispatcherServlet -->
    <servlet>
        <servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <!-- 编码过滤器 -->
    <filter>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

    <!--
        监听 JavaWeb 项目中的 ServletContext 对象的创建
        一旦创建，就创建一个 Spring 的容器对象放入其中
        private WebApplicationContext context;
        this.context = createWebApplicationContext(servletContext);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
    -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

</web-app>
````



### index.jsp

````jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>hello</title>
</head>
<body>
<h1>hello</h1>
</body>
</html>
````



IDEA 右侧 -> Maven -> 插件 -> tomcat7 -> tomcat7:run

![image](https://img2024.cnblogs.com/blog/3706506/202610/3706506-20261028180305289-1841574229.png)

服务器之后浏览器访问 localhost:8082。应该可以访问到。

![image](https://img2024.cnblogs.com/blog/3706506/202610/3706506-20261028184814449-28554531.png)



## 2.2 整合 Spring Security

### 添加依赖

- spring-security-core.jar：核心包，任何 SpringSecurity 的功能都需要此包
- spring-security-web.jar：web 工程必备，包含过滤器和相关的 web 安全的基础结构代码
- spring-security-config.jar：用于 xml 文件解析处理
- spring-security-tablibs.jar：动态标签库

```xml
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-config</artifactId>
  <version>5.8.16</version>
</dependency>
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-taglibs</artifactId>
  <version>5.8.16</version>
</dependency>
```

### web.xml 文件中配置 SpringSecurity

````xml
<!-- Spring Security 过滤器链 -->
<filter>
    <!-- 名称固定 -->
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
````



### 添加 SpringSecurity 配置文件 spring-security.xml

spring-security.xml 放在 resources 目录下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:security="http://www.springframework.org/schema/security"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/security
       http://www.springframework.org/schema/security/spring-security.xsd">

    <!--
        auto-config 表示自动加载 SpringSecurity 的配置文件
        use-expressions 表示使用 Spring 的 EL 表达式
    -->
    <security:http auto-config="true" use-expressions="true">
        <!--
            拦截资源
            pattern="/**" 拦截所有的资源
            access="hasAnyRole('ROLE_USER')" 表示只有 ROLE_USER 这个角色可以访问资源
        -->
        <security:intercept-url pattern="/**" access="hasAnyRole('ROLE_USER')"/>
    </security:http>

    <!-- 认证用户信息 -->
    <security:authentication-manager>
        <security:authentication-provider>
            <security:user-service>
                <!-- 设置一个账号 zhangsan 密码 123 {noop}表示不加密 具有的角色是 ROLE_USER -->
                <security:user name="zhangsan" authorities="ROLE_USER" password="{noop}123"/>
            </security:user-service>
        </security:authentication-provider>
    </security:authentication-manager>
</beans>
```

### 将 SpringSecurity 的配置文件引入到 Spring 中

修改 Spring 配置文件 applicationContext.xml，添加：

````xml
<!-- 引入 Spring Security 配置文件 -->
<import resource="classpath:spring-security.xml"/>
````

### 测试

浏览器访问：http://localhost:8082/index.jsp

由于引入了 SpringSecurity，所有不能在未登录的情况下访问服务资源。所以会自动跳转到登录页面，这是一个 SpringSecurity 内置的登录页面：

![image-20261028180635009](/home/yangyunhe/.config/Typora/typora-user-images/image-20261028180635009.png)

需要输入正确的用户和密码（即 zhangsan 和 123）才可以登录并访问到资源。

访问 http://localhost:8082/logout 则可以退出登录状态。