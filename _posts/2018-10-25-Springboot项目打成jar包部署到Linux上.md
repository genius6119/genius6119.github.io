---
layout:     post
title:      SpringBoot项目打成jar包部署到Linux上
subtitle:   SpringBoot1.5.18 + CentOS7.4
date:       2018-10-25
author:     Zwx
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - SpringBoot
    - Linux
---

---
## 用IDEA打包

直接用maven插件打包,位置：`Maven Projects`-`Lifecycle`-`package`

![](http://pic.zwxzzz.top/maven.png)
  
控制台输出`BUILD SUCCESS`时，说明打包成功
```
...
[INFO] --- maven-jar-plugin:2.6:jar (default-jar) @ demo ---
[INFO] Building jar: C:\Project2\demo\target\demo-1.0.0.jar
[INFO] 
[INFO] --- spring-boot-maven-plugin:1.5.17.BUILD-SNAPSHOT:repackage (default) @ demo ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 15.989 s
[INFO] Finished at: 2018-10-25T17:10:11+08:00
[INFO] ------------------------------------------------------------------------

Process finished with exit code 0
```

可以在pom.xml文件里控制jar包的名字和版本号
```xml
        <groupId>com.zwx</groupId>
	<artifactId>demo</artifactId>
	<version>1.0.0</version>
	<packaging>jar</packaging>
```

进入`\项目\target\ `目录下找到打包好的jar包，用Xftp工具放到linux服务器上，复制粘贴即可。

其实到这一步可以直接用:
```
java -jar /位置/jar包名.jar
```
运行了，但是直接这样做的话在停止springboot服务时很麻烦，需要找到这个jar包占用的进程，接着杀死相关进程。
所以为了方便后期维护，使用shell脚本进行启动、关闭和检查。

---
## 编写脚本

#### 启动脚本

使用Xshell连接到服务器后，在jar包同级目录下（为了方便），敲命令：`vim start.sh`，输入：

```

#!/bin/sh

rm -f tpid
nohup java -Xms1536m -Xmx1536m -jar demo.jar --spring.config.location=/appsystems/IFC/config/application.properties > launch.log 2>&1 &
echo $! > tpid
echo Start Success!
```
其中demo.jar 是要执行的jar包，launch.log是日志输出的位置。

访问ip+端口号:/接口名，如下图所示，表明SpringBoot部署启动成功：

![](http://pic.zwxzzz.top/4445.png)

同时可以看到目录下生成了日志文件：

![](http://pic.zwxzzz.top/xftp.png)

#### 停止脚本
输入命令：`vim stop.sh`，下面这段粘进去：

```
#!/bin/sh
APP_NAME=demo
tpid=`ps -ef|grep $APP_NAME|grep -v grep|grep -v kill|awk '{print $2}'`
if [ ${tpid} ]; then
    echo 'Stop Process...'
    kill -15 $tpid
fi
sleep 5
tpid=`ps -ef|grep $APP_NAME|grep -v grep|grep -v kill|awk '{print $2}'`
if [ ${tpid} ]; then
    echo 'Kill Process!'
    kill -9 $tpid
else
    echo 'Stop Success!'
fi
```

其中APP_NAME改成你自己的jar包名称。运行stop脚本,输出如下表示运行成功：

```
[root@iz2tx5qq7tthc7z sb_demo]# sh stop.sh 
Stop Process...
Stop Success!
```

访问接口,发现已经无法访问，说明脚本运行成功：

![](http://pic.zwxzzz.top/fail.png)

----
#### 检查脚本

vim check.sh,输入：

```
#!/bin/sh
APP_NAME=demo
tpid=`ps -ef|grep $APP_NAME|grep -v grep|grep -v kill|awk '{print $2}'`
if [ ${tpid} ]; then
        echo 'App is running.'
else
        echo 'App is NOT running.'
fi

```

效果：
```
[root@iz2tx5qq7tthc7z sb_demo]# sh check.sh 
   App is NOT running.
```