title: Spring Boot 添加JSP支持
date: 2015-08-01
categories: Spring Boot

---
[转载](#)
#### 创建Maven web project

使用Eclipse新建一个Maven Web Project ，项目取名为：spring-boot-jsp

#### 在pom.xml文件添加依赖

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.3.3.RELEASE</version>
	</parent>

依赖包：

	<!-- web支持: 1、webmvc; 2、restful; 3、jackjson支持; 4、aop ........ -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<!-- servlet依赖. -->
	<dependency>
		<groupId>javax.servlet</groupId>
		<artifactId>javax.servlet-api</artifactId>
		<scope>provided</scope>
	</dependency>
	<!--JSTL（JSP Standard TagLibrary，JSP标准标签库)是一个不断完善的开放源代码的JSP标签库，
		是由apache的jakarta小组来维护的。JSTL只能运行在支持JSP1.2和Servlet2.3规范的容器上，
		如tomcat 4.x。在JSP2.0中也是作为标准支持的。不然报异常信息：
		javax.servlet.ServletException:Circular view path [/helloJsp]: 
		would dispatch back to the current handler URL[/helloJsp] again.
		Check your ViewResolver setup!
		(Hint: This may be the resultof an unspecified view, 
		due to default view name generation.)
	 -->
	<dependency>
		<groupId>javax.servlet</groupId>
		<artifactId>jstl</artifactId>
	</dependency>
	<!-- tomcat的支持.-->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-tomcat</artifactId>
	<scope>provided</scope>
	</dependency>
		<groupId>org.apache.tomcat.embed</groupId>
		<artifactId>tomcat-embed-jasper</artifactId>
		<scope>provided</scope>
	</dependency>

Jdk编译版本：

	<build>
	      <finalName>spring-boot-jsp</finalName>
	      <plugins>
	             <plugin>
	                    <artifactId>maven-compiler-plugin</artifactId>
	                    <configuration>
	                          <source>1.7</source>
	                          <target>1.7</target>
	                    </configuration>
	             </plugin>
	      </plugins>
	</build>

#### application.properties配置

上面说了spring-boot 不推荐JSP，想使用JSP需要配置application.properties。 
添加src/main/resources/application.properties内容：

	#页面默认前缀目录
	spring.mvc.view.prefix=/WEB-INF/jsp/
	#响应页面默认后缀
	spring.mvc.view.suffix=.jsp
	#自定义属性，可以在Controller中读取
	application.hello=HelloAngel From application

#### 编写测试Controller

编写类：com.kfit.jsp.controller.HelloController：

	package com.kfit.jsp.controller;
	import java.util.Map;
	import org.springframework.beans.factory.annotation.Value;
	import org.springframework.stereotype.Controller;
	import org.springframework.web.bind.annotation.RequestMapping;
	/**
	 *测试
	 * @tearfive(QQ:415896243)
	 * @version v.0.1
	 */
	@Controller
	public class HelloController {
		//从 application.properties 中读取配置，如取不到默认值为HelloShanhy
		@Value("${application.hello:Hello Angel}")
		private String hello;
		@RequestMapping("/helloJsp")
		public StringhelloJsp(Map<String,Object> map){
			System.out.println("HelloController.helloJsp().hello="+hello);
			map.put("hello",hello);
			return"helloJsp";
		}
	}

#### 编写JSP页面

在 src/main 下面创建 webapp/WEB-INF/jsp 目录用来存放我们的jsp页面：helloJsp.jsp

	<%@page language="java"contentType="text/html;charset=UTF-8"pageEncoding="UTF-8"%>
	<!DOCTYPE html>
	<html>
		<head>
			<metahttp-equiv="Content-Type"content="text/html; charset=UTF-8">
			<title>Insert title here</title>
		</head>
		<body>
			helloJsp
			<hr/>
			${hello}
		</body>
	</html>

#### 编写启动类

编写App.java启动类：

	package com.kfit.jsp;
	import org.springframework.boot.SpringApplication;
	importorg.springframework.boot.autoconfigure.SpringBootApplication;
	importorg.springframework.boot.context.web.SpringBootServletInitializer;
	@SpringBootApplication
	public class App extends SpringBootServletInitializer {
		//@Override
		//protectedSpringApplicationBuilder configure(SpringApplicationBuilder application){
			//returnapplication.sources(App.class);
		//}
		public static voidmain(String[] args) {
			SpringApplication.run(App.class,args);
		}
	}
右键Run As  Java Application访问：[http://127.0.0.1:8080/helloJsp](#)可以访问到：

	helloJsp

---

	Hello Angel Fromapplication

特别说明：针对el表达式，类似${hello}这个对于servlet的版本是有限制的，2.4版本版本以下是不支持的，是无法进行识别的，请注意。