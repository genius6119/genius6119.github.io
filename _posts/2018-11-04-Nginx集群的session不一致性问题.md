---
layout:     post
title:      Nginx集群的session不一致性问题
subtitle:   ——出现原因与解决方案
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
## 解决方法

- 负载均衡规则设置为：ip_hash
    
    这样虽然能解决问题，但是万一其中一台服务器挂了，这个用户就访问不了这个服务了，用ip_hash只提高了服务器的并发能力，没有提高容灾能力。
    
- 数据库存session

    可以解决问题，但是会增加数据库压力，不推荐。

- Tomcat配置同步session

    会增加应用服务器压力，不推荐。
    
- Cookie存session
    
    Cookie不安全，若客户端禁用Cookie也无法进行session同步。    
    
- session外置，存缓存

    存到Redis或Memcache中,Session外置集中管理，推荐。
    
---
## Session存Redis例子

   见下一篇。