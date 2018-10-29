---
layout:     post
title:      @Transactional注解的用法和实例
subtitle:   自己写的声明式事务@Transactional
date:       2018-10-29
author:     Zwx
header-img: img/post-bg-digital-native.jpg
catalog: true
tags:
    - Sql
    - Java
---
---
## 参数

@Transactional可以配制那些参数及以其所代表的意义：

| 参数       | 意义           |
| ------------- |:-------------| 
| isolation     | 事务隔离级别 | 
| propagation | 事务传播机制     |
| readOnly | 事务读写性     |
| noRollbackFor     | 一组异常类，遇到时不回滚。默认为{}。      |
| noRollbackForClassName | 一组异常类名，遇到时不回滚，默认为{}    |
| rollbackFor | 一组异常类，遇到时回滚     |
| rollbackForClassName | 一组异常类名，遇到时回滚      |
| timeout | 超时时间，以秒为单位      |
| value | 可选的限定描述符，指定使用的事务管理器      |

#### isolation

`isolation`属性可配置的值有：

- Isolation.READ_COMMITTED      :使用各个数据库默认的隔离级别
- Isolation.READ_UNCOMMITTED    :读未提交数据(会出现脏读,不可重复读,幻读)
- Isolation.READ_COMMITTED      :读已提交的数据(会出现不可重复读,幻读)
- Isolation.REPEATABLE_READ     :可重复读(会出现幻读)
- Isolation.SERIALIZABLE        :串行化
 
###### 数据库默认隔离级别
- MYSQL: 默认为REPEATABLE_READ级别
- SQLSERVER: 默认为READ_COMMITTED
- Oracle 默认隔离级别 READ_COMMITTED

----
#### propagation
`propagation`属性可配置的值有：

| 传播机制        | 说明           |
| ------------- |:-------------| 
| Propagation.REQUIRED     | 如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是 最常见的选择，也是Spring的默认传播机制。 | 
| Propagation.SUPPORTS     | 支持当前事务，如果当前没有事务，就以非事务方式执行。      |
| Propagation.MANDATORY | 使用当前的事务，如果当前没有事务，就抛出异常。    |
| Propagation.REQUIRES_NEW | 新建事务，如果当前存在事务，把当前事务挂起。     |
| Propagation.NOT_SUPPORTED | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。     |
| Propagation.NEVER | 以非事务方式执行，如果当前存在事务，则抛出异常。     |
| Propagation.NESTED | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与 PROPAGATION_REQUIRED 类似的操作。      |

上一篇写事务的文章中已经把这两个属性说得很清楚了，这里就不再赘述了。

----
#### readOnly

默认情况下是false，可以显示指定为true， 告诉程序该方法下使用的是只读操作，如果进行其他非读操作，则会跑出异常；

- 概念：
从这一点设置的时间点开始（时间点a）到这个事务结束的过程中，其他事务所提交的数据，该事务将看不见！（查询中不会出现别人在时间点a之后提交的数据）

- 应用场合：
如果你一次执行单条查询语句，则没有必要启用事务支持，数据库默认支持SQL执行期间的读一致性； 
如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询SQL必须保证整体的读一致性，否则，在前条SQL查询之后，后条SQL查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用事务支持。

【注意是一次执行多次查询来统计某些信息，这时为了保证数据整体的一致性，要用只读事务】

----
#### rollbackForClassName/rollbackFor

Spring默认情况下会对运行期例外(RunTimeException)进行事务回滚。这个例外是unchecked，如果遇到checked意外就不回滚。

用来指明回滚的条件是哪些异常类或者异常类名。

---
#### noRollbackForClassName/noRollbackFor

用来指明不回滚的条件是哪些异常类或者异常类名。
