---
title: Springboot项目同时支持http和https访问
date: 2020-09-15 16:35:53
categories:
  - [Java, Springboot]
  - [技术日常]
tags:
  - Java
  - Springboot
top_image: /assets/blogpicture/car.jpg
---

因公司业务需求，需要项目同时支持http和https访问，在此记录如何实现
项目采用springboot 2.x搭建，实现与springboot 1.x略有不同，使用springboot 2.x的可以参考实现
<!-- more -->

## 实现步骤：
#### 1. 首先项目为sprintboot 2.x搭建，并引入web模块
#### 2. 将https需要的证书放入项目中
##### 2.1 证书来源
- 如果公司提供，则直接使用公司提供的证书
- 如果公司没有提供，也可自己使用Java自带的命令keytool来生成
	- windows下cmd打开命令黑窗口，输入以下命令（直接复制即可）
	```
	keytool -genkey -alias tomcat  -storetype PKCS12 -keyalg RSA -keysize 2048  -keystore keystore.p12 -validity 3650
	```
	- 按照提示输入生成证书所需信息，就会在系统的当前用户目录下生成一个keystore.p12文件（如果你修改了证书文件的名称那就是你修改的名字）
	- 简单的参数说明：
	```
	1. -storetype 指定密钥仓库类型 
	2. -keyalg 生证书的算法名称，RSA是一种非对称加密算法 
	3. -keysize 证书大小 
	4. -keystore 生成的证书文件的存储路径 
	5. -validity 证书的有效期
	```

##### 2.2 证书放置位置
- 证书可以放在项目的根目录下，即和pom文件同级的目录
- 证书也可放置在```src/main/resources```目录下
- 两者放置位置不同配置时证书路径写法稍有不同，下文会具体说明

#### 3. 在配置文件中配置支持https所需信息
```properties
# 支持https访问
# https访问的端口号
server.port=8443
# 证书的路径，根据证书放置位置不同，写法不同
# 如果证书放在根目录下，此处只需要写证书的名字即可，但项目打包部署时提示证书找不到，故建议放在resources文件夹下
# 如果证书放在 src/main/resources 下，则需写 classpath:keystore/server.keystore
server.ssl.key-store=classpath:keystore/server.keystore
# 证书的签名密码，如果是自己生成的证书在输入信息时会有输入
server.ssl.key-store-password=slipper
# 证书类型，常见的两种证书类型有：PKCS12和JKS，这里需要注意证书类型不能写错了，否则项目启动时会报错
server.ssl.keyStoreType=JKS
```
- 按照如上配置即可通过配置的8443端口实现https访问了
- 例如：https://127.0.0.1:8443/...

#### 4. 添加Java配置类使项目同时支持http访问
- 因一个项目只能配置一个 ```server.port```，所以要支持http访问需要用Java代码实现

4.1 配置文件中添加http端口配置
```properties
http.port=8060
```
4.2 书写Java配置类
- 此处需要注意很多博客中提到的```EmbeddedServletContainerFactory```相关类，在springboot 2.x版本中已经被弃用，需要使用```WebServerFactoryCustomizer<ConfigurableWebServerFactory>```这种写法
```java
/**
 * 
 * @function   http访问配置类
 *
 */
@Configuration
public class TomcatConfig {
    
    @Value("${http.port}")
    private int httpPort;

    @Bean
    public WebServerFactoryCustomizer<ConfigurableWebServerFactory> webServerFactoryCustomizer() {
        return new WebServerFactoryCustomizer<ConfigurableWebServerFactory>() {

            @Override
            public void customize(ConfigurableWebServerFactory factory) {
                if (factory instanceof TomcatServletWebServerFactory) {
                    TomcatServletWebServerFactory webServerFactory = (TomcatServletWebServerFactory)factory;
                    Connector connector = new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
                    // 设置http访问的端口号，不能与https端口重复，否则会报端口被占用的错误
                    connector.setPort(httpPort);
                    webServerFactory.addAdditionalTomcatConnectors(connector);
                }
            }
        };
    }
      
}
```
- 此时就可以使用8060端口http访问了
- 例如：http://127.0.0.1:8060/...

> 至此项目就实现了既能支持https访问，又能支持http访问

- 参考资料：
[springboot官方demo](https://github.com/spring-projects/spring-boot/blob/v2.0.0.RELEASE/spring-boot-samples/spring-boot-sample-tomcat-multi-connectors/src/main/java/sample/tomcat/multiconnector/SampleTomcatTwoConnectorsApplication.java)