# API数据序列化格式

## 0x1 背景

当前系统采用微服务架构设计，整个后台系统由多个互相解耦的微服务组成。这样就导致一个业务流程可能涉及多个服务之间HTTP调用，系统之间的通信采用JSON格式交换数据，但是这些内部数据并不需要传输到前端展示在页面上。因此我们可以选择一种效率更高的消息格式来传输内部数据，而非使用JSON这种纯文本的消息格式。

当前的消息格式主要分为文本格式和二进制格式，在web service中常用的文本数据格式为JSON和XML，二进格式比较多，完整的列表参考[Comparison_of_data_serialization_formats](https://en.wikipedia.org/wiki/Comparison_of_data_serialization_formats#cite_note-3)。

当前二进制数据交换格式比较多，经过比较我选择*Google Protocol Buffer(GPB)*作为JSON格式的比较对象，这里主要比较JSON格式和GPB格式的性能差异、易用性等。

## 0x2 JSON vs. GPB

### JSON

JSON格式简单、易懂，在web service中作为数据交换格式已经广泛使用，其优点有：

- 简单
- 可读性好
- 浏览器可以直接解析
- 数据可以自解释，无需定义schema

少数缺点如下：

- 体积大
- 不同的库性能差异比较大

### Protocol Buffer

[Protocol Buffer](https://developers.google.com/protocol-buffers/)是Google开发的数据序列化机制，其官方网页上有GPB与XML格式的数据对比：

- 比XML简单
- 比XML的体积小3-10倍
- 解析速度比XML块20-100倍
- 编程性更高，也更加清晰

我们更加关心GPB与JSON的对比，根据[http://www.infoq.com/presentations/RESTful-Web-Services-Orbitz](http://www.infoq.com/presentations/RESTful-Web-Services-Orbitz) 给出的数据，GPB的优点如下：

- 比JSON体积小
- 速度比JSON快7-10倍
- 客户端和服务端解耦，忽略未知字段

但是与JSON相比，GPB也有缺点：

- 需要定义Schema，降低开发效率
- 二进制难以阅读，不方便debug

完整的对比列表见 [ivm-serializer-comparison](https://github.com/eishay/jvm-serializers/wiki) 根据对比，推荐public API采用JSON格式，而internal API采用GPB格式。



## 0x3 Spring 集成 GPB

Spring从4.1开始支持GPB消息转换，因此如果需要在Spring框架中使用Protocolbuf，Spring需要升级到4.1+以上。

### 依赖关系(pom.xml)

```xml
<properties>
  <spring.version>4.2.4.RELEASE</spring.version>
</properties>
 
<dependency>
  <groupId>com.google.protobuf</groupId>
  <artifactId>protobuf-java</artifactId>
  <version>2.6.1</version>
</dependency>
```

### 消息转换

```xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.protobuf.ProtobufHttpMessageConverter"/>
    </mvc:message-converters>
</mvc:annotation-driven>
```

### IDEA 配置

1. 安装插件：Google Protocol Buffers support 
2. 在Compiler->Protocol Buffers Compiler中设置protoc路径

也可以配置Maven plugin实现代码自动生成。

```xml
<plugin>
  <groupId>com.google.protobuf.tools</groupId>
  <artifactId>maven-protoc-plugin</artifactId>
  <version>0.4.4</version>
  <configuration>
    <protocExecutable>/usr/local/bin/protoc</protocExecutable>
  </configuration>
  <executions>
    <execution>
      <id>generate-sources</id>
      <goals>
        <goal>compile</goal>
      </goals>
      <phase>generate-sources</phase>
      <configuration>
        <protoSourceRoot>${basedir}/src/main/proto/</protoSourceRoot>
        <includes>
          <param>**/*.proto</param>
        </includes>
      </configuration>
    </execution>
  </executions>
</plugin>
```



## 0x4 安装Protoc编译器

目前Ubuntu和OS X上libprotoc的版本并不一致，Ubuntu中的版本为2.5.0而OS X中的版本为2.6.1。根据[release notes](https://github.com/google/protobuf/releases) 2.5.0为2013年2月发布的版本，因此建议手动安装2.6.1版。

### macOS

```shell
> brew install protobuf
> protoc --version
libprotoc 2.6.1
```

### Ubuntu 14.04

手动安装时如果遇到configure错误，需要安装build-essential库。

```shell
> sudo apt-get install protobuf-compiler
> protoc --version
libprotoc 2.5.0

> sudo apt-get remove protobuf-compiler
> wget https://github.com/google/protobuf/releases/download/v2.6.1/protobuf-2.6.1.tar.gz
> tar -zxvf protobuf-2.6.1.tar.gz && cd protobuf-2.6.1
> ./configure
> make
> make check
> sudo make install

# 由于protobuf的默认安装路径为/usr/local/lib，并不在Ubuntu默认的LD_LIBRARY_PATH中，
# 因此需要手动添加LIBDIR到LD_LIBRARY_PATH环境变量中：
> sudo sh
> touch /etc/ld.so.conf.d/bprotobuf.conf
> cat >>bprotobuf.conf<<EOF
/usr/local/lib
EOF
> exit
> sudo ldconfig
> protoc --version
> libprotoc 2.6.1
```



## 0x5 引入Proto源码

首先需要根据消息格式定义的proto文件编译生成Java源码，然后在代码中直接使用即可。

1. 依据数据格式定义proto文件
2. 用命令行或者Idea插件编译proto文件生成Java源码
3. 使用编译生成的Java类包装数据

例如：

```java
package ping;

option java_package = "com.allinmoney.platform.utilservice.protos";
option java_outer_classname = "PingModel";
option optimize_for = SPEED;

message Ping {
    required string msg = 1;
    required string timestamp = 2;
    required string error = 3;
}
```

编译生成的Java源码类PingModel.java位于com.allinmoney.platform.utilservice.protos package中，在App代码中可以直接使用其Java类。

```java
@RequestMapping(value = "/v2/ping", method = RequestMethod.GET)
public ResponseEntity<PingModel.Ping> pingV2() {
    PingModel.Ping.Builder builder = PingModel.Ping.newBuilder();
    builder.setMsg("pong")
            .setError("0")
            .setTimestamp(new SimpleDateFormat(Constants.DATETIME_FORMAT).format(new Date()));
    logger.debug(builder.toString());
    return new ResponseEntity<PingModel.Ping>(builder.build(), HttpStatus.OK);
}
```



## 0x6 Protobuf2 和Protobuf3

Google在维护Protobuf version 2的同时，开发了version 3。目前v2和v3语义上是兼容的，但是根据文档，proto2的枚举类型不能在proto3语法中解析（[Using proton Message Type](https://developers.google.com/protocol-buffers/docs/proto3#other)）。

# Appendix

1. [Google Developers:Protocol-Buffers](https://developers.google.com/protocol-buffers/)
2. [5 Reasons to Use Protocol Buffers Instead of JSON For Your Next Service](http://blog.codeclimate.com/blog/2014/06/05/choose-protocol-buffers/)
3. [Serialization comparision](https://github.com/eishay/jvm-serializers/wiki)
4. [google protocol buffers vs Json vs XML](http://stackoverflow.com/questions/14028293/google-protocol-buffers-vs-json-vs-xml)
5. [Case Study: RESTful Web Services at Orbitz](http://www.infoq.com/presentations/RESTful-Web-Services-Orbitz)
6. [Using JAX-RS with Protocol Buffers for high-performance REST APIs](http://www.javarants.com/2008/12/27/using-jax-rs-with-protocol-buffers-for-high-performance-rest-apis/)
7. [Using Google Protocol Buffers with Spring MVC-based REST Services](https://spring.io/blog/2015/03/22/using-google-protocol-buffers-with-spring-mvc-based-rest-services)
8. [Spring4.1新特性 ---- Spring MVC增强 ](http://jinnianshilongnian.iteye.com/blog/2107205) 
9. [MessagePack,Protocol Buffers和Thrift序列化框架原理和比较说明](http://jimmee.iteye.com/blog/2042420)
10. [Performance Playground: Jackson vs. Protocol Buffers](http://technicalrex.com/2014/06/23/performance-playground-jackson-vs-protocol-buffers/)

