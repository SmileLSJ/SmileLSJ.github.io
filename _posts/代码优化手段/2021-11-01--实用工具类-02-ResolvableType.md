---
layout:     post
title:      "实用工具类-02-ResolvableType"
date:       2021-11-01 12:00:00
author:     "LSJ"
header-img: "img/post-bg-2015.jpg"
tags:
    - 实用工具类
---

原文出处：http://jinnianshilongnian.iteye.com/blog/1993608





[Spring4新特性——泛型限定式依赖注入](http://jinnianshilongnian.iteye.com/blog/1989330)

[Spring4新特性——核心容器的其他改进](http://jinnianshilongnian.iteye.com/blog/1989379)

[Spring4新特性——Web开发的增强](http://jinnianshilongnian.iteye.com/blog/1989381)

[Spring4新特性——集成Bean Validation 1.1(JSR-349)到SpringMVC](http://jinnianshilongnian.iteye.com/blog/1990081) 

[Spring4新特性——Groovy Bean定义DSL](http://jinnianshilongnian.iteye.com/blog/1991830)

[Spring4新特性——更好的Java泛型操作API](http://jinnianshilongnian.iteye.com/blog/1993608) 

[Spring4新特性——JSR310日期API的支持](http://jinnianshilongnian.iteye.com/blog/1994164)

[Spring4新特性——注解、脚本、任务、MVC等其他特性改进](http://jinnianshilongnian.iteye.com/blog/1995111) 

 

随着泛型用的越来越多，获取泛型实际类型信息的需求也会出现，如果用原生API，需要很多步操作才能获取到泛型，比如：

```java
ParameterizedType parameterizedType =   
    (ParameterizedType) ABService.class.getGenericInterfaces()[0];  
Type genericType = parameterizedType.getActualTypeArguments()[1];  
```

 

Spring提供的ResolvableType API，提供了更加简单易用的泛型操作支持，如：

```java
ResolvableType resolvableType1 = ResolvableType.forClass(ABService.class);  
resolvableType1.as(Service.class).getGeneric(1).resolve()  
```

对于获取更复杂的泛型操作ResolvableType更加简单。

 

假设我们的API是：

```java
public interface Service<N, M> {  
}  
  
@org.springframework.stereotype.Service  
public class ABService implements Service<A, B> {  
}  
  
@org.springframework.stereotype.Service  
public class CDService implements Service<C, D> {  
}  
```

如上泛型类非常简单。 

 

**1、得到类型的泛型信息**

```java
ResolvableType resolvableType1 = ResolvableType.forClass(ABService.**class);** 
```

通过如上API，可以得到类型的ResolvableType，如果类型被Spring AOP进行了CGLIB代理，请使用ClassUtils.getUserClass(ABService.class)得到原始类型。

 

可以通过如下得到泛型参数的第1个位置（从0开始）的类型信息

```java
resolvableType1.getInterfaces()[0].getGeneric(1).resolve() 
```

因为我们泛型信息放在 Service<A, B> 上，所以需要resolvableType1.getInterfaces()[0]得到；

 

通过getGeneric（泛型参数索引）得到某个位置的泛型；

resolve()把实际泛型参数解析出来

 

**2、得到字段级别的泛型信息**

假设我们的字段如下：

```java
@Autowired  
 private Service<A, B> abService;  
 @Autowired  
 private Service<C, D> cdService;  
  
 private List<List<String>> list;  
  
 private Map<String, Map<String, Integer>> map;  
  
 private List<String>[] array;  
```

通过如下API可以得到字段级别的ResolvableType

```java
ResolvableType resolvableType2 =  
                ResolvableType.forField(ReflectionUtils.findField(GenricInjectTest.class, "cdService"));  
```



 

然后通过如下API得到Service<C, D>的第0个位置上的泛型实参类型，即C

```java
resolvableType2.getGeneric(0).resolve() 
```



 

比如 List<List<String>> list;是一种嵌套的泛型用例，我们可以通过如下操作获取String类型：

```java
ResolvableType resolvableType3 =  
                ResolvableType.forField(ReflectionUtils.findField(GenricInjectTest.class, "list"));  
resolvableType3.getGeneric(0).getGeneric(0).resolve();  
```



 

更简单的写法

```java
resolvableType3.getGeneric(0, 0).resolve(); //List<List<String>> 即String  
```



 

比如Map<String, Map<String, Integer>> map;我们想得到Integer，可以使用：

```java
ResolvableType resolvableType4 =  
                ResolvableType.forField(ReflectionUtils.findField(GenricInjectTest.class, "map"));  
resolvableType4.getGeneric(1).getGeneric(1).resolve();  
```

更简单的写法 

```java
resolvableType4.getGeneric(1, 1).resolve() 
```



 

**3、得到方法返回值的泛型信息**

假设我们的方法如下：

```java
private HashMap<String, List<String>> method() {  
    return null;  
}   
```

得到Map中的List中的String泛型实参：

```java
ResolvableType resolvableType5 =  
                ResolvableType.forMethodReturnType(ReflectionUtils.findMethod(GenricInjectTest.class, "method"));  
resolvableType5.getGeneric(1, 0).resolve();  
```



 

**4、得到构造器参数的泛型信息**

假设我们的构造器如下：

```java
public Const(List<List<String>> list, Map<String, Map<String, Integer>> map) {  
}  
```

我们可以通过如下方式得到第1个参数（ Map<String, Map<String, Integer>>）中的Integer：

```java
ResolvableType resolvableType6 =  
                ResolvableType.forConstructorParameter(ClassUtils.getConstructorIfAvailable(Const.class, List.class, Map.class), 1);  
resolvableType6.getGeneric(1, 0).resolve();  
```



 

**5、得到数组组件类型的泛型信息**

如对于private List<String>[] array; 可以通过如下方式获取List的泛型实参String：

```java
ResolvableType resolvableType7 =  
                ResolvableType.forField(ReflectionUtils.findField(GenricInjectTest.class, "array"));  
resolvableType7.isArray();//判断是否是数组  
resolvableType7.getComponentType().getGeneric(0).resolve();  
```



 

**6、自定义泛型类型**

```java
ResolvableType resolvableType8 = ResolvableType.forClassWithGenerics(List.class, String.class);  
        ResolvableType resolvableType9 = ResolvableType.forArrayComponent(resolvableType8);  
resolvableType9.getComponentType().getGeneric(0).resolve();  
```

ResolvableType.forClassWithGenerics(List.class, String.class)相当于创建一个List<String>类型；

ResolvableType.forArrayComponent(resolvableType8);：相当于创建一个List<String>[]数组；

resolvableType9.getComponentType().getGeneric(0).resolve()：得到相应的泛型信息；

 



**7、泛型等价比较：**

```java
resolvableType7.isAssignableFrom(resolvableType9) 
```



如下创建一个List<Integer>[]数组，与之前的List<String>[]数组比较，将返回false。

```java
ResolvableType resolvableType10 = ResolvableType.forClassWithGenerics(List.class, Integer.class);  
ResolvableType resolvableType11= ResolvableType.forArrayComponent(resolvableType10);  
resolvableType11.getComponentType().getGeneric(0).resolve();  
resolvableType7.isAssignableFrom(resolvableType11);  
```

 

从如上操作可以看出其泛型操作功能十分完善，尤其在嵌套的泛型信息获取上相当简洁。目前整个Spring4环境都使用这个API来操作泛型信息。

 

如之前说的泛型注入：[Spring4新特性——泛型限定式依赖注入](http://jinnianshilongnian.iteye.com/blog/1989330)，通过在依赖注入时使用如下类实现：

GenericTypeAwareAutowireCandidateResolver

QualifierAnnotationAutowireCandidateResolver

ContextAnnotationAutowireCandidateResolver

 

还有如Spring的核心BeanWrapperImpl，以及整个Spring/SpringWevMVC的泛型操作都是替换为这个API了：GenericCollectionTypeResolver和GenericTypeResolver都直接委托给ResolvableType这个API。

 测试用例请参考[GenricInjectTest.java](https://github.com/zhangkaitao/spring4-showcase/blob/master/spring4-others/src/test/java/com/sishuok/spring4/generic/GenricInjectTest.java)。

