---
layout: post
title: Python正则表达式示例
category: 技术
tags: []
keywords: Python,正则表达式
description:
---

## 后向肯定断言 (?<=pattern)
工程代码中有一个kEnableWMCTimeLog的编译宏，若kEnableWMCTimeLog值为1，编译出的build可以输出性能日志。出于种种原因，开发同学不想在主干代码上打开这个宏。于是，我们新建了一个CI任务，在拉取工程代码后，工程真正编译前，新加一个task, 这个task能够利用正则表达式修改代码。简单来讲，就是希望能够匹配到kEnableWMCTimeLog后面的这个0，把0修改成1，然后保存代码，但是不匹配kEnableWMCTimeLog本身。

```c
// 工程代码
#define kEnableWMCTimeLog  0
```

```python
# 正则表达式
pattern = re.compile(u"(?<=\skEnableWMCTimeLog\s)\s*0")
new_code = re.sub(pattern, supplement, orignal_code)
```
触类旁通，
- 后向否定断言 (?<!pattern)
- 前向肯定断言 (?=pattern)
- 前向否定断言 (?!pattern)

## Named Group - 给匹配的分组起名

测试报告是html格式的，页面中所以img元素的src属性值即图片的网络地址

```html
<img width="180" src="http://xxx.yyy.zzz/static/CosmeticsHairDyeingTest.1.jpg">
```

如果想要匹配所有的图片地址，有以下正则表达式

```python
pattern = re.compile(u'(?<=src=")(?P<url>.*)(?=")')
for match in re.finditer(pattern, original_content):
	print "match.group(0) %s" % match.group(0)
	print "match.group('url') %s" %  match.group('url')
	print "*" * 60
```

程序输出如下

```shell
match.group(0) http://xxx.yyy.zzz/static/CosmeticsHairDyeingTest.1.jpg
match.group('url') http://xxx.yyy.zzz/static/CosmeticsHairDyeingTest.1.jpg
************************************************************
match.group(0) http://xxx.yyy.zzz/static/CosmeticsHairDyeingTest.baseline.jpg
match.group('url') http://xxx.yyy.zzz/static/CosmeticsHairDyeingTest.baseline.jpg
************************************************************
match.group(0) CosmeticsHairDyeingTest.diff.1.2.jpg
match.group('url') http://xxx.yyy.zzz/static/CosmeticsHairDyeingTest.diff.1.2.jpg
************************************************************
```

match.group等于match.group('url'), ```(?P<url>.*)```给匹配的内容起了个别名url一样，可以通过这个别名在MatchObject.group()引用被匹配的内容。

## re.sub(pattern, repl, string, count=0, flags=0)
上面的案例中简单地运用re.sub来实现文本替换，有的时候替换的string与被替换的string有一定的关系，有的时候有多处需要被替换成不一样的string，这个时候repl就派上用场了。官网上讲，

> repl can be a string or a function
> If repl is a function, it is called for every non-overlapping occurrence of pattern. The function takes a single match object argument, and returns the replacement string

接着Named Group中的例子往下讲，HTML格式的测试报告需要通过SMTP协议发送给项目组，公司的安全策略邮件中的HTTP链接都会被block掉。好在服务器本地有图片文件，可以为每一个图片创建一个MIMEImage，attach到MIMEMultipart再发出去。

```python
msgRoot = MIMEMultipart('related')

fp = open(<image_path>, 'rb')  
msgImage = MIMEImage(fp.read())  
fp.close()
msgImage.add_header('Content-ID', 'image_1')

msgRoot.attach(msgImage)  
```
但是，发出去的HTML文档需要将```<img src="HTTP地址">```替换成```<img src="cid:image_1">```，其中第一个img元素的src为"cid:image_1"，第二个img元素的src为"cid:image_2"，以此类推。这样接收到测试报告邮件的同事才能够看到图片。

```python
num = 1
def repl_func(matched):
  global num
  replacement = "cid:image_%d" % num
  num += 1
  return replacement

new_content = re.sub(pattern, repl_func, original_content)
```
