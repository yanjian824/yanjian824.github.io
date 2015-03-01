---
layout: post
title: 从Android hprof文件中导出bmp图像文件
category: 技术
tags: []
keywords: Android,Bitmap,Hprof
description:
---

### 问题背景

做终端APP内存测试时，常常会用DDMS dump出java hprof文件，利用MAT或者jhat这样的工具去分析被测APP的内存使用是否合理。

尤其IM，图片类应用，Top Retained Heap往往少不了Bitmap的身影。再深入一点思考，BitmapFactory.options设置了inSampleSize之后，图片缩小了多少倍，不同的配色方案(ARGB_8888, RGB_565)对图片大小有什么样的影响。

![](..\..\..\public\img\mat-bitmap.png)

稍有经验的同学可能会先通过引用关系找到目标instance，然后在MAT左侧的Inspector Tab的Attributes栏目中去观察mBuffer，mHeight, mWidth的大小。

![](..\..\..\public\img\mat-inspector.png)

面对二进制的数据总有点陌生的感觉，如何能把mBuffer转换成图像文件那该多好。

### 手动方法

首先记录下Inspector中Bitamp的mWidth，mHeight属性值。

接着，在dominator_tree视图中找到目标Bitamp，弹出右键菜单，选择【Copy】-> 【Save Value to File】。

![](..\..\..\public\img\mat-save-value-to-file.png)

用保存好的文件size/(mWidth*mHeight)来判断是RGBA_8888，还是RGB_565。

下面ffmpeg就排上用场了，注意RGBA_8888,RGB_565分别对应ffmpeg参数pix_fmt的rgba, rgb565。

```shell
ffmpeg -vcodec rawvideo -f rawvideo -pix_fmt <rgba|rgb565> -s <mWidth>x<mHeight> -i <raw_file> -f image2 -vcodec bmp <bmp_file>
```

运行结果如下，

```shell
ffmpeg -vcodec rawvideo -f rawvideo -pix_fmt rgba -s 120*16
0 -i 120x160.rgb -f image2 -vcodec bmp 120x160.bmp
ffmpeg version N-52045-g694fa00 Copyright (c) 2000-2013 the FFmpeg developers
  built on Apr 12 2013 16:54:51 with gcc 4.8.0 (GCC)
  configuration: --enable-gpl --enable-version3 --disable-w32threads --enable-av
isynth --enable-bzlib --enable-fontconfig --enable-frei0r --enable-gnutls --enab
le-iconv --enable-libass --enable-libbluray --enable-libcaca --enable-libfreetyp
e --enable-libgsm --enable-libilbc --enable-libmp3lame --enable-libopencore-amrn
b --enable-libopencore-amrwb --enable-libopenjpeg --enable-libopus --enable-libr
tmp --enable-libschroedinger --enable-libsoxr --enable-libspeex --enable-libtheo
ra --enable-libtwolame --enable-libvo-aacenc --enable-libvo-amrwbenc --enable-li
bvorbis --enable-libvpx --enable-libx264 --enable-libxavs --enable-libxvid --ena
ble-zlib
  libavutil      52. 26.100 / 52. 26.100
  libavcodec     55.  2.100 / 55.  2.100
  libavformat    55.  2.100 / 55.  2.100
  libavdevice    55.  0.100 / 55.  0.100
  libavfilter     3. 53.101 /  3. 53.101
  libswscale      2.  2.100 /  2.  2.100
  libswresample   0. 17.102 /  0. 17.102
  libpostproc    52.  3.100 / 52.  3.100
[rawvideo @ 024e9980] Estimating duration from bitrate, this may be inaccurate
Input #0, rawvideo, from '120x160.rgb':
  Duration: 00:00:00.04, start: 0.000000, bitrate: 15360 kb/s
    Stream #0:0: Video: rawvideo (RGBA / 0x41424752), rgba, 120x160, 15360 kb/s,
 25 tbr, 25 tbn, 25 tbc
File '120x160.bmp' already exists. Overwrite ? [y/N] y
Output #0, image2, to '120x160.bmp':
  Metadata:
    encoder         : Lavf55.2.100
    Stream #0:0: Video: bmp, bgra, 120x160, q=2-31, 200 kb/s, 90k tbn, 25 tbc
Stream mapping:
  Stream #0:0 -> #0:0 (rawvideo -> bmp)
Press [q] to stop, [?] for help
frame=    1 fps=0.0 q=-1.0 Lsize=N/A time=00:00:00.04 bitrate=N/A
video:75kB audio:0kB subtitle:0 global headers:0kB muxing overhead -100.028626%
```

来看看转换好的bmp图片吧。

![](..\..\..\public\img\bmp-120-160.bmp)

### 自动导出

作为一个懒惰的工程师，当然希望能够自动地从hprof文件中导出所有的bitmap。
这里我直接借鉴了jhat代码中解析hprof的部分，先把每个Bitamp的mBuffer中的二进制值分别写进单独的文件。

```java
Snapshot model = null;
try {
	model = com.sun.tools.hat.internal.parser.Reader.readFile(fileName, true, 0);
} catch (IOException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}

model.resolve(true);

JavaClass bitmapClass = model.findClass("android.graphics.Bitmap");
Enumeration instances = bitmapClass.getInstances(false);
while (instances.hasMoreElements()) {

	JavaObject instance = (JavaObject) instances.nextElement();

  final JavaThing[] things = instance.getFields();
  final JavaField[] fields = instance.getClazz().getFieldsForInstance();

  byte[] bytes = {0x0};
  String height = "";
  String width = "";

  for (int i = 0; i < fields.length; i++) {
  	String fieldName = fields[i].getName();
    if (fieldName.equals("mBuffer")) {
      if (things[i].getClass().getSimpleName().equals("JavaValueArray")) {
        bytes = ((JavaValueArray) things[i]).getRawBytes();
      } else {
        continue;
      }
    } else if (fieldName.equals("mHeight")) {
      height = things[i].toString();
    } else if (fieldName.equals("mWidth")) {
      width = things[i].toString();
    }
  }

  int bits = bytes.length / Integer.parseInt(width) / Integer.parseInt(height) * 8;
  File folder = new File(output);
  File rgbFile = new File(folder, instance.toString() + "_" + width + "x" + height + "_" + bits + ".rgb");

  try {
    FileOutputStream out = new FileOutputStream (rgbFile);
    out.write(bytes);
    out.close();
  } catch (IOException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
  }

}
```
