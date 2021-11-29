---
layout:     post
title:      "JDK-01-Collectors.toMap问题"
date:       2021-09-22 12:00:00
author:     "LSJ"
header-img: "img/post-bg-2015.jpg"
tags:
    - JDK问题
---

2018年11月7日 17:59:27 该bug貌似在java9中修复,欢迎补充
2019年3月19日 17:59:11 查看java11的toMap方法后,发现并没有修改任何实现

```java
Caused by: java.lang.NullPointerException
	java.util.HashMap.merge(HashMap.java:1224)
	java.util.stream.Collectors.lambda$toMap$58(Collectors.java:1320)
	java.util.stream.ReduceOps$3ReducingSink.accept(ReduceOps.java:169)
	java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1374)
	java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:481)
	java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:471)
	java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708)

```

**<font color='red'>原因是使用Map.merge方法合并时,merge不允许value为null导致的</font>**



方法源码:

```java
default V merge(K key, V value,
            BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        //在这里判断了value不可为null
        Objects.requireNonNull(value);
        V oldValue = get(key);
        V newValue = (oldValue == null) ? value :
                   remappingFunction.apply(oldValue, value);
        if(newValue == null) {
            remove(key);
        } else {
            put(key, newValue);
        }
        return newValue;
    }

```





解决方案:

```java
Map<String,Integer> videoGiftSumVoMap=videoGiftSum.stream().collect(Collectors.toMap
(callRecodVo -> Optional.ofNullable(callRecodVo).map
(CallRecodVo::getStatdate).orElse(0),callRecodVo -> Optional.ofNullable(callRecodVo).map
(CallRecodVo::getVideogiftdiamond).orElse(0), (key1, key2) -> key2));

```

使用optional判断空指针设置为null的默认值



**分析:**
因没有找到Map.merge方法为什么要检查Value Null的相关资料和官方回答,所以做以下推断:
Collectors.toMap可以使用ConcurrentHashMap为最终收集结构,而ConcurrentHashMap不允许Value为Null避免产生二义性(ConcurrentHashMap的key value不能为null，map可以？)和CAS的ABA问题,所以Map.merge为了兼容ConcurrentHashMap还有ConcurrentSkipListMap等多线程环境下使用的数据结构和使用CAS的实现不允许Value为Null
其他知识:key不能为null，是因为无法分辨是key没找到的原因所以为null，还是key值本身就为null。–key的二义性



原文地址：https://blog.csdn.net/wysnxzm/article/details/81260073
