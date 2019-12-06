## 深入理解迭代器
迭代器是Python中经常用的到一种数据结构，具有惰性加载的原则，可以帮我们节省很多内存。

![iterator](images/container.png)

```python
a=[1]*10000  
sys.getsizeof(a)   # 80072

b=itertools.repeat([1], 1000)
sys.getsizeof(b)   # 64
```
可以看到当我们使用使用迭代器后，内存占用非常少

### iterator
任何实现了__iter__和__next__方法的对象都是迭代器，__iter__返回迭代器自身，__next__返回下一个值，并且只能向前，无法后退，如果没有元素了，则会抛出Stopiteration异常。

```python
from itertools import count
counter = count(start=13)
next(counter)   # 13
next(counter)   # 14
```

### iterable
可迭代对象是实现了__next__方法的对象，像list、dict等都是可迭代对象。我们可以通过调用iter函数将一个可迭代对象转为迭代器
```python
a = [1, 2, 3]
print(isinstance(a, collections.Iterable))  # True
print(isinstance(a, collections.Iterator))   # False

# Get an iterator from an object.
x = iter(a)
print(isinstance(x, collections.Iterable))  # True
print(isinstance(x, collections.Iterator))   # True
```


### 自定义迭代器
从上面我们应该知道了迭代器需要实现哪些方法，那么我们只要实现了这些方法，也可以自定义一个迭代器

```python
class Fib:
    def __init__(self, max=10):
        self.a, self.b = 0, 1
        self.max = max
        self.cur = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.cur >= self.max:
            raise StopIteration
        self.a, self.b = self.b, self.a + self.b
        self.cur += 1
        return self.a


f = Fib()
x = [i for i in f]  # [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```
实现一个简单的斐波那契迭代器

### 一次消费
迭代器的特定就是向前迭代
```python
a = [1, 2, 3]
x = iter(a)
1 in x  # True
1 in x  # False
```


### Itertools
Itertools是生成迭代器的一个基础库

#### count
count(start=0, step=1)
```python
itertools.count(10) # 10, 11,12...
```

#### cycle
```python
itertools.cycle('ABC') # A, B, C
```

#### repeat
参数elem [,n]， 无限次或n次
```python
itertools.repeat(1, 10) # 1,1,1,1..1
```

#### islice
islice 创建一个迭代器，从start开始到stop， 步长是step，类似切片的参数

```python
# islice(iterable, [start, ] stop [, step]):

a = [i for i in range(100)]
b = itertools.islice(a, 0, 10)
next(b)  # 0
next(b)  # 1
...
next(b)  # 9
next(b)  # raise StopIteration
```

#### chain
chain(*iterables), 返回第一个迭代器的元素，耗尽后返回第二个迭代器的元素，直到所有的迭代器元素都耗尽

```python
a = [1, 2, 3]
b = ['a', 'b', 'c']
list(itertools.chain(a, b))  # [1, 2, 3, 'a', 'b', 'c']
```
这里就简单列举几个，其他的可以看下面的参考链接

### 参考
1. https://nvie.com/posts/iterators-vs-generators/
2. Itertools详解：https://juejin.im/post/5af56230f265da0b93485cca