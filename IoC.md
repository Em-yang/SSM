# Spring

### Spring框架两大核心机制

* IoC (控制反转)  / DI（依赖注入）
* AOP (面向切面编程)

Spring 是一个企业级开发框架，是软件设计层面的框架，优势在于可以将应用程序进行分层，开发者可以自主选择组件。

MVC：Struts2、Spring MVC

ORMapping：Hibernate、MyBatis、Spring Data



### 如何使用IoC

* 创建Maven工程，在 pom.xml 添加依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.yang</groupId>
    <artifactId>springioc</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-contest</artifactId>
            <version>5.0.11.RELEASE</version>
        </dependency>
    </dependencies>

</project>
```

* 创建实体类

```java
package com.yang.entity;
import lombok.Data;
@Data
public class Student {
    private long id;
    private String name;
    private int age;
}
```

* 传统的开发方式  手动 new 一个对象

```
Student student = new Student();
student.setId(1L);
student.setName("张三");
student.setAge(22);
System.out.println(student);
```

* 通过 IoC 创建对象，在配置文件中添加需要管理的对象，XML格式的配置文件，文件名可以自定义。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
        xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-4.3.xsd">

    <bean id="student" class="com.yang.entity.Student">
        <property name="id" value="1"></property>
        <property name="name" value="张三"></property>
        <property name="age" value="22"></property>
    </bean>

</beans>
```

* 从IoC 中获取对象，

  ​	1、通过 id 获取。

```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
Student student = (Student) applicationContext.getBean("student");
System.out.println(student);
```

​			2、通过运行时类来获取Bean

```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
Student student = (Student) applicationContext.getBean(Student.class);
System.out.println(student);
```

​			这种方式存在一个问题，配置文件中一个数据类型的对象只能有一个实例，否则会抛出异常，因为没有唯一的bean。



### 配置文件

* 通过配置 bean 标签来完成对象的管理

  1、id：对象名

  2、class：对象的模板类。（所有交给 IoC 容器来管理的类必须有无参构造函数，因为 Spring 的底层是通过反射机制来创建对象，调用的是无参构造）

  3、对象的成员变量通过 property 标签完成赋值

  ​		name：成员变量名

  ​		value：成员变量值（基本数据类型，String 可以直接赋值，如果是其他引用类型，不能通过 value 赋值）

  ​		ref：将 IoC 中的另外一个bean 赋给当前的成员变量 （DI）

  ```xml
  <bean id="student" class="com.yang.entity.Student">
      <property name="id" value="1"></property>
      <property name="name" value="张三"></property>
      <property name="age" value="22"></property>
      <property name="address" ref="address"></property>
  </bean>
  <bean id="address" class="com.yang.entity.Address">
      <property name="id" value="1"></property>
       <property name="name" value="西关街"></property>
  </bean>
  ```

###  IoC 底层原理

1、读取配置文件，解析XML

2、通过反射机制实例化配置文件中所配置的所有的 bean。

```java
package com.yang.ioc;

public interface ApplicationContext {
    public Object getBean(String id);
}
```

```java
package com.yang.ioc;

import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

import javax.print.Doc;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;

public class ClassPathXmlApplicationContext implements ApplicationContext {
    private Map<String,Object> ioc = new HashMap<String, Object>();
    public ClassPathXmlApplicationContext(String path) {
        try {
            SAXReader reader = new SAXReader();
            Document document = reader.read("./src/main/resources/"+path);
            Element root = document.getRootElement();
            //System.out.println(document);
            Iterator<Element> iterator = root.elementIterator();
            while (iterator.hasNext()){
                Element element = iterator.next();
                String id = element.attributeValue("id");
                String className = element.attributeValue("class");
                //通过反射机制创建对象
                Class clazz = Class .forName(className);
                //获取无参构造函数 创建目标对象
                Constructor constructor =  clazz.getConstructor();
                Object object = constructor.newInstance();
                //System.out.println(object);
                //给目标对象赋值
                Iterator<Element> beanIter = element.elementIterator();
                while (beanIter.hasNext()){
                    Element property = beanIter.next();
                    String name = property.attributeValue("name");
                    String valueStr = property.attributeValue("value");
                    String ref = property.attributeValue("ref");
                    if (ref == null){
                        String methodName ="set"+ name.substring(0,1).toUpperCase()+name.substring(1);
                        //System.out.println(methodName);
                        Field field = clazz.getDeclaredField(name);
                        Method method = clazz.getDeclaredMethod(methodName,field.getType());
                        //根据成员变量的数据类型将Value进行转换
                        Object value = null;
                        if (field.getType().getName() == "long"){
                            value = Long.parseLong(valueStr);
                        }
                        if (field.getType().getName() =="java.lang.String"){
                            value = valueStr;
                        }
                        if (field.getType().getName() == "int"){
                            value = Integer.parseInt(valueStr);
                        }
                        method.invoke(object,value);
                    }ioc.put(id,object);
                }
            }
        } catch (DocumentException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }

    @Override
    public Object getBean(String id) {
        return ioc.get(id);
    }
}

```

```java
package com.yang.ioc;

import com.yang.entity.Student;

public class Text {
    public static void main(String[] args){
        ApplicationContext application = new ClassPathXmlApplicationContext("spring.xml");
        Student student = (Student) application.getBean("student");
        System.out.println(student);
    }
    
}
```



### 通过有参构造创建 bean

* 在实体类中创建对应的有参构造函数。

```xml
 <!-- 通过 name -->
<bean id="student3" class="com.yang.entity.Student">
    <constructor-arg name="id" value="3"></constructor-arg>
    <constructor-arg name="name" value="小米"></constructor-arg>
    <constructor-arg name="age" value="20"></constructor-arg>
    <constructor-arg name="address" ref="address"></constructor-arg>
</bean>
```

```xml
<!-- 通过 index（下标） -->
<bean id="student3" class="com.yang.entity.Student">
    <constructor-arg index="0" value="3"></constructor-arg>
    <constructor-arg index="2" value="20"></constructor-arg>
    <constructor-arg index="1" value="小米"></constructor-arg>
    <constructor-arg index="3" ref="address"></constructor-arg>
</bean>
```



### 给 bean 注入集合

```xml
<bean id="student" class="com.yang.entity.Student">
    <property name="id" value="1"></property>
    <property name="name" value="张三"></property>
    <property name="age" value="22"></property>
    <property name="addresses">
        <list>
            <ref bean="address"></ref>
            <ref bean="address2"></ref>
        </list>
    </property>
</bean>

<bean id="address" class="com.yang.entity.Address">
        <property name="id" value="1"></property>
         <property name="name" value="西关街"></property>
    </bean>
    <bean id="address2" class="com.yang.entity.Address">
        <property name="id" value="2"></property>
        <property name="name" value="朝阳路"></property>
    </bean>
```



### scope 作用域

Spring 管理的 bean 是根据 scope 来生产的，表示 bean 的作用域，共4种，默认是单例。

* singleton：单例，表示通过 IoC 容器获取的 bean 是唯一的。
* prototype：原型，表示通过 IoC 容器获取的 bean 是不同的。
* request：请求，表示在一次 HTTP 请求内有效。
* session：会话，表示在一个用户会话内有效。

request 和 session 只适用于 Web 项目，多数情况下，使用单例和原型较多。

prototype 模式当业务代码获取 IoC 容器中的 bean 时，Spring 才去调用无参构造创建对应的 bean

singleton 模式无论业务代码是否获取 IoC 容器中的 bean，Spring 在加载 spring.xml 时就会创建 bean。



### Spring 的继承

与 Java 的继承不同，Java 是类层面的继承，子类可以继承父类的内部结构信息；Spring 是对象层面的继承，子对象可以继承父对象的属性值，并且可以覆盖。

```xml
<bean id="student" class="com.yang.entity.Student">
    <property name="id" value="1"></property>
    <property name="name" value="张三"></property>
    <property name="age" value="22"></property>
    <property name="addresses">
        <list>
            <ref bean="address"></ref>
            <ref bean="address2"></ref>
        </list>
    </property>
</bean>
<bean id = "stu" class="com.yang.entity.Student" parent="student">
    <!-- 覆盖父对象的 name 属性值-->
    <property name="name" value="李四"></property>
 </bean>
<bean id="address" class="com.yang.entity.Address">
    <property name="id" value="1"></property>
     <property name="name" value="西关街"></property>
</bean>
<bean id="address2" class="com.yang.entity.Address">
    <property name="id" value="2"></property>
    <property name="name" value="朝阳路"></property>
</bean>
```

Spring 的继承关注点在于具体的对象，而不在类，即不同的两个类的实例化对象可以完成继承，前提是子对象必须包含父对象的所有属性，同时可以在此基础上添加其他的属性。



### Spring 的依赖

与继承相似，依赖也是描述 bean 和 bean 之间的一种关系，配置依赖之后，被依赖的 bean 一定先创建，再创建依赖的 bean， A 依赖于 B，先创建 B，再创建 A。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-4.3.xsd">

    <bean id="user" class="com.yang.entity.User" depends-on="student"></bean>
    <bean id="student" class="com.yang.entity.Student"></bean>
    
</beans>
```



### Spring 的 p 命名空间

p 命名空间是对 IoC/DI 的简化操作，使用 p 命名空间可以更加方便的完成 bean 的配置以及 bean 之间的依赖注入。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-4.3.xsd">

        <bean id="student" class="com.yang.entity.Student" p:id="1" p:name="张三" p:age="22" p:address-ref="address"></bean>
        <bean id="address" class="com.yang.entity.Address" p:id="1" p:name="科技路"></bean>
</beans>
```



### Spring 的工厂方法

IoC 通过工厂模式创建 bean 的方式有两种：

* 静态工厂方式
* 实例工厂方式



#### 静态工厂方式

```java
package com.yang.entity;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Car {
    private long id;
    private String name;
}
```

```java
package com.yang.test;

import com.yang.entity.Car;
import com.yang.factory.StaticCarFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Test4 {
    public static void main(String[] args){
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring-factory.xml");
        Car car = (Car) applicationContext.getBean("car");
        System.out.println(car);
    }

}
```

```xml
<!-- 配置静态工厂创建 Car -->
  <bean id="car" class="com.yang.factory.StaticCarFactory" factory-method="getCar">
        <constructor-arg value="2"></constructor-arg>
  </bean>
```



#### 实例工厂方式

```java
package com.yang.factory;

import com.yang.entity.Car;
import jdk.internal.org.objectweb.asm.tree.IincInsnNode;

import java.util.HashMap;
import java.util.Map;

public class InstanceFactory {
    private Map<Long,Car> carMap;
    public InstanceFactory(){
        carMap = new HashMap<Long, Car>();
        carMap.put(1L,new Car(1L,"宝马"));
        carMap.put(2L,new Car(2L,"宝马"));
    }
    public Car getCar(long id){
        return carMap.get(id);
    }
}

```

```xml
<!-- 配置实例工厂的bean -->
<bean id="carFactory" class="com.yang.factory.InstanceFactory"></bean>

<!-- 配置实例工厂创建 Car -->
<bean id="car2" factory-bean="carFactory" factory-method="getCar">
    <constructor-arg value="1"></constructor-arg>
</bean>
```



### IoC 自动装载 （Autowire）

IoC 负责创建对象，DI 负责完成对象的依赖注入，通过配置 property 标签的 ref 属性来完成；同时 Spring 提供了另外一种更加简便的依赖注入方式：自动装载，不需要手动配置 property，IoC 容器会自动选择 bean 完成注入。

自动装载有两种方式：

* byName：通过属性名自动装载
* byType：通过属性的数据类型自动装载

1、byName

```xml
<bean id="car" class="com.yang.entity.Car">
    <property name="id" value="1"></property>
    <property name="name" value="宝马"></property>
</bean>
<bean id="person" class="com.yang.entity.Person" autowire="byName">
    <property name="id" value="11"></property>
    <property name="name" value="颤三"></property>
</bean>
```

byName：自动装载的 bean 的 id 需得和成员变量名一致，否侧就会报错。

2、byType

```xml
<bean id="car2" class="com.yang.entity.Car">
    <property name="id" value="1"></property>
    <property name="name" value="宝马"></property>
</bean>
<bean id="person" class="com.yang.entity.Person" autowire="byType">
    <property name="id" value="11"></property>
    <property name="name" value="颤三"></property>
</bean>
```

byType 使用时需要注意，如果同时存在两个及以上的符和条件的 bean 时，自动装载就会抛出异常。