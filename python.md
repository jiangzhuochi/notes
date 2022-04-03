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

## warnings 模块

出现了一些需要让用户知道的问题，但又不想停止程序，这时候我们可以使用警告

首先导入警告模块：`import warnings`，在需要的地方，我们使用 `warnings` 中的 `warn` 函数：

```python
warn(msg, WarningType = UserWarning)
def month_warning(m):
    if not 1<= m <= 12:
        msg = "month (%d) is not between 1 and 12" % m
        warnings.warn(msg, RuntimeWarning)
month_warning(13)
```

有时候我们想要忽略特定类型的警告，可以使用 `warnings` 的 `filterwarnings` 函数：`filterwarnings(action, category)`

将 `action` 设置为 `'ignore'` 便可以忽略特定类型的警告：

```python
warnings.filterwarnings(action = 'ignore', category = RuntimeWarning)
month_warning(13)
```

参阅[警告 - 警告控制](https://www.docs4dev.com/docs/zh/python/3.7.2rc1/all/library-warnings.html)

## array 元素的比对

我们可以直接使用运算符进行比较，比如判断数组中元素是否大于某个数：

```python
from numpy import array
a = array([[ 0, 1, 2, 3, 4, 5],
           [10,11,12,13,14,15],
           [20,21,22,23,24,25],
           [30,31,32,33,34,35]])

a > 10
```

输出

```python
array([[False, False, False, False, False, False],
       [False,  True,  True,  True,  True,  True],
       [ True,  True,  True,  True,  True,  True],
       [ True,  True,  True,  True,  True,  True]])
```

判断数组中元素大于10的元素赋值为 -10 

```python
a[a > 10] = -10
a
```

输出

```python
array([[  0,   1,   2,   3,   4,   5],
       [ 10, -10, -10, -10, -10, -10],
       [-10, -10, -10, -10, -10, -10],
       [-10, -10, -10, -10, -10, -10]])
```

但是当数组元素较多时，查看输出结果便变得很麻烦，这时我们可以使用 `all()` 方法，直接比对矩阵的所有对应的元素是否满足条件。假如判断某个区间的值是否全是大于 20:

```python
from numpy import array
a = array([[ 0, 1, 2, 3, 4, 5],
           [10,11,12,13,14,15],
           [20,21,22,23,24,25],
           [30,31,32,33,34,35]])

a[1:3,1:3]
# array([[11, 12],
#        [21, 22]])

np.all(a[1:3, 1:3] > 20)
# False
```

使用 `any()` 来判断数组某个区间的元素是否存在大于 20 的元素:

```python
np.any(a[1:3,1:3] > 20)

True
```

## list 与 str 切片的坑

```python
b = [1, 2, 3]
a = b
a[0] -= 1
print(a)
print(b)

[0, 2, 3]
[0, 2, 3]
```

用等号 a 就是 b，b 就是 a！
似乎它们的地址是一样的？
注意可变对象与不可变对象
参见 [Python 函数](https://www.runoob.com/python/python-functions.html)

不要用等号复制列表，使用 copy 方法，或者 `b = copy.deepcopy(a)` 更安全

```python
b = [1, 2, 3]
a = b.copy()
a[0] -= 1
print(a)
print(b)

[0, 2, 3]
[1, 2, 3]
```

字符串切片不对原来的字符串产生任何影响，而是生成一个新的字符串

```python
>>> b = '123123'
>>> b[1:2]
'2'
>>> b
'123123'
>>> a = '12312312'
>>> a = a[1:2]
>>> a
'2'
```

## yield 迭代器

```python
def fibonacci(n):
    a, b = 0, 1
    for i in range(n):
        yield a
        a, b = b, a+b
        # 右边的 b, a+b 会返回一个tuple 
        # 然后这个左边的 a, b 会分别赋值为这个tuple里的第一个和第二个

f = fibonacci(10)
print(f)
print(list(f))
for i in f:
    print(i)


<generator object fibonacci at 0x00000224285B6CC8>
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

yield 返回迭代器。迭代器没有回头路，上面代码 `list` 方法已将迭代器遍历，因此该迭代器 f 失效，for 语句不会打印出任何数；除非用 `g = fibonacci(10)`，用 for 遍历 g

## 全排列的迭代算法

```python
def permutations(arr, pos, end):
 
    if pos == end:
        print(arr)
 
    else:
        for i in range(pos, end):
 
            arr[i], arr[pos] = arr[pos], arr[i] # 交换
            permutations(arr, pos+1, end)
            arr[i], arr[pos] = arr[pos], arr[i] # 复原
 
arr = ["a", "b", "c"]
permutations(arr, 0, 3)
'''
当指针指向第一个元素a时，它可以是其本身a，还可以和b，c进行交换，故有3种可能
当第一个元素a确定以后，指针移向第二位置，第二个位置可以和其本身b及其后的元素c进行交换,又可以形成两种排列
当指针指向第三个元素c的时候，这个时候其后没有元素了，此时，则确定了一组排列，输出。
每次输出后要把数组恢复为原来的样子。

简单来说，它的思想即为，确定第1位，对n-1位进行全排列，确定第二位，对n-2位进行全排列
'''
```

可以看到，参数 `pos` 和 `end` 控制了他们之间的位置的全排列（包括 `pos` 而不包括 `end`）

## Python3 中的编码

首先明确，Python3 内部只有 Unicode 和其他

- 显示的字符（str 类型）背后全是 Unicode，它们可以看作一回事
- Unicode 就是显示的字符（或 \u 开头），除此之外显示的都是 \x 开头
- 解码字节串，编码字符串（decode bytes, encode str）DBES
- str 有 encode 方法，指定 encoding='utf-8' 参数，表示从 Unicode 编码到 utf-8（bytes）
- bytes 有 decode 方法，指定 encoding='gbk' 参数，表示从 gbk 解码到 Unicode

**例子一**

假如从网上爬取了一个 utf-8 网页，它在计算机里的存储形式就是 utf-8，而 open() 似乎会默认将以 gbk 格式读取，就出错了。例如

`UnicodeDecodeError: 'gbk' codec can't decode byte 0xad in position 21: illegal multibyte sequence`

于是在 open 时，指定参数 encoding='utf-8' 即可

**例子二**

将一个 utf-8 存储的中文文章转换成 gbk 存储

先用 open 只读模式读入到 Python，由于默认是 encoding='gbk'，需要指定 'utf-8'。相当于告诉 open：嘿，我这个文件是 utf-8 写的，你要用（Unicode <==> utf-8）双射表阅读这个文件，将 utf-8 翻译成 Unicode 记下来！

接着用写模式直接往另一个文件写，由于默认是 encoding='gbk'，因此不需要指定。相当于告诉 open：嘿，你要用（Unicode <==> gbk）双射表将你记下来的 Unicode 翻译成 gbk 给我送到文档里！

```python
with open('test1.txt', encoding='utf-8') as f:
    txt = f.read()
    print(txt)

with open('output.txt', mode='w') as g:
    aa = g.write(txt)
```

注意，官方文档中 open 的 encoding 参数默认为 None，但实际效果似乎是 gbk，和系统有关？

更深入的编码知识，参考

https://zhuanlan.zhihu.com/p/37359861

http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html

## 类中 \_\_str\_\_ 与 print

在类里定义

```python
    def __str__(self):
        if self.isEmpty():
            return '\nempty stack'

        item_max_len = max([len(str(i)) for i in self.items])
        template = '{:^{item_max_len}}'
        str_ = '\n'
        for item in reversed(self.items):
            str_ = str_ + '| '  + template.format(item, item_max_len=item_max_len) + ' |\n'
            str_ = str_ + (item_max_len+4) * '-'
            if item != self.items[0]:
                str_ = str_ + '\n'
        return str_
```

能够在 print() 时调用，打印 \_\_str\_\_ 返回的内容，上面就是我给 pythonds 库的 Stack 类加的，能够打印出栈里现有的元素

```python
from pythonds import Stack
s = Stack()
s.push(13)
s.push(234)
s.push('asdf')
print(s)
s.pop()
print(s)
```

输出为

```
| asdf |
--------
| 234  |
--------
|  13  |
--------

| 234 |
-------
| 13  |
-------
```

## enumerate 带下标迭代器

enumerate 还支持传入参数。比如在某些场景当中，我们希望下标从1开始，而不再是0开始，我们可以额外多传入一个参数实现这点：

```python
for i, item in enumerate(items, 1):
```

如果我们迭代的是一个多元组数组，我们需要注意要将 index 和 value 区分开。举个例子：

```
data = [(1, 3), (2, 1), (3, 3)]
```

在不用 enumerate 的时候，我们有两种迭代方式，这两种都可以运行

```python
for x, y in data:

for (x, y) in data:
```

但是如果我们使用 enumerate 的话，由于引入了一个 index，我们必须要做区分

```python
for i, x, y in enumerate(data):
```

会报错，所以我们可以按下面的方法：

```python
for i, (x, y) in enumerate(data):

for i, tuple_ in enumerate(data):
```

这样，后者的 `tuple_` 是元组
