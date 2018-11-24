---
title: 深入Java注解
catalog: true
date: 2018-11-08 01:00:39
subtitle: Drive Into Java Annotation
author: Rancho
header-img: header.png
tags:
    -  编程札记
---

近日暂别热爱且把玩多年的Python, 转向Java阵营. 转型期间遇到的首个confusion便是注解, 一方面是它长得很像Python装饰器, 另一方面是搬砖仿写时出镜率贼高, 但又特别陌生.

## 注解的本质

`java.lang.annotation.Annotation`有这么句话, 用于描述『注解』

> The Common interface extended by all annotation types.
所有的注解类型都继承自本接口.

虽然抽象但言简意赅, 再来看看常见的`Override` 注解的JDK源码实现

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {

}
```

它本质上其实就是继承了`Annatation`接口的接口

```java
public interface Override extends Annotation{

}
```

一个注解准确意义上讲, 只是一种特殊的注释而已, 如果没有解析它的代码, 可能连注释都不如. 而解析一个类或者方法的注解往往有两种方式, 一种是编译期间的直接扫描, 一种是运行时反射. 反射的事情后面再说, 而编译期间的扫描, 指的是编译器在将Java代码编译成字节码的过程中, 如果检测到了某个类或者方法被注释所修饰, 这时编译器就会对这些注解进行特定处理.

`@Override`就是个典型的例子, 一旦编译器发现了它, 就会检查当前方式签名是否真的重写了父类中的方法, 也就是看父类中是否有同样的方法签名.

这种情况只适用于那些编译器熟知的注解类, 编译器厂商并不会为个人定制的注解提供服务, 当然他们也不知道该如何处理你的注解, 往往只是根据该注解的作用范围, 来选择是否编译进字节码文件里, 仅此而已.

## 元注解
`元注解`就是用于修饰注解的注解, 在`@Override`的定义中, `@Target`, `@Retention`就是所谓的『元注解』. 它一般用于指定某个注解的生命周期以及作用目标等信息.

Java中有以下几种『元注解』

- @Target: 注解的作用目标
- @Retention: 注解的声明周期
- @Documented: 注解是否被包含在JavaDoc文档中
- @Inherited: 注解是否被子类所继承

### Target
`@Target`用于指明被修饰的注解可以作用的目标, 是用来修饰一个方法, 一个类? 还是仅用于修饰字段属性的

`@Target`的定义
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    /**
     * Returns an array of the kinds of elements an annotation type
     * can be applied to.
     * @return an array of the kinds of elements an annotation type
     * can be applied to
     */
    ElementType[] value();
}
```

TODO. 不明白怎么不会出现循环定义的问题

我们可以通过以下方式来给注解传值

    @Target(value={ElementType.FIELED})

其中, ElementType是个枚举类型, 它的值有

- ElementType.TYPE：允许被修饰的注解作用在类、接口和枚举上
- ElementType.FIELD：允许作用在属性字段上
- ElementType.METHOD：允许作用在方法上
- ElementType.PARAMETER：允许作用在方法参数上
- ElementType.CONSTRUCTOR：允许作用在构造器上
- ElementType.LOCAL_VARIABLE：允许作用在本地局部变量上
- ElementType.ANNOTATION_TYPE：允许作用在注解上
- ElementType.PACKAGE：允许作用在包上

### Retention
`@Retention`用于指明当前注解的生命周期

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    /**
     * Returns the retention policy.
     * @return the retention policy
     */
    RetentionPolicy value();
}
```

同样的, 这里的RetentionPolicy也是个枚举类型, 它的值有

- RetentionPolicy.SOURCE：注解仅在编译期可见，不会写入 class 文件
- RetentionPolicy.CLASS：注解在类加载阶段丢弃，会写入 class 文件
- RetentionPolicy.RUNTIME：永久保存，可以反射获取

`@Document`和`@inherited`比较简单, 就不细说. 前者在我们执行JavaDoc文档打包时会被保存进doc文档, 后者表示该注解是可以被继承的, 即该类的子类将自动继承父类的该注解.

## Java三大内置注解
除了上述四种元注解外，JDK 还为我们预定义了另外三种注解，它们是

- @Override
- @Deprecated
- @SuppressWarnings

### Override
`@Override`前面已经提到, 定义如下

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {

}
```

现在我们知道, 该注解仅对方法起作用, 而且在编译结束即被丢弃. 所以, `@Override`是一种典型的『标记式注解』, 仅被编译器感知.

### Deprecated
`@Deprecated`的基本定义如下

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE})
public @interface Deprecated {

}
```

依然是一种『标记式注解』, 永久存在, 而且可以修饰所有类型. 作用是标记当前的类或方法或字段等已经不再被推荐使用

当然编译器并不会强制你做什么修改

### SuppressWarnings
`@SuppressWarnings`主要用来压制Java警告, 其定义是

```java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {
    /**
     * The set of warnings that are to be suppressed by the compiler in the
     * annotated element.  Duplicate names are permitted.  The second and
     * successive occurrences of a name are ignored.  The presence of
     * unrecognized warning names is <i>not</i> an error: Compilers must
     * ignore any warning names they do not recognize.  They are, however,
     * free to emit a warning if an annotation contains an unrecognized
     * warning name.
     *
     * <p> The string {@code "unchecked"} is used to suppress
     * unchecked warnings. Compiler vendors should document the
     * additional warning names they support in conjunction with this
     * annotation type. They are encouraged to cooperate to ensure
     * that the same names work across multiple compilers.
     * @return the set of warnings to be suppressed
     */
    String[] value();

}
```

它有一个value属性需要在使用时主动传值, 然后编译器就会跳过对应类型的警告

(其实后两个感觉不常用, 怎么就上榜了三大内置注解呢?)

## 注解与反射
上述内容简单理清了注解的定义和使用方式, 现在我们从虚拟机的层面看看, 注解的本质到底是什么

首先, 我们自定义一个注解类型

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @Author: Rancho Cooper
 * @Date 2018/11/9
 * @Desc:
 */
@Target(value= {ElementType.FIELD, ElementType.METHOD})
@Retention(value = RetentionPolicy.RUNTIME)
public @interface Comment {
    String value();
}

```

我们让`Comment`注解只能修饰字段和方法, 并且注解一直存活, 以便在运行时反射获取

前面我们说, 虚拟机(编译器实现)规范定义了一系列和注解相关的属性表, 无论是字段, 方法或是类本身, 如果被注解修饰了, 就会被写入字节码文件里. 属性表有以下几种

- RuntimeVisibleAnnotations：运行时可见的注解
- RuntimeInVisibleAnnotations：运行时不可见的注解
- RuntimeVisibleParameterAnnotations：运行时可见的方法参数注解
- RuntimeInVisibleParameterAnnotations：运行时不可见的方法参数注解
- AnnotationDefault：注解类元素的默认值

而对于一个类或者接口来说, Class类中提供了以下方法用于反射注解

- getAnnotation：返回指定的注解
- isAnnotationPresent：判定当前元素是否被指定注解修饰
- getAnnotations：返回所有的注解
- getDeclaredAnnotation：返回本元素的指定注解
- getDeclaredAnnotations：返回本元素的所有注解，不包含父类继承而来的

方法, 字段中相关的反射注解的方式基本类似. 我们来实际操作一下. 首先, 随便写个Main, 并使用上面的Comment注解

```java
import java.lang.reflect.Method;

public class Main {

    @Comment("hello")
    public static void main(String[] args) throws Exception {
        Class cls = Main.class;
        Method method = cls.getMethod("main", String[].class);
        Comment hello = method.getAnnotation(Comment.class);
    }

}
```

然后设置虚拟机启动参数, 用于捕获JDK动态代理类

    -Dsun.misc.ProxyGenerator.saveGeneratedFiles=true


运行程序后, 项目下就会多出输出目录, 打开`com/sun/proxy`下的文件, 便可以看到虚拟机动态代理机制生成的代理类
(idea中双击打开直接能看到反编译后的代码, 不需额外折腾反编译工具)

不难发现

```java
import java.lang.reflect.InvocationHandler;

public final class $Proxy1 extends Proxy implements Comment {
    // ommit some insignificant source code

    private static Method m1;
    private static Method m2;
    private static Method m4;
    private static Method m0;
    private static Method m3;

    public $Proxy1(InvocationHandler var1) throws  {
        super(var1);
    }

    public final String value() throws  {
        try {
            return (String)super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final Class annotationType() throws  {
        try {
            return (Class)super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

        static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m4 = Class.forName("Comment").getMethod("annotationType");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m3 = Class.forName("Comment").getMethod("value");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```


在得到的代理类中, 实现了接口Comment, 并重写了所有方法, 包括`value`方法以及从`Annotation`继承而来的方法

而且这个代理类中有个构造函数, 它接受`InvocationHandler`. AnnotationInvocationHandler是Java中专门用于处理注解的Handler. 源码见[这里](https://github.com/frohoff/jdk8u-jdk/blob/master/src/share/classes/sun/reflect/annotation/AnnotationInvocationHandler.java)(费了不少功夫才找到... )

```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable {
    private static final long serialVersionUID = 6182022883658399397L;
    private final Class<? extends Annotation> type;
    private final Map<String, Object> memberValues;

    AnnotationInvocationHandler(Class<? extends Annotation> type, Map<String, Object> memberValues) {
        Class<?>[] superInterfaces = type.getInterfaces();
        if (!type.isAnnotation() ||
            superInterfaces.length != 1 ||
            superInterfaces[0] != java.lang.annotation.Annotation.class)
            throw new AnnotationFormatError("Attempt to create proxy for a non-annotation type.");
        this.type = type;
        this.memberValues = memberValues;
    }
    ...
}
```
其构造参数接受`memberValues`, 它是个键值对, 键是我们注解的属性名称, 值是其被定义时赋的值. 再往下是一个`invoke`方法

```java
public Object invoke(Object proxy, Method method, Object[] args) {
    String member = method.getName();
    Class<?>[] paramTypes = method.getParameterTypes();

    // Handle Object and Annotation methods
    if (member.equals("equals") && paramTypes.length == 1 &&
        paramTypes[0] == Object.class)
        return equalsImpl(args[0]);
    if (paramTypes.length != 0)
        throw new AssertionError("Too many parameters for an annotation method");

    switch(member) {
    case "toString":
        return toStringImpl();
    case "hashCode":
        return hashCodeImpl();
    case "annotationType":
        return type;
    }

    // Handle annotation member accessors
    Object result = memberValues.get(member);

    if (result == null)
        throw new IncompleteAnnotationException(type, member);

    if (result instanceof ExceptionProxy)
        throw ((ExceptionProxy) result).generateException();

    if (result.getClass().isArray() && Array.getLength(result) != 0)
        result = cloneArray(result);

    return result;
}
```

前面的动态代理类代理了注解接口中所有的方法, 实际上代理类中任何方法的调用, 最终都会被转到这里来. `invoke`的入参之一是被调用的方法实例, 首先会拿到方法实例的名字, 如果被调用的是`equals`走了特殊流程(emmm, 没看明白), 如果是toString, hashCode, annotationType的话, 会直接返回AnnotationInvocationHandler 预定义的方法实现.

而如果没有匹配到以上四种方法, 说明当前的方法调用是自定义注解字节声明的方法, 例如我们`Comment`注解的value方法. 这种情况下, 将从我们的注解map中获取注解属性对应的值

总的来讲, 当我们通过键值对的形式来给注解赋值时, 比如`@Comment(value="doc it")`, 并用注解修饰某元素. 那么编译器将在编译期间扫描作用对象(类或方法等)上的注解, 然后做基本检查, 如果注解允许被作用在当前位置(元注解的作用), 则将注解信息写入该元素的属性表

然后, 当你在运行时进行反射时, 虚拟机会将所有生命周期在`RUNTIME`的注解取出, 存入一个map, 并传递给一个`AnnotationInvocationHandler`实例.

最后, 虚拟机将采用JDK动态代理机制生成一个目标注解的代理类, 并初始化好处理器

这样, 一个注解的实例就创建出来了. 它本质上是个代理类
