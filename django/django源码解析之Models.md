### Model源码解析
一个模型是关于您的数据的单一的，最终的信息来源。 它包含您正在存储的数据的基本字段和行为。通常，每个模型映射到单个数据库表。
Model的源码中djang.db.models下的base.py中：
```python
clsss Model(six.with_metaclass(ModelBase)):
    pass
```
Model类继承了ModelBase，ModelBase和Model在同一个位置。
```python
class ModelBase(type):
    """
    Metaclass for all models.
    所有models的元类。
    """
    def __new__(cls, name, bases, attrs):
        """
        param name：项目中模型的名字
        param bases: 模型的基类
        param attrs: 模型的属性
        """
        super_new = super(ModelBase, cls).__new__  # 获取内置函数__new__()

        # Also ensure initialization is only performed for subclasses of Model
        # (excluding Model class itself).
        # 确保对Model的子类进行初始化，不包含Model本身
        parents = [b for b in bases if isinstance(b, ModelBase)]
        if not parents:
            return super_new(cls, name, bases, attrs)  # 对子类初始化

        module = attrs.pop("__module__")  # __module__ is the module name in #which the class was defined 获取模块的名字
        new_attrs = {'__module__': module}  # 创建新属性
        classcell = attrs.pop('__classcell__', None)  # 获取__classcell__属性
        if classcell is not None:
            new_attrs['__classcell__'] = classcell  # 因为使用type.__new__创建对象时，必须传入__classcell__，所以需要获取到__classcell__属性
        new_class = super_new(cls, name, bases, new_attrs)
        attr_meta = attrs.pop('Meta', None)  # 获取每个模型的Meta类
        abstract = getattr(attr_meta, 'abstract', False)  # 获取每个模型Meta中的abstract属性，没有就返回False

        if not attr_meta:
            # 如果模型没有Meta类，就从新建的new_class中获取
            meta = getattr(new_class, 'Meta', None)
        else:
            meta = attr_meta
        base_meta = getattr(new_class, '_meta', None)

        app_label = None  # app的名字
        app_config = apps.get_containing_app_config(module)  # 从配置中获取app的相关配置，即每个app下面的apps.py中的AppConfig

        # 如果获取不到app_label，而且该模型不是abstract的，则报错
        if getattr(meta, 'app_label', None) is None:
            if app_config is None:
                if not abstract:
                    raise RuntimeError(
                        "Model class %s.%s doesn't declare an explicit "
                        "app_label and isn't in an application in "
                        "INSTALLED_APPS." % (module, name)
                    )

            else:
                app_label = app_config.label  # 从配置中获取app的label

        # 在new_class中的_meta中添加一个Options对象，初始化为(meta, app_label)元组
        new_class.add_to_class('_meta', Options(meta, app_label))
        if not abstract:
            # 如果不是抽象类，则添加不存在以及多对象返回异常
            new_class.add_to_class(
                'DoesNotExist',
                subclass_exception(
                    str('DoesNotExist'),
                    tuple(
                        x.DoesNotExist for x in parents if hasattr(x, '_meta') and not x._meta.abstract
                    ) or (ObjectDoesNotExist,),
                    module,
                    attached_to=new_class))
            new_class.add_to_class(
                'MultipleObjectsReturned',
                subclass_exception(
                    str('MultipleObjectsReturned'),
                    tuple(
                        x.MultipleObjectsReturned for x in parents if hasattr(x, '_meta') and not x._meta.abstract
                    ) or (MultipleObjectsReturned,),
                    module,
                    attached_to=new_class))
            if base_meta and not base_meta.abstract:
                # Non-abstract child classes inherit some attributes from their
                # non-abstract parent (unless an ABC comes before it in the
                # method resolution order).
                """
                    非抽象子类从其非抽象父进程继承一些属性（除非ABC在方法解析顺序中出现）。
                """
                if not hasattr(meta, 'ordering'):
                    new_class._meta.ordering = base_meta.ordering
                if not hasattr(meta, 'get_latest_by'):
                    new_class._meta.get_latest_by = base_meta.get_latest_by

        is_proxy = new_class._meta.proxy  # 是否为代理模型

        # If the model is a proxy, ensure that the base class
        # hasn't been swapped out.
        if is_proxy and base_meta and base_meta.swapped:
            raise TypeError("%s cannot proxy the swapped model '%s'." % (name, base_meta.swapped))

        # Add all attributes to the class.
        for obj_name, obj in attrs.items():
            new_class.add_to_class(obj_name, obj)  # 将所有属性添加到new_class中

        # 在此模型上声明的任何类型的所有字段
        new_fields = chain(
            new_class._meta.local_fields,
            new_class._meta.local_many_to_many,
            new_class._meta.private_fields
        )
        field_names = {f.name for f in new_fields}
 

```
