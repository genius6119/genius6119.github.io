---
layout:     post
title:      Nginx集群的session不一致性问题
subtitle:   ——出现原因与解决方案的笔记
date:       2018-11-05
author:     Zwx
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Linux
---

## Session
   简单的说就是保存用户登陆信息和唯一标识的一个东西
   
---
## 演示
修改接口返回信息,多返回一条sessionId：
```java
public class HelloController {
    @ApiOperation(value = "这是一个作用为Hello的类",notes = "备注说明")
    @RequestMapping(value = "/hello", method= RequestMethod.GET)
    public String hello(int i, HttpSession session){
        System.out.println(i);
        return "Hello！！！！"+"_"+i+"_"+session.getId();
    }
}
```
负载均衡规则设置为轮询，多次发请求的效果：

![](http://pic.zwxzzz.top/session.png)

发现每一次请求的session都不一样，很难受。

--- 
## 原因

