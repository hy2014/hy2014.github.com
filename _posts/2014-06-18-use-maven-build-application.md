---
layout: post
title: "使用Maven 插件构建Java应用"
description: ""
category: Java
tags: [Maven]
---
{% include JB/setup %}
今天主要介绍使用maven的插件构建一个基础的java application的结构。 

为什么强调Java Application？

	以前大多数开发的都是Java Web的应用程序， 大家把应用程序build成一个War包， 扔到Tomcat或者Java EE的服务器上。 
	但是最后大家发现， 几乎很少一个Java Web容器同时部署多个工程， 一般只部署一个应用。 所以，现在有很多应用的开发， 最后并不build成一个WAR包， 如果需要Web功能， 内置一个Jetty的容器， 提供Web服务就可以了。

一个基本的Java Application的结构如下所示:

	--Projct
	 - bin
	 - lib
	 - conf
	 - log
	 - webapps(optional)
	 - xxxxx-${version}.jar
	 - readme.txt

下面我将一步一步介绍构建成这个结构的使用的插件:
首先我们的配置文件是放在conf这个folder下， 所以在build jar的时候需要过滤掉配置文件.
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <configuration>
	<excludes>
		<exclude>**/*.properties</exclude>
		<exclude>**/*.xml</exclude>
		<exclude>**/*.csv</exclude>
		<exclude>**/*.txt</exclude>
	</excludes>
  </configuration>
</plugin>
```

然后我们需要使用appassembler-maven-plugin将我们依赖的jar文件放到一个目录下面
```xml
<plugin>
	<groupId>org.codehaus.mojo</groupId>
	<artifactId>appassembler-maven-plugin</artifactId>
	<version>1.5</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>create-repository</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<repositoryName>lib</repositoryName>
		<repositoryLayout>flat</repositoryLayout>
	</configuration>
</plugin>
```
接下来就是使用maven-assembly-plugin来build这种结构, 关于这个插件的具体用法. 
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
	<version>2.4</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>single</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<descriptors>
			<descriptor>src/main/assembly/binary-deployment.xml</descriptor>
		</descriptors>
	</configuration>
</plugin>
```
重点在binary-deployment.xml这个文件, 这里只介绍怎么build lib folder这个文件；
在这里， 我们把上一步build的jar文件挪到libfolder下面， 并且exclude了我们自己的jar文件。
```xml
<id>${the_id}</id>
<formats>
	<format>tar.gz</format>
</formats>
<fileSets>
	<fileSet>
		<directory>${project.build.directory}/appassembler/lib</directory>
		<outputDirectory>lib</outputDirectory>
		<excludes>
			<exclude>${project.build.finalName}.jar</exclude>
		</excludes>
		<fileMode>0644</fileMode>
		<directoryMode>0755</directoryMode>
	</fileSet>
</fileSets>
```
这样一个基本的Java Application的结构build好了。

