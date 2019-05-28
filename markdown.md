
# 1. 斜体和粗体

*斜体*

**粗体**

# 2. 分级标题

```
一级标题
===

二级标题
---
```

# 3. 链接

[百度链接](www.baidu.com)

# 4. 无序列表

* 无序列表1
+ 无序列表2
- 无序列表3

* 无序列表4
    + 无序列表5
        - 无序列表6
        - 无序列表7
    + 无序列表8
* 无序列表9

# 5. 有序列表

1. 有序列表1
2. 有序列表2
3. 有序列表3

# 6. 文字引用

> 离离原上草，
> 一岁一枯荣。
> 野火烧不尽，
> 春风吹又生。

# 7. 行内代码块

`行内代码`

# 8. 代码块

    代码块

# 9. 图像

![图像](https://www.zybuluo.com/static/img/logo.png)

# 10. 内容目录

[TOC]

# 11. 标签分类

Tags: 1 2 3

# 12. 删除线

~~删除线~~

# 13. 注脚

注脚[^footnote]

注脚[^footnote2]

# 14. LaTeX 公式

$E=mc^2$

$$\sum_{i=1}^n a_i=0$$

$$f(x_1,x_x,\ldots,x_n) = x_1^2 + x_2^2 + \cdots + x_n^2 $$

$$\sum^{j-1}_{k=0}{\widehat{\gamma}_{kj} z_k}$$

# 15. 加强代码块

```
$ sudo apt-get install vim-gnome
```

```python
@requires_authorization
def somefunc(param1='', param2=0):
    '''A docstring'''
    if param1 > param2: # interesting
        print 'Greater'
    return (param2 - param1 + 1) or None

class SomeClass:
    pass

>>> message = '''interpreter
... prompt'''
```

``` javascript
/**
* nth element in the fibonacci series.
* @param n >= 0
* @return the nth element, >= 0.
*/
function fib(n) {
  var a = 1, b = 1;
  var tmp;
  while (--n >= 0) {
    tmp = a;
    a += b;
    b = tmp;
  }
  return a;
}

document.write(fib(10));
```

# 16. 流程图

```flow
st=>start: Start:>https://www.zybuluo.com
io=>inputoutput: verification
op=>operation: Your Operation
cond=>condition: Yes or No?
sub=>subroutine: Your Subroutine
e=>end

st->io->op->cond
cond(yes)->e
cond(no)->sub->io
```

# 12. 表格支持

| 项目        | 价格   |  数量  |
| --------   | -------:  | :------:  |
| 计算机     | \$1600 |   5     |
| 手机        |   \$12   |   12   |
| 管线        |    \$1    |  234  |


    <i class="icon-weibo"></i>
