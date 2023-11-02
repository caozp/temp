# AAC

是一种专为声音数据设计的文件压缩格式。他的目的是为了取代MP3格式，与MP3不同，它采用了全新的算法进行编码，更加高效

## AAC音频格式
### ADIF (Audio Data Interchange Format)
这种格式只需要在文件开头存一个很小的头，包括采样率，采样大小，声道数量等基本信息，就可以对文件进行解读。这种格式**只能从头开始解码，常用在磁盘文件中**。

------



### ADTS (Audio Data transport Stream)
**这种格式每一帧前面都有一个同步字，占用7-9个字节，好处是可以在音频流的任何位置开始解码**，他类似于数据流格式。因为每一帧前面都有同步字，所以ADTS文件要比ADIF增加一些数据量

有的时候当你编码AAC裸流的时候，会遇到写出来的AAC文件并不能在PC和手机上播放，很大的可能就是AAC文件的每一帧里缺少了ADTS头信息文件的包装拼接。

只需要加入头文件ADTS即可。一个AAC原始数据块长度是可变的，对原始帧加上ADTS头进行ADTS的封装，就形成了ADTS帧。

[adts头部格式分析](https://www.cnblogs.com/zhangxuan/p/8809245.html)

网上找了一段代码，根据上文来分析下组成部分

```
    /**
     * 写入ADTS头部数据
     *
     * @param packet
     * @param packetLen
     */
    public static void addADTStoPacket(byte[] packet, int packetLen) {
        int profile = 2; // AAC LC
        int freqIdx = 4; // 44.1KHz
        int chanCfg = 2; // CPE

        //syncword，比如为1，即0xff
        packet[0] = (byte) 0xFF;
        packet[1] = (byte) 0xF9;
        packet[2] = (byte) (((profile - 1) << 6) + (freqIdx << 2) + (chanCfg >> 2));
        packet[3] = (byte) (((chanCfg & 3) << 6) + (packetLen >> 11));
        packet[4] = (byte) ((packetLen & 0x7FF) >> 3);
        packet[5] = (byte) (((packetLen & 7) << 5) + 0x1F);
        packet[6] = (byte) 0xFC;
    }
```

packet[0],packet[1]  FFF9  = 1111  1111 1111 1001。其中，第十三位为1，也就是MPEG标识符为1，1标识MPEG-2;Layer固定为00，第十六位为1，也就是没有CRC校验。

接着是profile 它有两位，也就是packet[2]的前两位。profile = MPEG-4 Audio Object Type - 1。AAC LC在MPEG-4中等于2，所以也就是1左移6位。 

packet[2] 中间4位的值为**sampling_frequency_index**，也就是这个freqIdx 左移两位

后面是private_bit。私有位，编码时设置为0，解码时忽略。

packet[2]的最后一位加packet[3]的前两位构成了chanCfg。chanCfg = 010。右移两位也就是00。也就是packet[2] 最后一位为0，总体packet[2]  =  01 01 00 00。然后packet[3]的前两位为10。

上面式子中 chanCfg & 3 即 010 & 011  = 010 这里&3主要应该是去除首位后左移6位。

中间4位都为0。然后到了可变头信息。packet[3]最后两位应该是13位packetLen的头两位，所以右移11位。

packet[4]  为packetLen的第三位到第11位

packet[5] 位packetLen的最后三位 + 11111

packet[6] 为 11 11 11 00



## 使用MediaCodec对音频进行AAC硬编码

Android中可以使用MediaCodec来访问底层的媒体编解码器，可以对媒体进行编/解码

1. 我们要将文件数据进行AAC编码，需要先初始化一个MediaCodec对象，设置他的MediaFormat为`MediaFormat.MIMETYPE_AUDIO_AAC`。

```
private void initMediaCodec() throws IOException {
        MediaFormat encodeFormat = MediaFormat.createAudioFormat(MediaFormat.MIMETYPE_AUDIO_AAC, 44100, 2);
        encodeFormat.setInteger(MediaFormat.KEY_BIT_RATE, 44100);//比特率
        encodeFormat.setInteger(MediaFormat.KEY_AAC_PROFILE, MediaCodecInfo.CodecProfileLevel.AACObjectLC);
        encodeFormat.setInteger(MediaFormat.KEY_MAX_INPUT_SIZE, 100 * 1024);
        mediaCodec = MediaCodec.createEncoderByType(MediaFormat.MIMETYPE_AUDIO_AAC);
        //创建一个解码器
        
        mediaCodec.configure(encodeFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
        encodeBufferInfo = new MediaCodec.BufferInfo();
    }
```

2. 调用mediaCodec的start方法开始编码

3. 同时将pcm数据传入到下面这个方法中进行AAC编码

```
public void onFrame(byte[] data, OutputStream out){
            int inputIndex = mediaCodec.dequeueInputBuffer(-1);//获取输入缓存的index
            if (inputIndex >= 0) {
                ByteBuffer inputByteBuf = mediaCodec.getInputBuffer(inputIndex);
                inputByteBuf.clear();
                inputByteBuf.put(data);//添加数据
                inputByteBuf.limit(data.length);//限制ByteBuffer的访问长度
                mediaCodec.queueInputBuffer(inputIndex, 0, data.length, 0, 0);//把输入缓存塞回去给MediaCodec
            }

            int outputIndex = mediaCodec.dequeueOutputBuffer(encodeBufferInfo, 0);//获取输出缓存的index
            while (outputIndex >= 0) {
                //获取缓存信息的长度
                int byteBufSize = encodeBufferInfo.size;
                //添加ADTS头部后的长度
                int bytePacketSize = byteBufSize + 7;
                //拿到输出Buffer
                ByteBuffer  outPutBuf = mediaCodec.getOutputBuffer(outputIndex);
                outPutBuf.position(encodeBufferInfo.offset);
                outPutBuf.limit(encodeBufferInfo.offset+encodeBufferInfo.size);

                byte[]  targetByte = new byte[bytePacketSize];
                //添加ADTS头部
                addADTStoPacket(targetByte, bytePacketSize);
            /*
            get（byte[] dst,int offset,int length）:ByteBuffer从position位置开始读，读取length个byte，并写入dst下
            标从offset到offset + length的区域
             */
                outPutBuf.get(targetByte,7,byteBufSize);

                outPutBuf.position(encodeBufferInfo.offset);

                try {
                    out.write(targetByte); //写入文件
                } catch (IOException e) {
                    e.printStackTrace();
                }
                //释放
                mediaCodec.releaseOutputBuffer(outputIndex,false);
                outputIndex = mediaCodec.dequeueOutputBuffer(encodeBufferInfo, 0);
            }

        }
```

