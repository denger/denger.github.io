---
title: MyBatis+Spring 基于接口编程的原理分析
layout: post
category: sourcecode
---

##### 整合Spring3及MyBatis3 

对于整合Spring及Mybatis不作详细介绍，可以参考： [MyBatis 3 User Guide Simplified Chinese.pdf](http://www.google.com.hk/url?sa=t&rct=j&q=MyBatis+3+User+Guide+Simplified+Chinese.pdf&source=web&cd=2&ved=0CDkQFjAB&url=http%3A%2F%2Fmybatis.googlecode.com%2Ffiles%2FMyBatis-3-User-Guide-Simplified-Chinese.pdf&ei=_mlZUdmSPMWpiAf4uYHIDw&usg=AFQjCNHAL9KPhzfilVnZloJ-yZZ2tXWeYQ&bvm=bv.44442042,d.dGI)，贴出我的主要代码如下： 


{% highlight java %}
package org.denger.mapper;  
  
import org.apache.ibatis.annotations.Param;  
import org.apache.ibatis.annotations.Select;  
import org.denger.po.User;  
  
public interface UserMapper {  
  
    @Select("select * from tab_uc_account where id=#{userId}")  
    User getUser(@Param("userId") Long userId);  
} 
{% endhighlight java %}

##### application-context.xml

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"  
    xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.0.xsd">  
  
  
    <!-- Provided by annotation-based configuration  -->  
    <context:annotation-config/>  
  
    <!--JDBC Transaction  Manage -->  
    <bean id="dataSourceProxy" class="org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy">  
        <constructor-arg>  
            <ref bean="dataSource" />  
        </constructor-arg>  
    </bean>  
  
    <!-- The JDBC c3p0 dataSource bean-->  
    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close">  
        <property name="driverClass" value="com.mysql.jdbc.Driver" />  
        <property name="jdbcUrl" value="jdbc:mysql://127.0.0.1:3306/noah" />  
        <property name="user" value="root" />  
        <property name="password" value="123456" />  
    </bean>  
  
    <!--MyBatis integration with Spring as define sqlSessionFactory  -->  
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">  
        <property name="dataSource" ref="dataSource" />  
    </bean>  
  
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">  
        <property name="sqlSessionFactory"  ref="sqlSessionFactory"/>  
        <property name="basePackage" value="org.denger.mapper"></property>  
    </bean>  
</beans>
{% endhighlight xml %}

##### Test Case

{% highlight java %}
package org.denger.mapper;  
  
import org.junit.Assert;  
import org.junit.Test;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.test.context.ContextConfiguration;  
import org.springframework.test.context.junit4.AbstractJUnit4SpringContextTests;  
  
@ContextConfiguration(locations = { "/application-context.xml"})  
public class UserMapperTest extends AbstractJUnit4SpringContextTests{  
  
    @Autowired  
    public UserMapper userMapper;  
      
    @Test  
    public void testGetUser(){  
        Assert.assertNotNull(userMapper.getUser(300L));  
    }  
} 
{% endhighlight java %}

##### 实现原理分析
对于以上极其简单代码看上去并无特殊之处，主要亮点在于 UserMapper 居然不用实现类，而且在调用 getUser 的时候，也是使用直接调用了UserMapper实现类，那么Mybatis是如何去实现 UserMapper的接口的呢？ 可能你马上能想到的实现机制就是通过[动态代理](http://hi.baidu.com/erics_lele/blog/item/4717dbea081421d0d439c91b.html)方式，好吧，看看MyBatis整个的代理过程吧。

首先在Spring的配置文件中看到下面的Bean： 
{% highlight xml %}
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">  
    <property name="sqlSessionFactory"  ref="sqlSessionFactory"/>  
    <property name="basePackage" value="org.denger.mapper"></property>  
</bean> 
{% endhighlight xml %}
以上的MapperScannerConfigurer class的注释中描述道：
> 从base 包中搜索所有下面所有 interface，并将其注册到 Spring Bean容器中，其注册的class bean是MapperFactoryBean。

好吧，看看它的注册过程，下面方法来从　MapperScannerConfigurer中的Scanner类中抽取，下面方法在初始化以上application-content.xml文件时就会进行调用。　主要用于是搜索 base packages 下的所有mapper class,并将其注册至 spring 的 benfinitionHolder中。

{% highlight java%}
/** 
* Calls the parent search that will search and register all the candidates. Then the 
* registered objects are post processed to set them as MapperFactoryBeans 
*/  
@Override  
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    //#1  
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);  
    if (beanDefinitions.isEmpty()) {  
            logger.warn("No MyBatis mapper was found in '" + MapperScannerConfigurer.this.basePackage  
                        + "' package. Please check your configuration.");  
    } else {  
         //#2  
         for (BeanDefinitionHolder holder : beanDefinitions) {  
                GenericBeanDefinition definition = (GenericBeanDefinition) holder.getBeanDefinition();  
            if (logger.isDebugEnabled()) {  
                logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '"+ definition.getBeanClassName() + "' mapperInterface");  
            }  
           //#3  
          definition.getPropertyValues().add("mapperInterface", definition.getBeanClassName());  
          definition.setBeanClass(MapperFactoryBean.class);  
     }  
    return beanDefinitions;  
}  
{% endhighlight java%}
\#1: 了解Spring初始化Bean过程的人可能都知道，Spring 首先会将需要初始化的所有class先通过BeanDefinitionRegistry进行注册，并且将该Bean的一些属性信息(如scope、className、beanName等)保存至BeanDefinitionHolder中；Mybatis这里首先会调用Spring中的ClassPathBeanDefinitionScanner.doScan方法，将所有Mapper接口的class注册至BeanDefinitionHolder实例中，然后返回一个Set<BeanDefinitionHolder>，其它包含了所有搜索到的Mapper class BeanDefinitionHolder对象。

\#2: 首先，由于 \#1 中注册的都是接口class， 可以肯定的是接口是不能直接初始化的；实际 \#2 中for循环中替换当前所有 holder的 className为 MapperFactoryBean.class，并且将 mapper interface的class name setter 至 MapperFactoryBean 属性为 mapperInterface 中，也就是 \#3 代码所看到的。

再看看 MapperFactoryBean，它是直接实现了 Spring 的　FactoryBean及InitializingBean　接口。其实既使不看这两个接口，当看MapperFactoryBean的classname就知道它是一个专门用于创建 Mapper 实例Bean的工厂。

至此，已经完成了Spring与mybatis的整合的初始化配置的过程。

接着，当我在以上的 Test类中，需要注入 UserMapper接口实例时，由于mybatis给所有的Mapper 实例注册都是一个MapperFactory的工厂，所以产生UserMapper实现仍需要 MapperFactoryBean来进行创建。接下来看看 MapperFactoryBean的处理过程。

先需要创建Mapper实例时，首先在 MapperFactoryBean中执行的方法是：
{% highlight java%}
/** 
 * {@inheritDoc} 
 */  
public void afterPropertiesSet() throws Exception {  
    Assert.notNull(this.sqlSession, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");  
    Assert.notNull(this.mapperInterface, "Property 'mapperInterface' is required");  

    Configuration configuration = this.sqlSession.getConfiguration();  
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {  
        configuration.addMapper(this.mapperInterface);  
    }  
} 
{% endhighlight java%}
上面方法中先会检测当前需要创建的 mapperInterface在全局的 Configuration中是否存在，如果不存在则添加。呆会再说他为什么要将 mapperInterface 添加至 Configuration中。该方法调用完成之后，就开始调用 getObject方法来产生mapper实例，看到MapperFactoryBean.getObject的该方法代码如下： 

{% highlight java%}
/** 
 * {@inheritDoc} 
 */  
public T getObject() throws Exception {  
    return this.sqlSession.getMapper(this.mapperInterface);  
}
{% endhighlight java%}
通过Debug可以看到 sqlSession及mapperInterface对象：

![alt "DEBUG IMAGE"](/images/2011/05/27/1.png)

到目前为止我们的 UserMapper 实例实际上还并未产生；再进入 org.mybatis.spring.SqlSessionTemplate 中的 getMapper 方法，该方法将 this 及 mapper interface class 作为参数传入 org.apache.ibatis.session.Configuration的getMapper() 方法中，代码如下：
{% highlight java%}
/** 
 * {@inheritDoc} 
 */  
public <T> T getMapper(Class<T> type) {  
    return getConfiguration().getMapper(type, this);  
}
{% endhighlight java%}
于是，再进入 org.apache.ibatis.session.Configuration.getMapper 中代码如下： 
{% highlight java%}
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {  
    return mapperRegistry.getMapper(type, sqlSession);  
} 
{% endhighlight java%}
其中 mapperRegistry保存着当前所有的 mapperInterface class.　那么它在什么时候将 mapperinterface class 保存进入的呢？其实就是在上面的 afterPropertiesSet 中通过 configuration.addMapper(this.mapperInterface) 添加进入的。 

再进入 org.apache.ibatis.binding.MapperRegistry.getMapper 方法，代码如下：
{% highlight java%}
/** 
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {  
  //首先判断当前knownMappers是否存在mapper interface class. 
  //因为前面说到 afterPropertiesSet 中已经将当前的 mapperinterfaceclass 添加进入了。  
  if (!knownMappers.contains(type))  
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");  
  try {  
    return MapperProxy.newMapperProxy(type, sqlSession);  
  } catch (Exception e) {  
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);  
  }  
}
{% endhighlight java%}
嗯，没错，看到　MapperProxy.newMapperProxy后可以肯定的是它确实采用的代理模式，再进入一看究竟吧：
{% highlight java%}
public static <T> T newMapperProxy(Class<T> mapperInterface, SqlSession sqlSession) {  
    ClassLoader classLoader = mapperInterface.getClassLoader();  
    Class[] interfaces = new Class[]{mapperInterface};  
    MapperProxy proxy = new MapperProxy(sqlSession);  
    return (T) Proxy.newProxyInstance(classLoader, interfaces, proxy);  
  }
{% endhighlight java%}
JDK的动态代理就不说了，至此，基本Mybatis已经完成了Mapper实例的整个创建过程，也就是你在具体使用 UserMapper.getUser 时，它实际上调用的是 MapperProxy，因为此时 所返回的 MapperProxy是实现了 UserMapper接口的。只不过 MapperProxy拦截了所有对userMapper中方法的调用，如下： 

{% highlight java%}
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
   try {  
    //如果调用不是 object 中默认的方法(如equals之类的)  
     if (!OBJECT_METHODS.contains(method.getName())) {  
       //因为当前MapperProxy代理了所有 Mapper，所以他需要根据当前拦截到的方法及代理对象获取 MapperInterface  class，也就是我这里的 UserMapper.class  
       final Class declaringInterface = findDeclaringInterface(proxy, method);  
       //然后再根据UserMapper.class、及当前调用的Method(也就是getUser)及SqlSession构造一个 MapperMethod，在这里面会获取到 getUser方法上的 @Select() 的SQL，然后再通过 sqlSession来进行执行  
       final MapperMethod mapperMethod = new MapperMethod(declaringInterface, method, sqlSession);  
       //execute执行数据库操作，并返回结果集  
       final Object result = mapperMethod.execute(args);  
       if (result == null && method.getReturnType().isPrimitive()) {  
         throw new BindingException("Mapper method '" + method.getName() + "' (" + method.getDeclaringClass() + ") attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");  
       }  
       return result;  
     }  
   } catch (SQLException e) {  
     e.printStackTrace();  
   }  
   return null;  
 } 
{% endhighlight java%}

为什么说它拦截了呢？可以看到, 它并没有调用 method.invoke(object)方法，因为实际上 MapperProxy只是动态的 implement 了UserMapper接口，但它没有真正实现 UserMapper中的任何方法。至于结果的返回，它也是通过 MapperMethod.execute 中进行数据库操作来返回结果的。说白了，就是一个实现了 MapperInterface 的 MapperProxy 实例被MapperProxy代理了，可以debug看看　userMapper实例就是 MapperProxy。

--
