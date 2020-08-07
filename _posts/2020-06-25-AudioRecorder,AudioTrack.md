---
layout: post
title: 'AudioRecord,AudioTrack'
date: 2020-06-25
author: qzhuorui
color: rgb(98,170,255)
tags: 音视频
---



> 用于记录学习音视频知识中，关于音频的初级知识，本次只涉及PCM相关，不涉及编解码



# Android AudioRecord,AudioTrack 录制播放音频

## 一、 AudioRecord 录制PCM

### 1.参数描述：

AudioRecord：Android提供用于实现录音功能。得到数据为PCM数据。

```java
//构造函数
public AudioRecord(int audioSource, int sampleRateInHz, int channelConfig, 
                   int audioFormat, int bufferSizeInBytes)
```

1. audioSource：音频源（Default，MIC等
2. sampleRateInHz：采样率（每秒采样次数，常用有8K,44.1K
3. channelConfig：声道（单声道或双声道
4. audioFormat：采样大小（8bit或16bit，采样大小越大，音质越好
5. bufferSizeInBytes：采集数据的缓冲区（一般通过getMinBufferSize获取

```java
public int SAMPLE_RATE = 16000; //采样率 8K或16K
public int CHANNEL_CONFIG = AudioFormat.CHANNEL_IN_MONO; //音频通道(单声道)
public int AUDIO_FORMAT = AudioFormat.ENCODING_PCM_16BIT; //音频格式，采样大小
public int AUDIO_SOURCE = MediaRecorder.AudioSource.DEFAULT;  //音频源（麦克风）
```

### 2.开始录制：

```java
//录制涉及到的方法有两个，如下：1.开始录制；2.读取到数据
mAudioRecord.startRecording();
mAudioRecord.read(data, 0, buffSize);
```

具体使用为：

1. 调用start开始录制，此时开启一个线程，在线程中while不断的读取录制的数据（应该有一个标志位isStarted，release和stop时需要修改状态），读到的数据放入字节数组中，这个字节数大小为 

`SAMPLE_RATE * 20 / 1000 * (AudioFormat.ENCODING_PCM_16BIT == AUDIO_FORMAT ? 16 : 8) / 8;`

```java
byte[] data = new byte[buffSize];\\上面计算出的size
dataLen = mAudioRecord.read(data, 0, buffSize);
```

read返回的是int，如果==0就sleep(2)后continue，如果>0就存放到一个队列中 `pcmBuffer.enqueue(data, dataLen);`（自定义一个QueueArray，注意线程安全）

2. 在此方法末再开启一个线程去驱动一个task，这个task用于从pcmBuffer中dequeue数据

`byte[] tmp = pcmBuffer.dequeue(buffSize);` 此时就拿到了PCM数据，可以交给编码器进行对应的编码。（这里只总结PCM数据的获取，关于编码器放后）

### 3.停止录制：

1. 如果isStarted且mAudioRecord!=null，那么就`mAudioRecord.stop(); ` 还有清除pcmBuffer，并且修改isStarted状态，同时remove上面的task。
2. 当真正退出程序时，可能还需要再加一步release操作。

```java
if (mAudioRecord != null) {
            mAudioRecord.release();
            mAudioRecord = null;
        }
        isStarted = false;
```



## 二、AudioTrack播放PCM

### 1.参数描述：

Android中可以使用AudioTrack播放PCM。但是关于数据有两种加载模式：

1. MODE_STREAM：音频数据不断写入AudioTrack中，会有延迟
2. MODE_STATIC：将音频数据一次性写入，无延迟，但是适合小文件，大文件可能内存不足，需要先write再play

```java
//初始化
mAudioTrack = new AudioTrack(AudioManager.STREAM_MUSIC, AudioRate, AudioOutChannel, AudioFormater, playBufferMinSize, AudioTrack.MODE_STREAM);
```

### 2.具体使用：

```java
mAudioTrack.play();
mInputStream = new FileInputStream(mFile);
while (mInputStream.available() > 0) {
                int readSize = mInputStream.read(bufferbytes);
                mAudioTrack.write(bufferbytes, 0, readSize);
            }
mAudioTrack.stop();
```

