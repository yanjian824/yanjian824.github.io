---
layout: post
title: Python装饰器用法示例 - 检查函数的参数类型
category: 技术
tags: []
keywords: Python,Decorator
description:
---

关于python decorator, 这里就不累述了，推荐左耳朵耗子的博文[Python修饰器的函数式编程](http://coolshell.cn/articles/11265.html)。
下面主要分享自己在工作中遇到的一个案例。

## 问题背景

最近一段时间在做图像类应用的测试，其中涉及到图像算法采用opencv python bindings来实现，比如下面这几个函数接口。

```python
def get_perceptual_hash(source):
    """计算图像的Perceptual Hash(http://en.wikipedia.org/wiki/Perceptual_hashing)
    """
    # 省略若干代码

def diff_image(base_image, compare_image, diff_image_prefix):
    """生成base_image, compare_image的差异文件
    """
    # 省略若干代码
```

团队内部的测试代码默认采用utf8编码的字符串，或者是unicode。在locale为中文情况下，opencv python bindings的API只接受gbk格式的文件路径字符串。那么问题就来了，我需要在读取图像文件做算法之前，对source/base_image/compare_image这些参数一个个做类型转换转成gbk编码？

```python
# 非装饰器版本
def get_perceptual_hash(srouce):  

    if isinstance(source, unicode):
        source = source.encode('gbk')
    elif isinstance(srouce, str):
        source = source.decode('utf8').encode('gbk')
    else:
        raise Exception('')

    # 省略若干代码

def diff_image(base_image, compare_image, diff_image_prefix):

    for image in [base_image, compare_image]:
        if isinstance(image, unicode):
            image = image.encode('gbk')
        elif isinstance(image, str):
            image = image.decode('utf8').encode('gbk')
        else:
            raise Exception('')
    # 省略若干代码
```

这样显然繁琐了点，也不容易维护。我们完全可以写个装饰器，在真正的逻辑执行前完成参数的类型转换。

## 解决方案

首先，在python里面函数也是对象，那来看看这个对象有些什么样的属性吧？

[Data model](https://docs.python.org/2/reference/datamodel.html)
- co_argcount is the number of positional arguments
- co_varnames is a tuple containing the names of the local variables (starting with the argument names)

```python
>>> def func(a, b):
...     print "a is ", a
...     print "b is ", b
...
>>> func.func_code.co_names
2
>>> func.func_code.co_varnames
('a', 'b')
```

理解了上面这段示例，就可以动手写装饰器函数了，暂时只处理函数的匿名参数*args

```python
def convert_arguments_to_unicode(arguments):

    def make_wrapper(func):

        def wrapper(*args, **kwargs):

            # 拿到被装饰函数的参数名列表
            code = func.func_code
            names = list(code.co_varnames[:code.co_argcount])

            # args类型是tuple, tuple是不可变对象
            args_in_list = list(args)

            # 装饰arguments
            for argument in arguments:
                num = names.index(argument)
                value = args[num]
                if isinstance(value, str):
                    value = value.decode('utf8').encode('gbk')
                elif isinstance(value, unicode):
                    value = value.encode('gbk')

                args_in_list[num] = value

            new_args = tuple(args_in_list)

            return func(*new_args, **kwargs)

        return wrapper

    return make_wrapper
```

再来看看装饰器版本的代码吧, 是不是简洁明了。

```python
# 装饰器版本
@convert_arguments_to_unicode(['source'])
def get_perceptual_hash(srouce):  
    # 省略若干代码

@convert_arguments_to_unicode(['base_image', 'compare_image'])
def diff_image(base_image, compare_image, diff_image_prefix):
    # 省略若干代码
```
