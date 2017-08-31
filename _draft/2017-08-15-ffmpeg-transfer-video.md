---
layout: post
title:  "用ffmpeg处理视频转换"
date: Tue, 15 Aug 2017 23:14:32 +0800
categories: ffmpeg
tags: ffmpeg video encode
---

ffmpeg是一个开源的集多媒体转码、格式转换、合并/分割视频等功能于一身的强大工具，很多GUI视频处理软件，甚至播放器背后都离不开ffmpeg的支持。最近看了些ffmpeg处理H.264编码的视频的文章，觉得收获挺多的，在此记录下ffmpeg处理H.264和Xvid编码的具体用法，以后可能随时用得到。

对于视频文件，现在常见的都是用一种容器（封装格式）封装上一个视频轨，一个或多个音频轨，有些视频文件还会封装一个或多个字幕轨。目前最为流行的封装格式主要是mp4和mkv，当然也有部分使用avi封装以及少量的rm/rmvb，wmv等，但这些都只是一种封装格式，里面的视频和音频的具体编码格式并不唯一。

说道编码格式，音频中的mp3大家早就被大家所熟知，现在还有AAC，AC3及DTS等音频编码格式也是很流行的。而视频的编码格式，rmvb文件使用的real 8/9/10、wmv文件使用的wmv9等目前在网络上流传的很少了，主流的影视作品目前都不再使用这些编码格式，现在最常见到的影视作品一般都是使用H.264（H.264还有一个名字叫做MPEG-4AVC）编码视频轨，当然也有使用divx/xvid编码视频轨的。

一般avi封装的文件大多使用divx/xvid视频编码加mp3/ac3音频编码，而mp4和mkv封装的文件更多使用H.264视频编码加aac/ac3/dts音频编码。

关于封装格式和编码格式的关系，打个比方：要把几张图片打包成一个压缩包，图片文件可以是jpg格式的，也可以是png或者gif格式的，这就是编码格式。可以打包成rar，也可以是zip或者7z等，而这个就是封装格式。所以说封装格式只是把不同的视频源，音频源及可能的字幕源打包在一起，并能正确识别其包含的各种不同内容，而编码格式才是真正压缩、存储视频、音频等多媒体内容的格式。

ffmpeg可以处理很多种封装格式和编码格式，本文主要想记录使用ffmpeg处理H.264视频编码和AAC音频编码的相关知识。下文会提到x264，这是H.264的开源实现，x264说的就是H.264。

## ffmpeg工具用法

ffmpeg是一个命令行工具包，包含一系列工具，常用的有：

- ffmpeg转码视频
- ffprobe查看视频文件详细的编码及封装信息
- ffplay是一个简单的播放器

ffmpeg基本的用法如下：

```bash
ffmpeg [options] [[infile options] -i infile]... {[outfile options] outfile}...
```

其中[options]用于指定ffmpeg全局选项，一般不用。[infile options]是针对输入视频文件指定某些参数（如跳过多长内容等），一般也不用。`-i infile`就是指定要转码的输入文件。[outfile options]指定输出文件的选项，主要就是通过这些输出文件选项来定义转码输出的具体格式，最后的outfile是输入文件名。

要注意的两点，一是输入文件的封装格式和编码格式都不用指定，ffmpeg会自动判断，主要是指定输出文件的详细参数来定制转码格式。二是输入文件参数和输出文件参数的位置要严格按照命令指示，输出参数必须是在`-i infile`之后及输出文件之前。

## 转码H.264/AAC

对于视频编码，有一个很重要的参数就是码率（单位是bps）表示每秒钟的文件大小（bit），在视频源、分辨率、帧率等都一致的情况下，高码率一般比地码率更加清晰，但是高码率带来的就是文件尺寸的增加（相同时长的视频，每秒的大小增加了，文件尺寸肯定也会增大）。在网上下载影视作品，同一部蓝光原盘转码且编码格式、分辨率等都一样的情况下，有些压制组释出的1080p电影会有几十GB，而有些的只有几GB大小，这之间的差异主要就是码率的不同。我们也能看出来，在播放器和显示设备都一样的前提下，同一部电影几十个G的文件观看起来会比几个G的文件质量高。

码率可以分为固定码率（CBR - Constant Bit Rate）和可变码率（VBR - Variable Bit Rate）。字面意思就可以理解，固定码率每秒钟的大小是固定，所以固定码率编码时知道视频文件的时长就能确定最终文件大小，而可变码率无法预测最终文件大小。可变码率在画面高速运动且画面色彩等变化较大时使用高码率，而在更多静态内容时使用低码率，这样在保证视频画面质量的同时，能尽可能减小视频文件体积。

### 恒定质量的可变码率编码

先给出我常用的H.264转码的命令：

```bash
# 真人影视
ffmpeg -i <infile> -c:v libx264 -crf 23 -preset veryslow -tune film -c:a aac -strict -2 -b:a 128k <outfile>
# 动画片
ffmpeg -i <infile> -c:v libx264 -crf 25 -preset veryslow -tune animation -c:a aac -strict -2 -b:a 128k <outfile>
```

其中的参数解释如下：

- `-c:v libx264`：指定输出文件的视频采用x264编码。
- `-crf 23`：crf是x264的一种保证恒定质量的动态编码方式，这个是最简便的x264编码用法，但是其效果非常好。这个参数表示码率选用固定质量模式（一种动态编码方式），取值范围在0~51之间，0表示最高质量（无损编码），51表示最低质量（画面几乎都是马赛克）。一般取值在18~28之间即可，默认值23。
- `-preset veryslow`：权衡压缩效率和编码速度，如果对转换时间要求不多，尽量选择处理的慢点。默认值是medium，其他可接受值参见x264手册。
- `tune film`：调整选项，在preset之上进一步针对视频做最佳化，一般真人电影选择film，动画片可以选择animation，这个选项可以不用。
- `-c:a aac`：指定输出文件的音频采用aac编码。
- `-strict -2`：因为ffmpeg使用aac编码还处于实验性阶段，该参数强制ffmpeg可以使用实验性的aac编码。
- `-b:a 128k`：指定音频编码的平均码率为128kbps。

采用crf这种编码方式，H.264会在自动计算动态码率，而且最终能在只用较小体积的情况下保证画面质量，在不需要精确控制编码参数的情况下，推荐使用。

### 两遍动态码率编码

H.264可以通过参数指定编码的码率，这种指定的码率其实是一个平均码率，H.264会自动计算所需动态码率并尽量保证整个视频文件的平均码率趋于所指定的码率。采用这种编码方式，可以教精确的控制最终文件，用到的命令如下：

```bash
ffmpeg -i <infile> -c:v libx264 -b:v 2500k -preset veryslow -tune film -c:a aac -strict -2 -b:a 128k <outfile>
```

这条命令中通过`-b:v 2500k`指定视频的平均码率为2500kbps，但是由于这种平均码率的实时计算无法考虑到整个视频文件的完整画面情况，为更加优化动态码率，一般采用两遍编码的方式，命令如下：

```bash
ffmpeg -i <infile> -c:v libx264 -b:v 2500k -preset veryslow -tune film -pass 1 --slow-firstpass -an /dev/null
ffmpeg -i <infile> -c:v libx264 -b:v 2500k -preset veryslow -tune film -pass 2 -c:a aac -strict -2 -b:a 128k <outfile>
```

第一条命令先把整个视频文件全部扫描一遍，生成一个记录视频全局码率变化的状态文件，一般在第一遍时将输出文件设置为空设备（/dev/null）表示不生成编码后的视频文件，仅仅统计全局码率状态。到第二条命令时做第二遍编码，这次编码会读取第一次编码生成的状态统计文件，建立一个最佳化的平均码率编码。

采用两遍编码（平均编码）方式，最主要的是要根据视频分辨率及期待的视频质量，指定一个比较合理的平均码率，这个需要一定的经验积累或者参考各大压片小组的作品。一般来说在保证视频源是蓝光原盘等真正高质量源的情况下，转码1080p的H.264视频平均码率分配到8000kbps以上质量会比较好，而720p的视频平均码率也应该在5000kbps以上。

也可以在第一遍时使用crf编码方式，第二遍再指定具体的平均码率。需要注意的是，采用两遍编码方式，实际上做了两次H.264转码，所以花费的时间也是最慢的。

### 固定码率编码

H.264并未提供提供选项支持固定码率，但是可以在指定平均码率的同时，限定最大和最小码率，从而间接实现类似“固定码率”编码的效果，命令如下：

```bash
ffmpeg -i <infile> -c:v libx264 -b:v 2500k -minrate 2500k -maxrate 2500k -preset veryslow -tune film -c:a aac -strict -2 -b:a 128k <outfile>
```

这里通过`-minrate`和`-maxrate`参数限制了H.264编码的最小和最大动态码率，且和平均码率相同，所以会以一个固定码率2500kbps来编码视频。虽然可以编码固定码率，但是最好不要这么做，这样编码的视频可能是质量最差的一种。

### 无损编码

开业使用`-qp 0`或者`-crf 0`的编码方式实现无损编码，但是提倡使用`-qp 0`实现无损编码，不过这样的编码方式码率可能会非常大，无损压缩的命令如下：

```bash
# 快速无损编码
ffmpeg -i <infile> -c:v libx264 -qp 0 -preset ultrafast -c:a aac -strict -2 -b:a 128k <outfile>
# 高压缩比无损编码
ffmpeg -i <infile> -c:v libx264 -qp 0 -preset veryslow -c:a aac -strict -2 -b:a 128k <outfile>
```

## 转码Xvid/mp3

Xvid视频编码加上mp3音频编码可以看作是前几年DVD的编码标准，现在已是H.264的天下，这里不再做过多解释，只给出命令参考：

```bash
ffmpeg -i <infile> -c:v mpeg4 -vtag xvid -q:v 3 -c:a libmp3lame -b:a 128k <outfile>
```

## 封装格式

前面也提到了视频的封装格式，常见主要就mp4、mkv、avi、flv、wmv、rm等，ffmpeg命令行指定输出文件时使用不同后缀，ffmpeg会自动使用该封装器封装视频、音频和字幕成为一个文件。

```bash
# 输出文件使用mp4封装
ffmpeg -i <infile> [options] out.mp4
# 输出文件使用mkv封装
ffmpeg -i <infile> [options] out.mkv
# 输出文件使用avi封装
ffmpeg -i <infile> [options] out.avi
# 不重编码，只转换封装格式（avi到mp4，其他类似）
ffmpeg -i in.avi -c copy out.mp4
```

需要注意的是，有些封装器不支持封装字幕轨或者多个音频轨等，这种情况下可能会失败。

另一种用法在于分离出单独的视频或者音频文件：

```bash
# 从infile中提取出视频轨，其中‘-an’表示输出文件中禁用音频轨
ffmpeg -i in.mp4 -c:v copy -an out.mp4
# 从infile中提取出音频轨，其中‘-vn’表示输出文件中禁用视频轨
ffmpeg -i in.avi -c:a copy -vn out.mp3
```

当然也可以在分离视频或音频时转码到不同编码格式。



## 高级用法

### 视频分辨率缩小

ffmpeg提供视频过滤器，可以实现多种过滤效果，通过scale过滤器可以缩小视频分辨率。

比如，你想把一个原本是`1920x1080`分辨率的视频转码成H.264并将分辨率缩小到`1280x720`，可以通过如下命令实现：

```bash
ffmpeg -i <infile> -c:v libx264 -crf 23 --preset veryslow -tune film -vf scale=-2:720 -c:a aac -strict -2 -b:a 128k <outfile>
```

参数`-vf scale=-2:720`的值冒号前面的值是指定输出视频的分辨率宽度，-2表示根据源视频的长宽比对照输出高度缩放宽度，而冒号后面的720指定输出视频的分辨率高度为720。也可以指定具体的宽度，而高度用-2表示等比例缩放。

### 旋转/翻转视频

可以通过transpose视频过滤器实现旋转视频的效果。

比如，要把一个视频顺时针旋转90度：

```bash
ffmpeg -i <infile> -c:v libx264 -crf 23 --preset veryslow -tune film -vf transpose=1 -c:a aac -strict -2 -b:a 128k <outfile>
```

transpose参数的取值可以为：

- 0	表示顺时针旋转90度并垂直翻转
- 1	表示顺时针旋转90度
- 2	表示逆时针旋转90度
- 3	表示逆时针旋转90度并垂直翻转

### 截取视频片段

比如，想把一个视频文件中从2分20秒开始到5分30秒之间的片段截取出来，可以使用命令：

```bash
# 只截取视频片段，保持源视频的编码格式
ffmpeg -i <infile> -c copy -ss 00:02:20 -to 00:05:30 <outfile>
# 截取视频片段，并用H.264重新编码
ffmpeg -i <infile> -c:v libx264 -crf 23 -preset veryslow -tune film -c:a aac -strict -2 -b:a 128k -ss 00:02:20 -to 00:05:30 <outfile>
```

### 合并多个视频片段为一整段

过滤器concat可以合并视频片段，具体可以参考[concat](https://trac.ffmpeg.org/wiki/Concatenate)，我目前只测试过mp4文件的合并，命令如下：

```bash
echo "file '/tmp/001.mp4'" > videolist.txt
echo "file '/tmp/002.mp4'" >> videolist.txt
echo "file '/tmp/003.mp4'" >> videolist.txt
ffmpeg -f concat -safe 0 -i videolist.txt -c copy out.mp4
```

该命令按照输入文件的先后顺序将001，002和003这三个视频文件合并成一个out.mp4文件，需要注意的是这里的输入文件要用绝对路径，如果文件路径是相对路径，参数`-safe 0`就不需要了。

Shell中可以用一条命令完成上述4条命令的任务：

```bash
ffmpeg -f concat -i <(for f in /tmp/001.mp4 /tmp/002.mp4 /tmp/003.mp4; do echo "file '$f'"; done) -c copy out.mp4
```

### 调整视频长宽比

有的视频长宽比不正常，看起来画片好像被强行挤窄或者拉宽，ffmpeg当然也可以调整到正常的长宽比，命令如下：

```bash
ffmpeg -i <infile> -c copy -aspect 16:9 <outfile>
```

`-c copy`表明不需要重新编码视频和音频，而`-aspect 16:9`表示仅把视频画面的长宽比改成16:9，当然可以根据具体的视频信息改成4:3或者16:10等常见的长宽比。

### 裁剪视频（去黑边）

有些视频画面会有额外的黑边，想去掉这些黑边就要用到ffmpeg的截取过滤器，可以参考[ffmpeg使用crop去除黑边](http://www.cnperler.com/?p=120)，这里有对于该滤镜的详细解释。

```bash
# 首先获取到我们要裁剪的画面的大小和位置
ffmpeg -i <infile> -vf "cropdetect=24:16:0" -f null -
# 该命令会自动检测出除黑边之外的画面在整个画面中的大小和位置信息并输出类似"crop=640:256:0:36"的信息
# 接下来用ffmpeg的crop滤镜裁剪视频画面即可
ffmpeg -i <infile> -vf "crop=640:256:0:36" <outfile>
```

### 添加额外信息

影视文件中的视频轨或者整个的封装格式，都可以添加一些额外信息，比如创建时间，标题，字幕轨是中文还是英文等信息。

常见的几个参数如下：

```bash
ffmpeg -i <infile> [options] <metadata> <outfile>
# 添加标题信息
-metadata title="My Title"
# 指定字幕轨语言
-metadata:s:a:0 language=chi
# 视频创建时间
-metadata creation_time="2010-12-17 06:30:00"
```
