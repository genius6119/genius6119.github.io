---
layout:     post
title:      使用SpringBoot容易犯的错误整理
subtitle:   在一个空的SpringBoot项目上的例子
date:       2018-10-18
author:     Zwx
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - SpringBoot
---

----
## 前言
[SpringBoot](https://spring.io/projects/spring-boot)是Spring社区2014年发布的一个轻量级微服务框架，解决了Spring“配置地狱”的问题。

该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。

其实SpringBoot的本质还是ssm，只是不用进行复杂的xml配置，以及内置了tomcat。

我在初学JavaWeb时，照着教学视频敲过一个SpringBoot的项目，这几天工作上的活少，自己又从头搭了一个SpringBoot的框架，把遇到的问题记录下来。

---
## 新建SpringBoot项目

虽然可以直接在IDEA中新建，但我还是习惯进入[spring.start.io](https://start.spring.io/)网页设置。两种都很方便，这里介绍使用网页下载的方式：

进入网页：
Group设置组织名、Artifact设置项目名、选择Maven Project，Javelin项目，版本随便选 = = 
![](http://pgoj9ayje.bkt.clouddn.com/start.png)
点击Generate Project按钮,就可以得到一个空的SpringBoot项目了，解压后用idea直接打开就行。

---
## SpringBoot配置
打开项目，结构如图所示：
![](http://pgoj9ayje.bkt.clouddn.com/mulu.png)
配置文件都放在resources下，除了.xml配置文件外，SpringBoot还支持.yml和.properties配置文件。个人认为，后两者都比.xml好很多。

对于同一段数据库配置，

.xml配置文件长这样：
```$xslt
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
		  destroy-method="close">
		<property name="driverClassName" value="com.mysql.jdbc.Driver" />
		<property name="url" value="jdbc:mysql://localhost:3306/dbname  " />
		<property name="username" value="root" />
		<property name="password" value="root" />
</bean>
```
.properties配置文件长这样：
```$xslt
spring.datasource.url = jdbc:mysql://localhost:3306/dbname  
spring.datasource.username = root  
spring.datasource.password = root  
spring.datasource.driverClassName = com.mysql.jdbc.Driver  
```

.yml配置文件长这样：
```$xslt
spring:  
  datasource:  
    url : jdbc:mysql://localhost:3306/dbname  
    username : root  
    password : root  
    driverClassName : com.mysql.jdbc.Driver  
```

.yml的可读性和简洁性都比.xml好太多。当然，这个东西也是仁者见仁。

---
## Controller 接口404问题

必须把Controller类放到XXXApplication.java同级或同级包的下层目录中,不然启动Application.main(SpringBoot启动方法)启动时会扫描不到。

就像这样：

![](http://pgoj9ayje.bkt.clouddn.com/mulu2.png)

---
##Mapper.xxx方法找不到

用Springboot连数据库测试时候，报出"Invalid bound statement (not found)"错误：

![](http://pgoj9ayje.bkt.clouddn.com/error.png)

经检查是maven的pom.xml文件中忘记加上：
```$xslt
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
            </resource>
        </resources>
```

注意：`<resources>`是加在`<build>`标签里的。

---
## 上传文件大小限制

使用mutipartfile上传文件时，报错：
```
org.apache.tomcat.util.http.fileupload.FileUploadBase$FileSizeLimitExceededException: The field pic exceeds its maximum permitted size of 1048576 bytes.
```

查询原因得知，原来SpringBoot默认上传文件大小为1Mb，在application.yml配置文件中配以下就好：
```$xslt
spring:
  http:
    multipart:
      maxFileSize: 100Mb
      maxRequestSize: 1000Mb
```

---
## SpringBoot定时器

在xxxAplication.java 的类头上加注解：
```
@SpringBootApplication
@EnableScheduling
public class MumushowApplication {

```
在方法上加上注解：
```
    @Scheduled(cron = "0/5 * * * * ? ")
    public String test (){
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");	//设置日期格式
        System.out.println(df.format(new Date()));				// new Date()为获取当前系统时间
        return null;
    }
```
控制台输出结果：
```
2018-10-18 16:57:00
2018-10-18 16:57:05
2018-10-18 16:57:10
2018-10-18 16:57:15
2018-10-18 16:57:20
2018-10-18 16:57:25
2018-10-18 16:57:30
```




