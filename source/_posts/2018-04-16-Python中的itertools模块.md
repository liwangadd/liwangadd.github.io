---
title: Python中的itertools模块
tags: Python
categories: Python
abbrlink: 4f4399bb
date: 2018-04-16 09:45:12
---

#### **无限迭代器**

- count(firstval=0, step=1)

  创建一个从firstval（默认值为0）开始，以step（默认值为1）为步长的无限整数迭代器。此迭代器不支持长整数，如果超出了sys.maxint，计数器将溢出并继续从-sys.maxint-1开始计算。

  ```python
  for x, y in zip(count(1), ['a', 'b', 'c']):
      print(x, y)
  输出:
  1 a
  2 b
  3 c
  ```

- cycle(iterable)

  对iterable中的元素反复执行循环，返回生成的迭代器

  ```python
  index = 0
  for value in cycle('abcd'):
      index += 1
      if index == 10:
          break
      print(value)
  输出：
  a b c d a b c d a 
  ```

- repeat(object, [,items])

  返回一个迭代器，反复生成object，如果给定times，则重复次数为items，否则为无限

  ```python
  for value in repeat('windylee', 5):
      print(value, end=' ')
  输出：
  windylee windylee windylee windylee windylee 
  ```

<!-- more -->

#### 有限迭代器**

- chain(iterable1, iterable2, iterable3, ...)

  chain接收多个可迭代对象作为参数，将他们连接起来，作为一个新的迭代器返回

  ```python
  for value in chain([1, 2, 3], ['a', 'b', 'c']):
      print(value, end=' ')
  输出：
  1 2 3 a b c 
  ```

- compress(data, selectors)

  compress可用于对数据进行筛选，当selectors的元素为true时，则保留data对应位置的元素

  ```python
  for value in compress([1, 2, 3, 4, 5], [True, False, False, True, True]):
      print(value, end=' ')
  输出:
  1 4 5 
  ```

- dropwhile(predicate, iterable)

  创建一个迭代器，只要函数predicate(item)为True，就丢弃iterable中的项，如果predicate返回False，就会生成iterable中的项和所有后续项。

  ```python
  def predict(item):
      if item < 3:
          return True
      return False

  for value in dropwhile(predict, [1, 2, 3, 4, 5]):
      print(value, end=' ')
  输出：
  3 4 5 
  ```

- takewhile(predicate, iterable)

  创建一个迭代器，只要函数predite(item)为True，就保留iterable中的项，如果predicate返回False，就会丢弃iterable中的项和所有的后续项。相当于`dropwhile`的反操作。

  ```python
  def predict(item):
      if item < 3:
          return True
      return False

  for value in takewhile(predict, [1, 2, 3, 4, 5]):
      print(value, end=' ')
  输出：
  1 2 
  ```

- groupby(iterable[, keyfunc])

  返回一个产生按照keyfunc的返回值进行分组的值集合的迭代器。iterable是一个可迭代对象，keyfunc是分组函数，用于对iterable的**连续项**进行分组。如果不指定，则默认将iterable中连续相同的项作为一组，否则将keyfunc返回值相同的连续项作为一组。

  ```python
  def predict(item):
      if item < 3 or item > 7:
          return True
      return False

  for key, value in groupby([1, 2, 3, 4, 5, 5, 6, 7, 8, 9, 10], predict):
      print(key, list(value))
  输出：
  True [1, 2]
  False [3, 4, 5, 5, 6, 7]
  True [8, 9, 10]
  ```

- starmap(function, iterable)

  创建一个迭代器，对于iterable中每一项item生成值`func(*item)`。

  ```python
  for value in starmap(lambda x, y: x * y, [(0, 5), (1, 6), (2, 7), (3, 8), (4, 9)]):
      print(value, end=' ')
  输出：
  0 6 14 24 36 
  ```

- tee(iterable, [, n])

  用于从iterable创建n个独立的迭代器，以元组的形式返回，n的默认值是2。由于生成的项会被缓存，并在所有新创建的迭代器中使用，所以一定要注意不要再调用`tee()`之后使用原始迭代器iterable，否则缓存机制可能无法正常工作。

  ```python
  iter1, iter2 = tee([1, 2, 3])
  for v1 in iter1:
      print(v1, end=' ')
  print()
  for v2 in iter2:
      print(v2, end=' ')
  输出：
  1 2 3 
  1 2 3 
  ```

- zip_longest(\*iterables, fillvalue=None)

  类似于内置函数zip，只不过迭代完最长的序列为止，短序列默认用None补齐。

  ```python
  for v1, v2 in zip_longest([1, 2, 3], ['a', 'b', 'c', 'd']):
      print(v1, v2)
  输出：
  1 a
  2 b
  3 c
  None d
  ```

- accumulate(iterable, [,func])

  创建一个迭代器，返回从第一项到当前项的子列表执行reduce函数的值。func是一个二元函数

  ```python
  >>> list(itertools.accumulate([1, 2, 3, 4], func=operator.add))
  [1, 3, 6, 10]
  ```

#### **组合生成器**

- product(\*iterables [,repeat=1])

  创建一个迭代器，生成多个可迭代对象的笛卡尔积，跟嵌套for循环等价，repeat用于指定重复生成序列的次数。

  ```python
  for value in accumulate(['a', 'b', 'c']):
      print(value)
  输出：
  a
  ab
  abc
  ```

- permutations(iterable, [, r]])

  创建一个迭代器，生成iterable中元素的排列，r用于指定生成排列的长度，如果省略r，生成的序列长度与iterable长度相同。

  ```python
  for value in permutations([1, 2, 3]):
      print(list(value))
  输出：
  [1, 2, 3]
  [1, 3, 2]
  [2, 1, 3]
  [2, 3, 1]
  [3, 1, 2]
  [3, 2, 1]
  ```

- combinations(iterable, r)

  创建一个迭代器，生成iterable中所有长度为r的序列。序列中元素的排列顺序和iterable中元素顺序相同，序列中元素不重复

  ```python
  for value in combinations([1, 2, 3], 2):
      print(list(value))
  输出：
  [1, 2]
  [1, 3]
  [2, 3]
  ```

- combinations_with_replacement(iterable, [, r])

  功能和`combinations`类似，不过该函数生成的序列中允许元素重复。

  ```python
  for value in combinations_with_replacement([1, 2, 3], 2):
      print(list(value))
  输出：
  [1, 1]
  [1, 2]
  [1, 3]
  [2, 2]
  [2, 3]
  [3, 3]
  ```

  ​