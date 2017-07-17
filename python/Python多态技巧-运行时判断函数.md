### Python多态技巧，运行时判断函数
- Java，C++等的多态，需要继承的方法在基类定义以后，才能编译通过；
- Python是动态的，代码也是数据，数据不需要申明，因此可以在运行时判断

```python
class Base(object):
    def __init__(self, name):
        self.name = name

    def print_name(self):
        print(self.function, self.name)  # 这里可以直接使用子类的函数，如果self具有此函数的话


class Sub(Base):
    def __init__(self):
        super(Sub, self).__init__('Sub')

    def function(self):
        return 'Sub Function'


if __name__ == '__main__':
    sub = Sub()
    sub.print_name()
```
