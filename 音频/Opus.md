# Opus

Opus是一个完全开放的、免费的、多功能的音频编解码器。 它在交互式的语音和音乐在互联网中的传输方面有着无与伦比的优势，但是同样致力于存储和流媒体应用程序。它是由互联网工程任务组(IETF)制定的标准，标准格式为RFC 6716，由Skype的SILK编解码器和Xiph.Org的CELT编解码器合并发展而来，号称音频编解码器中的瑞士军刀

[github](https://github.com/xiph/opus)

Opus可以处理广泛的音频应用程序，包括IP电话、视频会议、游戏内聊天、甚至远程现场音乐表演。从低比特率窄带语音到非常高质量的立体声音乐，它都可以适用。技术特点：

- 6 kb /s到510 kb / s的比特率
- 采样率从8 kHz（窄带）到48 kHz（全频）
- 帧大小从2.5毫秒到60毫秒
- 支持恒定比特率（CBR）和可变比特率（VBR）
- 从窄带到全频段的音频带宽
- 支持语音和音乐
- 支持单声道和立体声
- 支持多达255个频道（多数据流的帧）
- 可动态调节比特率，音频带宽和帧大小
- 良好的鲁棒性丢失率和数据包丢失隐藏（PLC）
- 浮点和定点实现



## API文档

https://www.opus-codec.org/

### opus_encoder_create

首先通过如下**opus_encoder_create**，创建编码器对象

```
OpusEncoder∗ opus_encoder_create (
    opus_int32 Fs,
    int channels,
    int application,
    int ∗ error )
```

其中Fs为采样率，该值必须是8000，12000，16000，24000，48000中的一个

channels 通道数：1 或者 2

application 该mode为

* **OPUS_APPLICATION_VOIP** 在给定的语音信号比特率下，它可以提供最佳质量。
* **OPUS_APPLICATION_AUDIO** 对于大多数非语音信号(例如音乐)它可以在给定的比特率下提供最佳质量。使用此模式使用与音乐和混合内容，广播以及要求少于15毫秒编码延迟的应用。
* **OPUS_APPLICATION_RESTRICTED_LOWDLEAY** 它配置了低延迟模式，近用了语音优化模式来稍微减少延迟。此模式只能在新初始化或新充值的编码器上，因为它更改编解码延迟



返回OpusEncoder：Opus编码器，包含了编码器的全部状态。

### opus_encoder_ctl

通过宏来生成请求和后来的参数

* **OPUS_SET_VBR**    两个值，

  可变比特率default  VBR(1)，确切类型由OPUS_SET_VBR_CONSTRAINT控制；

  另一个恒定比特率CBR(0)，对于比特率非常低的LPC /混合模式，这可以导致明显的质量下降

* **OPUS_SET_VBR_CONSTRAINT** 该设置为CBR时，忽略

  默认1，受约束的VBR

* **OPUS_SET_BITRATE** 比特率

* **OPUS_SET_COMPLEXITY** 复杂度 最低1，最高10

* **OPUS_SET_SIGNAL** 信号类型，

  OPUS_AUTO

  OPUS_SIGNAL_VOICE 选择LPC或混合模式的偏差阈值

  OPUS_SIGNAL_MUSIC 选择MDCT模式的偏差阈值。

* **OPUS_SET_LSB_DEPTH**  编码器的信号深度,在8-24间

* **OPUS_SET_EXPERT_FRAME_DURATION** 



### 编码音频数据

编码音频数据也分为两种方式，一种是float型，一种是int型

```
opus_int32 opus_encode( OpusEncoder∗ st, 
                       const opus_int16∗ pcm, 
                       int frame_size, 
                       unsigned char∗ data, 
                       opus_int32 max_data_bytes)
```

举例：**opus_int32 len** = **opus_encode(enc, pcm, 960, out, 4000);**

参数：解码器状态指针、音频原始数据、以样本个数为单位的帧长度、编码完成的数据、最大的编码后数据长度

约束：opus为了对一个帧进行编码，必须正确地用音频数据的帧(2.5, 5, 10, 20, 40 or 60 ms)来调用opus_encode()或opus_encode_float()函数。

比如，在48kHz的采样率下，opus_encode()参数中的合法的frame_size（单通道的帧大小）值只有：120, 240, 480, 960, 1920, 2880。即：
    frame_size = 采样率 * 帧时间*通道数。

另外 通常我们record的是byte数组，而sizeof(opus_16) = 2，所以传进来的byte数组大小 最好是2*frame_size



或者

```
opus_int32 opus_encode_float(OpusEncoder* st, 
                       const float* pcm, 
                       int frame_size, 
                       unsigned char* data, 
                       opus_int32 max_data_bytes);
```


举例：**opus_int32 len** = **opus_encode(enc, pcm, 960, out, 4000);**
参数：同上

在这两种编码方式中，浮点型编码是更加损耗CPU，但是因此获得的编码精度更高，所以音质更加精准，而16位in整型编码是更多机器支持的，编码精度稍低，对CPU的损耗也小一点。



### 解码音频数据

```
int opus_decode (
        OpusDecoder ∗ st,
        const unsigned char ∗ data,
        opus_int32 len,
        opus_int16 ∗ pcm,
        int frame_size,
        int decode_fec )
```

data 为输入，输出为pcm。sizeof(unsigned char) = 1;sizeof(opus_16) = 2;

decode_fec 为0或者1，是否请求对纠错数据进行编码。



