### Signal(django.dispatch)
Django提供一个信号分发器，允许解耦的应用在框架的其它地方发生操作时会被通知到。简单来说，信号允许特定的sender通知一组receiver某些操作已经发生。这在多处代码和同一事件有关联的情况下很有用。

#### 监听信号
你需要注册一个receiver函数来接受信号，它在信号通过使用Signal.connect()方式发送时被调用。

##### Receiver函数
首先，我们需要定义接收器函数。接收器可以是Python函数或者方法：
```python
def my_callback(sender, **kwargs):
    print("Request finished!")
```
注意函数接受一个sender参数，以及通配符关键字(**kwargs)；所有信号处理器都必须接受这些参数。

##### 连接Receiver函数
有两种方法可以将一个接收器连接到信号。你可以采用手动的方法：
```python
from django.core.signals import request_finished

request_finished.connect(my_callback)
```
或者使用receiver()装饰器来自动连接：
```python
from django.core.signals import request_finished
from django.dispatch import receiver

@receiver(request_finished)
def my_callback(sender, **kwargs):
    print("Request finished!")
```
现在，在HTTP请求结束时，会调用my_callback函数。
严格来说，信号处理和注册的代码应该放在你想要的任何地方，但是推荐避免放在应用的根模块和models模块中，以尽量减少产生导入代码的副作用。实际上，信号处理通常定义在应用相关的signals子模块中。信号receivers 在你应用配置类中的ready() 方法中连接。如果你使用receiver() 装饰器，只需要在ready() 内部导入signals子模块就可以。

##### 连接由指定的sender发送的信号
一些信号会发送多次，但是你只想接收这些信号的一个确定的子集。例如，考虑 django.db.models.signals.pre_save信号，它在模型保存之前发送。大多数情况下，你并不需要知道所有模型何时保存 -- 只需要知道一个特定的模型何时保存。
在这些情况下，你可以通过注册来接收只由特定sender发出的信号。对于django.db.models.signals.pre_save的情况，sender是被保存的模型类，所以你可以认为你只需要由某些模型发出的信号：
```python
from django.db.models.signals import pre_save
from django.dispatch import receiver
from myapp.models import MyModel


@receiver(pre_save, sender=MyModel)
def my_handler(sender, **kwargs):
    ...
```
my_handler函数只在MyModel实例保存时被调用。不同的信号使用不同的对象作为它们的sender。

##### 防止重复的信号
在一些情况下，连接receiver 到信号的代码可能会执行多次。这会使你的receiver 函数被注册多次，并且导致它对于同一信号事件被调用多次。
如果这样的行为会导致问题（例如在任何时候模型保存时使用信号来发送邮件），可以传递一个唯一的标识符作为dispatch_uid 参数来标识你的receiver 函数。标识符通常是一个字符串，虽然任何可计算哈希的对象都可以。最后的结果是，对于每个唯一的dispatch_uid值，你的receiver 函数都只绑定到信号一次：
```python
from django.core.signals import request_finished

request_finished.connect(my_callback, dispatch_uid="my_unique_identifier")
```

### 定义和发送信号
你的应用可以利用信号功能来提供自己的信号。

#### 定义信号
class Signal([providing_args=list])
所有信号都是 django.dispatch.Signal 的实例。providing_args是一个列表，由信号将提供给监听者的参数名称组成。理论上是这样，但是实际上并没有任何检查来保证向监听者提供了这些参数。
```python
import django.dispatch

pizza_done = django.dispatch.Signal(providing_args=["toppings", "size"])
```
这段代码声明了pizza_done信号，它向接受者提供toppings和 size 参数。
要记住你可以在任何时候修改参数的列表，所以首次尝试的时候不需要完全确定API。

#### 发送信号
Django中有两种方法用于发送信号。
Signal.send(sender, **kwargs)
Signal.send_robust(sender, **kwargs)
调用 Signal.send()或者Signal.send_robust()来发送信号。你必须提供sender 参数（大多数情况下它是一个类），并且可以提供尽可能多的关键字参数。
例如，通过下面的方法来发送我们的pizza_done信号：
```python
class PizzaStore(object):
    ...

    def send_pizza(self, toppings, size):
        pizza_done.send(sender=self.__class__, toppings=toppings, size=size)
        ...
```
send() 和send_robust()都会返回一个含有二元组的列表 [(receiver, response), ... ]，它代表了被调用的接收器函数和他们的响应值。
send() 与 send_robust()在处理receiver 函数产生的异常时有所不同。send()  不会 捕获任何由receiver产生的异常。它会简单地让错误往上传递。所以在错误产生的情况，不是所有receiver 都会获得通知。
send_robust()捕获所有继承自Python Exception类的异常，并且确保所有receiver 都能得到信号的通知。如果发生错误，错误实例会在产生错误的receiver 的二元组中返回。

### 断开信号
Signal.disconnect([receiver=None, sender=None, weak=True, dispatch_uid=None])
调用Signal.disconnect()来断开信号的接收器。Signal.connect()中描述了所有参数。如果接收器成功断开，返回 True ，否则返回False。
receiver 参数表示要断开的已注册receiver。如果使用dispatch_uid标识receiver，它可以为None。

读了上面，你是不是觉得，哦，这完全和Linux下的信号是完全两回事嘛。是的，完全是两回事，Linux的信号是软件中断，提供一种处理异步事件的方法，信号是系统定义好的，可用作进程之间传递信息的一种方式。但是django的信号只是一个普通类而已，不能跨进程，其实就是函数的调用。


### django内置的几个信号
#### Model signals
- pre_init
- post_init
- pre_save
- post_save
- pre_delete
- post_delete
- class_prepared

#### Management signals
- post_sysncdb

#### Request/Response signals
- request_started
- request_finished
- got_request_exception

#### Test signals
- template_rendered


