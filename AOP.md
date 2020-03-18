# AOP

AOP:  Aspect Oriented Programming 面向切面编程

优点:

* 降低模块之间的耦合度
* 使系统更容易扩展
* 更好的代码复用
* 非业务代码更加集中，不分散，便于统一管理
* 业务代码更加简洁纯粹，不掺杂其他代码的影响

AOP 是对面向对象编程的一个补充，在运行时，动态的将代码切入到类的指定方法、指定位位置上的编程思想就是面向切面编程。将不同方法的同一个位置抽象成一个切面对象，对该切面对象进行编程就是 AOP。



### 使用

* 创建 Maven 工程，pom.xml 添加

  ```xml
  <dependencies>
      <dependency>
          <groupId>org.springframework</groupId>
          <artifactId>spring-aop</artifactId>
          <version>5.0.11.RELEASE</version>
      </dependency>
  	 <dependency>
              <groupId>org.springframework</groupId>
              <artifactId>spring-context</artifactId>
              <version>5.0.11.RELEASE</version>
          </dependency>
      <dependency>
          <groupId>org.springframework</groupId>
          <artifactId>spring-aspects</artifactId>
          <version>5.0.11.RELEASE</version>
      </dependency>
  </dependencies>
  ```

* 创建一个计算器接口 Cal，定义4个方法

```java
package com.yang.utils;

public interface Cal {
    public int add(int num1,int num2);
    public int sub(int num1,int num2);
    public int mul(int num1,int num2);
    public int div(int num1,int num2);

}
```

* 创建接口的实现类 CalImpl

```java
package com.yang.utils.impl;

import com.yang.utils.Cal;

public class CalImpl implements Cal {
    public int add(int num1, int num2) {
        System.out.println("add方法的参数是["+num1+","+num2+"]");
        int result = num1+num2;
        System.out.println("add方法的结果是"+result);
        return result;
    }

    public int sub(int num1, int num2) {
        System.out.println("sub方法的参数是["+num1+","+num2+"]");
        int result = num1-num2;
        System.out.println("sub方法的结果是"+result);
        return result;
    }

    public int mul(int num1, int num2) {
        System.out.println("mul方法的参数是["+num1+","+num2+"]");
        int result = num1*num2;
        System.out.println("mul方法的结果是"+result);
        return result;
    }

    public int div(int num1, int num2) {
        System.out.println("div方法的参数是["+num1+","+num2+"]");
        int result = num1/num2;
        System.out.println("div方法的结果是"+result);
        return result;
    }
}
```

上述代码中，日志信息和业务逻辑的耦合性很高，不利于系统的维护，使用AOP 可以进行优化，如何来实现AOP？使用动态代理的方式来实现。

给业务代码找一个代理，打印日志信息的工作交给代理来做，这样的话业务代码就只需要关注自身的业务即可。

```java
package com.yang.utils;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.Arrays;

/***
 * 这个类的功能是创建代理类
 */
public class MyInvocationHandler implements InvocationHandler {
    //接收委托对象
    private Object object = null;
    //返回代理对象
    public Object bind(Object object){
        this.object = object;
        return Proxy.newProxyInstance(object.getClass().getClassLoader(),object.getClass().getInterfaces(),this);
    }
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(method.getName()+"方法的参数是"+Arrays.toString(args));
        Object result = method.invoke(this.object,args);
        System.out.println(method.getName()+"方法的结果是"+result);
        return result;
    }
}
```

以上是通过动态代理实现AOP的过程，比较复杂，不好理解，Spring 框架对 AOP 进行了封装，使用Spring 框架可以用面向对象的思想类实现AOP。

Spring 框架中不需要创建 InvocationHandler，只需要创建一个切面对象，将所有的非业务代码在切面对象中完成即可。Spring 框架底层会自动根据切面类以及目标类生成一个代理对象。

```java
//切面对象 LoggerAspect
package com.yang.aop;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;
import java.util.Arrays;
@Aspect
@Component
public class LoggerAspect {
    @Before("execution(public int com.yang.utils.impl.CalImpl.*(..))")
    public void before(JoinPoint joinPoint){
        //获取方法名
        String name = joinPoint.getSignature().getName();
        //获取参数
        String args = Arrays.toString(joinPoint.getArgs());
        System.out.println(name+"方法的参数是："+args);
    }
    @After("execution(public int com.yang.utils.impl.CalImpl.*(..))")
    public void after(JoinPoint joinPoint){
        //获取方法名
        String name = joinPoint.getSignature().getName();
        System.out.println(name+"方法执行完毕");
    }

    @AfterReturning(value ="execution(public int com.yang.utils.impl.CalImpl.*(..))",returning = "result")
    public void afterReturning(JoinPoint joinPoint,Object result){
        //获取方法名
        String name = joinPoint.getSignature().getName();
        System.out.println(name+"方法的结果是"+result);
    }

    @AfterThrowing(value ="execution(public int com.yang.utils.impl.CalImpl.*(..))",throwing = "exception")
    public void afterThrowing(JoinPoint joinPoint,Exception exception){
        //获取方法名
        String name = joinPoint.getSignature().getName();
        System.out.println(name+"方法抛出了异常"+exception);
    }
}
```

LoggerAspect 类定义处添加的两个注解：

* @Aspect：表示该类是切面类
* @Component：将该类的对象注入到 IoC 容器。

具体方法处添加的注解：

*  @Before、@After、@AfterReturning、@AfterThrowing：表示方法执行的具体位置和时机。

CalImpl 也需要添加 @Component ，交给 IoC 容器来管理。

```java
package com.yang.utils.impl;
import com.yang.utils.Cal;
import org.springframework.stereotype.Component;
@Component
public class CalImpl implements Cal {
    public int add(int num1, int num2) {
        int result = num1+num2;
        return result;
    }

    public int sub(int num1, int num2) {
        int result = num1-num2;
        return result;
    }

    public int mul(int num1, int num2) {
        int result = num1*num2;
        return result;
    }

    public int div(int num1, int num2) {
        int result = num1/num2;
        return result;
    }
}
```

在 spring.xml  中配置 AOP.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:aop="http://www.springframework.org/schema/aop"
        xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-4.3.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-4.3.xsd
">

    <!-- 自动扫描 -->
    <context:component-scan base-package="com.yang"></context:component-scan>
    <!-- 是Aspect 注解生效，为目标类生成代理对象 -->
    <!-- autoproxy自动代理 -->
    <aop:aspectj-autoproxy></aop:aspectj-autoproxy>
</beans>
```

context:component-scan 将 com.yang 包中的所有类进行扫描，如果该类同时添加了 @Component，则将该类扫描到 IoC 容器中，即 IoC 管理它的对象。

aop:aspectj-autoproxy 让 Spring 框架结合切面类和目标类自动生成动态代理对象。



* 切面：横切关注点被模块化的抽象对象。
* 通知：切面对象完成的工作。
* 目标：被通知的对象，即被横切的对象。
* 代理：切面、通知、目标混合之后的对象。
* 连接点：通知要插入业务代码的具体位置。
* 切点：AOP 通过切点定位到连接点。