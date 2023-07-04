---
title: Netty常见问题
date: 2023-06-21 22:06:22
tags:
 - Netty
 - java
categories:
 - Netty
comments: true
---

# Netty常见问题

### 1. Could not find artifact io.netty:netty-tcnative:jar:windows-x86_32:2.0.25.Final in centra

netty编译时，如果是32位系统，则会报错”Could not find artifact io.netty:netty-tcnative:jar:windows-x86_32:2.0.25.Final in centra“。

![image-20230614154337227](img/Netty常见问题/image-20230614154337227.png)

在netty-parent模块下pom.xml文件第306行开始，能够看见tcnative.classifier。其中，classifier表示是一个二级分类（区分不同的jdk，版本号等）

在这里是想要通过os.detected.classifier获取一个classifier，以达到获取系统版本号的的目的。

**解决方法：**

1. 将JDK、IDEA、操作系统这三者都更改为64位，这样能够保证下载的是64为jar包
2. 直接指定jar包的版本为32位。

### 2.java:程序包io.netty.util.collection不存在

![image-20230614161752830](img/Netty常见问题/image-20230614161752830.png)

在netty-common模块内，会通过模板和脚本（templates和script包）生成java文件。因此也能在pom.xml文件中找到驱动脚本执行的groovy包，设置是在编译的时候执行codegen.groovy文件。

```xml
<executions>
  <execution>
    <id>generate-collections</id>
    <phase>generate-sources</phase>
    <goals>
      <goal>execute</goal>
    </goals>
    <configuration>
      <source>${project.basedir}/src/main/script/codegen.groovy</source>
    </configuration>
  </execution>
</executions>
```

**解决方案**：但是有时候不会自动编译，可以自己直接执行编译命令，也可也在idea中点击maven compile执行。代码将会生成到target里面。

![image-20230614162645764](img/Netty常见问题/image-20230614162645764.png)

![image-20230617222109130](img/Netty常见问题/image-20230617222109130.png)
