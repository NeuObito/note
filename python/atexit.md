### atexit
- atexit定义了一个简单的用于注册程序退出时的回调函数，在这个函数中我们可以做一些资源清理的操作，如果程序非正常退出，或者调用了os._exit()，注册的回调函数不会被调用
- 我们也可以通过sys.exitfunc来注册回调，但通过它只能注册一个回调，而且还不支持参数。所以建议大家使用atexit来注册回调函数。但千万不要在程序中同时使用这两种方式，否则通过atexit注册的回调可能不会被正常调用。其实通过查阅atexit的源码，你会发现原来它内部是通过sys.exitfunc来实现的，它先把注册的回调函数放到一个列表中，当程序退出的时候，按先进后出的顺序调用注册的回调。如果回调函数在执行过程中抛出了异常，atexit会打印异常的文字信息，并继续执行下一下回调，直到所有的回调都执行完毕，它会重新抛出最后接收到的异常。
- 如果使用的是2.6+, 还可以使用装饰器来使用回调函数

#### 实例
```python
import atexit

def goodbye(name, adjective):
    print "Goodbye, %s, it was %s to meet you." % (name, adjective)

atexit.register(goodbye, 'Donny', 'nice')

# or
atexit.register(goodbye, adjective="nice", name="Donny")

# or
'''
@atexit.register
def goodbye():
    print "You are now leaving the Python sector."

'''
```