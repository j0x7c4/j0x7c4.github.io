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

Drools是一个bussiness rule management system (BRMS)，实现了规则引擎中广泛使用的Ｒete算法

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

{% codeblock %}
  <role rolename="tomcat"/>
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <role rolename="manager-jmx"/>
  <role rolename="manager-status"/>
  <role rolename="admin"/>
  <role rolename="kie-server"/>
  <user username="tomcat" password="tomcat" roles="tomcat,manager-gui,manager-script,manager-jmx,manager-status,admin,kie-server"/>
{% endcodeblock %}

## Tomcat安装kie workbench需要的JAR
* http://central.maven.org/maven2/org/codehaus/btm/btm-tomcat55-lifecycle/2.1.4/btm-tomcat55-lifecycle-2.1.4.jar
* http://central.maven.org/maven2/com/h2database/h2/1.3.161/h2-1.3.161.jar
* http://central.maven.org/maven2/javax/transaction/jta/1.1/jta-1.1.jar
* http://central.maven.org/maven2/org/slf4j/slf4j-api/1.7.2/slf4j-api-1.7.2.jar
* http://central.maven.org/maven2/org/slf4j/slf4j-jdk14/1.7.2/slf4j-jdk14-1.7.2.jar
* http://central.maven.org/maven2/org/kie/kie-tomcat-integration/6.3.0.Final/kie-tomcat-integration-6.3.0.Final.jar
* http://central.maven.org/maven2/org/jboss/spec/javax/security/jacc/jboss-jacc-api_1.4_spec/1.0.2.Final/jboss-jacc-api_1.4_spec-1.0.2.Final.jar
* https://repository.jboss.org/nexus/content/repositories/thirdparty-releases/org/slf4j/slf4j-api/1.7.2.jbossorg-1/slf4j-api-1.7.2.jbossorg-1.jar

## 使用kie-workbench和kie-server

* kie-workbench: http://docs.jboss.org/drools/release/6.3.0.Final/drools-docs/html/ch18.html

* kie-server: http://docs.jboss.org/drools/release/6.3.0.Final/drools-docs/html/ch22.html

## 会被坑的地方
* Tomcat如果通过bin/shutdown.sh关闭，再次重新启动时workbench会报错(previous error)无法启动。因为之前启动workbench会有一个9418的端口被占用了(git)，需要手动Kill那个进程，具体用`lsof -i:9418`查到对应的pid，然后再kill -9杀掉进程。

## 有问题的地方
* 目前通过workbench中建立的container,　在kie-server的REST api`/services/rest/server/containers`中看不到，应该是没有正确创建。