---
title: 『译』测试固件那些事
catalog: true
date: 2017-07-26 14:57:33
subtitle: Pytest fixtures explicit, modular, scalable
header-img:
tags:
    - 札记
    - pytest
---

测试固件的目的是提供一个固定的基线。在此基础上，测试可以可靠且反复地执行。pytest fixture对典型的xUnit风格的setup/teardown功能提供了显著的改进。

## Fixtures as Function arguments
通过参数传递，测试函数能够接收固件对象。pytest fixture基于setup/teardown等xUnit风格上做了令人兴奋的改进：
- 固件具有显式名称，通过在参数列表中声明它们，便能在测试函数、模块、类或整个项目中激活。
- 固件以模块化的方式实现，因为每个固件都会触发特定的功能，它自己也可以使用其他的固件。
- 固件管理十分灵活，从简单的单元到复杂的功能测试，并且允许根据配置和组件选项将固件参数化，并在类，模块或整个测试会话中重用固件。

此外，pytest继续支持经典的xUnit风格设置。你可以混合使用这两种风格，也可以逐步地从经典到新风格。
另外，基于unittest和nosed的项目也能很容易扩展为pytest

## Fixtures as Function arguments
测试函数可以通过传递参数来接收固件对象。对于每个参数名称，都对应了该名称的固件函数提供的固件对象。固件的注册是通过 `@pytest.fixture`来注册的。让我们看一个简单的包含fixture和测试函数的测试模块:

```python
# content of ./test_smtpsimple.py
import pytest

@pytest.fixture
def smtp():
    import smtplib
    return smtplib.SMTP("smtp.gmail.com", 587, timeout=5)

def test_ehlo(smtp):
    response, msg = smtp.ehlo()
    assert response == 250
    assert 0 # for demo purposes
```

在这里， 我们在`test_ehlo`测试函数的参数列表中显式传入了所需的 `smtp`固件。pytest会发现并调用被标记为`smtp`的`@pytest.fixture`

## Sharing a fixture across tests in a module or class or session
如果你想复用声明的fixture，只需在声明时显式指定其作用域即可

```python
@pytest.fixture(scope='function')
def function_only_modular_smtp():
    pass


@pytest.fixture(scope='module')
def module_shared_smtp():
    pass


@pytest.fixture(scope='class')
def class_shared_smtp():
    pass


@pytest.fixture(scope='session')
def whole_testing_shared_smtp():
    pass
```

## Fixture finalization/ executing teardown code
不同于unittest中通过定义teardown函数的方式，pytest中只需通过`yield`关键字就能完成teardown工作

```python
@pytest.fixture(scope="module")
def smtp():
    smtp = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)  # do something
    yield smtp  # provide the fixture value
    print("teardown smtp")    # do the teardown thing
    smtp.close()
```

如果你的teardown包含复杂的工作，也可以选择下面这种方式。
通过向固件传入`request`对象，便能内省测试上下文。

```python
@pytest.fixture(scope="module")
def smtp(request):
    smtp = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)
    def fin():
        print ("teardown smtp")
        smtp.close()
    request.addfinalizer(fin)
    return smtp  # provide the fixture value
```
不同于第一种方式，`addfinalizer`方法支持注册多个finalizer并且能保证代码一定被执行(yield只是一种语法糖，并不能保证这一点)

## Fixtures can introspect the requesting test context
如刚才展示的，`request`对象允许在定义固件时访问其上下文。假设我们的smtp需要动态地获取服务器，它的单元测试代码可能如下所示

```python
smtpserver = 'mail.python.org'

def test_showhelo(smtp):
    assert 0, smtp.helo()
```

在定义固件时，通过`request`便能动态获取到这个值
```python
@pytest.fixture(scope='module')
def smtp(request):
    server = getattr(request.module, 'smtpserver', 'smtp.gmail.com') # default value
    smtp = smtplib.SMTP(server, 587, timeout=5)
    yield smtp
    print ("finalizing %s (%s)" % (smtp, server))
    smtp.close()
```

