---
layout: post
title: Python装饰器用法示例 - 检查函数的参数类型
category: 技术
tags: []
keywords: Python,Decorator
description:
---

关于python decorator, 这里就不累述了，推荐左耳朵耗子的博文[Python修饰器的函数式编程](http://coolshell.cn/articles/11265.html)。下面主要分享自己在工作中遇到的一个案例。

## 问题背景

最近一段时间在做图像类应用的测试，其中涉及到图像算法采用opencv python bindings来实现，比如下面这几个函数接口。

```python
def get_perceptual_hash(source):
    """计算图像的Perceptual Hash(http://en.wikipedia.org/wiki/Perceptual_hashing)
    """
    # 省略若干代码

def diff_image(base_image, compare_image, diff_image_prefix):
    """生成base_image, compare_image的差异图片
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

这样的代码显得繁琐，也不容易维护。可以写个【装饰器函数】，接受【被装饰函数】的参数名作为【装饰器函数】的参数，在真正的算法逻辑执行前完成参数的类型转换。嗯，用代码描述是这样的。
```python
@convert_arguments_to_unicode(['param_2'])
def func(param_1, param_2):
  pass
```
上面的例子中，函数func的逻辑真正执行前，param_2已经完成了类型转换。


## 解决方案

问题的难点，怎么从被装饰函数的参数列表(*args, **kwargs)中，找到指定名字的参数并完成修改。在python语言中，函数也是对象，那么这个对象有哪些属性可以来利用呢？经过一番搜索，在官网上找到了答案。

[Data model](https://docs.python.org/2/reference/datamodel.html)
- co_argcount is the number of positional arguments
- co_varnames is a tuple containing the names of the local variables (starting with the argument names)

```python
>>> def func(param_1, param_2):
...     print "param_1 is ", param_1
...     print "param_2 is ", param_2
...
>>> func.func_code.co_names
2
>>> func.func_code.co_varnames
('param_1', 'param_2')
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
