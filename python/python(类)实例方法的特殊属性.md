### Python实例方法的特殊属性
我们通过下面这个例子来说明：
```python
class A(object):
    def foo(self):
        ''' A method '''

        print("Hello World!")

    bar = foo

    @classmethod
    def class_method(cls, arg):
        print(str(arg))


a = A()
```

#### 实例方法的只读属性
与自定义函数的特殊属性相比，实例方法具有__self__, __func__这两个函数所不具有的只读属性；此外，方法的__doc__,__name__,__module__属性也都是只读属性。对于实例而言，其__self__属性为实例本身。
```python
print(a.foo.__self__)
<__main__.A object at 0x0000000002285C18>
print(a)
<__main__.A object at 0x0000000002285C18>
```
而__func__属性则返回方法所对应的底层函数：
```python
print(a.foo)
<bound method A.foo of <__main__.A object at 0x0000000002255C18>>
print(a.foo.__func__)
<function foo at 0x0000000002267AC8>
```
注意a.foo与a.foo.__func__的区别。
至于__doc__,__name__,__module__属性则与函数相应属性的值一直，所不同的是方法的这些属性均为只读属性，不可改写：
```python
print(a.foo.__doc__)
 A method 
print(a.foo.__name__)
foo
print(a.foo.__module__)
__main__
```

#### 实例方法可通过底层函数访问函数属性
不过，实例方法也可以通过其底层的function对象(通过__func__属性获得)访问函数所具有的特殊属性，如：
```python
print(a.foo.__func__.__code__)
<code object foo at 00000000020AF5B0, file "F:\python_method.py", line 12>
```
因此，诸如 __doc__,__name__,__module__等属性的值就可以通过底层函数相应的特殊属性进行改写：
```python
a.foo.__doc__ = "A method about foo"
AttributeError: attribute '__doc__' of 'instancemethod' objects is not writable

print(a.foo.__func__.__doc__)
 A method 

a.foo.__func__.__doc__ = "Can be changed through func doc"
print(a.foo.__doc__)
Can be changed through func doc

a.foo.__name__ = "foo2"
AttributeError: 'instancemethod' object has no attribute '__name__'

a.foo.__func__.__name__ = "foo2"
print(a.foo.__name__)
foo2
```

#### 底层函数的唯一性
当一个类的实例方法是通过其他实例方法创建的，则其他实例方法所对应的底层函数并非其所创建的实例方法，而是其所创建的实例方法所对应的底层函数：
```python
print(a.bar)
<bound method A.foo of <__main__.A object at 0x0000000002245C18>>

print(a.bar.__func__)
<function foo at 0x0000000002257AC8>
```
上例中，通过a.foo创建了a.bar，但是a.bar.__func__并不是a.foo而是a.foo.__func__:
```python
print(a.bar.__func__)
<function foo at 0x00000000021F7B38>

print(a.foo.__func__)
<function foo at 0x00000000021F7B38>

print(a.foo)
<bound method A.foo of <__main__.A object at 0x00000000021E5C50>>
```

#### 实例方法的首位参数为实例本身
实例方法执行时，其底层函数的首位参数为实例本身，下面两行代码的执行结果是一样的：
```python
a.foo()
Hello World!
a.foo.__func__(a)
Hello World!
```

#### 类实例方法的首位参数是类本身
当一个实例方法(严格来说是类实例方法)是由类创建的，则其__self__属性是其类本身：
```python
print(a.class_method.__self__)
<class '__main__.A'>
```
事实上，通过类方法建立的(类)实例方法，在调用底层函数时(下例是A.class_method.__func__),其首位参数(也即__self__)是类本身，这一点与实例方法执行时有所区别。
```python
a.class_method('dog')
dog
A.class_method('dog')
dog
A.class_method.__func__(A, 'dog')
dog
```
类实例方法，本身也是bound method，这与实例方法一致：
```python
print(a.class_method)
<bound method type.class_method of <class '__main__.A'>>
print(a.foo)
<bound method A.foo of <__main__.A object at 0x0000000002255C18>>
print(A.class_method)
<bound method type.class_method of <class '__main__.A'>>
print(A.foo)
<unbound method A.foo>
```
只不过，一个绑定了实例，一个绑定了类。

### 总结
- Python 3 有两种 bound method, 一种是 instance method,一种是 class method（class instance method）,两种都可以被实例访问；
- 对于 instance method 其 __self__ 属性值为 instance 本身，而 class method 其属性值则为 class 本身；
- 不管是 instance method 还是 class method ，执行时，都需要调用与之对应的底层函数（underlying function，通过 __func__ 访问）,底层函数的首位参数通过 __self__ 获得， 对于 instance method 其为该实例本身，对于 class method 则为该类本身；
- bound method 可以通过对应的底层函数，访问函数的所有特殊属性。

### 原文出自：https://segmentfault.com/a/1190000005701971
- 非常感谢。
