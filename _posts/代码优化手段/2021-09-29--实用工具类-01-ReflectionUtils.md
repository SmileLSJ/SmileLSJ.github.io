---
layout:     post
title:      "实用工具类-01-ReflectionUtil"
date:       2021-09-29 12:00:00
author:     "LSJ"
header-img: "img/post-bg-2015.jpg"
tags:
    - 实用工具类
---

### 问题点

* 在项目中有时候我们会使用到反射的功能,如果使用最原始的方法来开发反射的功能的话肯能会比较复杂，需要处理一大堆异常以及访问权限等问题。



### 解决

* spring中提供了ReflectionUtils，这个反射的工具类，如果项目使用spring框架的话，使用这个工具可以简化反射的开发工作。

* 我们的目标是根据bean的名称、需要调用的方法名、和要传递的参数来调用该bean的特定方法。



### 案例

下面直接上代码：

```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.util.ReflectionUtils;

import java.lang.reflect.Method;

/**
 * Created with IntelliJ IDEA.
 * User: 焦一平
 * Date: 2015/6/15
 * Time: 18:22
 * To change this template use File | Settings | File Templates.
 */
@Service
public class ReInvokeService {
    @Autowired
    private SpringContextsUtil springContextsUtil;

    public void reInvoke(String beanName,String methodName,String[] params){
        Method method = ReflectionUtils.findMethod(springContextsUtil.getBean(beanName).getClass(), methodName, String.class, String.class, Boolean.class,String.class);
        Object[] param1 = new Object[3];
        param1[0]=params[0];
        param1[1]=params[1];
        param1[2]=true;
        ReflectionUtils.invokeMethod(method, springContextsUtil.getBean(beanName), param1);
    }
}
```



ReflectionUtils.findMethod()方法的签名是这样的：
public static Method findMethod(Class<?> clazz, String name, Class<?>... paramTypes)
依次需要传入 class对象、方法名、参数类型



SpringContextsUtil 这个工具类实现了 ApplicationContextAware接口，可以访问ApplicationContext的相关信息，代码如下：

```
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

/**
 * Created with IntelliJ IDEA.
 * User: 焦一平
 * Date: 2015/6/15
 * Time: 18:36
 * To change this template use File | Settings | File Templates.
 */
@Component
public class SpringContextsUtil implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    public Object getBean(String beanName) {
        return applicationContext.getBean(beanName);
    }

    public <T> T getBean(String beanName, Class<T> clazs) {
        return clazs.cast(getBean(beanName));
    }
    @Override
    public void setApplicationContext(ApplicationContext applicationContext)
            throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```



 



