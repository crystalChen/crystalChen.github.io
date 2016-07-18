---
layout: post
categories: [Spring]
tags: [Spring]
code: true
title: Spring InitializingBean的afterPropertiesSet方法
---

org.springframework.beans.factory包下有一个接口是InitializingBean 只有一个方法：  

```
/**
  * Invoked by a BeanFactory after it has set all bean properties supplied  
  * (and satisfied BeanFactoryAware and ApplicationContextAware).  
  * <p>This method allows the bean instance to perform initialization only  
  * possible when all bean properties have been set and to throw an  
  * exception in the event of misconfiguration.  
  * @throws Exception in the event of misconfiguration (such  
  * as failure to set an essential property) or if initialization fails.   
  */
 void afterPropertiesSet() throws Exception;
 ```

这个方法将在所有的属性被初始化后调用,但是会在init前调用，但是主要的是如果是延迟加载的话，则马上执行。  
所以可以在类上加上注解：  
import org.springframework.context.annotation.Lazy;  
@Lazy(false)  
这样spring容器初始化的时候afterPropertiesSet就会被调用，只需要实现InitializingBean接口就行。    
init-method 与afterPropertiesSet 都是在初始化bean的时候执行，执行顺序是afterPropertiesSet 先执行，init-method 后执行，从BeanPostProcessor的作用，可以看出最先执行的是postProcessBeforeInitialization，然后是afterPropertiesSet，然后是init-method，然后是postProcessAfterInitialization。