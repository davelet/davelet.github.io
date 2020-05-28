---
layout: post
title: Spring MVC中带有BindingResult的接口报错NoSuchMethodException的问题
categories: [dev]
tags: [java]
---

现在开发Java web项目基本都是Spring框架的天下。而对接前端也基本使用Spring MVC，包括spring cloud。前端接口进行参数校验时，一种常用的方法是在参数里面增加@Valid注解，然后在参数签名中传入BindingResult result紧邻待校验参数来获取异常校验信息。

> 虽然@Valid和@Validation似乎功能一样，但我遇见的问题时，起码@Validation没有像@Valid一样能够嵌套校验。

今天遇到一个问题刚开始让人很懵逼。我需要通过aop拦截mvc的方法（过去一般叫controller中的action），然后通过反射拿到当前的方法引用，以判断方法上面是否有A或B或C或D注解，如果有则拿到注解的引用。

通过反射获取方法很简单，大家使用反射的话都是一样的：
```java 
public void filter(JoinPoint point) {
        Object[] args = point.getArgs();
        Class<?>[] argTypes = new Class[point.getArgs().length];
        for (int i = 0; i < args.length; i++) {
            argTypes[i] = args[i].getClass();
        }
        Method method;
        try { 
            // 这里通过反射拿到方法引用
            method = point.getTarget().getClass().getMethod(point.getSignature().getName(), argTypes);
        } catch (NoSuchMethodException | SecurityException e) {
            throw new RuntimeException(e);
        }
       if (method != null) {
           A bodyIndicator = method.getAnnotation(A.class);
           B bodyIndicator = method.getAnnotation(B.class);
           C bodyIndicator = method.getAnnotation(C.class);
           D bodyIndicator = method.getAnnotation(D.class);
            // 其他逻辑
       }
}
```
流程就是先通过aop的参数值获取其类型集合，然后调用java.lang.Class.getMethod(String name, Class<?>... parameterTypes)拿到方法。然后很快测试人员告诉我后台报错：
```
java.lang.NoSuchMethodException: XXXServiceImpl.update(XXXVO, org.springframework.validation.BeanPropertyBindingResult)
        at java.lang.Class.getMethod(Class.java:1786)
        at XXXXAspect.filter(XXXXAspect.java:87)
        at sun.reflect.GeneratedMethodAccessor193.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:629)
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:611)
        at org.springframework.aop.aspectj.AspectJMethodBeforeAdvice.before(AspectJMethodBeforeAdvice.java:43)
        at org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor.invoke(MethodBeforeAdviceInterceptor.java:51)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:168)
        at org.springframework.aop.aspectj.MethodInvocationProceedingJoinPoint.proceed(MethodInvocationProceedingJoinPoint.java:97)
```
也就是说aop找不到正在执行的方法。方开始一眼望去很是纳闷：怎么会没有呢，怎么可能没有呢？！

于是在getMehod那里加了断点，一步步走进去，发现在下面的地方竟然返回了false：
<div align="center">
<img src="/images/post/jvmmethod.png" title="" />
</div>

既然是false，就看一下两个参数类型分别是啥。我的方法定义是
```java
void update(@RequestBody XXXVO vo, BindingResult result);
```
而看到报错信息中的类型并非BindingResult，而是BeanPropertyBindingResult。

到这里原因就清晰了：BindingResult是一个接口，在实际传参的时候传入的是其实现类。所以修改代码如下：
```java 
for (int i = 0; i < args.length; i++) {
    argTypes[i] = args[i].getClass();
    if (args[i] instanceof BindingResult) {
        // BindingResult的需要转换类型
        argTypes[i] = BindingResult.class;
    }
}
```

---

那么最后思考一个问题：如果一个方法接收的是Object类型参数，那么传进去的值类型是什么都有可能，怎么匹配到这个方法呢？