## 深入理解生成器
生成器是一种特殊的迭代器，当每次调用yield，函数就会暂停。生成器的好处就是节省内存空间。生成器有两种类型：生成器函数和生成器表达式

### 生成器表达式
将列表推导式[]改成()就行
```python
gen=(i for i in range(10))
gen  # <generator object <genexpr> at 0x104d21960>
isinstance(gen, types.GeneratorType)   # True
isinstance(gen, collections.Iterator)  # True
isinstance(gen, collections.Iterable)  # True
```

### 生成器函数yield

```python
def func():
    yield 1
    yield 2
    yield 3


f = func()
print(next(f))  # 1
print(next(f))  # 2
print(next(f))  # 3
```

#### next
通过next(next其实也是调用__next__)可以进行迭代使用，当达到最后一个元素之后，就会抛出StopIteration

#### send
通过send我们可以想生成器函数发送数据，第一次发送必须是next之后，否则会报错，当然我们可以先调用send(None), 等价于next

`send`可以向生成器传值的同时会让生成器前进到下一个`yield`语句，并将`yield`右侧的值作为返回值

```python
def func():
    x = yield 1
    r = yield x
    print(r)


f = func()
f.send(None)
print(f.send("ok"))
print(next(f))
```
首先我们先调用send初始化生成器，然后再`send("ok")`, 会将x进行覆盖，然后传给r，最后返回的就是"ok"

#### 协程状态
`GEN_CREATE` 等待开始执行
`GEN_RUNNING` 解释器正在执行
`GEN_SUSPENDED` 在yield表达式处暂停
`GEN_CLOSED` 执行结束
使用`inspect.getgeneratorstat(generator)`可以查看

```python
from inspect import getgeneratorstate

def md():
    yield 1
    yield 2


m = md()

print(getgeneratorstate(m))  # GEN_CREATED

m.send(None)

print(getgeneratorstate(m))  # GEN_SUSPENDED

m.send('A')

print(getgeneratorstate(m))  # GEN_SUSPENDED

try:
    m.send('B')
except:
    pass

print(getgeneratorstate(m))  # GEN_CLOSED
```

#### yield from
yield from是[pep380](https://www.python.org/dev/peps/pep-0380/)后加入的，可以将一个可迭代对象直接转为生成器

```python
def func():
    # 不用在这样写
    # for i in range(10):
    #     yield i
    yield from [i for i in range(10)]

for i in func():
    print i
```

#### throw
throw(typ[,val[,tb]])可以在yield出抛出某个异常, 如果捕获到这个错误，那么生成器会前进到下个yield，并将产出值做为throw的返回值，否则中止生成器

```python
def generator():
    try:
        yield 'a'
    except ZeroDivisionError as e:
        print("catch ", e)
    yield 'b'


gen = generator()
print(next(gen))     
print(gen.throw(ZeroDivisionError, 'error')) 

### 输出
# a
# catch  error
# b
```

#### return
生成器返回值，在python2中是不能使用return做返回值的，但是python3中却可以


```python
def generator():
    yield
    return 'a'


gen = generator()
next(gen)

try:
    next(gen)
except StopIteration as e:
    print("catch ", e)
```
执行上面返回的是catch e，当运行return，会把return的返回值做为StopIteration的值传递出去

#### close
停止生成器，我们可以调用f.close(), close其实就是抛出一个GeneratorExit异常，让生成器退出，如果一个生成器函数被close掉后，还继续使用send或者next调用，会抛出异常StopIteration

```python
def generator():
    yield 1
    yield 2
    yield 3


gen = generator()
print(next(gen))
gen.close()
print(next(gen))
```