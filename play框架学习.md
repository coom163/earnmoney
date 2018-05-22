# play框架学习

## 一、快速构建一个demo

Play的架构是“ROR”风格，为经典的MVC架构框架,可以见下图

![play-package](./picture/play-package.png)
- demo是实现一个简单的公司员工列表

### 目录架构如下

#### app

```
app
  - controller
  - models
  - views
  - services
  - filter
```

几乎所有的业务代码都在app的目录下，最前面的就是controllers、models和views。Play框架也提供了Filter，因此可以在app目录下添加filter目录，专门管理filter的业务逻辑。

##### models

创建类

```
case class Employee（
	id: Long,
	name: String,
	sex: String,
	positions: String
）
```

*case class与scala中普通类的区别：1、构造器中的参数如果不被声明为var的话，它默认的话是val类型的，但一般不推荐将构造器中的参数声明为var ；2、自动创建伴生对象，同时实现子apply方法，在使用时可以不直接显示new对象；3、伴生对象同样会帮我们实现unapply方法，从而可以将case class应用于模式匹配；4、实现自己的toString、hashCode、copy、equals方法*

##### services

```
class EmployeeService{
    val jilen = Employee(
    	id = 1,
    	name = "jilen",
    	sex = "男"，
    	position = "全栈工程师"
    )
    
    val yison = Employee(
    	id = 2,
    	name = "Yison",
    	sex = "女",
    	position = "程序员鼓励师"
    )
    
    def getEmployees: Seq[Employee] = Seq(jilen,yison)
}
```

##### views

创建一个名为employeeList.scala.html文件，主要实现数据的渲染

```
@(employees: Seq[Employee])

<table class="employee-list">
  <tr>
    <th>编号</th>
    <th>姓名</th>
    <th>性别</th>
    <th>职位</th>
  </tr>
  @for(e <- employees){
    <tr>
      <td>@e.id</td>
      <td>@e.name</td>
      <td>@e.sex</td>
      <td>@e.position</td>
    </tr>
  }
</table>
```

##### controllers

```
class EmployeeController @Inject() (
	cc:ControllerComponents
)extends AbstractController(cc){
    val employeeService = new EmployeeService
    
    def employeeList = Action{implicit request =>
    	val employees = employeeService.getEmployees()
    	Ok(views.html.emplyeeList(employees))
    }
}
```

```
def routes: PartialFunction[RequestHeader, Handler] = {	
	case controllers_EmployController_employeeList7_route(params) =>
      call { 
        controllers_EmployController_employeeList7_invoker.call(EmployController_4.employeeList)
      }
}
```

*可见其实 routes 在 play! 中的实现是一个方法，它是一个「偏函数」当某个请求被匹配到了就调用相应的方法，如果没有匹配到则报错，所以我们也可以自己实现某个路由，而不用 play! 的这种方式，当然用 play! 约定好会更加清晰和简单。*



#### conf

```
conf
  - application.conf
  - routes
```

在conf下面，主要放置整个项目的配置文件和路由文件

##### application.conf

该文件是配置Play应用的一系列的信息，例如secret key，数据库信息等。

##### routes

应用程序的路由都是在route中实现的，这些路由就是应用程序的入口

```
GET    /employee/employee-list    controllers.EmployeeController.employeeList
```

![play-mvc](./picture/play-mvc.png)



#### build.sbt

```
name := "HelloWorld"
organization := "com.shawdubie"

version := "1.0-SNAPSHOT"

lazy val root = (project in file(".")).enablePlugins(PlayScala)

scalaVersion := "2.12.2"

libraryDependencies += guice
libraryDependencies += "org.scalatestplus.play" %% "scalatestplus-play" % "3.1.0" % Test
```

### 效果如下：

![3](./picture/3.PNG)

## 二、依赖注入

Play框架从2.4开始推荐使用Guice来作为依赖注入。采用依赖注入最大的好处就是解耦。Play框架支持两种方式的依赖注入：运行时依赖注入和编译时依赖注入。

### 运行时依赖注入

假设我们的EmployeeService需要依赖DatabaseAccessService，那就这样去实现：

```
package services

import models._
import javax.inject._

class EmployeeService @Inject() (db:DBService){
    ...
}
```

在代码中，引入了import javax.inject._，并且可以看到多了一个@Inject()注解，我们实现运行时依赖注入就采用该注解。

*@Inject 必须放在类名后面，构造参数前面*

运行时依赖注入，是在程序运行时进行依赖注入，但是她不能再编译时进行校验，为了能让程序在编译时就能实现对依赖注入的校验，play支持编译时依赖注入。

### 编译时依赖注入

由于使用自定义的ApplicationLoader 出现问题，采用的方案是[macwire](https://github.com/adamw/macwire) 。

基于上面的controller、service、model、view

第一步添加sbt依赖：

```
libraryDependencies += "com.softwaremill.macwire" %% "macros" % "2.3.0" % "provided"
```

第二步在service的包中，添加一个特质ServicesModule.scala

```
package services

trait ServicesModule {

  import com.softwaremill.macwire._

  lazy val greetingService = wire[EmployeeService]

}
```

第三步在app下创建一个特质EmployeesModule，并且继承ServicesModule

```
import controllers.GreeterController
import play.api.i18n.Langs
import play.api.mvc.ControllerComponents
import services.ServicesModule

trait EmployeesModule extends ServicesModule {

  import com.softwaremill.macwire._

  lazy val greeterController = wire[EmployeeController]

  def langs: Langs

  def controllerComponents: ControllerComponents
}
```

第四步，新建一个MyApplicationLoader的类

```
import _root_.controllers.AssetsComponents
import com.softwaremill.macwire._
import play.api.ApplicationLoader.Context
import play.api._
import play.api.i18n._
import play.api.mvc._
import play.api.routing.Router
import router.Routes

/**
 * Application loader that wires up the application dependencies using Macwire
 */
class MyApplicationLoader extends ApplicationLoader {
  def load(context: Context): Application = new MyComponents(context).application
}

class MyComponents(context: Context) extends BuiltInComponentsFromContext(context)
  with EmployeesModule
  with AssetsComponents
  with I18nComponents 
  with play.filters.HttpFiltersComponents {

  lazy val router: Router = {
    // add the prefix string in local scope for the Routes constructor
    val prefix: String = "/"
    wire[Routes]
  }
}
```

第五步，在conf/application.conf中添加一行代码，以启用编译时依赖注入

```
play.application.loader=MyApplicationLoader
```

最后，在conf/routes添加一行,与之前是一样的，添加了一个入口

```
GET    /employee/employee-list    controllers.EmployeeController.employeeList
```

效果图：

![3](./picture/3.PNG)



由于上面的代码是参考官方的教程，网上有一篇[博文](http://shawdubie.com/notes/dependency-injection2) ，可能更加简洁，下回有时间可以去尝试一下。

## 三、Play框架与其他web框架的优势
