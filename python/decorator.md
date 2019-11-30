## 理解装饰器
装饰器是Python里面经常使用到的一个功能，通过装饰器我们能在不修改函数的情况下为函数增加一些功能，装饰器通过语法糖调用

### 装饰器

#### 普通装饰器

```python
def logger(func):
    def decorator(*args, **kwargs):
        print("func start")
        func(*args, **kwargs)
    return decorator

@logger
def func(name):
    print("my name is %s" % name)


func("leon")
```
这是个装饰器的简单例子，整个调用其实就相当于logger(func(name)), 如果是有多个装饰的情况下，那么调用顺序是怎样的？

```python
@A
@B
@C
def func():
    pass
```
上面执行的的过程就是A(B(C(func())))

#### wraps
我们先看一个例子
```python
def logger(func):
    def decorator(*args, **kwargs):
        print("func start")
        func(*args, **kwargs)
    return decorator

@logger
def func(name):
    print("my name is %s" % name)


func("leon")
print(func.__name__)   # decorator
```
当我们调用func.__name__ 原本上应该输出func，但是因为加入了装饰器，导致__name__、__doc__等一些构造函数被改变了，那么如果我们如何解决这个问题呢？
这里就需要用到functools.wraps，我们看下加入wraps试下

```python
from functools import wraps
def logger(func):
    @wraps(func)
    def decorator(*args, **kwargs):
        print("func start")
        func(*args, **kwargs)
    return decorator

@logger
def func(name):
    print("my name is %s" % name)


func("leon")
print(func.__name__)    # func
```
这里__name__是我们预期的，wraps到底做了些什么事情，这里我们看下源码

```python
WRAPPER_ASSIGNMENTS = ('__module__', '__name__', '__qualname__', '__doc__',
                       '__annotations__')
WRAPPER_UPDATES = ('__dict__',)
def update_wrapper(wrapper,
                   wrapped,
                   assigned = WRAPPER_ASSIGNMENTS,
                   updated = WRAPPER_UPDATES):
    """Update a wrapper function to look like the wrapped function

       wrapper is the function to be updated
       wrapped is the original function
       assigned is a tuple naming the attributes assigned directly
       from the wrapped function to the wrapper function (defaults to
       functools.WRAPPER_ASSIGNMENTS)
       updated is a tuple naming the attributes of the wrapper that
       are updated with the corresponding attribute from the wrapped
       function (defaults to functools.WRAPPER_UPDATES)
    """
    for attr in assigned:
        try:
            value = getattr(wrapped, attr)
        except AttributeError:
            pass
        else:
            setattr(wrapper, attr, value)
    for attr in updated:
        getattr(wrapper, attr).update(getattr(wrapped, attr, {}))
    # Issue #17482: set __wrapped__ last so we don't inadvertently copy it
    # from the wrapped function when updating __dict__
    wrapper.__wrapped__ = wrapped
    # Return the wrapper so this can be used as a decorator via partial()
    return wrapper

def wraps(wrapped,
          assigned = WRAPPER_ASSIGNMENTS,
          updated = WRAPPER_UPDATES):
    """Decorator factory to apply update_wrapper() to a wrapper function

       Returns a decorator that invokes update_wrapper() with the decorated
       function as the wrapper argument and the arguments to wraps() as the
       remaining arguments. Default arguments are as for update_wrapper().
       This is a convenience function to simplify applying partial() to
       update_wrapper().
    """
    return partial(update_wrapper, wrapped=wrapped,
                   assigned=assigned, updated=updated)

```
从源码上看wraps最终是调用了update_wrapper， 这里会将___name__,__doc__等一些构造函数从原函数赋值到wrapped的的函数中

#### 带参数的装饰器
如果我们想给装饰器加一些参数呢，这个要怎么做？

```python
from functools import wraps

def need_logger(need=True):
    def logger(func):
        @wraps(func)
        def decorator(*args, **kwargs):
            if need:
                print("func start")
            func(*args, **kwargs)
        return decorator
    return logger

@need_logger(need=True)
def func(name):
    print("my name is %s" % name)


func("leon")
```
如果装饰器要带参数的话，我们需要再把装饰器加一层，这个是函数的，我们也可以用类实现带参数的装饰器

```python
class CustomLogger():
    def __init__(self, need=True):
        self.need = need

    def __call__(self, func):
        @wraps(func)
        def decorator(*args, **kwargs):
            if self.need:
                print("func start 2")
            func(*args, **kwargs)
        return decorator


@CustomLogger(need=True)
def func(name):
    print("my name is %s" % name)
```
第二种方法是不是有点不一样的感觉，通过类，我们可以将装饰器封装成一个完成包，去集成我们自己的功能

### 内置的三个装饰器
这里我们讲一下Python里面内置的装饰器

#### property
通过@property我们可以将一个函数变成一个属性进行调用，很大程度的方便了我们使用
```python
class Demo():
    def __init__(self):
        self.value = 0

    @property
    def get_value(self):
        return self.value
    
    @get_value.setter
    def set_value(self, value):
        self.value = value


d = Demo()
print(d.get_value)   # 0

d.set_value = 1

print(d.get_value)   # 1
```

#### classmethod
classmethod和类绑定，不是和实例绑定, classmethod接收一个类参数cls, 可以修改类的状态，并将其作用到所有的类实例上
```python
class Demo():
    value = 1

    @classmethod
    def get_value2(cls):
        cls.value += 1
        return cls.value


print(Demo.get_value2())  # 2
print(Demo.value)    # 2
```
可以看到，我们通过类方法修改的值已经作用域所有的类实例了

#### staticmethod
静态方法跟普通函数没什么区别，通过类和实例都可以直接该函数

```python
class Demo():
    @staticmethod
    def get_value(cls):
        return "ok"

print(Demo.get_value())  # ok
```

