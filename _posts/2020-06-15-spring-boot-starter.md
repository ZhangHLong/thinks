---
layout: post
title: SpringBoot-starter 
categories: springboot
description: SpringBoot-starter 
keywords: SpringBoot-starter 
---

## 开发一个SpringBoot-starter

### 1、开发步骤：
1. 新建一个Maven工程， 定义Pom依赖
2. 新建一个配置类，写好配置项和默认配置项，指定配置项前缀
3. 新建一个自动装配类，使用@Configuration @Bean 来进行自动装配
4. 新建一个spring.factories 文件，指定starter的自动装配类

<img src="/images/posts/spring/starter-1.png"   />

### 2、实例

#### 1. 配置项：
 **MyJsonProperties**
 ```$xslt
@SuppressWarnings("ConfigurationProperties")
@Data
@ConfigurationProperties(prefix = "long.json")
public class MyJsonProperties {

    public static final String DEFAULT_NAME = "long";

    private  String name = DEFAULT_NAME;
}

```

**MyJsonAutoConfiguration**

```$xslt
@Configuration
@ConditionalOnClass(MyJsonProperties.class)
@EnableConfigurationProperties(MyJsonProperties.class)
public class MyJsonAutoConfiguration {

    @Autowired
    private MyJsonProperties myJsonProperties;

    @Bean
    public MyJsonService myJsonService(){
        MyJsonService myJsonService = new MyJsonService();
        myJsonService.setName(myJsonProperties.getName());

        return myJsonService;
    }

}
```

#### 2. 服务接口

**MyJsonService**

```$xslt
@Data
public class MyJsonService {

    private String name;


    @Bean
    public String myJsonService(){

        return name + "@" + JSON.toJSONString(name);
    }
}
```

#### 3. 配置项
 **src/main/resources/META-INF/spring.factories**
 
```$xslt
com.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.test.myjson.config.MyJsonAutoConfiguration
```

### 3、使用

<img src="/images/posts/spring/starter-2.png"   />

```$xslt
server:
  port: 8080
# 动态扩展配置信息，默认采用starter内部配置
long:
    json:
      name: "zhanghuilong"
```
