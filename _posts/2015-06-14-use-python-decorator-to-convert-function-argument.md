---
layout: post
title: 装饰器用法示例 - 检查函数的参数
category: 技术
tags: []
keywords: Python,Decorator
description:
---

关于python decorator, 这里就不累述了，推荐左耳朵耗子的博文[Python修饰器的函数式编程](http://coolshell.cn/articles/11265.html)。下面分享自己在具体工作中遇到的一个例子。

## 问题背景

最近一段时间，在做图像类应用的测试，其中涉及到图像算法采用opencv python bindings来实现，比如下面这几个函数接口。

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


团队内部的测试代码默认采用utf8编码的字符串，或者是unicode。在locale为中文情况下，opencv python bindings的API只接受gbk格式的字符串。
那么问题就来了，我需要在读取图像文件做算法之前，对source/base_image/compare_image这些参数做一个的类型转换，转成gbk编码。

```python
def get_perceptual_hash(srouce):  
    if isinstance(source, unicode):
        source = source.encode('gbk')
    elif isinstance(srouce, str):
        source = source.decode('utf8').encode('gbk')
    else:
        raise Exception('')

    # 省略若干代码

def diff_image(base_image, compare_image, compare_image, diff_image_prefix):
    for image in [base_image, compare_image]:
        if isinstance(image, unicode):
            image = image.encode('gbk')
        elif isinstance(image, str):
            image = image.decode('utf8').encode('gbk')
        else:
            raise Exception('')
    # 省略若干代码
```
