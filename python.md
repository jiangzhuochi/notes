# Python tips

[官方文档教程](https://docs.python.org/zh-cn/3/tutorial/index.html)

## 根据字典的值去查找键

```python
def find_key(d, val):
    """
    根据值找出键值对，返回包含键值元组的列表；若查找的值不在字典中，返回None。
    第一个参数填被查找的字典，第二个参数填要查找的值。
    """
    kv = []
    for k, v in d.items():
        if v == val:
            kv.append((k, v))

    if kv:
        return kv
    else:
        return None
```

注意，字典不可以直接 `for k, v in d:`

## 合并两个字典

```python
x = {}
y = {}
z = {**x, **y}
```

如果 y 里面有和 x 重复的 key，则后面的值会覆盖前面的值，即 y 覆盖 x
这种只在 Python 3.5 以上有用

## for 循环中访问元素的下标

```python
scores = []
for index, s in enumerate(scores):
    pass
```

## sorted 给字典里的值排序

```python
scores = {}
s = sorted(scores.items(), key = lambda item: item[1], reverse = True)
new_s = {k: v for k, v in s}
```

第二条语句生成的是排好序的元组的列表，见 [Python sorted() 函数](https://www.runoob.com/python/python-func-sorted.html)
第三条语句是将元组的列表变为字典的方法，当然，字典是无序的

## 集合的交并差

```python
a = {1, 2, 3, 4}
b = {2, 4, 5}
x = a.intersection(b)
y = a.union(b)
z = a.difference(b)
print(x, y, z) # {2, 4} {1, 2, 3, 4, 5} {1, 3}
```

```

```

## in not in 判断包含关系

```python
>>> a = 'this is maishu coding'
>>> b = 'maishu'
>>> str.__contains__(a, b)
True
>>> str.__contains__(b, a)
False
```

判断前一个字符串是否包含后一个字符串，不建议使用这种方法，因为这是私有的

使用如下方法

```python
>>> b in a
True
>>> a in b
False
>>> b not in a
False
>>> a not in b
True
```

也可以用来判断元素是否在列表中

## append 和 extend 的区别

```python
>>> a = [1, 2, 3]
>>> b = [4, 5]
>>> a.append(b)
>>> a
[1, 2, 3, [4, 5]]

>>> a = [1, 2, 3]
>>> a.extend(b)
>>> a
[1, 2, 3, 4, 5]

>>> a = [1, 2, 3]
>>> a + b
[1, 2, 3, 4, 5]
```

对元组同样适用

## reversed 反转

reversed 函数返回一个反转的迭代器。

以下是 reversed 的语法

`reversed(seq)`

seq -- 要转换的序列，可以是 tuple, string, list 或 range

返回一个反转的迭代器，并不改变 seq

## map 映射

其用法为：
`map(aFun, aSeq)`，返回迭代类型，可以用 list 方法变为列表
将函数 aFun 应用到序列 aSeq 上的每一个元素上，返回一个迭代器，不管这个序列原来是什么类型
事实上，根据函数参数的多少，map 可以接受多组序列，将其对应的元素作为参数传入函数：

```python
def divid(a, b):
    """
    除法
    :param a: number 被除数
    :param b: number 除数
    :return: 商和余数
    """
    quotient = a // b
    remainder = a % b
    return quotient, remainder

a = (10, 6, 7)
b = [2, 5, 3]
print(list(map(divid,a,b)))
```

输出 `[(5, 0), (1, 1), (2, 1)]`

## reduce 归约

依次调用函数，直到最后只剩下一个结果为止
比如说我们有一个数组 [a, b, c, d] 和一个函数 f，我们计算 reduce(f, [a, b, c, d]) 其实就等价于 f(f(f(a, b), c), d)。和 map 不同的是，reduce 最后得到一个结果，而不是一个迭代器或者是 list

```python
from functools import reduce

def f(a, b):
    return a + b
    
print(reduce(f, [1, 2, 3, 4]))
```

最终得到的结果当然是 10，同样，我们也可以将reduce中的方法定义成匿名函数，一样不影响最终的结果：`print(reduce(lambda x, y: x + y, [1, 2, 3, 4]))`

## filter 过滤

编写一个函数来判断元素是否合法。通过调用 filter，会自动将这个函数应用到容器当中所有的元素上，最后只会保留运行结果是 True 的元素，而过滤掉那些是 False 的元素

要保留 list 当中的奇数而过滤掉偶数 `arr = [1, 3, 2, 4, 5, 8]`
`[i for i in arr if i % 2 > 0 ]`

而使用 filter 会非常方便：`list(filter(lambda x: x % 2 > 0, arr))`

## compress 过滤

在 itertools 当中有一个方法叫做 compress，通过 compress 我们可以实现根据一个序列的条件过滤另一个序列

有两个数组：

```
student = ['xiaoming', 'xiaohong', 'xiaoli', 'emily']
scores = [60, 70, 80, 40]
```

我们想要获取所有考试及格的同学的 list

```python
from itemtools import compress

>>> passed = [i > 60 for i in scores]
>>> print(passed)
[False, True, True, False]

>>> list(compress(student, passed))
['xiaohong', 'xiaoli']
```

需要注意的是 filter 和 compress 返回的都是一个迭代器，我们要获取它们的值，需要手动转换成 list
