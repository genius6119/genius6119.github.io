---
layout:     post
title:      SpringBoot项目整合Swagger框架
subtitle:   SpringBoot1.5.18 + Swagger2
date:       2018-10-23
author:     Zwx
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - SpringBoot
    - Java
---

----
## 前言：

参加工作这几个月，做的项目都是前后端分离的:前端渲染页面，发送请求给后端，后端接受请求处理参数再返回给前端。

前后端的联系只有接口(API),因此，API文档变得特别重要。

在我工作的第一个项目上，我还不知道有Swagger这个东西，写的接口都是通过手写API文档说明，然后钉钉发消息的形式向前端解释，很麻烦也很难受。

现在了解到了Swagger这个框架，记录一下整合过程。

----
## 配置

因为是maven项目，所以pom.xml里添加依赖（版本号可变）：

```xml
        	<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger2</artifactId>
			<version>2.2.2</version>
		</dependency>

		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger-ui</artifactId>
			<version>2.2.2</version>
		</dependency>
```

然后在xxxapplication.java的同级目录下创建配置类Swagger2.java：

```java
package com.zwx.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * Swagger2配置类
 * 在与spring boot集成时，放在与Application.java同级的目录下。
 * 通过@Configuration注解，让Spring来加载该类配置。
 * 再通过@EnableSwagger2注解来启用Swagger2。
 */
@Configuration
@EnableSwagger2
public class Swagger2 {

    /**
     * 创建API应用
     * apiInfo() 增加API相关信息
     * 通过select()函数返回一个ApiSelectorBuilder实例,用来控制哪些接口暴露给Swagger来展现，
     * 本例采用指定扫描的包路径来定义指定要建立API的目录。
     *
     * @return
     */
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.zwx.demo"))
                .paths(PathSelectors.any())
                .build();
    }

    /**
     * 创建该API的基本信息（这些基本信息会展现在文档页面中）
     * 访问地址：http://项目实际地址/swagger-ui.html
     * @return
     */
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot中使用Swagger2构建RESTful APIs")
                .description("更多请关注http://www.baidu.com")
                .termsOfServiceUrl("http://www.baidu.com")
                .contact("sunf")
                .version("1.0")
                .build();
    }
}
```

注意：`.apis(RequestHandlerSelectors.basePackage("com.zwx.demo"))`里的地址是你controller类的上层目录，一定要写对。


接着在你的xxxAplication.java类头上加开启Swagger的注解：
```java
@SpringBootApplication
@EnableSwagger2
public class DemoApplication {
	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```


----
## 运行

配置到其实这里直接可以运行访问了，启动项目，访问`项目地址：端口号:/swagger-ui.html`效果如下:
![](http://pic.zwxzzz.top/swa.png)

点击开发现有很多重复的：

![](http://pic.zwxzzz.top/123.png)

原因：要指定RequestMapping的method:
```java
@RequestMapping(value = "/test",method = RequestMethod.POST)
```

---
## 注解

Swagger是通过注解的方式自动生成API文档的，主要的注解有：
```
@Api：用在请求的类上，表示对类的说明
    tags="说明该类的作用，可以在UI界面上看到的注解"
    value="该参数没什么意义，在UI界面上也看到，所以不需要配置"
 
@ApiOperation：用在请求的方法上，说明方法的用途、作用
    value="说明方法的用途、作用"
    notes="方法的备注说明"
 
@ApiImplicitParams：用在请求的方法上，表示一组参数说明
    @ApiImplicitParam：用在@ApiImplicitParams注解中，指定一个请求参数的各个方面
        name：参数名
        value：参数的汉字说明、解释
        required：参数是否必须传
        paramType：参数放在哪个地方
            · header --> 请求参数的获取：@RequestHeader
            · query --> 请求参数的获取：@RequestParam
            · path（用于restful接口）--> 请求参数的获取：@PathVariable
            · body（不常用）
            · form（不常用）    
        dataType：参数类型，默认String，其它值dataType="Integer"       
        defaultValue：参数的默认值
 
@ApiResponses：用在请求的方法上，表示一组响应
    @ApiResponse：用在@ApiResponses中，一般用于表达一个错误的响应信息
        code：数字，例如400
        message：信息，例如"请求参数没填好"
        response：抛出异常的类
 
@ApiModel：用于响应类上，表示一个返回响应数据的信息
            （这种一般用在post创建的时候，使用@RequestBody这样的场景，
            请求参数无法使用@ApiImplicitParam注解进行描述的时候）
    @ApiModelProperty：用在属性上，描述响应类的属性

```

例子：

```java
@Api(value = "这是一个Hello类" ,tags = {"Hello TAGS"})
@RestController
public class HelloController {
    @ApiOperation(value = "这是一个作用为Hello的类",notes = "备注说明")
    @RequestMapping(value = "/hello", method= RequestMethod.GET)
    public String hello(){
        System.out.println("hello");
        return "Hello！！！！";
    }
}
```

效果：

![](http://pic.zwxzzz.top/234.png)

例子2：
```java
    @ResponseBody
    @RequestMapping(value = "/updatePassword", method= RequestMethod.GET)
    @ApiOperation(value="修改用户密码", notes="根据用户id修改密码")
    @ApiImplicitParams({
            @ApiImplicitParam(paramType="query", name = "userId", value = "用户ID", required = true, dataType = "Integer"),
            @ApiImplicitParam(paramType="query", name = "password", value = "旧密码", required = true, dataType = "String"),
            @ApiImplicitParam(paramType="query", name = "newPassword", value = "新密码", required = true, dataType = "String")
    })
    public String updatePassword(@RequestParam(value="userId") Integer userId, @RequestParam(value="password") String password,
                                 @RequestParam(value="newPassword") String newPassword){
        if(userId <= 0 || userId > 2){
            return "未知的用户";
        }
        if(StringUtils.isEmpty(password) || StringUtils.isEmpty(newPassword)){
            return "密码不能为空";
        }
        if(password.equals(newPassword)){
            return "新旧密码不能相同";
        }
        return "密码修改成功!";
    }
```

效果：
![](http://pic.zwxzzz.top/456.png)


----




参考：


[Swagger官网](http://swagger.io/)

[https://blog.csdn.net/sanyaoxu_2/article/details/80555328](https://blog.csdn.net/sanyaoxu_2/article/details/80555328)
