## 深入理解__slots__

### Why use slots?
python里面每个类默认都有个`__dict__`属性，这个里面存储了一个对象的属性，通过这个我们可以动态的设置属性，但是在有些情况（当创建的对象成千上万）下，`__dict__`会浪费很多内存，有个可以减少内存使用的方式就是使用__slots__。

有篇比较出名的文章就是说明这种情况：[`Saving 9 GB of RAM with Python’s __slots__`](http://tech.oyster.com/save-ram-with-python-slots/)

### slots
`__slots__`的文档在[slots](https://docs.python.org/3/reference/datamodel.html#slots)


#### Without `__slots__`
```python
class MyClass:
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

#### With `__slots__`
```python
class MyClass:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y
```
当使用了`__slots__`将不会有`__dict__`属性，并且无法动态添加属性


### 内存使用
使用前：
```python
class A:
    def __init__(self, x, y):
        self.x = x
        self.y = y

import resource
mem_init = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
num = 1024*256
x = [A(1,1) for i in range(num)]
mem_done = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss

print(mem_init)   # 5369856
print(mem_done)   # 66310144
```

使用后：
```python
class A:
    def __init__(self, x, y):
        self.x = x
        self.y = y

import resource
mem_init = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
num = 1024*256
x = [A(1,1) for i in range(num)]
mem_done = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss

print(mem_init)   # 5390336
print(mem_done)   # 36188160
```
内存使用是mem_done-mem_init，由于本人测试是Mac，所以单位是Byte，可以看到在使用了`__slots__`后内存，减少了50%左右。当需要实例化的对象超过上千，那么使用`__slots__`将极大的较少内存，使用了`__slots__`其实是舍弃了一些灵活性，大家应该视情况使用.