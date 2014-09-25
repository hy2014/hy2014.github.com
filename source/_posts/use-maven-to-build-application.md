title: use maven to build application
date: 2014-06-18 17:07:05
tags: maven
description: "使用Maven 构建一个完整的application的目录结构"
category: maven
---

今天主要介绍使用maven的插件构建一个完整的java应用结构。 

为什么强调Java 应用？

	通常大多数开发的都是基于Java Web的应用程序， 大家把应用程序build成一个War包, 扔到Tomcat或者Java EE的服务器上。 
	实际情况是一个Java Web容器不会同时部署多个工程. 所以，现在有很多应用的开发， 最后并不build成一个WAR包， 如果需要Web功能， 内置一个Jetty的容器， 提供Web服务就可以了。

<!-- more -->

一个基本的Java Application的结构如下所示:

	--Projct_Name
	 - bin
	 - lib
	 - conf
	 - log
	 - webapps(optional)
	 - xxxxx-${version}.jar
	 - readme.txt

> bin目录存放启动该运用的脚本.
> lib存放该项目依赖的jar包.
> conf存放启动该项目的配置文件.
> webapps是可选，如果需要Web容器， 这里存放的是Web.xml的配置。
> xxxxx-${version}.jar是该项目的代码块.

下面我将介绍构建成这个结构的使用的插件:
####1. 首先我们的配置文件是放在conf这个folder下， 所以在build jar的时候需要过滤掉配置文件.
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

####2. 然后我们需要使用appassembler-maven-plugin将我们依赖的jar文件放到一个目录下面
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

####3. 紧接着就是使用maven-assembly-plugin来build这种结构, 关于这个插件的具体用法. 
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
####4. 重点在binary-deployment.xml这个文件, 在这里，我们把步骤2中, 所有的依赖jar包copy到lib folder下, 由于我们自己项目的jar也在里面， 需要exclude掉.
```xml
<id>${app_version}</id>
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