---
layout: post
title: 'MediaExtractor、MediaMuxer 分离和合成MP4'
date: 2020-07-04
author: qzhuorui
color: rgb(98,170,255)
tags: 音视频
---



## MediaExtractor、MediaMuxer 分离和合成MP4

### 一、MediaExtractor：

#### 1.概述：

音视频分离器，将一些格式的视频分离出 视频轨道和音频轨道。

主要流程和API：

1. setDataSource(String path)：设置数据源（本地文件地址，HTTP协议的网络码流地址
2. getTrackCount()：获取通道数
3. getTackFormat(int index)：获取通道格式（获取码流的详细格式/配置信息
4. readSampleData(ByteBuffer byteBuf,int offset)：获取指定通道中的数据
5. getSampleTime()：获取当前时间戳
6. advance()：下一帧
7. release()：释放资源

```java
int videoTrackIndex = -1;
int audioTrackIndex = -1;

for(int i = 0; i < mMediaExtractor.getTrackCount(); i++) {
    //获取码流的详细格式/配置信息
    MediaFormat format = mMediaExtractor.getTrackFormat(i);

    String mime = format.getString(MediaFormat.KEY_MIME);
    if(mime.startsWith("video/")) {
        videoTrackIndex = i;
    }
    else if(mime.startsWith("audio/")) {
        audioTrackIndex = i;
    }
}
```

获取到媒体文件的详细信息之后，就可以选择指定的通道，分离和读取数据

```java
mMediaExtractor.selectTrack(videoTrackIndex); //选择读取视频数据
while(true) {
    int sampleSize = mMediaExtractor.readSampleData(buffer, 0);  //读取一帧数据,获取指定通道数据
    if(sampleSize < 0) {
        break;
    }
    mMediaExtractor.advance(); //移动到下一帧
}
mMediaExtractor.release(); //读取结束后，要记得释放资源
```

### 二、MediaMuxer：

#### 1.概述：

音视频合成器，将 视频和音频合成 相应的格式。

1. MediaMuxer(String path ,int format)：初始化输出文件名称，输出文件格式（mpeg4
2. addTrack(MediaFormat format)：添加轨道，返回track index，在writeSampleData中使用
3. start()：开始合成文件
4. writeSampleData(int , ByteBuffer , MediaCodec.BufferInfo)：写数据
5. stop()：停止合成文件
6. release()：释放资源

#### 2.使用：

1.创建对象后，一个比较重要的操作就是`addTrack` ，添加数据通道，需要传入MediaFormat对象，MediaFormat即媒体格式类，用于描述媒体的格式参数（视频帧率，音频采样率等

两种方式：

1. 使用MediaExtractor.getTrackFormat()解析得到的MediaFormat对象（实际很少使用

2. 手动创建MediaFormat对象

   1. ```java
      MediaFormat.createVideoFormat(MIMETYPE_VIDEO_AVC,width,height); 
      MediaFormat.createAudioFormat(MIMETYPE_AUDIO_AAC, sampleRate, channelCount);
      ```

   2. 注意：手动创建的话，一定要设置"csd-0"和"csd-1"

2.通过addTrack()后，记录下对应的trackIndex，就可以使用MediaMuxer.writeSampleData()，向mp4文件中写入数据了。

1. 注意：writeSampleData()最后一个参数是BufferInfo，必须填入“正确”的值

```java
BufferInfo info = new BufferInfo();
info.offset = 0;
info.size = sampleSize;//数据大小
info.flags = MediaCodec.BUFFER_FLAG_SYNC_FRAME;//是否为关键帧
info.presentationTimeUs = timestamp;//必须给出正确的时间戳！单位是微妙us
```

2. 最终记得关闭和释放

```java
mMediaMuxer.stop();
mMediaMuxer.release();
```

### 三、实际使用：

1. 根据实际需求，初始化参数（path，fileName，createTime，fps等，需要帧率是为了计算视频时间戳，且有一个预录需求，需要帧率设置一个圆环buffer）

   1. ```java
      //根据参数初始化MediaMuxer
      mediaMuxer = new MediaMuxer(filePath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
      //手动创建对应MediaFormat
      videoFormat = MediaFormat.createVideoFormat(MediaFormat.MIMETYPE_VIDEO_AVC, param.width, param.height);
      audioFormat = MediaFormat.createAudioFormat(MediaFormat.MIMETYPE_AUDIO_AAC, 8000, 1);
      ```

   2. 此时MediaMuxer创建成功，等待 编码器返回数据，进行下一步操作。

2. 关于编码器的介绍放后面，在此只说过程。

   1. 开启编码器  mediacodec.start()
      1. 编码器拿到csd数据，设置给mediaFormat，再给muxer.addTrack(format)，此时也有了音频或视频的track
      2. muxer准备就绪
   2. 开始transmit
      1. muxer.start()
      2. 拿到编码器返回的数据，开始在程序内传输编码后的数据
   3. 从编码器拿到回调过来的数据和数据类型（音频或视频），只要编码器有数据回调进来，就不断的mixData，生成对应的muxerBuffer（自己封装）。并且把muxerBuffer放入一个ArrayList中。
   4. 与此同时我们还需要实现真正的mix方法，这里开启一个线程去驱动一个task，这个task用于从刚才保存muxerBuffer的list中while的去拿数据（while条件为标志位）,拿到的muxerBuffer在把各个字段值赋值给BufferInfo（循环中复用，不必多次创建Mediacodec.BufferInfo）
   5. 此时就需要执行写入操作了。这里格外需要注意线程同步问题。标志位和线程同步要处理好，且还要对时间戳进行是否合理的判断处理。最终调用`mediaMuxer.writeSampleData(index, buffer, info);`

   



