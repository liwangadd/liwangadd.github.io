---
title: Python中的functools模块
tags: 
- Python 
- 标准库
categories: Python
abbrlink: f6b0a560
date: 2018-03-22 09:22:20
---

### **functools.partial(func[, \*args][, \*\*keywords])**

该函数通过包装手法，允许我们"重新定义"函数。返回用args和keywords填充了func指定位置参数后的函数，调用该函数只需传入原函数未填充的参数。

```python
from functools import partial
basetwo = partial(int, base=2)
basetwo('10010')
```

basetwo('10010')实际上等价于调用int('10010', base=2)

### @functools.singledispatch

使用过面向对象语言的同学，肯定熟悉各种方法的重载。虽然Python不支持方法重载的，但是我们可以添加@functools.singledispatch注解来动态指定相应的方法所接收的参数类型。

```python
from functools import singledispatch
class TestClass(object):
  @singledispatch
  def test_method(arg, verbose=False):
    if verbose:
      print('Let me just say,', end=' ')
    print(arg)
  @test_method.register(int)
  def _(arg):
    print('Strength in numbers, eh?', end=' ')
  @test_method.register(list)
  def _(arg):
    print('Enumerate this:')
```

通过@test_method.register(int)和@test_method.register(list)指定当test_method的第一个参数为int或者list的时候，分别调用不同的方法来进行处理。

### **functools.update_wrapper(\*\*wrapper, wrapped)**

默认partial对象没有`__name__`和`__doc__`，这种情况下，对于装饰器函数非常难以debug。使用该函数可以将被封装函数的这些属性复制到封装函数中。

```python
def my_decorator(f):
  def wrapper(*args, **kwds):
    print('print in wrapper')
    return f(*args, **kwds)
  return update_wrapper(wrapper, f)
@my_decorator
def example():
    print('print in example')
```

### @functools.wraps

该注释内部调用了`functools.update_wrapper()`函数，简化编码复杂度。

```python
from functools import warps
def my_decorator(f):
    @warps(f)
    def wrapper(*args, **kwds):
        print 'print in wrapper'
        return f(*args, **kwds)
   	return wrapper
@my_decorator
def example():
    print 'print in example'
```

<!--more-->

### @functools.lru_cache(maxsize=None, typed=False)

@lru_cache用于缓存函数返回的结果，对于重复的调用直接从缓存中获取返回值。如果maxsize设置为None，则缓存没有上界。如果typed设置为true，则对于不同的参数类型会区别缓存。

```python
from functools import lru_cache
@luc_cache(None)
def add(x, y):
    print('calculating: %s + %s' % (x, y))
    return x + y
print(add(1, 2))
print(add(1, 2))
print(add(2, 3))
```

输出结果：

```python
calculating: 1 + 2
3
3
calculating: 2 + 3
5
```

### @functools.total_ordering

这是一个类装饰器，用于自动实现类的比较运算。若自定义类中实现了`__eq__`和`__lt__`、`__gt__`、`__ne__`、`__le__`、`__ge__`中的一个，带有@total_ordering装饰的类会自动实现剩下的函数 。

#### **functools.cmp_to_key(func)**

该函数用于将旧式的比较函数转化为key(关键值)函数并应用于接受key function的工具类中(sorted(), min(), max(), heapq.nsamllest(), heapq.nlargest(), itertools,groupby())

```python
from functools import cmp_to_key
sorted(iterable, key=cmp_to_key(cmp_func))
```

### **functools.reduce(func, iterable, [, initializer])**

与python2中的内建函数reduce等价
