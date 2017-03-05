#用Gradle开发Web项目#

Java服务端的Web组件(JavaEE)提供动态扩展能力允许你在web容器或者应用服务器中运行你的程序，就像Servlet这个名字的意思，接收客户端的请求返回响应，在MVC架构中充当控制器的角色，Servlet的响应通过视图组件--JSP来渲染，下图展示了一个典型的MVC架构的Java应用。

![](/images/dag15.png)

WAR(web application archive)用来捆绑Web组件、编译生成的class文件以及其他资源文件如部署描述符、HTML、JavaScript和CSS文件，这些文件组合在一起就形成了一个Web应用，要运行Java Web应用，WAR文件需要部署在一个服务器环境---Web容器。

Gradle提供拆箱插件用来打包WAR文件以及部署Web应用到本地的Servlet容器，接下来我们就来学习怎么把Java应用编程Web应用。

**添加Web组件**

接下来我们将创建一个Servlet，ToDoServlet，用来接收HTTP请求，执行CRUD操作，并将请求传递给JSP。你需要写一个todo-list.jsp文件，这个页面知道怎么去渲染todo items，提供一些UI组件比如按钮和指向CURD操作的链接，下图是用户检索和渲染todo items的流程：

![](/images/dag16.png)

**Web 控制器**

下面这个就是web 控制器ToDoServlet，用来处理所有的URL请求：

	package com.manning.gia.todo.web;

	public class ToDoServlet extends HttpServlet {
		private ToDoRepository toDoRepository = new InMemoryToDoRepository();

		@Override
		protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    		String servletPath = request.getServletPath();
    		String view = processRequest(servletPath, request);
    		RequestDispatcher dispatcher = request.getRequestDispatcher(view);
    		dispatcher.forward(request, response);
	 
		}

		private String processRequest(String servletPath, HttpServletRequest request) {

    		if(servletPath.equals("/all")) {
        		List<ToDoItem> toDoItems = toDoRepository.findAll();
        		request.setAttribute("toDoItems", toDoItems);
        		return "/jsp/todo-list.jsp";
    		}
    		else if(servletPath.equals("/delete")) {
    	       	(...)
    		}
    		(...)
    		return "/all";
		}

	}


对于每一个接收的请求，获取Servlet路径，基于CRUD操作在processRequest方法中处理请求，然后通过请求分派器请求传递给todo-list.jsp。

**使用War和Jetty插件**

Gradle支持构建和运行Web应用，接下来我将介绍两个web应用开发的插件War和Jetty，War插件继承了Java插件用来给web应用开发添加一些约定、支持打包War文件。Jetty是一个很受欢迎的轻量级的开源Web容器，Gradle的Jetty插件继承War插件，提供部署应用程序到嵌入的容器的任务。

既然War插件继承了Java插件，所有你在应用了War插件后无需再添加Java插件，即使你添加了也没有负面影响，应用插件是一个非幂等的操作，因此对于一个指定的插件只运行一次。添加下面这句到你的build.gradle脚本中：
	apply plugin: 'war'

除了Java插件提供的约定外，你的项目现在多了Web应用的源代码目录，将打包成war文件而不是jar文件，Web应用默认的源代码目录是src/main/webapp,如果所有的源代码都在正确的位置，你的项目布局应该是类似这样的：

![](/images/dag17.png)
![](/images/dag18.png)

你用来实现Web应用的帮助类不是java标准的一部分，比如javax.servlet.HttpServlet,在运行build之前，你应该确保你声明了这些外部依赖，War插件引入了两个新的依赖配置，用于Servlet依赖的配置是providedCompile，这个用于那些编译器需要但是由运行时环境提供的依赖，你现在的运行时环境是Jetty，因此用provided标记的依赖不会打包到WAR文件里面，运行时依赖比如JSTL这些在编译器不需要，但是运行时需要，他们将成为WAR文件的一部分。

	dependencies {
	   providedCompile 'javax.servlet:servlet-api:2.5'
	   runtime 'javax.servlet:jstl:1.1.2'
	}


build Web项目和Java项目一样，运行gradle build后打包的WAR文件在目录build/libs下，输出如下：

	$ gradle build
	:compileJava
	:processResources UP-TO-DATE
	:classes
	:war
	:assemble
	:compileTestJava UP-TO-DATE
	:processTestResources UP-TO-DATE
	:testClasses UP-TO-DATE
	:test
	:check
	:build

War插件确保打包的WAR文件符合JAVA EE规范，war任务拷贝web应用源目录src/main/webapp到WAR文件的根目录，编译的class文件保存在WEB-INF/classes,运行时依赖的库放在WEB-INF/libs,下面显示了WAR文件todo-webapp-0.1.war的目录结构：

![](/images/dag19.png)

##自定义WAR插件##

假设你把所有的静态文件放在static目录，所有的web组件放在webfiles，目录结构如下：

![](/images/dag20.png)


	//Changes web application source directory

	webAppDirName = 'webfiles'

	//Adds directories css and jsp to root of WAR file archive
	war {
	from 'static'
	}

**在嵌入的Web容器中运行**

嵌入式的Servlet容器完全不知到你的应用程序知道你提供准确的classpath和源代码目录，你可以手工编程提供，Jetty插件给你完成了所有的工作，你只需要添加下面一条命令：
	apply plugin: 'jetty'

运行Web应用需要用到的任务是jettyRun,启动Jetty容器并且无需创建WAR文件，这个命令的输出应该类似这样的：

	$ gradle jettyRun
	:compileJava
	:processResources UP-TO-DATE
	:classes
	> Building > :jettyRun > Running at http://localhost:8080/todo-webapp-jetty

最后一行Jetty插件给你提供了一个URL用来监听HTTP请求，打开浏览器输入这个链接就能看到你编写的Web应用了。

Jetty插件默认监听的端口是8080，上下文路径是todo-webapp-jetty,你也可以自己配置成想要的：

	jettyRun {
	   httpPort = 9090
	   contextPath = 'todo'
	}

这样你就把监听端口改成了9090,上下文目录改成了todo。






