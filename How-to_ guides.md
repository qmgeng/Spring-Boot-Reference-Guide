### How-to指南

本章节将回答一些常见的"我该怎么做"类型的问题，这些问题在我们使用Spring Boot时经常遇到。这绝不是一个详尽的列表，但它覆盖了很多方面。

如果遇到一个特殊的我们没有覆盖的问题，你可能想去查看[stackoverflow.com](http://stackoverflow.com/tags/spring-boot)，看是否有人已经给出了答案;这也是一个很好的提新问题的地方（请使用`spring-boot`标签）。

我们也乐意扩展本章节;如果想添加一个'how-to'，你可以给我们发一个[pull请求](http://github.com/spring-projects/spring-boot/tree/master)。

### Spring Boot应用

* 解决自动配置问题

Spring Boot自动配置总是尝试尽最大努力去做正确的事，但有时候会失败并且很难说出失败原因。

在每个Spring Boot `ApplicationContext`中都存在一个相当有用的`ConditionEvaluationReport`。如果开启`DEBUG`日志输出，你将会看到它。如果你使用`spring-boot-actuator`，则会有一个`autoconfig`的端点，它将以JSON形式渲染该报告。可以使用它调试应用程序，并能查看Spring Boot运行时都添加了哪些特性（及哪些没添加）。

通过查看源码和javadoc可以获取更多问题的答案。以下是一些经验：
1. 查找名为`*AutoConfiguration`的类并阅读源码，特别是`@Conditional*`注解，这可以帮你找出它们启用哪些特性及何时启用。
将`--debug`添加到命令行或添加系统属性`-Ddebug`可以在控制台查看日志，该日志会记录你的应用中所有自动配置的决策。在一个运行的Actuator app中，通过查看`autoconfig`端点（`/autoconfig`或等效的JMX）可以获取相同信息。
2. 查找是`@ConfigurationProperties`的类（比如[ServerProperties](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/ServerProperties.java)）并看下有哪些可用的外部配置选项。`@ConfigurationProperties`类有一个用于充当外部配置前缀的`name`属性，因此`ServerProperties`的值为`prefix="server"`，它的配置属性有`server.port`，`server.address`等。在运行的Actuator应用中可以查看`configprops`端点。
3. 查看使用`RelaxedEnvironment`明确地将配置从`Environment`暴露出去。它经常会使用一个前缀。
4. 查看`@Value`注解，它直接绑定到`Environment`。相比`RelaxedEnvironment`，这种方式稍微缺乏灵活性，但它也允许松散的绑定，特别是OS环境变量（所以`CAPITALS_AND_UNDERSCORES`是`period.separated`的同义词）。
5. 查看`@ConditionalOnExpression`注解，它根据SpEL表达式的结果来开启或关闭特性，通常使用解析自`Environment`的占位符进行计算。

6. 启动前自定义Environment或ApplicationContext

每个`SpringApplication`都有`ApplicationListeners`和`ApplicationContextInitializers`，用于自定义上下文（context）或环境(environment)。Spring Boot从`META-INF/spring.factories`下加载很多这样的内部使用的自定义。有很多方法可以注册其他的自定义：

1. 以编程方式为每个应用注册自定义，通过在SpringApplication运行前调用它的`addListeners`和`addInitializers`方法来实现。
2. 以声明方式为每个应用注册自定义，通过设置`context.initializer.classes`或`context.listener.classes`来实现。
3. 以声明方式为所有应用注册自定义，通过添加一个`META-INF/spring.factories`并打包成一个jar文件（该应用将它作为一个库）来实现。

`SpringApplication`会给监听器（即使是在上下文被创建之前就存在的）发送一些特定的`ApplicationEvents`，然后也会注册监听`ApplicationContext`发布的事件的监听器。查看Spring Boot特性章节中的[Section 22.4, “Application events and listeners” ](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-application-events-and-listeners)可以获取一个完整列表。

* 构建ApplicationContext层次结构（添加父或根上下文）

你可以使用`ApplicationBuilder`类创建父/根`ApplicationContext`层次结构。查看'Spring Boot特性'章节的[Section 22.3, “Fluent builder API” ](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-fluent-builder-api)获取更多信息。

* 创建一个非web（non-web）应用

不是所有的Spring应用都必须是web应用（或web服务）。如果你想在main方法中执行一些代码，但需要启动一个Spring应用去设置需要的底层设施，那使用Spring Boot的`SpringApplication`特性可以很容易实现。`SpringApplication`会根据它是否需要一个web应用来改变它的`ApplicationContext`类。首先你需要做的是去掉servlet API依赖，如果不能这样做（比如，基于相同的代码运行两个应用），那你可以明确地调用`SpringApplication.setWebEnvironment(false)`或设置`applicationContextClass`属性（通过Java API或使用外部配置）。你想运行的，作为业务逻辑的应用代码可以实现为一个`CommandLineRunner`，并将上下文降级为一个`@Bean`定义。

### 属性&配置

* 外部化SpringApplication配置

SpringApplication已经被属性化（主要是setters），所以你可以在创建应用时使用它的Java API修改它的行为。或者你可以使用properties文件中的`spring.main.*`来外部化（在应用代码外配置）这些配置。比如，在`application.properties`中可能会有以下内容：
```java
spring.main.web_environment=false
spring.main.show_banner=false
```
然后Spring Boot在启动时将不会显示banner，并且该应用也不是一个web应用。

* 改变应用程序外部配置文件的位置

默认情况下，来自不同源的属性以一个定义好的顺序添加到Spring的`Environment`中（查看'Sprin Boot特性'章节的[Chapter 23, Externalized Configuration](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config)获取精确的顺序）。

为应用程序源添加`@PropertySource`注解是一种很好的添加和修改源顺序的方法。传递给`SpringApplication`静态便利设施（convenience）方法的类和使用`setSources()`添加的类都会被检查，以查看它们是否有`@PropertySources`，如果有，这些属性会被尽可能早的添加到`Environment`里，以确保`ApplicationContext`生命周期的所有阶段都能使用。以这种方式添加的属性优先于任何使用默认位置添加的属性，但低于系统属性，环境变量或命令行参数。

你也可以提供系统属性（或环境变量）来改变该行为：

1. `spring.config.name`（`SPRING_CONFIG_NAME`）是根文件名，默认为`application`。
2. `spring.config.location`（`SPRING_CONFIG_LOCATION`）是要加载的文件（例如，一个classpath资源或一个URL）。Spring Boot为该文档设置一个单独的`Environment`属性，它可以被系统属性，环境变量或命令行参数覆盖。

不管你在environment设置什么，Spring Boot都将加载上面讨论过的`application.properties`。如果使用YAML，那具有'.yml'扩展的文件默认也会被添加到该列表。

详情参考[ConfigFileApplicationListener](http://github.com/spring-projects/spring-boot/tree/master/spring-boot/src/main/java/org/springframework/boot/context/config/ConfigFileApplicationListener.java)

* 使用'short'命令行参数

有些人喜欢使用（例如）`--port=9000`代替`--server.port=9000`来设置命令行配置属性。你可以通过在application.properties中使用占位符来启用该功能，比如：
```java
server.port=${port:8080}
```
**注**：如果你继承自`spring-boot-starter-parent` POM，为了防止和Spring-style的占位符产生冲突，`maven-resources-plugins`默认的过滤令牌（filter token）已经从`${*}`变为`@`（即`@maven.token@`代替了`${maven.token}`）。如果已经直接启用maven对application.properties的过滤，你可能也想使用[其他的分隔符](http://maven.apache.org/plugins/maven-resources-plugin/resources-mojo.html#delimiters)替换默认的过滤令牌。

**注**：在这种特殊的情况下，端口绑定能够在一个PaaS环境下工作，比如Heroku和Cloud Foundry，因为在这两个平台中`PORT`环境变量是自动设置的，并且Spring能够绑定`Environment`属性的大写同义词。

* 使用YAML配置外部属性

YAML是JSON的一个超集，可以非常方便的将外部配置以层次结构形式存储起来。比如：
```json
spring:
    application:
        name: cruncher
    datasource:
        driverClassName: com.mysql.jdbc.Driver
        url: jdbc:mysql://localhost/test
server:
    port: 9000
```
创建一个application.yml文件，将它放到classpath的根目录下，并添加snakeyaml依赖（Maven坐标为`org.yaml:snakeyaml`，如果你使用`spring-boot-starter`那就已经被包含了）。一个YAML文件会被解析为一个Java `Map<String,Object>`（和一个JSON对象类似），Spring Boot会平伸该map，这样它就只有1级深度，并且有period-separated的keys，跟人们在Java中经常使用的Properties文件非常类似。
上面的YAML示例对应于下面的application.properties文件：
```java
spring.application.name=cruncher
spring.datasource.driverClassName=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost/test
server.port=9000
```
查看'Spring Boot特性'章节的[Section 23.6, “Using YAML instead of Properties”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config-yaml)可以获取更多关于YAML的信息。

* 设置生效的Spring profiles

Spring `Environment`有一个API可以设置生效的profiles，但通常你会设置一个系统profile（`spring.profiles.active`）或一个OS环境变量（`SPRING_PROFILES_ACTIVE`）。比如，使用一个`-D`参数启动应用程序（记着把它放到main类或jar文件之前）：
```shell
$ java -jar -Dspring.profiles.active=production demo-0.0.1-SNAPSHOT.jar
```
在Spring Boot中，你也可以在application.properties里设置生效的profile，例如：
```java
spring.profiles.active=production
```
通过这种方式设置的值会被系统属性或环境变量替换，但不会被`SpringApplicationBuilder.profiles()`方法替换。因此，后面的Java API可用来在不改变默认设置的情况下增加profiles。

想要获取更多信息可查看'Spring Boot特性'章节的[Chapter 24, Profiles](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-profiles)。

* 根据环境改变配置

一个YAML文件实际上是一系列以`---`线分割的文档，每个文档都被单独解析为一个平坦的（flattened）map。

如果一个YAML文档包含一个`spring.profiles`关键字，那profiles的值（以逗号分割的profiles列表）将被传入Spring的`Environment.acceptsProfiles()`方法，并且如果这些profiles的任何一个被激活，对应的文档被包含到最终的合并中（否则不会）。

示例：
```json
server:
    port: 9000
---

spring:
    profiles: development
server:
    port: 9001

---

spring:
    profiles: production
server:
    port: 0
```
在这个示例中，默认的端口是9000，但如果Spring profile 'development'生效则该端口是9001，如果'production'生效则它是0。

YAML文档以它们遇到的顺序合并（所以后面的值会覆盖前面的值）。

想要使用profiles文件完成同样的操作，你可以使用`application-${profile}.properties`指定特殊的，profile相关的值。

* 发现外部属性的内置选项

Spring Boot在运行时将来自application.properties（或.yml）的外部属性绑定进一个应用中。在一个地方不可能存在详尽的所有支持属性的列表（技术上也是不可能的），因为你的classpath下的其他jar文件也能够贡献。

每个运行中且有Actuator特性的应用都会有一个`configprops`端点，它能够展示所有边界和可通过`@ConfigurationProperties`绑定的属性。

附录中包含一个[application.properties](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#common-application-properties)示例，它列举了Spring Boot支持的大多数常用属性。获取权威列表可搜索`@ConfigurationProperties`和`@Value`的源码，还有不经常使用的`RelaxedEnvironment`。

### 内嵌的servlet容器

* 为应用添加Servlet，Filter或ServletContextListener

Servlet规范支持的Servlet，Filter，ServletContextListener和其他监听器可以作为`@Bean`定义添加到你的应用中。需要格外小心的是，它们不会引起太多的其他beans的热初始化，因为在应用生命周期的早期它们已经被安装到容器里了（比如，让它们依赖你的DataSource或JPA配置就不是一个好主意）。你可以通过延迟初始化它们到第一次使用而不是初始化时来突破该限制。

在Filters和Servlets的情况下，你也可以通过添加一个`FilterRegistrationBean`或`ServletRegistrationBean`代替或以及底层的组件来添加映射（mappings）和初始化参数。

* 禁止注册Servlet或Filter

正如[以上讨论](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-add-a-servlet-filter-or-servletcontextlistener)的任何Servlet或Filter beans将被自动注册到servlet容器中。为了禁止注册一个特殊的Filter或Servlet bean，可以为它创建一个注册bean，然后禁用该bean。例如：
```java
@Bean
public FilterRegistrationBean registration(MyFilter filter) {
    FilterRegistrationBean registration = new FilterRegistrationBean(filter);
    registration.setEnabled(false);
    return registration;
}
```

* 改变HTTP端口

在一个单独的应用中，主HTTP端口默认为8080，但可以使用`server.port`设置（比如，在application.properties中或作为一个系统属性）。由于`Environment`值的宽松绑定，你也可以使用`SERVER_PORT`（比如，作为一个OS环境变）。

为了完全关闭HTTP端点，但仍创建一个WebApplicationContext，你可以设置`server.port=-1`（测试时可能有用）。

想获取更多详情可查看'Spring Boot特性'章节的[Section 26.3.3, “Customizing embedded servlet containers”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-customizing-embedded-containers)，或[ServerProperties](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/ServerProperties.java)源码。

* 使用随机未分配的HTTP端口

想扫描一个未使用的端口（为了防止冲突使用OS本地端口）可以使用`server.port=0`。

* 发现运行时的HTTP端口

你可以通过日志输出或它的EmbeddedServletContainer的EmbeddedWebApplicationContext获取服务器正在运行的端口。获取和确认服务器已经初始化的最好方式是添加一个`ApplicationListener<EmbeddedServletContainerInitializedEvent>`类型的`@Bean`，然后当事件发布时将容器pull出来。

使用`@WebIntegrationTests`的一个有用实践是设置`server.port=0`，然后使用`@Value`注入实际的（'local'）端口。例如：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = SampleDataJpaApplication.class)
@WebIntegrationTest("server.port:0")
public class CityRepositoryIntegrationTests {

    @Autowired
    EmbeddedWebApplicationContext server;

    @Value("${local.server.port}")
    int port;

    // ...

}
```
* 配置SSL

SSL能够以声明方式进行配置，一般通过在application.properties或application.yml设置各种各样的`server.ssl.*`属性。例如：
```json
server.port = 8443
server.ssl.key-store = classpath:keystore.jks
server.ssl.key-store-password = secret
server.ssl.key-password = another-secret
```
获取所有支持的配置详情可查看[Ssl](http://github.com/spring-projects/spring-boot/tree/master/spring-boot/src/main/java/org/springframework/boot/context/embedded/Ssl.java)。

**注**：Tomcat要求key存储（如果你正在使用一个可信存储）能够直接在文件系统上访问，即它不能从一个jar文件内读取。Jetty和Undertow没有该限制。

使用类似于以上示例的配置意味着该应用将不在支持端口为8080的普通HTTP连接。Spring Boot不支持通过application.properties同时配置HTTP连接器和HTTPS连接器。如果你两个都想要，那就需要以编程的方式配置它们中的一个。推荐使用application.properties配置HTTPS，因为HTTP连接器是两个中最容易以编程方式进行配置的。获取示例可查看[spring-boot-sample-tomcat-multi-connectors](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-tomcat-multi-connectors)示例项目。

* 配置Tomcat

通常你可以遵循[Section 63.7, “Discover built-in options for external properties”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-discover-build-in-options-for-external-properties)关于`@ConfigurationProperties`（这里主要的是`ServerProperties`）的建议，但也看下`EmbeddedServletContainerCustomizer`和各种你可以添加的Tomcat-specific的`*Customizers`。

Tomcat APIs相当丰富，一旦获取到`TomcatEmbeddedServletContainerFactory`，你就能够以多种方式修改它。或核心选择是添加你自己的`TomcatEmbeddedServletContainerFactory`。

* 启用Tomcat的多连接器（Multiple Connectors）

你可以将一个`org.apache.catalina.connector.Connector`添加到`TomcatEmbeddedServletContainerFactory`，这就能够允许多连接器，比如HTTP和HTTPS连接器：
```java
@Bean
public EmbeddedServletContainerFactory servletContainer() {
    TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory();
    tomcat.addAdditionalTomcatConnectors(createSslConnector());
    return tomcat;
}

private Connector createSslConnector() {
    Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
    Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
    try {
        File keystore = new ClassPathResource("keystore").getFile();
        File truststore = new ClassPathResource("keystore").getFile();
        connector.setScheme("https");
        connector.setSecure(true);
        connector.setPort(8443);
        protocol.setSSLEnabled(true);
        protocol.setKeystoreFile(keystore.getAbsolutePath());
        protocol.setKeystorePass("changeit");
        protocol.setTruststoreFile(truststore.getAbsolutePath());
        protocol.setTruststorePass("changeit");
        protocol.setKeyAlias("apitester");
        return connector;
    }
    catch (IOException ex) {
        throw new IllegalStateException("can't access keystore: [" + "keystore"
                + "] or truststore: [" + "keystore" + "]", ex);
    }
}
```
* 在前端代理服务器后使用Tomcat

Spring Boot将自动配置Tomcat的`RemoteIpValve`，如果你启用它的话。这允许你透明地使用标准的`x-forwarded-for`和`x-forwarded-proto`头，很多前端代理服务器都会添加这些头信息（headers）。通过将这些属性中的一个或全部设置为非空的内容来开启该功能（它们是大多数代理约定的值，如果你只设置其中的一个，则另一个也会被自动设置）。
```java
server.tomcat.remote_ip_header=x-forwarded-for
server.tomcat.protocol_header=x-forwarded-proto
```
如果你的代理使用不同的头部（headers），你可以通过向application.properties添加一些条目来自定义该值的配置，比如：
```java
server.tomcat.remote_ip_header=x-your-remote-ip-header
server.tomcat.protocol_header=x-your-protocol-header
```
该值也可以配置为一个默认的，能够匹配信任的内部代理的正则表达式。默认情况下，受信任的IP包括 10/8, 192.168/16, 169.254/16 和 127/8。可以通过向application.properties添加一个条目来自定义该值的配置，比如：
```java
server.tomcat.internal_proxies=192\\.168\\.\\d{1,3}\\.\\d{1,3}
```
**注**：只有在你使用一个properties文件作为配置的时候才需要双反斜杠。如果你使用YAML，单个反斜杠就足够了，`192\.168\.\d{1,3}\.\d{1,3}`和上面的等价。

另外，通过在一个`TomcatEmbeddedServletContainerFactory` bean中配置和添加`RemoteIpValve`，你就可以完全控制它的设置了。

* 使用Jetty替代Tomcat

Spring Boot starters（特别是spring-boot-starter-web）默认都是使用Tomcat作为内嵌容器的。你需要排除那些Tomcat的依赖并包含Jetty的依赖。为了让这种处理尽可能简单，Spring Boot将Tomcat和Jetty的依赖捆绑在一起，然后提供单独的starters。

Maven示例：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
Gradle示例：
```gradle
configurations {
    compile.exclude module: "spring-boot-starter-tomcat"
}

dependencies {
    compile("org.springframework.boot:spring-boot-starter-web:1.3.0.BUILD-SNAPSHOT")
    compile("org.springframework.boot:spring-boot-starter-jetty:1.3.0.BUILD-SNAPSHOT")
    // ...
}
```
* 配置Jetty

通常你可以遵循[Section 63.7, “Discover built-in options for external properties”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-discover-build-in-options-for-external-properties)关于`@ConfigurationProperties`（此处主要是ServerProperties）的建议，但也要看下`EmbeddedServletContainerCustomizer`。Jetty API相当丰富，一旦获取到`JettyEmbeddedServletContainerFactory`，你就可以使用很多方式修改它。或更彻底地就是添加你自己的`JettyEmbeddedServletContainerFactory`。

* 使用Undertow替代Tomcat

使用Undertow替代Tomcat和[使用Jetty替代Tomcat](https://github.com/qibaoguang/Spring-Boot-Reference-Guide/edit/master/How-to_%20guides.md)非常类似。你需要排除Tomat依赖，并包含Undertow starter。

Maven示例：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```
Gradle示例：
```gradle
configurations {
    compile.exclude module: "spring-boot-starter-tomcat"
}

dependencies {
    compile 'org.springframework.boot:spring-boot-starter-web:1.3.0.BUILD-SNAPSHOT")
    compile 'org.springframework.boot:spring-boot-starter-undertow:1.3.0.BUILD-SNAPSHOT")
    // ...
}

```
* 配置Undertow

通常你可以遵循[Section 63.7, “Discover built-in options for external properties”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-discover-build-in-options-for-external-properties)关于`@ConfigurationProperties`（此处主要是ServerProperties和ServerProperties.Undertow），但也要看下`EmbeddedServletContainerCustomizer`。一旦获取到`UndertowEmbeddedServletContainerFactory`，你就可以使用一个`UndertowBuilderCustomizer`修改Undertow的配置以满足你的需求。或更彻底地就是添加你自己的`UndertowEmbeddedServletContainerFactory`。

* 启用Undertow的多监听器（Multiple Listeners）

往`UndertowEmbeddedServletContainerFactory`添加一个`UndertowBuilderCustomizer`，然后添加一个监听者到`Builder`：
```java
@Bean
public UndertowEmbeddedServletContainerFactory embeddedServletContainerFactory() {
    UndertowEmbeddedServletContainerFactory factory = new UndertowEmbeddedServletContainerFactory();
    factory.addBuilderCustomizers(new UndertowBuilderCustomizer() {

        @Override
        public void customize(Builder builder) {
            builder.addHttpListener(8080, "0.0.0.0");
        }

    });
    return factory;
}
```
* 使用Tomcat7

Tomcat7可用于Spring Boot，但默认使用的是Tomcat8。如果不能使用Tomcat8（例如，你使用的是Java1.6），你需要改变classpath去引用Tomcat7。

- 通过Maven使用Tomcat7

如果正在使用starter pom和parent，你只需要改变Tomcat的version属性，比如，对于一个简单的webapp或service：
```xml
<properties>
    <tomcat.version>7.0.59</tomcat.version>
</properties>
<dependencies>
    ...
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    ...
</dependencies>
```
- 通过Gradle使用Tomcat7

你可以通过设置`tomcat.version`属性改变Tomcat的版本：
```gradle
ext['tomcat.version'] = '7.0.59'
dependencies {
    compile 'org.springframework.boot:spring-boot-starter-web'
}
```
* 使用Jetty8

Jetty8可用于Spring Boot，但默认使用的是Jetty9。如果不能使用Jetty9（例如，因为你使用的是Java1.6），你只需改变classpath去引用Jetty8。你也需要排除Jetty的WebSocket相关的依赖。

- 通过Maven使用Jetty8

如果正在使用starter pom和parent，你只需添加Jetty starter，去掉WebSocket依赖，并改变version属性，比如，对于一个简单的webapp或service：
```xml
<properties>
    <jetty.version>8.1.15.v20140411</jetty.version>
    <jetty-jsp.version>2.2.0.v201112011158</jetty-jsp.version>
</properties>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.eclipse.jetty.websocket</groupId>
                <artifactId>*</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
```
- 通过Gradle使用Jetty8

你可以设置`jetty.version`属性并排除相关的WebSocket依赖，比如对于一个简单的webapp或service：
```gradle
ext['jetty.version'] = '8.1.15.v20140411'
dependencies {
    compile ('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
    }
    compile ('org.springframework.boot:spring-boot-starter-jetty') {
        exclude group: 'org.eclipse.jetty.websocket'
    }
}
```
* 使用@ServerEndpoint创建WebSocket端点

如果想在一个使用内嵌容器的Spring Boot应用中使用@ServerEndpoint，你需要声明一个单独的ServerEndpointExporter @Bean：
```java
@Bean
public ServerEndpointExporter serverEndpointExporter() {
    return new ServerEndpointExporter();
}
```
该bean将用底层的WebSocket容器注册任何的被`@ServerEndpoint`注解的beans。当部署到一个单独的servlet容器时，该角色将被一个servlet容器初始化方法履行，ServerEndpointExporter bean也就不是必需的了。

* 启用HTTP响应压缩

Spring Boot提供两种启用HTTP压缩的机制;一种是Tomcat特有的，另一种是使用一个filter，可以配合Jetty，Tomcat和Undertow。

- 启用Tomcat的HTTP响应压缩

Tomcat对HTTP响应压缩提供内建支持。默认是禁用的，但可以通过application.properties轻松的启用：
```java
server.tomcat.compression: on
```
当设置为`on`时，Tomcat将压缩响应的长度至少为2048字节。你可以配置一个整型值来设置该限制而不只是`on`，比如：
```java
server.tomcat.compression: 4096
```
默认情况下，Tomcat只压缩某些MIME类型的响应（text/html，text/xml和text/plain）。你可以使用`server.tomcat.compressableMimeTypes`属性进行自定义，比如：
```java
server.tomcat.compressableMimeTypes=application/json,application/xml
```
- 使用GzipFilter开启HTTP响应压缩

如果你正在使用Jetty或Undertow，或想要更精确的控制HTTP响应压缩，Spring Boot为Jetty的GzipFilter提供自动配置。虽然该过滤器是Jetty的一部分，但它也兼容Tomcat和Undertow。想要启用该过滤器，只需简单的为你的应用添加`org.eclipse.jetty:jetty-servlets`依赖。

GzipFilter可以使用`spring.http.gzip.*`属性进行配置。具体参考[GzipFilterProperties](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/GzipFilterProperties.java)。

### Spring MVC

* 编写一个JSON REST服务

在Spring Boot应用中，任何Spring `@RestController`默认应该渲染为JSON响应，只要classpath下存在Jackson2。例如：
```java
@RestController
public class MyController {

    @RequestMapping("/thing")
    public MyThing thing() {
            return new MyThing();
    }

}
```
只要MyThing能够通过Jackson2序列化（比如，一个标准的POJO或Groovy对象），[localhost:8080/thing](http://localhost:8080/thing)默认响应一个JSON表示。有时在一个浏览器中你可能看到XML响应因为浏览器倾向于发送XML
响应头。

* 编写一个XML REST服务

如果classpath下存在Jackson XML扩展（jackson-dataformat-xml），它会被用来渲染XML响应，示例和JSON的非常相似。想要使用它，只需为你的项目添加以下的依赖：
```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```
你可能也想添加对Woodstox的依赖。它比JDK提供的默认Stax实现快很多，并且支持良好的格式化输出，提高了namespace处理能力：
```xml
<dependency>
    <groupId>org.codehaus.woodstox</groupId>
    <artifactId>woodstox-core-asl</artifactId>
</dependency>
```
如果Jackson的XML扩展不可用，Spring Boot将使用JAXB（JDK默认提供），不过你需要为MyThing添加额外的注解`@XmlRootElement`：
```java
@XmlRootElement
public class MyThing {
    private String name;
    // .. getters and setters
}
```
想要服务器渲染XML而不是JSON，你可能需要发送一个`Accept: text/xml`头部（或使用浏览器）。

* 自定义Jackson ObjectMapper

在一个HTTP交互中，Spring MVC（客户端和服务端）使用HttpMessageConverters协商内容转换。如果classpath下存在Jackson，你就已经获取到Jackson2ObjectMapperBuilder提供的默认转换器。

创建的ObjectMapper（或用于Jackson XML转换的XmlMapper）实例默认有以下自定义属性：

- `MapperFeature.DEFAULT_VIEW_INCLUSION`禁用
- `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`禁用

Spring Boot也有一些简化自定义该行为的特性。

你可以使用当前的environment配置ObjectMapper和XmlMapper实例。Jackson提供一个扩展套件，可以用来简单的关闭或开启一些特性，你可以用它们配置Jackson处理的不同方面。这些特性在Jackson中使用5个枚举进行描述的，并被映射到environment的属性上：

|Jackson枚举|Environment属性|
|------|:-------|
|`com.fasterxml.jackson.databind.DeserializationFeature`|`spring.jackson.deserialization.<feature_name>=true|false`|
|`com.fasterxml.jackson.core.JsonGenerator.Feature`|`spring.jackson.generator.<feature_name>=true|false`|
|`com.fasterxml.jackson.databind.MapperFeature`|`spring.jackson.mapper.<feature_name>=true|false`|
|`com.fasterxml.jackson.core.JsonParser.Feature`|`spring.jackson.parser.<feature_name>=true|false`|
|`com.fasterxml.jackson.databind.SerializationFeature`|`spring.jackson.serialization.<feature_name>=true|false`|

例如，设置`spring.jackson.serialization.indent_output=true`可以开启漂亮打印。注意，由于[松绑定](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config-relaxed-binding)的使用，`indent_output`不必匹配对应的枚举常量`INDENT_OUTPUT`。

如果想彻底替换默认的ObjectMapper，你需要定义一个该类型的`@Bean`并将它标记为`@Primary`。

定义一个Jackson2ObjectMapperBuilder类型的`@Bean`将允许你自定义默认的ObjectMapper和XmlMapper（分别用于MappingJackson2HttpMessageConverter和MappingJackson2XmlHttpMessageConverter）。

另一种自定义Jackson的方法是向你的上下文添加`com.fasterxml.jackson.databind.Module`类型的beans。它们会被注册入每个ObjectMapper类型的bean，当为你的应用添加新特性时，这就提供了一种全局机制来贡献自定义模块。

最后，如果你提供任何MappingJackson2HttpMessageConverter类型的`@Beans`，那它们将替换MVC配置中的默认值。同时，也提供一个HttpMessageConverters类型的bean，它有一些有用的方法可以获取默认的和用户增强的message转换器。

想要获取更多细节可查看[Section 65.4, “Customize the @ResponseBody rendering”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-customize-the-responsebody-rendering)和[WebMvcAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/WebMvcAutoConfiguration.java)源码。

* 自定义@ResponseBody渲染

Spring使用HttpMessageConverters渲染`@ResponseBody`（或来自`@RestController`的响应）。你可以通过在Spring Boot上下文中添加该类型的beans来贡献其他的转换器。如果你添加的bean类型默认已经包含了（像用于JSON转换的MappingJackson2HttpMessageConverter），那它将替换默认的。Spring Boot提供一个方便的HttpMessageConverters类型的bean，它有一些有用的方法可以访问默认的和用户增强的message转换器（有用，比如你想要手动将它们注入到一个自定义的`RestTemplate`）。

在通常的MVC用例中，任何你提供的WebMvcConfigurerAdapter beans通过覆盖configureMessageConverters方法也能贡献转换器，但不同于通常的MVC，你可以只提供你需要的转换器（因为Spring Boot使用相同的机制来贡献它默认的转换器）。最终，如果你通过提供自己的` @EnableWebMvc`注解覆盖Spring Boot默认的MVC配置，那你就可以完全控制，并使用来自WebMvcConfigurationSupport的getMessageConverters手动做任何事。

具体参考[WebMvcAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/WebMvcAutoConfiguration.java)源码。

* 处理Multipart文件上传

Spring Boot采用Servlet 3 `javax.servlet.http.Part` API来支持文件上传。默认情况下，Spring Boot配置Spring MVC在单个请求中每个文件最大1Mb，最多10Mb的文件数据。你可以覆盖那些值，也可以设置临时文件存储的位置（比如，存储到`/tmp`文件夹下）及传递数据刷新到磁盘的阀值（通过使用MultipartProperties类暴露的属性）。如果你需要设置文件不受限制，例如，可以设置`multipart.maxFileSize`属性值为`-1`。

当你想要接收部分（multipart）编码文件数据作为Spring MVC控制器（controller）处理方法中被`@RequestParam`注解的MultipartFile类型的参数时，multipart支持就非常有用了。

具体参考[MultipartAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/MultipartAutoConfiguration.java)源码。

* 关闭Spring MVC DispatcherServlet

Spring Boot想要服务来自应用程序root `/`下的所有内容。如果你想将自己的servlet映射到该目录下也是可以的，但当然你可能失去一些Boot MVC特性。为了添加你自己的servlet，并将它映射到root资源，你只需声明一个Servlet类型的`@Bean`，并给它特定的bean名称`dispatcherServlet`（如果只想关闭但不替换它，你可以使用该名称创建不同类型的bean）。

* 关闭默认的MVC配置

完全控制MVC配置的最简单方式是提供你自己的被`@EnableWebMvc`注解的`@Configuration`。这样所有的MVC配置都逃不出你的掌心。

* 自定义ViewResolvers

ViewResolver是Spring MVC的核心组件，它负责转换`@Controller`中的视图名称到实际的View实现。注意ViewResolvers主要用在UI应用中，而不是REST风格的服务（View不是用来渲染`@ResponseBody`的）。Spring有很多你可以选择的ViewResolver实现，并且Spring自己对如何选择相应实现也没发表意见。另一方面，Spring Boot会根据classpath上的依赖和应用上下文为你安装一或两个ViewResolver实现。DispatcherServlet使用所有在应用上下文中找到的解析器（resolvers），并依次尝试每一个直到它获取到结果，所以如果你正在添加自己的解析器，那就要小心顺序和你的解析器添加的位置。

WebMvcAutoConfiguration将会为你的上下文添加以下ViewResolvers：

- bean id为`defaultViewResolver`的InternalResourceViewResolver。这个会定位可以使用DefaultServlet渲染的物理资源（比如，静态资源和JSP页面）。它在视图（view name）上应用了一个前缀和后缀（默认都为空，但你可以通过`spring.view.prefix`和`spring.view.suffix`外部配置设置），然后查找在servlet上下文中具有该路径的物理资源。可以通过提供相同类型的bean覆盖它。
- id为`beanNameViewResolver`的BeanNameViewResolver。这是视图解析器链的一个非常有用的成员，它可以在View被解析时收集任何具有相同名称的beans。
- id为`viewResolver`的ContentNegotiatingViewResolver只会在实际View类型的beans出现时添加。这是一个'主'解析器，它的职责会代理给其他解析器，它会尝试找到客户端发送的一个匹配'Accept'的HTTP头部。这有一篇有用的，关于你需要更多了解的[ContentNegotiatingViewResolver](https://spring.io/blog/2013/06/03/content-negotiation-using-views)的博客，也要具体查看下源码。通过定义一个名叫'viewResolver'的bean，你可以关闭自动配置的ContentNegotiatingViewResolver。
- 如果使用Thymeleaf，你将有一个id为`thymeleafViewResolver`的ThymeleafViewResolver。它会通过加前缀和后缀的视图名来查找资源（外部配置为`spring.thymeleaf.prefix`和`spring.thymeleaf.suffix`，对应的默认为'classpath:/templates/'和'.html'）。你可以通过提供相同名称的bean来覆盖它。
- 如果使用FreeMarker，你将有一个id为`freeMarkerViewResolver`的FreeMarkerViewResolver。它会使用加前缀和后缀（外部配置为`spring.freemarker.prefix`和`spring.freemarker.suffix`，对应的默认值为空和'.ftl'）的视图名从加载路径（外部配置为`spring.freemarker.templateLoaderPath`，默认为'classpath:/templates/'）下查找资源。你可以通过提供一个相同名称的bean来覆盖它。
- 如果使用Groovy模板（实际上只要你把groovy-templates添加到classpath下），你将有一个id为`groovyTemplateViewResolver`的Groovy TemplateViewResolver。它会使用加前缀和后缀（外部属性为`spring.groovy.template.prefix`和`spring.groovy.template.suffix`，对应的默认值为'classpath:/templates/'和'.tpl'）的视图名从加载路径下查找资源。你可以通过提供一个相同名称的bean来覆盖它。
- 如果使用Velocity，你将有一个id为`velocityViewResolver`的VelocityViewResolver。它会使用加前缀和后缀（外部属性为`spring.velocity.prefix`和`spring.velocity.suffix`，对应的默认值为空和'.vm'）的视图名从加载路径（外部属性为`spring.velocity.resourceLoaderPath`，默认为'classpath:/templates/'）下查找资源。你可以通过提供一个相同名称的bean来覆盖它。

具体参考：  [WebMvcAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/WebMvcAutoConfiguration.java)，[ThymeleafAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/thymeleaf/ThymeleafAutoConfiguration.java)，[FreeMarkerAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/thymeleaf/ThymeleafAutoConfiguration.java)，[GroovyTemplateAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/thymeleaf/ThymeleafAutoConfiguration.java)，[VelocityAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/thymeleaf/ThymeleafAutoConfiguration.java)。

### 日志

Spring Boot除了commons-logging  API外没有其他强制性的日志依赖，你有很多可选的日志实现。想要使用[Logback](http://logback.qos.ch/)，你需要包含它，及一些对classpath下commons-logging的绑定。最简单的方式是通过依赖`spring-boot-starter-logging`的starter pom。对于一个web应用程序，你只需添加`spring-boot-starter-web`依赖，因为它依赖于logging starter。例如，使用Maven：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
Spring Boot有一个LoggingSystem抽象，用于尝试通过classpath上下文配置日志系统。如果Logback可用，则首选它。如果你唯一需要做的就是设置不同日志的级别，那可以通过在application.properties中使用`logging.level`前缀实现，比如：
```java
logging.level.org.springframework.web: DEBUG
logging.level.org.hibernate: ERROR
```
你也可以使用`logging.file`设置日志文件的位置（除控制台之外，默认会输出到控制台）。

想要对日志系统进行更细粒度的配置，你需要使用正在说的LoggingSystem支持的原生配置格式。默认情况下，Spring Boot从系统的默认位置加载原生配置（比如对于Logback为`classpath:logback.xml`），但你可以使用`logging.config`属性设置配置文件的位置。

* 配置Logback

如果你将一个logback.xml放到classpath根目录下，那它将会被从这加载。Spring Boot提供一个默认的基本配置，如果你只是设置日志级别，那你可以包含它，比如：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    <logger name="org.springframework.web" level="DEBUG"/>
</configuration>
```
如果查看spring-boot jar包中的默认logback.xml，你将会看到LoggingSystem为你创建的很多有用的系统属性，比如：
- ${PID}，当前进程id
- ${LOG_FILE}，如果在Boot外部配置中设置了`logging.file`
- ${LOG_PATH}，如果设置了`logging.path`（表示日志文件产生的目录）

Spring Boot也提供使用自定义的Logback转换器在控制台上输出一些漂亮的彩色ANSI日志信息（不是日志文件）。具体参考默认的`base.xml`配置。

如果Groovy在classpath下，你也可以使用logback.groovy配置Logback。

* 配置Log4j

Spring Boot也支持[Log4j](http://logging.apache.org/log4j/1.2)或[Log4j 2](http://logging.apache.org/log4j/2.x)作为日志配置，但只有在它们中的某个在classpath下存在的情况。如果你正在使用starter poms进行依赖装配，这意味着你需要排除Logback，然后包含你选择的Log4j版本。如果你不使用starter poms，那除了你选择的Log4j版本外还要提供commons-logging（至少）。

最简单的方式可能就是通过starter poms，尽管它需要排除一些依赖，比如，在Maven中：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j</artifactId>
</dependency>
```
想要使用Log4j 2，只需要依赖`spring-boot-starter-log4j2`而不是`spring-boot-starter-log4j`。

**注**：使用Log4j各版本的starters都会收集好依赖以满足common logging的要求（比如，Tomcat中使用`java.util.logging`，但使用Log4j或 Log4j 2作为输出）。具体查看Actuator Log4j或Log4j 2的示例，了解如何将它用于实战。

* 使用YAML或JSON配置Log4j2

除了它的默认XML配置格式，Log4j 2也支持YAML和JSON配置文件。想要使用其他配置文件格式来配置Log4j 2，你需要添加合适的依赖到classpath。为了使用YAML，你需要添加`com.fasterxml.jackson.dataformat:jackson-dataformat-yaml`依赖，Log4j 2将查找名称为`log4j2.yaml`或`log4j2.yml`的配置文件。为了使用JSON，你需要添加`com.fasterxml.jackson.core:jackson-databind`依赖，Log4j 2将查找名称为`log4j2.json`或`log4j2.jsn`的配置文件

### 数据访问

* 配置一个数据源

想要覆盖默认的设置只需要定义一个你自己的DataSource类型的`@Bean`。Spring Boot提供一个工具构建类DataSourceBuilder，可用来创建一个标准的DataSource（如果它处于classpath下），或者仅创建你自己的DataSource，然后将它和在[Section 23.7.1, “Third-party configuration”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config-3rd-party-configuration)解释的一系列Environment属性绑定。

比如：
```java
@Bean
@ConfigurationProperties(prefix="datasource.mine")
public DataSource dataSource() {
    return new FancyDataSource();
}
```
```java
datasource.mine.jdbcUrl=jdbc:h2:mem:mydb
datasource.mine.user=sa
datasource.mine.poolSize=30
```
具体参考'Spring Boot特性'章节中的[Section 28.1, “Configure a DataSource”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-configure-datasource)和[DataSourceAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jdbc/DataSourceAutoConfiguration.java)类源码。

* 配置两个数据源

创建多个数据源和创建第一个工作都是一样的。如果使用针对JDBC或JPA的默认自动配置，你可能想要将其中一个设置为`@Primary`（然后它就能被任何`@Autowired`注入获取）。
```java
@Bean
@Primary
@ConfigurationProperties(prefix="datasource.primary")
public DataSource primaryDataSource() {
    return DataSourceBuilder.create().build();
}

@Bean
@ConfigurationProperties(prefix="datasource.secondary")
public DataSource secondaryDataSource() {
    return DataSourceBuilder.create().build();
}
```
* 使用Spring Data仓库

Spring Data可以为你的`@Repository`接口创建各种风格的实现。Spring Boot会为你处理所有事情，只要那些`@Repositories`接口跟你的`@EnableAutoConfiguration`类处于相同的包（或子包）。

对于很多应用来说，你需要做的就是将正确的Spring Data依赖添加到classpath下（对于JPA有一个`spring-boot-starter-data-jpa`，对于Mongodb有一个`spring-boot-starter-data-mongodb`），创建一些repository接口来处理`@Entity`对象。具体参考[JPA sample](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-data-jpa)或[Mongodb sample](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-data-mongodb)。

Spring Boot会基于它找到的`@EnableAutoConfiguration`来尝试猜测你的`@Repository`定义的位置。想要获取更多控制，可以使用`@EnableJpaRepositories`注解（来自Spring Data JPA）。

* 从Spring配置分离`@Entity`定义

Spring Boot会基于它找到的`@EnableAutoConfiguration`来尝试猜测你的`@Entity`定义的位置。想要获取更多控制，你可以使用`@EntityScan`注解，比如：
```java
@Configuration
@EnableAutoConfiguration
@EntityScan(basePackageClasses=City.class)
public class Application {

    //...

}
```
* 配置JPA属性

Spring Data JPA已经提供了一些独立的配置选项（比如，针对SQL日志），并且Spring Boot会暴露它们，针对hibernate的外部配置属性也更多些。最常见的选项如下：
```java
spring.jpa.hibernate.ddl-auto: create-drop
spring.jpa.hibernate.naming_strategy: org.hibernate.cfg.ImprovedNamingStrategy
spring.jpa.database: H2
spring.jpa.show-sql: true
```
（由于宽松的数据绑定策略，连字符或下划线作为属性keys作用应该是等效的）`ddl-auto`配置是个特殊情况，它有不同的默认设置，这取决于你是否使用一个内嵌数据库（create-drop）。当本地EntityManagerFactory被创建时，所有`spring.jpa.properties.*`属性都被作为正常的JPA属性（去掉前缀）传递进去了。

具体参考[HibernateJpaAutoConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/orm/jpa/HibernateJpaAutoConfiguration.java)和[JpaBaseConfiguration](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/orm/jpa/JpaBaseConfiguration.java)。

* 使用自定义的EntityManagerFactory

为了完全控制EntityManagerFactory的配置，你需要添加一个名为`entityManagerFactory`的`@Bean`。Spring Boot自动配置会根据是否存在该类型的bean来关闭它的实体管理器（entity manager）。

* 使用两个EntityManagers

即使默认的EntityManagerFactory工作的很好，你也需要定义一个新的EntityManagerFactory，因为一旦出现第二个该类型的bean，默认的将会被关闭。为了轻松的实现该操作，你可以使用Spring Boot提供的EntityManagerBuilder，或者如果你喜欢的话可以直接使用来自Spring ORM的LocalContainerEntityManagerFactoryBean。

示例：
```java
// add two data sources configured as above

@Bean
public LocalContainerEntityManagerFactoryBean customerEntityManagerFactory(
        EntityManagerFactoryBuilder builder) {
    return builder
            .dataSource(customerDataSource())
            .packages(Customer.class)
            .persistenceUnit("customers")
            .build();
}

@Bean
public LocalContainerEntityManagerFactoryBean orderEntityManagerFactory(
        EntityManagerFactoryBuilder builder) {
    return builder
            .dataSource(orderDataSource())
            .packages(Order.class)
            .persistenceUnit("orders")
            .build();
}
```
上面的配置靠自己基本可以运行。想要完成作品你也需要为两个EntityManagers配置TransactionManagers。其中的一个会被Spring Boot默认的JpaTransactionManager获取，如果你将它标记为`@Primary`。另一个需要显式注入到一个新实例。或你可以使用一个JTA事物管理器生成它两个。

* 使用普通的persistence.xml

Spring不要求使用XML配置JPA提供者（provider），并且Spring Boot假定你想要充分利用该特性。如果你倾向于使用`persistence.xml`，那你需要定义你自己的id为'entityManagerFactory'的LocalEntityManagerFactoryBean类型的`@Bean`，并在那设置持久化单元的名称。

默认设置可查看[JpaBaseConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/orm/jpa/JpaBaseConfiguration.java)

* 使用Spring Data JPA和Mongo仓库

Spring Data JPA和Spring Data Mongo都能自动为你创建Repository实现。如果它们同时出现在classpath下，你可能需要添加额外的配置来告诉Spring Boot你想要哪个（或两个）为你创建仓库。最明确地方式是使用标准的Spring Data `@Enable*Repositories`，然后告诉它你的Repository接口的位置（此处*即可以是Jpa，也可以是Mongo，或者两者都是）。

这里也有`spring.data.*.repositories.enabled`标志，可用来在外部配置中开启或关闭仓库的自动配置。这在你想关闭Mongo仓库，但仍旧使用自动配置的MongoTemplate时非常有用。

相同的障碍和特性也存在于其他自动配置的Spring Data仓库类型（Elasticsearch, Solr）。只需要改变对应注解的名称和标志。

* 将Spring Data仓库暴露为REST端点

Spring Data REST能够将Repository的实现暴露为REST端点，只要该应用启用Spring MVC。

Spring Boot暴露一系列来自`spring.data.rest`命名空间的有用属性来定制化[RepositoryRestConfiguration](http://docs.spring.io/spring-data/rest/docs/current/api/org/springframework/data/rest/core/config/RepositoryRestConfiguration.html)。如果需要提供其他定制，你可以创建一个继承自SpringBootRepositoryRestMvcConfiguration的`@Configuration`类。该类功能和RepositoryRestMvcConfiguration相同，但允许你继续使用`spring.data.rest.*`属性。

### 数据库初始化

一个数据库可以使用不同的方式进行初始化，这取决于你的技术栈。或者你可以手动完成该任务，只要数据库是单独的过程。

* 使用JPA初始化数据库

JPA有个生成DDL的特性，这些可以设置为在数据库启动时运行。这可以通过两个外部属性进行控制：

- `spring.jpa.generate-ddl`（boolean）控制该特性的关闭和开启，跟实现者没关系
- `spring.jpa.hibernate.ddl-auto`（enum）是一个Hibernate特性，用于更细力度的控制该行为。更多详情参考以下内容。

* 使用Hibernate初始化数据库

你可以显式设置`spring.jpa.hibernate.ddl-auto`，标准的Hibernate属性值有`none`，`validate`，`update`，`create`，`create-drop`。Spring Boot根据你的数据库是否为内嵌数据库来选择相应的默认值，如果是内嵌型的则默认值为`create-drop`，否则为`none`。通过查看Connection类型可以检查是否为内嵌型数据库，hsqldb，h2和derby是内嵌的，其他都不是。当从内存数据库迁移到一个真正的数据库时，你需要当心，在新的平台中不能对数据库表和数据是否存在进行臆断。你也需要显式设置`ddl-auto`，或使用其他机制初始化数据库。

此外，启动时处于classpath根目录下的import.sql文件会被执行。这在demos或测试时很有用，但在生产环境中你可能不期望这样。这是Hibernate的特性，和Spring没有一点关系。

* 使用Spring JDBC初始化数据库

Spring JDBC有一个DataSource初始化特性。Spring Boot默认启用了该特性，并从标准的位置schema.sql和data.sql（位于classpath根目录）加载SQL。此外，Spring Boot将加载`schema-${platform}.sql`和`data-${platform}.sql`文件（如果存在），在这里platform是`spring.datasource.platform`的值，比如，你可以将它设置为数据库的供应商名称（hsqldb, h2, oracle, mysql, postgresql等）。Spring Boot默认启用Spring JDBC初始化快速失败特性，所以如果脚本导致异常产生，那应用程序将启动失败。脚本的位置可以通过设置`spring.datasource.schema`和`spring.datasource.data`来改变，如果设置`spring.datasource.initialize=false`则哪个位置都不会被处理。

你可以设置`spring.datasource.continueOnError=true`禁用快速失败特性。一旦应用程序成熟并被部署了很多次，那该设置就很有用，因为脚本可以充当"可怜人的迁移"-例如，插入失败时意味着数据已经存在，也就没必要阻止应用继续运行。

如果你想要在一个JPA应用中使用schema.sql，那如果Hibernate试图创建相同的表，`ddl-auto=create-drop`将导致错误产生。为了避免那些错误，可以将`ddl-auto`设置为“”（推荐）或“none”。不管是否使用`ddl-auto=create-drop`，你总可以使用data.sql初始化新数据。

* 初始化Spring Batch数据库

如果你正在使用Spring Batch，那么它会为大多数的流行数据库平台预装SQL初始化脚本。Spring Boot会检测你的数据库类型，并默认执行那些脚本，在这种情况下将关闭快速失败特性（错误被记录但不会阻止应用启动）。这是因为那些脚本是可信任的，通常不会包含bugs，所以错误会被忽略掉，并且对错误的忽略可以让脚本具有幂等性。你可以使用`spring.batch.initializer.enabled=false`显式关闭初始化功能。

* 使用一个高级别的数据迁移工具

Spring Boot跟高级别的数据迁移工具[Flyway](http://flywaydb.org/)(基于SQL)和[Liquibase](http://www.liquibase.org/)(XML)工作的很好。通常我们倾向于Flyway，因为它一眼看去好像很容易，另外它通常不需要平台独立：一般一个或至多需要两个平台。

- 启动时执行Flyway数据库迁移

想要在启动时自动运行Flyway数据库迁移，需要将`org.flywaydb:flyway-core`添加到你的classpath下。

迁移是一些`V<VERSION>__<NAME>.sql`格式的脚本（`<VERSION>`是一个下划线分割的版本号，比如'1'或'2_1'）。默认情况下，它们存放在一个`classpath:db/migration`的文件夹中，但你可以使用`flyway.locations`（一个列表）来改变它。详情可参考flyway-core中的Flyway类，查看一些可用的配置，比如schemas。Spring Boot在[FlywayProperties](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/flyway/FlywayProperties.java)中提供了一个小的属性集，可用于禁止迁移，或关闭位置检测。

默认情况下，Flyway将自动注入（`@Primary`）DataSource到你的上下文，并用它进行数据迁移。如果你想使用一个不同的DataSource，你可以创建一个，并将它标记为`@FlywayDataSource`的`@Bean`-如果你这样做了，且想要两个数据源，记得创建另一个并将它标记为`@Primary`。或者你可以通过在外部配置文件中设置`flyway.[url,user,password]`来使用Flyway的原生DataSource。

这是一个[Flyway示例](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-flyway)，你可以作为参考。

- 启动时执行Liquibase数据库迁移

想要在启动时自动运行Liquibase数据库迁移，你需要将`org.liquibase:liquibase-core`添加到classpath下。

主改变日志（master change log）默认从`db/changelog/db.changelog-master.yaml`读取，但你可以使用`liquibase.change-log`进行设置。详情查看[LiquibaseProperties](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/liquibase/LiquibaseProperties.java)以获取可用设置，比如上下文，默认的schema等。

这里有个[Liquibase示例](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-liquibase)可作为参考。



 