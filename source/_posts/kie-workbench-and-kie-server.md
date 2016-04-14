---
title: Kie Workbench 和 Kie server
date: 2016-04-14 00:57:39
tags:
  - drools
  - kie-server
  - kie-workbench
categories:
  - 后端工程师
  - 规则引擎
---
最近尝试用规则引擎去处理一些事务逻辑。在开源的规则引擎系统里用的比较广泛的就是Drools啦。

## drools, kie-workbench, kie-server是些什么东西

### drools

Drools是一个bussiness rule management system (BRMS)，实现了规则引擎中广泛使用的Rete算法

### kie-workbench

KIE (knowledge is everything)包含了主体drools之外，还有其他一些附加功能。

workbench是一个在线工具，包括drools项目的建立，编辑，版本控制，构建和部署等很多功能，是一个web　IDE.

### kie-server

kie-server是一个独立的REST服务，主要是执行规则。

## 安装Tomcat
* 去tomcat官网下个tomcat8+的版本就行了(至少7+)
* 解压放在合适的地方(如果时Linux, 建议放在`/usr/local/`下面（不要问为什么，我有强迫症）
* 在环境变量中加入`export TOMCAT_HOME=/usr/local/apache-tomcat-8.0.33`
* 在$TOMCAT_HOME/conf/tomcat-users.xml中加入role和user

```xml
  <role rolename="tomcat"/>
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <role rolename="manager-jmx"/>
  <role rolename="manager-status"/>
  <role rolename="admin"/>
  <role rolename="kie-server"/>
  <user username="tomcat" password="tomcat" roles="tomcat,manager-gui,manager-script,manager-jmx,manager-status,admin,kie-server"/>
```

## Tomcat安装kie workbench需要的JAR
* http://central.maven.org/maven2/org/codehaus/btm/btm-tomcat55-lifecycle/2.1.4/btm-tomcat55-lifecycle-2.1.4.jar
* http://central.maven.org/maven2/com/h2database/h2/1.3.161/h2-1.3.161.jar
* http://central.maven.org/maven2/javax/transaction/jta/1.1/jta-1.1.jar
* http://central.maven.org/maven2/org/slf4j/slf4j-api/1.7.2/slf4j-api-1.7.2.jar
* http://central.maven.org/maven2/org/slf4j/slf4j-jdk14/1.7.2/slf4j-jdk14-1.7.2.jar
* http://central.maven.org/maven2/org/kie/kie-tomcat-integration/6.3.0.Final/kie-tomcat-integration-6.3.0.Final.jar
* http://central.maven.org/maven2/org/jboss/spec/javax/security/jacc/jboss-jacc-api_1.4_spec/1.0.2.Final/jboss-jacc-api_1.4_spec-1.0.2.Final.jar
* https://repository.jboss.org/nexus/content/repositories/thirdparty-releases/org/slf4j/slf4j-api/1.7.2.jbossorg-1/slf4j-api-1.7.2.jbossorg-1.jar

## 安装kie-workbench

workbench在drools的官网能下载，我用的是6.3.0-final的版本。

将下载的war包放在tomcat的webapps目录下面，会自动解压（前提是tomcat正在运行）。然后会发现多了一个workbench的目录，里面有个README.txt，具体描述安装过程，其中需要的jar包在上面提供了下载链接。


## 安装kie-server

在官网下载下来的ZIP包中包含了3个war包，我用的是tomcat，所以选择了-webc的war包放在webapps目录下面。

## 配置setenv.sh

在$TOMCAT_HOME/bin目录下建立setenv.sh，配置如下：

```bash
CATALINA_OPTS="-Xmx512M -XX:MaxPermSize=512m -Dbtm.root=$CATALINA_HOME \
    -Dbitronix.tm.configuration=$CATALINA_HOME/conf/btm-config.properties \
    -Djbpm.tsr.jndi.lookup=java:comp/env/TransactionSynchronizationRegistry \
    -Djava.security.auth.login.config=$CATALINA_HOME/webapps/kie-drools-wb/WEB-INF/classes/login.config \
    -Dorg.jboss.logging.provider=jdk \
    -Dorg.kie.server.controller=http://localhost:8080/kie-drools-wb/rest/controller \
    -Dorg.kie.server.controller.user=tomcat \
    -Dorg.kie.server.controller.pwd=tomcat \
    -Dorg.kie.server.id=sherry-kie-server \
    -Dorg.kie.server.location=http://localhost:8080/kie-server-webc/services/rest/server"
```

其中的org.ki.server的配置项在workbench的readme中没有提到，我在掉了很多次坑之后终于发现了这些配置项的重要性。主要是org.kie.server.controller一定配置，不然通过workbench建立的container会找不到endpoints. 用kie-server的REST　api也看不到container的信息。

* org.kie.server.controller: 这个是维持kie-server配置的控制器，它的地址就是workbench路径的/rest/controller

* org.kie.server.controller.user: 登陆workbench的用户名

* org.kie.server.controller.pwd: 登陆workbench的密码

## 使用kie-workbench和kie-server

* kie-workbench: http://docs.jboss.org/drools/release/6.3.0.Final/drools-docs/html/ch18.html

* kie-server: http://docs.jboss.org/drools/release/6.3.0.Final/drools-docs/html/ch22.html

## 配置好的container

![container](/images/container.png) 

## 会被坑的地方
* Tomcat如果通过bin/shutdown.sh关闭，再次重新启动时workbench会报错(previous error)无法启动。因为之前启动workbench会有一个9418的端口被占用了(git)，需要手动Kill那个进程，具体用`lsof -i:9418`查到对应的pid，然后再kill -9杀掉进程。

## 有问题的地方
* 如果workbench中建立的container,　在kie-server的REST api`/services/rest/server/containers`中看不到，应该是org.kie.server.controller没有配置正确。