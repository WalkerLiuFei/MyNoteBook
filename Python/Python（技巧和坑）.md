# Python（技巧和坑）

## 知识点

1. **python 的 from ...import 和 import 的区别**
 ```python
from urllib import request
# access request directly.
mine = request()

import urllib.request
# used as urllib.request
mine = urllib.request()
 ```

















[Python中几个内置函数](http://devopstarter.info/pythonkai-fa-zhi-mapreduce/)

```python
#两个list，取(x - y) + (y - x)
x=[{'a': 1, 'b': 2}, {'c': 3}, {'d': 4}]  
y=[{'a': 1}, {'c': 3}, {'e': 5}]

filter(lambda z: (x+y).count(z)<2, (x+y))  
```

```python
#flatten out nested sublist
#result: [ 1, 2, 3, 4, 5 ]
import operator  
reduce( operator.concat, [ [ 1, 2 ], [ 3, 4 ], [ ], [ 5 ] ], [ ] )  

```

```python
#多项式求和
import operator  
def evaluate (a, x):  
       xi = map( lambda i: x**i, range( 0, len(a)))
       axi = map(operator.mul, a, xi)
       return reduce( operator.add, axi, 0 )
```


```python
#数据库SQL
reduce( max, map( Camera.pixels, filter(  
        lambda c: c.brand() == "Nikon", cameras ) ) )

#maybe equals
SELECT max(pixels)  
FROM cameras  
WHERE brand = “Nikon”

#There.
#cameras is a sequence
#where clause is a filter
#pixels is a map
#max is a reduce
```
```python
#一行并发
import urllib2  
from multiprocessing.dummy import Pool as ThreadPool

urls = [  
           'http://www.python.org',
           'http://www.google.com',
           'http://www.baidu.com',
           'http://www.python.org/community/',
           'http://www.saltstack.com'
           ]

#pool = ThreadPool()
pool = ThreadPool(4) # Sets the pool size to 4

result = pool.map(urllib2.urlopen, urls)  
pool.close()  
pool.join()  
```

**原生的通过set list parameters 进行 in (x,x,x)的查询**
```python
placeholders= ', '.join(['%s']*len(article_ids))  # "%s, %s, %s, ... %s"

query = 'SELECT name FROM table WHERE article_id IN ({})'.format(placeholders)

cursor.execute(query, tuple(article_ids))
```
2. Python3.6 的解码有问题，再安装一些依赖时报编码错误，解决办法： https://github.com/pypa/pip/issues/4251