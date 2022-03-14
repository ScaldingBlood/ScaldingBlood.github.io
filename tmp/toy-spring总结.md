---
title: toy-spring总结
date: 2019-10-27 17:48:35
tags:
---

### Intro
面试需要，翻了下Spring原理，主要设计ioc和aop部分的知识。其中有个简化的Spring源码实现叫[toy-spring](http://www.tianxiaobo.com/2018/01/18/%E8%87%AA%E5%B7%B1%E5%8A%A8%E6%89%8B%E5%AE%9E%E7%8E%B0%E7%9A%84-Spring-IOC-%E5%92%8C-AOP-%E4%B8%8A%E7%AF%87/)。读了下，对两者的实现原理有了些认识，简单总结下。

### IOC
Bean之间的依赖关系不再由Bean自己控制，而是交给Spring来管理，在需要使用相应的bean时由Spring提供。具体控制和管理这些Bean的是`BeanFactory`的实现，使用其暴露的`getBean`方法进行调用。在`BeanFactory`的实现中存在一个保存了bean与id的映射的map，从而实现bean的查找与注入。

##### 加载过程
管理的bean是懒加载的，通过著名的`BeanDefinition`来实现。具体来说，toy-spring有一个`XmlBeanFactory`的实现类，也就是说，需要加载的类可以在xml文件中定义。`XmlBeanFactory`会使用`XmlBeanReader`解析定义的bean，并把解析出来的内容作为`BeanDefinition`的实例。`XmlBeanFactory`会获取`XmlBeanReader`解析得到的`BeanDefinition`与id的map。之后在调用`getBean`方法时，会先在`BeanDefinition`中查找是否存在对应bean的饮用，若不存在则未被加载过。通过`BeanDefinition`中定义的类型信息创建新实例，并通过反射调用set方法，把`BeanDefinition`中记录的属性信息设置到实例中。如果存在引用类型的属性，则递归进行解析再设置属性。`BeanDefinition`中的引用属性信息通过单独的类型进行记录，同于和值属性的类型进行区分。最后把实例保存在`BeanDefinition`中并返回。

##### Aware的注入
通过是否实现`BeanFactoryAware`借口来判断是否需要注入BeanFactory实现从而使得被注入的bean可以借助其`getBean`方法来加载bean。

### AOP
切面的设计更加巧妙和复杂一些。其利用了动态代理的技术，从而可以修改原有类方法的调用实现。即存在两个类，一个是原有类，另一个是切入的类。通过动态代理Proxy生成第三个类。在这里主要是在原有方法被调用时，在执行方法之前会先执行切面中定义的方法。项目中使用的是JDK提供的动态代理技术。切入的类需要实现`InvocationHandler`接口。代理类通过`Proxy.newProxyInstance`生成。

##### 切面
切面(Aspect)由切点(Pointcut)和通知(Advice)组成。这里的通知借助AspectJ的`MethodInterceptor`来实现，通知中通过`MethodInvocation`来调用原有对象方法。

##### 在Spring中使用AOP
在Spring中使用AOP也即能够在加载类的时候根据条件织入相应的切面，另外实现切面、`MethodInterceptor`、通知等bean的管理。
首先是相关bean的管理：和其他的bean加载类似，一样由`BeanFactory`管理，并由bean的类型来将相应类型的切面bean都加载出来。具体来说，会把我们实现的通知实例化并注入切面。然后根据传入的切点表达式，生成切点对象。包括类型过滤和方法匹配去实现织入的判定。

织入切面则是借助了`BeanPostProcessor`接口来实现。在`BeanFactory`在实例化bean之后，在返回之前，将遍历所有实现了`BeanPostProcessor`接口的bean，并调用其`postProcessAfterInitialization`方法。在这里我们实现动态代理的生成并返回。具体来说，项目中`AspectJAwareAdvisorAutoProxyCreator`实现了`BeanPostProcessor`接口，并且实现了`BeanFactoryAware`接口使用`BeanFactory`加载所有的切面对象。遍历所有的切面，寻找是否存在类匹配的对象。若存在，创建`AdvisedSupport`对象传入切面的方法过滤器、通知、和具体bean的实例、类型、接口信息。代理工厂会把`AdvisedSupport`传给`JdkDynamicAopProxy`获取实际的代理类并返回。从而完成方法的织入。

### Outro
总体来说toyspring的技术还是很简单的，但是类的依赖继承关系有点杂乱，我稍微整理了下放在我的[仓库](https://github.com/scaldingblood)内。关注点分离和接耦的思想在大型项目中的地位还是很高的，便于设计出清晰易于扩展的系统。