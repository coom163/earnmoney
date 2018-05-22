# Scala基于jmockit进行单元测试

## 一、环境

网上有很多关于jmockit的案例，但是大多是基于java的教程。本文是基于scala来编写jmockit的测试用例。首先需要在sbt的工程中添加以下两个依赖。

```
"org.jmockit" % "jmockit" % "1.18"
"de.leanovate.play-mockws" %% "play-mockws" % "2.6.2"
```



## 二、jmockit简介

Jmockit有两种mock的方式：

- Behavior-oriented(Expectations & Verifivations)
- State-oriented(MockUp<GenericType)

通俗点讲，Behavior-oriented是基于行为的mock，对mock目标代码的行为进行模仿，更像黑盒测试。State-oriented 是基于状态的mock，是站在目标测试代码内部的。可以对传入的参数进行检查、匹配，才返回某些结果，类似白盒。而State-oriented的 new MockUp基本上可以mock任何代码或逻辑。非常强大。

## 三、实践

### 1、模拟类

Jmockit基本上有三个步骤：

- 打桩。指定要打桩类和函数，模拟返回结果。这里是new Mockup
- 调用被测方法。被测逻辑执行过程中，之前打桩数据生效。
- 判断测试结果是否符合预期。

通过下面的方法来模拟类和方法

```java
new Mockup[类](){
    @Mock
    def 方法 ={
    	模拟的返回值
    }
}
```

被测方法如下

```scala
@Singleton
class UserService @Inject()(
  conf: Configuration,
  userDao: UserDao,
  wsClient: WSClient,
  logUrlCache: AsyncCacheApi)(implicit ec: ExecutionContext) {

  def addUser(userId: String): Future[Boolean] = {
    userDao.insert(userId).map { res =>
      if (res == 0) {
        Logger.error("error")
        false
      } else {
        true
      }
    }
  }

}
```

根据被测方法编写mock，新建一个UserServiceMock

```scala
import javax.inject.Singleton
import mockit.{Mock, MockUp}
import services.UserService
import scala.concurrent.Future

@Singleton
object UserServiceMock {

  def setTestMode()={                                  //打桩
    new MockUp[UserService]() {
      @Mock
      def addUser(userId: String): Future[Boolean] ={  //模拟的是addUser方法
        println("this is mock method")                 
        Future.successful(true)                        //返回成功的模拟信息
      }
    }
  }

}
```

然后新建一个UserserviceTest，测试框架是[scalatest](http://www.scalatest.org/user_guide/selecting_a_style)，采用的风格是FunSuite。

```scala
import base.BaseSuite
import com.typesafe.config.ConfigFactory
import dao.UserDao
import mock.UserServiceMock
import play.api.Configuration
import play.api.cache.AsyncCacheApi
import play.api.libs.ws.WSClient
import scala.concurrent.ExecutionContext.Implicits.global

class UserServiceTest  extends BaseSuite {

  var userService: UserService  = null

  override protected def beforeAll(): Unit = {
    initService()                       //初始化类
  }

  private def initService(): Unit = {
    val configuration: Configuration = Configuration(ConfigFactory.load())
    val userDao = fakeApp.injector.instanceOf(classOf[UserDao])
    val ws = fakeApp.injector.instanceOf(classOf[WSClient])
    val logUrlCache = fakeApp.injector.instanceOf(classOf[AsyncCacheApi])
    userService = new UserService(configuration,userDao, ws,logUrlCache)  //
  }
  
  test("Userservice first mock example") {
    UserServiceMock.setTestMode()         //初始化mock方法
    val a = userService.addUser("123")    //调用被测方法
    var msg:String = null
    a onComplete {                        //由于被测方法类型是Future[Boolean],所以是用onComplete
      case r => println(r.toString)
        msg = r.toString
    }
    Thread.sleep(10000)                   //主要是为了等待被测方法返回数据
    assert(msg == "Success(true)")        //判断返回值是否符合预期
  }
}
```

执行结果如下：

```
Testing started at 14:11 ...
[info] application - ApplicationTimer demo: Starting application at 2018-05-22T06:11:43.372Z.
this is mock method
Success(true)

Process finished with exit code 0

```



到这里，用Jmockit来进行mock类的例子就结束了，接下去在实际项目中可能还有其他的例子，等下回遇到时再继续添加。
