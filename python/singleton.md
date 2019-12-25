## 单例模式

### 包导入模式

```python
class Singleton:
    pass

singleton = Singleton()
```

### 装饰器模式

```python
def SingletonDec(cls):
    _instance = {}

    def _singleton(*args, **kwargs):
        if cls not in _instance:
            _instance[cls] = cls(*args, **kwargs)
        return _instance[cls]

    return _singleton

@SingletonDec
class Foo:
    pass
```


### 类

```python
class Singleton:
    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):
            cls._instance = super().__new__(cls)
        return cls._instance
```


### 元类

```python
class Singleton(type):
    def __call__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):
            cls._instance = super().__call__(*args, **kwargs)
        return cls._instance

class Foo(metaclass=Singleton):
    pass
```