---
title: 在JUnit中定制Runner
catalog: true
date: 2019-05-29 15:56:22
author: Rancho
header-img:
subtitle: JUnit4 Customer Runners
tags:
- 编程札记
- JUnit
- Runner
---

## 前言
本文将快速介绍如何在`JUnit`测试框架中使用自定义`Runner`来运行单测
当前, 这需要配合`@RunWith`注解

## 准备
首先, 添加项目依赖

```java
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
</dependency>
```

## 实现Runner
JUnit框架提供了抽象类`Runner`, 它定义了两个抽象方法
```java
public abstract Description getDescription();

public abstract void run(RunNotifier var1);
```


另外, 它还需要实现`Describable`接口, 即实现`getDescription()` 方法
接下来我们实现一个`CustomerRunner`

```java
import org.junit.Test;
import org.junit.runner.Description;
import org.junit.runner.Runner;
import org.junit.runner.notification.RunNotifier;

import java.lang.reflect.Method;

/**
 * @author rancho
 * @date 2019-05-29
 */
public class CustomerRunner extends Runner {

    private Class testedClass;

    public CustomerRunner(Class testedClass) {
        super();
        this.testedClass = testedClass;
    }

    /**
     * implements Describable interface
     * @return Description
     */
    @Override
    public Description getDescription() {
        return Description.createTestDescription(testedClass, "A Customer runner");
    }

    /**
     * invoke the target tested methods using reflection
     * @param notifier used for firing events that have information about the test progress
     */
    @Override
    public void run(RunNotifier notifier) {
        System.out.println("running the tests based on CustomerRunner" + testedClass);

        try {
            Object testedObject = testedClass.newInstance();
            for (Method method : testedClass.getMethods()) {
                if (method.isAnnotationPresent(Test.class)) {
                    notifier.fireTestStarted(Description.createTestDescription(testedClass, method.getName()));
                    method.invoke(testedObject);
                    notifier.fireTestFinished(Description.createTestDescription(testedClass, method.getName()));
                }
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

在这个实现类中, 我们定义了一个接受类参数的构造函数, 这是JUnit框架对Runner的规范. 单测运行时, JUnit会将目标类传给Runner的构造器.
`getDescription`方法用于返回描述信息, 而最关键的就在于`run`方法实现, 通过反射来调用目标方法.
另外, 通过`run`方法的入参`RunNotifier`, 我们可以出发具有各种测试进度信息的事件. 这里我们仅在目标方法调用前后触发事件. 感兴趣的话可以自行翻下RunNotifier中定义的其他方法

接下来, 我们随便写个测试用例来用下这个`Runner`
```java
import org.junit.Test;
import org.junit.runner.RunWith;

import static org.junit.Assert.*;

/**
 * @author rancho
 * @date 2019-05-29
 */
@RunWith(CustomerRunner.class)
public class CustomerRunnerTest {

    Calculator calculator = new Calculator();

    @Test
    public void testCalculator() {
        System.out.println("start Calculator");
        assertEquals("addition", 8, calculator.add(5, 3));
    }

}

class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
}
```

`Runner`是JUnit中的低级运行器, 相比之下, 其子类`ParentRunner`和`BlockJUnit4Runner`更加易于定制和扩展
`ParentRunner`同样是个抽象类, 它会以分层的方式来运行测试
一般情况, 如果你想对Runner做定制, 从`BlockJUnit4Runner`来派生子类是不错的选择


## 小结
通过定制JUnit Runner, 开发人员可以自由控制测试执行的整个过程. 一些流行的第三方Runner实现还有`SpringJUnit4ClassRunner`, `MockitoJUnitRunner`, `HierarchicalContextRunner`等
如果你想runwith multiple runner, 可以考虑从多个父类派生出新的子类Runner
而在PowerMock中, 这个问题是通过代理的方式解决的(`@PowerMockRunnerDelegate`), 也是个不错的思路
