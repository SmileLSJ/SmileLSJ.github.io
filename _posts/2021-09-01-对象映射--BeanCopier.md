---
layout:     post
title:      "BeanCopier"
subtitle:   "BeanCopier"
date:       2021-09-01 12:00:00
author:     "LSJ"
header-img: "img/post-bg-2015.jpg"
tags:
    - 对象复制
    - Java工具类
---



本文将简要介绍CGLIB代码包结构以及核心类的基本功能，然后通过介绍BeanCopier的使用例子，将其作为引子对相关源码实现进行分析。

## CGLIB代码包结构

### 1.core

- ClassGenerator（接口） & 

  AbstractClassGenerator

  （实现类）

  - 作为cglib代码中 **最核心的调度者** ，封装了类创建的主要流程，并支持一些缓存操作、命名策略（NamingPolicy）、代码生成策略（GeneratorStrategy）等。
  - 其中，protected Object create(Object key) 作为模版方法，定义了类的生成过程。同时将变化点进行封装，供继承类自主实现。

- GeneratorStrategy（接口） & DefaultGeneratorStrategy（默认实现类）

  - 控制ClassGenerator生成class的字节码。
  - 为继承类预留了两个抽象方法，可以对生成的字节码进行操作。

- NamingPolicy（接口） & DefaultNamingPolicy（默认实现类）

  - 用于控制生成类的命名规则。
  - 一般的命名规则：
    - 被代理类名 + $$ + CGLIB核心处理类 + "ByCGLIB" + $$ + key的hashCode。
    - 示例：FastSource<span>$$</span>FastClassByCGLIB<span>$$</span>e1a36bab.class。

- KeyFactory

  - 每个生成类都会在cglib的缓存中存在唯一的key与之对应，这个key就通过KeyFactory进行生成。

- DebuggingClassWriter

  - 被DefaultGeneratorStrategy调用,将生成类转为字节码输出。
  - 将生成的字节码写入到文件中(debugLocation)。

- ClassEmitter & CodeEmitter

  - 封装了ASM的实现，提供对类和方法的字节码操作。

- 工具类

  - EmitUtils：封装了一些字节码操作的基本函数。
  - ReflectUtils ：封装JDK中的反射操作。

### 2.beans

- BeanCopier：用于两个bean之间，同名属性间的拷贝。
- BulkBean：用于两个bean之间，自定义get&set方法间的拷贝。
- BeanMap：针对POJO Bean与Map对象间的拷贝。
- BeanGenerator：根据Map<String,Class>properties的属性定义，动态生成POJO Bean类。
- ImmutableBean:同样用于两个bean间的属性拷贝，但生成的bean不允许调用set方法，也就是说，生成的对象是不可变的。

### 3.reflect

- FastClass & FastMethod
  - FastClass机制就是对一个类的方法建立索引，通过索引来直接调用相应的方法.

### 4.proxy

- Enhancer: 用于生成动态代理类

此处略过了部分与本文无关的类，重点关注core包中的类即可~

 更多细节可以参考[这里](https://link.jianshu.com?t=http://www.iteye.com/topic/799827)。

## BeanCopier实现机制

### 1.BeanCopier的使用

顾名思义，BeanCopier是用于在两个bean之间进行属性拷贝的。BeanCopier支持两种方式，一种是不使用Converter的方式，仅对两个bean间属性名和类型完全相同的变量进行拷贝。另一种则引入Converter，可以对某些特定属性值进行特殊操作。

 代码如下所示。

不使用Converter的例子



```java
    public void testSimple() {
        // 动态生成用于复制的类,false为不使用Converter类
        BeanCopier copier = BeanCopier.create(MA.class, MA.class, false);

        MA source = new MA();
        source.setIntP(42);
        MA target = new MA();

        // 执行source到target的属性复制
        copier.copy(source, target, null);

        assertTrue(target.getIntP() == 42);
    }
```

使用Converter的例子



```java
    public void testConvert() {
        // 动态生成用于复制的类,并使用Converter类
        BeanCopier copier = BeanCopier.create(MA.class, MA.class, true);

        MA source = new MA();
        source.setIntP(42);
        MA target = new MA();

        // 执行source到target的属性复制
        copier.copy(source, target, new Converter() {

            /**
             * @param sourceValue source对象属性值
             * @param targetClass target对象对应类
             * @param methodName targetClass里属性对应set方法名,eg.setId
             * @return
             */
            public Object convert(Object sourceValue, Class targetClass, Object methodName) {
                if (targetClass.equals(Integer.TYPE)) {
                    return new Integer(((Number)sourceValue).intValue() + 1);
                }
                return sourceValue;
            }
        });

        assertTrue(target.getIntP() == 43);
    }
```

很简单吧~

 核心代码就只有两行:

 copier = BeanCopier.create // 生成用于两个bean间进行复制的类 

 copier.copy(source, target, converter) // 执行复制

### 2.性能分析

| 场景                                 | 耗时（1000000次调用） | 原理       |
| ------------------------------------ | :-------------------: | ---------- |
| 直接使用get&set方法                  |         22ms          | 直接调用   |
| 使用BeanCopiers（不使用Converter）   |         22ms          | 修改字节码 |
| 使用BeanCopiers（使用Converter）     |         249ms         | 修改字节码 |
| 使用BeanUtils                        |        12983ms        | 反射       |
| 使用PropertyUtils（不使用Converter） |        3922ms         | 反射       |



 从上面数据可以看出，不使用Converter时，BeanCopiers与直接调用get&set方法性能相当。

### 3.一次调用流程

接下来，我们来看看在简单的调用背后，cglib替我们做了哪些事。

#### (1)CGLIB做了什么

CGLIB的核心在于通过操作字节码生成类，来实现原本需要通过反射或者一堆代码才能实现的逻辑。在我们刚刚的例子里(注意， **是不带Converter的例子** )，CGLIB在背后悄悄替我们生成了两个类，我们先来稍微窥探一下这两个生成类，然后接下来的时间我们都将用来分析，cglib是如何生成这两个类的。

- 第一个类



```java
public class MA$$BeanCopierByCGLIB$$d9c04262 extends BeanCopier {
    public MA$$BeanCopierByCGLIB$$d9c04262() {
    }

    public void copy(Object var1, Object var2, Converter var3) {
        MA var10000 = (MA)var2;
        MA var10001 = (MA)var1;
        var10000.setBooleanP(((MA)var1).isBooleanP());
        var10000.setByteP(var10001.getByteP());
        var10000.setCharP(var10001.getCharP());
        var10000.setDoubleP(var10001.getDoubleP());
        var10000.setFloatP(var10001.getFloatP());
        var10000.setId(var10001.getId());
        var10000.setIntP(var10001.getIntP());
        var10000.setLongP(var10001.getLongP());
        var10000.setName(var10001.getName());
        var10000.setShortP(var10001.getShortP());
        var10000.setStringP(var10001.getStringP());
    }
}
```

先不用太介意这个奇葩的类名，先看看这个类生成的代码在做什么事，它通过生成拷贝属性值的代码，来完成我们需要的拷贝逻辑。这个生成类也就是我们前面例子里的copier(`BeanCopier copier = BeanCopier.create(MA.class, MA.class, false);`)对应的Class。

- 第二个类



```java
public class BeanCopier$BeanCopierKey$$KeyFactoryByCGLIB$$f32401fd extends KeyFactory implements BeanCopierKey {
    // 源类名
    private final String FIELD_0;
    // 目标类名
    private final String FIELD_1;
    // 是否使用Converter
    private final boolean FIELD_2;

    public BeanCopier$BeanCopierKey$$KeyFactoryByCGLIB$$f32401fd() {
    }

    public Object newInstance(String var1, String var2, boolean var3) {
        return new BeanCopier$BeanCopierKey$$KeyFactoryByCGLIB$$f32401fd(var1, var2, var3);
    }

    public int hashCode() {
        return ((95401 * 54189869 + (this.FIELD_0 != null?this.FIELD_0.hashCode():0)) * 54189869 + (this.FIELD_1 != null?this.FIELD_1.hashCode():0)) * 54189869 + (this.FIELD_2 ^ 1);
    }

    public boolean equals(Object var1) {
          ...
    }

    public String toString() {
        StringBuffer var10000 = new StringBuffer();
        var10000 = (this.FIELD_0 != null?var10000.append(this.FIELD_0.toString()):var10000.append("null")).append(", ");
        return (this.FIELD_1 != null?var10000.append(this.FIELD_1.toString()):var10000.append("null")).append(", ").append(this.FIELD_2).toString();
    }
}
```

第二个类就不如第一个那么一目了然了，它这就是我们前面讲解代码包结构时，core包中的KeyFactory生成的key，它作为 **类一（第一个类）** 的唯一标识，在cglib的缓存Map中作为key。

 这个类包含一个默认的构造函数、一个newInstance的工厂方法用于创建新的实例，以及重写的hashCode、equals和toString方法。我们后面会对它进行详细说明。

 在下文中，我们将简称 **第一个类** 为 **类一** ， **第二个类** 为 **类二**。

#### (2)从BeanCopier#create开始

在浏览了生成的类一和类二后，我们从BeanCopier的调用代码入手。代码省去了不影响主流程的细节。



```java
// 一次调用
BeanCopier copier = BeanCopier.create(MA.class, MA.class, true);

abstract public class BeanCopier
{
    public static BeanCopier create(Class source, Class target, boolean useConverter) {
        Generator gen = new Generator();
        gen.setSource(source);
        gen.setTarget(target);
        gen.setUseConverter(useConverter);

        // 调用类创建方法
        return gen.create();
    }

    public static class Generator extends AbstractClassGenerator {

        public BeanCopier create() {
            // 1.通过KEY_FACTORY创建key实例
            Object key = KEY_FACTORY.newInstance(source.getName(), target.getName(), useConverter);
            // 2.调用AbstractClassGenerator#create创建copy类
            return (BeanCopier)super.create(key);
        }
        ...
    }
}
```

BeanCopier#create只做了一件事情，新建了一个Generator实例，并调用了Generator#create方法。

BeanCopier$Generator#create就稍微复杂一点了：

1. 通过KEY_FACTORY创建key实例
2. 调用AbstractClassGenerator#create创建copy类

这个KEY_FACTORY.newInstance是不是有点眼熟了，我们刚刚提到的 **类二** 中，就有这个newInstance方法。由此可以猜测，KEY_FACTORY应该是 **类二** 的一个实例。

#### (3)KEY_FACTORY的由来

我们留意下KEY_FACTORY在BeanCopier中是如何定义的。



```java
    private static final BeanCopierKey KEY_FACTORY =
      (BeanCopierKey)KeyFactory.create(BeanCopierKey.class);
```

查看源码，可以整理出接下来调用链路大致如下:

> KeyFactory#create
>
> > -> KeyFactory$Generator#create
> >
> > > -> AbstractClassGenerator#create

OK，看到了AbstractClassGenerator#create方法，重头戏来了。

#### (4)AbstractClassGenerator#create方法流程

这个方法封装了类创建的主要流程。为了便于阅读，去掉了部分不重要的代码。



```csharp
protected Object create(Object key) {
    Class gen = null;

    synchronized (source) {
        ClassLoader loader = getClassLoader();
        Map cache2 = null;
        cache2 = (Map) source.cache.get(loader);
        
        /** 1.尝试加载缓存 **/
        // 如果缓存不存在,则新建空的缓存
        if (cache2 == null) {
            cache2 = new HashMap();
            cache2.put(NAME_KEY, new HashSet()); // NAME_KEY对应的Set集合用于去重
            source.cache.put(loader, cache2);
        }
        // 如果缓存存在,且要求使用缓存
        else if (useCache) {
            // 通过key获取缓存中的生成类(拿到的是引用[WeakReference],调用ref.get()拿到类本身)
            Reference ref = (Reference) cache2.get(key);
            gen = (Class) ((ref == null) ? null : ref.get());
        }

        this.key = key;
        
        /** 2.如果不能从缓存中查找到生成类,则新建类 **/
        if (gen == null) {
            // strategy.generate中调用了子类里的generateClass函数
            // 并返回生成的字节码
            byte[] b = strategy.generate(this);

            String className = ClassNameReader.getClassName(new ClassReader(b));

            // 将className放入NAME_KEY对应的Set中
            getClassNameCache(loader).add(className);
            
            // 根据返回的字节码生成类
            gen = ReflectUtils.defineClass(className, b, loader);
        }

        if (useCache) {
            // 在缓存中放入新生成的类
            cache2.put(key, new WeakReference(gen));
        }
        /** 3.根据生成类，创建实例并返回 **/
        return firstInstance(gen);
    }
    /** 3.根据生成类，创建实例并返回 **/
    return firstInstance(gen);
}
```

结合代码，分析下具体流程。

1. 尝试加载缓存，大致流程代码已经交代得很清楚，这里简单介绍几个实现细节

   - 关于source.cache

     - source.cache用于缓存生成类，是一个两层嵌套Map，第一层key值为classloader，第二层key值为生成类对应的唯一索引名（在这里就是"BeanCopierKey"啦）

     - source.cache使用了 

       WeakHashMap

       - WeakHashMap的key值为弱引用（WeakReference）。如果一个WeakHashMap的key被回收，那么它对应用的value也将被自动的被移除。这也是为什么要使用classloader作为key，当classloader被回收，使用这个classloader加载的类也应该被回收，在这时将这个键值对移除是合理的。

   - 在第二层Map中，出现了唯一一个不和谐的key值：NAME_KEY。它对应的Set存储了当前缓存的所有生成类的类名，用于检测生成类的类名是否重复。

2. 如果不能从缓存中查找到生成类,则新建类

   - (1) 根据生成策略（GeneratorStrategy）生成字节码

     - 我们可以看看默认的DefaultGeneratorStrategy是如何实现的

       ![img](https:////upload-images.jianshu.io/upload_images/1455237-c89ef7559b239a7e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1048/format/webp)

       DefaultGeneratorStrategy

       - 首先，统一调用了子类(KeyFactory)的Generator#generateClass函数，完成了类的构建。也可以看做是完成了对类的定义。我们将在[(5)KeyFactory#generateClass方法流程](#section_5)具体说明。
       - 然后，调用DebuggingClassWriter#toByteArray转为字节码输出。关于DebuggingClassWriter，它通过封装ClassWriter，实现了从定义的类结构到字节码的转换工作。

   - (2) 根据字节码创建类,其原理是通过反射调用ClassLoader#defineClass方法。

   - (3) 如果要求使用缓存，则将新生成的类放入缓存中。

     - 这里有个小问题值得讨论下~。我们都知道，GC判断能否回收这个对象，是检查当前这个对象是否还被其他对象强引用。因此，代码里将新生成的类加了一层弱引用后放入缓存中，以保证缓存的引用不影响生成类的释放。咋一看这是很合理的，但是类的回收和普通对象的回收不太一样，它要求必须满足 **“加载该类的ClassLoader已经被回收”** 的条件才允许将这个类回收。 **当满足这个条件时** ，按照前面的介绍，由于我们的source.cache本就是WeakHashMap，classloader所对应的键值对已经被回收，那么生成类是否使用WeakReference已经无所谓了（反正这层引用已经失效了）。

3. 根据生成类，创建实例并返回

   - firstInstance方法由子类自定义实现，KeyFactory的实现是直接通过反射调用生成类的newInstance方法，设置入参为null。

#### (5)KeyFactory#generateClass方法流程

\#generateClass作为模板方法，由各子类实现，用于自定义子类想要生成的类结构。



```dart
        public void generateClass(ClassVisitor v) {
            ClassEmitter ce = new ClassEmitter(v);

            // 对定义key工厂类结构的接口进行判断，判断该接口是否只有newInstance一个方法，newInstance的返回值是否为Object
            Method newInstance = ReflectUtils.findNewInstance(keyInterface);
            if (!newInstance.getReturnType().equals(Object.class)) {
                throw new IllegalArgumentException("newInstance method must return Object");
            }

            // 获取newInstance的入参类型，此处使用ASM的Type来定义
            Type[] parameterTypes = TypeUtils.getTypes(newInstance.getParameterTypes());
            // 创建class
            ce.begin_class(Constants.V1_2,
                           Constants.ACC_PUBLIC,
                           getClassName(),
                           KEY_FACTORY,
                           new Type[]{ Type.getType(keyInterface) },
                           Constants.SOURCE_FILE);
            //生成默认构造函数
            EmitUtils.null_constructor(ce);
            //生成newInstance 工厂方法
            EmitUtils.factory_method(ce, ReflectUtils.getSignature(newInstance));

            //生成有参构造方法
            int seed = 0;
            CodeEmitter e = ce.begin_method(Constants.ACC_PUBLIC,
                                            TypeUtils.parseConstructor(parameterTypes),
                                            null);
            e.load_this();
            e.super_invoke_constructor();
            e.load_this();
            for (int i = 0; i < parameterTypes.length; i++) {
                seed += parameterTypes[i].hashCode();

                //为每一个入参生成一个相同类型的类字段
                ce.declare_field(Constants.ACC_PRIVATE | Constants.ACC_FINAL,
                                 getFieldName(i),
                                 parameterTypes[i],
                                 null);
                e.dup();
                e.load_arg(i);
                e.putfield(getFieldName(i));
            }
            e.return_value();
            e.end_method();

            //生成hashCode函数
            e = ce.begin_method(Constants.ACC_PUBLIC, HASH_CODE, null);
            int hc = (constant != 0) ? constant : PRIMES[(int)(Math.abs(seed) % PRIMES.length)];
            int hm = (multiplier != 0) ? multiplier : PRIMES[(int)(Math.abs(seed * 13) % PRIMES.length)];
            e.push(hc);
            for (int i = 0; i < parameterTypes.length; i++) {
                e.load_this();
                e.getfield(getFieldName(i));
                EmitUtils.hash_code(e, parameterTypes[i], hm, customizer);
            }
            e.return_value();
            e.end_method();

            //生成equals函数，在equals函数中对每个入参都进行判断
            e = ce.begin_method(Constants.ACC_PUBLIC, EQUALS, null);
            Label fail = e.make_label();
            e.load_arg(0);
            e.instance_of_this();
            e.if_jump(e.EQ, fail);
            for (int i = 0; i < parameterTypes.length; i++) {
                e.load_this();
                e.getfield(getFieldName(i));
                e.load_arg(0);
                e.checkcast_this();
                e.getfield(getFieldName(i));
                EmitUtils.not_equals(e, parameterTypes[i], fail, customizer);
            }
            e.push(1);
            e.return_value();
            e.mark(fail);
            e.push(0);
            e.return_value();
            e.end_method();

            // toString
            e = ce.begin_method(Constants.ACC_PUBLIC, TO_STRING, null);
            e.new_instance(Constants.TYPE_STRING_BUFFER);
            e.dup();
            e.invoke_constructor(Constants.TYPE_STRING_BUFFER);
            for (int i = 0; i < parameterTypes.length; i++) {
                if (i > 0) {
                    e.push(", ");
                    e.invoke_virtual(Constants.TYPE_STRING_BUFFER, APPEND_STRING);
                }
                e.load_this();
                e.getfield(getFieldName(i));
                EmitUtils.append_string(e, parameterTypes[i], EmitUtils.DEFAULT_DELIMITERS, customizer);
            }
            e.invoke_virtual(Constants.TYPE_STRING_BUFFER, TO_STRING);
            e.return_value();
            e.end_method();

            ce.end_class();
        }
```

还记得我们在一开始提到的 **类二** 么，它就是靠这个方法定义出来滴~

大致的流程已经可以通过代码梳理出来,这里就不展开了。

 值得提及的细节是流程中类名的生成问题，是通过调用AbstractClassGenerator定义的getClassName()实现：

1. 其中用于去重的nameCache，是从[(4)AbstractClassGenerator#create方法流程](#section_4)中提到的NAME_KEY中获取。

2. namingPolicy是命名策略，默认规则(DefaultNamingPolicy中定义)是： **被代理类名 + <span>"$$"</span> + CGLIB核心处理类 + "ByCGLIB" + <span>"$$"</span> + key的hashCode** 

    代码如下:



```java
    private String getClassName(final ClassLoader loader) {
        // 获取现在缓存的所有className
        final Set nameCache = getClassNameCache(loader);

        return namingPolicy.getClassName(namePrefix, source.name, key, new Predicate() {

            // 根据nameCache去重
            public boolean evaluate(Object arg) {
                return nameCache.contains(arg);
            }
        });
    }
```



 PS：之所以要为类二重写hashCode和equals方法，是因为类二的实例是作为key存在在HashMap中的。

最后，对于代码中关于ClassEmitter & MethodEmitter是如何封装ASM的实现，以及它如何配合ClassVisitor（包含ClassWriter/DebuggingClassWriter）实现转换字节码等细节，内容较为繁琐，可以单独开一篇博文来讲啦~。

至此，终于把KEY_FACTORY的创建讲完了（可以跳回[(3)KEY_FACTORY的由来](#section_3) 瞅瞅）~ 

 还记得大明湖畔的BeanCopier$Generator#create吗？我们回顾一下。



```java
        public BeanCopier create() {
            // 1.通过KEY_FACTORY创建key实例
            Object key = KEY_FACTORY.newInstance(source.getName(), target.getName(), useConverter);
            // 2.调用AbstractClassGenerator#create创建copy类
            return (BeanCopier)super.create(key);
        }
```

我们已经说过，这个create方法一共做了两件事情：

1. 通过KEY_FACTORY创建key实例；
2. 调用AbstractClassGenerator#create创建copy类。

在前面的流程中，我们已经搞定了第一件事情：成功创建了KEY_FACTORY（生成了 **类二** ，并创建了类二的第一个实例），并根据KEY_FACTORY.newInstance方法，我们为即将被创建的copier准备一个唯一的key值了。这里，key的唯一性由三个元素共同决定：源类名、目标类名、以及是否需要使用Converter（可以跳到 [(1)CGLIB做了什么](#section_1) 回顾下生成的类二）。

 那么，接下来我们将着手第二件事情： **类一** 的生成流程。

类一的生成与key的生成类似，也是通过AbstractClassGenerator#create方法完成类生成，因此我们只需要关注其中的变化点，也就是BeanCopier#generateClass。

#### (6)BeanCopier#generateClass方法流程

按照前面流程的介绍，我们知道generateClass方法就是用于定义类结构的，这里也不例外。



```dart
       public void generateClass(ClassVisitor v) {
            Type sourceType = Type.getType(source);
            Type targetType = Type.getType(target);
            ClassEmitter ce = new ClassEmitter(v);

            // 创建class
            ce.begin_class(Constants.V1_2,
                           Constants.ACC_PUBLIC,
                            // 类名,在AbstractClassGenerator#getClassName中生成
                           getClassName(),
                           BEAN_COPIER,
                           null,
                           Constants.SOURCE_FILE);

            // 生成默认构造函数
            EmitUtils.null_constructor(ce);

            // [BEGIN COPY METHOD]生成copy方法
            CodeEmitter e = ce.begin_method(Constants.ACC_PUBLIC, COPY, null);
            PropertyDescriptor[] getters = ReflectUtils.getBeanGetters(source);
            PropertyDescriptor[] setters = ReflectUtils.getBeanSetters(target);

            // names可根据属性名查找对应的getter方法
            Map names = new HashMap();
            for (int i = 0; i < getters.length; i++) {
                names.put(getters[i].getName(), getters[i]);
            }

            Local targetLocal = e.make_local();
            Local sourceLocal = e.make_local();
            if (useConverter) {
                e.load_arg(1);
                e.checkcast(targetType);
                e.store_local(targetLocal);
                e.load_arg(0);                
                e.checkcast(sourceType);
                e.store_local(sourceLocal);
            } else {
                e.load_arg(1);
                e.checkcast(targetType);
                e.load_arg(0);
                e.checkcast(sourceType);
            }
            // 为每个属性生成赋值语句
            for (int i = 0; i < setters.length; i++) {
                PropertyDescriptor setter = setters[i];
                PropertyDescriptor getter = (PropertyDescriptor)names.get(setter.getName());

                if (getter != null) {
                    MethodInfo read = ReflectUtils.getMethodInfo(getter.getReadMethod());
                    MethodInfo write = ReflectUtils.getMethodInfo(setter.getWriteMethod());
                    if (useConverter) {
                        Type setterType = write.getSignature().getArgumentTypes()[0];
                        e.load_local(targetLocal);
                        e.load_arg(2);
                        e.load_local(sourceLocal);
                        e.invoke(read);
                        e.box(read.getSignature().getReturnType());
                        EmitUtils.load_class(e, setterType);
                        e.push(write.getSignature().getName());
                        e.invoke_interface(CONVERTER, CONVERT);
                        e.unbox_or_zero(setterType);
                        e.invoke(write);
                        // 如果property类型相同
                    } else if (compatible(getter, setter)) {
                        e.dup2();
                        e.invoke(read);
                        e.invoke(write);
                    }
                }
            }
            e.return_value();
            e.end_method();
            //[END COPY METHOD]

            ce.end_class();
        }
```

通过BeanCopier$Generator#generateClass方法，我们得到了类一的完整定义，并为它定义了相应的copy方法。
 接下来就是继续[(4)AbstractClassGenerator#create方法流程](#section_4)的老路，通过反射获取类一的实例，也就是前面代码里的copier。剩下的事情，只需要调用`copier.copy(source, target, null)`，就可以完成source bean到target bean的属性拷贝了。

 一次BeanCopier#create的调用流程也大功告成~。

### 更多细节

1. 我们注意到KeyFactory和BeanCopier都有一个继承自AbstractClassGenerator的名为Generator的内部类，而AbstractClassGenerator封装了类创建的主要流程，只需要将 **“想要创建一个啥样的类”** 这段逻辑，提取出来封装为模板方法 **generateClass** ，供继承类实现。同样的，BulkCopier、BeanGenerator、Enhancer等类也是如此实现。
2. BeanCopier#create为什么要返回一个生成类的实例，而不是直接返回生成类。
   - 应该是因为如果直接返回生成类，调用方还得用反射调用构造函数获得实例，在使用上不如直接返回生成类的实例方便。
3. 为什么在BeanCopier中，需要专门为key值生成一个对象。
   - 因为有些生成类需要multi-vaules key来标识这个生成类。比如我们的copier，需要“源类名、目标类名、以及是否需要使用Converter”三个因素共同保证这个生成类的唯一性。

## Tips

1. BeanCopier流程虽然比较简单，但分析流程中已经涉及到对关键类AbstractClassGenerator、KeyFactory、GeneratorStrategy、NamingPolicy等的使用，基于此可以轻松阅读后续的BulkBean、BeanGenerator等类。Enhancer&FastClass的实现要更特殊一些，后续将作进一步介绍。
2. 关于如何查看生成的class文件，在代码里加入
    `System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "文件路径");`
3. 需要自行验证的地方，结合CGLIB提供的测试代码跑一下即可。
   - 文中代码版本为cglib-RELEASE_3_2_0





## 参考文献

1. [http://www.javacodegeeks.com/2013/12/cglib-the-missing-manual.html](https://link.jianshu.com?t=http://www.javacodegeeks.com/2013/12/cglib-the-missing-manual.html)
2. [http://www.iteye.com/topic/799827](https://link.jianshu.com?t=http://www.iteye.com/topic/799827)
3. [http://www.cnblogs.com/cruze/p/3843996.html](https://link.jianshu.com?t=http://www.cnblogs.com/cruze/p/3843996.html)
4. [http://cglib.sourceforge.net/apidocs/index.html](https://link.jianshu.com?t=http://cglib.sourceforge.net/apidocs/index.html)

