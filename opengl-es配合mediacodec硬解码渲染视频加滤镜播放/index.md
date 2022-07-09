# OpenGL ES：配合MediaCodec硬解码渲染（视频加滤镜播放）


## 注意点

1.	MediaCodec 解码后的原始数据，格式为yuv，而OpenGL所能渲染的格式为rgb，因此我们需要使用扩展库中的**扩展纹理**

    **GLES11Ext.GL_TEXTURE_EXTERNAL_OES**

而它的作用就是实现YUV格式到RGB的自动转化。

**片段着色器中需使用扩展采样器：**

    uniform samplerExternalOES sTexture

2.	**COLOR_FormatYUV420Flexible**

YUV420Flexible并不是一种确定的YUV420格式，而是包含COLOR_FormatYUV411Planar, COLOR_FormatYUV411PackedPlanar, COLOR_FormatYUV420Planar, COLOR_FormatYUV420PackedPlanar, COLOR_FormatYUV420SemiPlanar和COLOR_FormatYUV420PackedSemiPlanar。在API 21引入YUV420Flexible的同时，它所包含的这些格式都deprecated掉了。

那么为什么所有的解码器都支持YUV420Flexible呢？官方没有说明这点，但我猜测，只要解码器支持YUV420Flexible中的任意一种格式，就会被认为支持YUV420Flexible格式。也就是说，几乎所有的解码器都支持YUV420Flexible代表的格式中的一种或几种。

此处引用：
https://www.polarxiong.com/archives/Android-MediaCodec视频文件硬件解码-高效率得到YUV格式帧-快速保存JPEG图片-不使用OpenGL.html
## **原理步骤：**

 1. 生成一个oes纹理，并且以此得到一个SurfaceTexture，并设置帧可用回调监听，可用时请求渲染，再得到一个 Surface，并将它回调给外部MediaCodec 配置并启动解码器。（在ESL上下文环境中生成，因此需要通过回调给外部使用）
 2.  渲染完一帧后，需调用`        surfaceTexture.updateTexImage()  //更新纹理数据`
 3. onFrameAvailable可用时，请求GLSurfaceView渲染重绘即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190719111816383.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NDEwMjM2,size_16,color_FFFFFF,t_70)

## 解码器

**MediaCodec的准备：**

 1. 使用MediaExtract选择相应的视频/音频轨道，获取相应的格式信息（MediaFormat）。
 2. 使用MdiaCodec的类接口创建codec对象，配置音视频格式以及**surface**。（ OpenGL回调出来的  Surface surface = new Surface(surfaceTexture);）
 3. 启动解码器。
 4. 启动一个线程，循环从MediaExtract读取媒体文件数据到MdiaCodec的输入缓冲区，直至文件尾。
 5. 从MdiaCodec中取出被成功解码的buffer的 index 和 buffer的信息。index无误则把此buffer渲染到surface上去。

**MediaCodec代码如下：**
```
package com.example.myapplication.ui.video;

import android.media.MediaCodec;
import android.media.MediaCodecInfo;
import android.media.MediaExtractor;
import android.media.MediaFormat;

import android.util.Log;
import android.view.Surface;


import java.io.IOException;
import java.nio.ByteBuffer;

public class VideoCodec {

    private static final long TIMEOUT_USEC = 10000;
    private String TAG = "VideoCodec";
    private MediaCodec videoDecoder;
    private MediaExtractor mediaExtractor;
    private boolean isAvailable = false;

	//视频路径
    private String path;
    //OpenGL回调出来的 Surface Surface surface = new Surface(surfaceTexture);
    //surfaceTexture设置帧可用监听
    //用于解码后数据的输出，交给OpenGL渲染
    private Surface surface;
    public VideoCodec(String path,Surface surface) {
        mediaExtractor = new MediaExtractor();
        this.path = path;
        this.surface = surface;
    }

	//传入指定的视频或音频格式，例如：video/、audio/
	//返回当前格式轨道的index 
    public int getTrackIndex(String mime) {
        int trackCount = mediaExtractor.getTrackCount();
        int videoIndex=-1;
        for (int i = 0; i < trackCount; i++) {
            MediaFormat format = mediaExtractor.getTrackFormat(i);
            if (format.getString(MediaFormat.KEY_MIME).startsWith(mime)) {
                videoIndex = i;
                break;
            }
        }
        return videoIndex;
    }

    private boolean isPlaying = false;

    public void start(){
        isPlaying = true;
        DecoderMP4Thread thread = new DecoderMP4Thread();
        thread.start();
    }

	//视频的解码放到子线程中进行，防止阻塞主线程
    private class DecoderMP4Thread extends Thread {
        @Override
        public void run() {

            try {
            	//首先设置待解码文件的路径
                mediaExtractor.setDataSource(path);
                int videoIndex = getTrackIndex("video/");
                 //当前index无效则直接退出
                if (videoIndex < 0) {
                    return;
                }
                //选取当前轨道
                mediaExtractor.selectTrack(videoIndex);
               //获取视频的格式信息
                MediaFormat mediaFormat = mediaExtractor.getTrackFormat(videoIndex);
                //设置解码支持的格式
                mediaFormat.setInteger(MediaFormat.KEY_COLOR_FORMAT, MediaCodecInfo.CodecCapabilities.COLOR_FormatRGBFlexible);
                String mime = mediaFormat.getString(MediaFormat.KEY_MIME);
                //通过MIME创建解码器
                videoDecoder = MediaCodec.createDecoderByType(mime);
                //配置
                videoDecoder.configure(mediaFormat, surface, null, 0);
            } catch (IOException e) {
                e.printStackTrace();
            }
            //启动解码器
            videoDecoder.start();
            
            int frameIndex = 0;
            boolean isVideoOver = false;
            // 开始循环，一直到视频资源结束
            MediaCodec.BufferInfo videoBufferInfo = new MediaCodec.BufferInfo();
		    // 开始的时间
            long startMs = System.currentTimeMillis();  
            // 当前Thread 没有被中断
            while (!Thread.interrupted()) {
                if (!isPlaying) {
                    continue;
                }
                if (!isVideoOver) {
                    // 视频没有结束  提取一个单位的视频资源放到 解码器(mediaCodec) 缓冲区中
                    isVideoOver = putBufferToMediaCodec(mediaExtractor, videoDecoder);
                }

                // 返回一个被成功解码的buffer的 index 或者是一个信息  同时更新 videoBufferInfo 的数据
                int outputBufferIndex = videoDecoder.dequeueOutputBuffer(videoBufferInfo, TIMEOUT_USEC);
                switch (outputBufferIndex) {
                    case MediaCodec.INFO_OUTPUT_FORMAT_CHANGED:
                        Log.v(TAG, "format changed");
                        break;
                    case MediaCodec.INFO_TRY_AGAIN_LATER:
                        Log.v(TAG, "超时");
                        break;
                    case MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED:
                        //outputBuffers = videoCodec.getOutputBuffers();
                        Log.v(TAG, "output buffers changed");
                        break;
                    default:
                        //延时操作
                        //如果缓冲区里的可展示时间>当前视频播放的总时间，就休眠一下 展示当前的帧，
                        sleepRender(videoBufferInfo, startMs);
                        
                        //渲染为true就会渲染到surface   configure() 设置的surface
                        videoDecoder.releaseOutputBuffer(outputBufferIndex, true);
                        
                        frameIndex ++;
                        Log.v(TAG, "frameIndex   " + frameIndex);
                        break;
                }

                if ((videoBufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                    Log.v(TAG, "buffer stream end");
                    break;
                }
            }
            //释放资源
            videoDecoder.stop();
            videoDecoder.release();
            mediaExtractor.release();
        }
    }

    /**
     * 将缓冲区传递至解码器
     * 如果到了文件末尾，返回true;否则返回false
     */
    private boolean putBufferToMediaCodec(MediaExtractor extractor, MediaCodec decoder) {
        boolean isMediaEOS = false;
        // 解码器  要填充有效数据的输入缓冲区的索引 —————— 此id的缓冲区可以被使用
        int inputBufferIndex = decoder.dequeueInputBuffer(TIMEOUT_USEC);

        if (inputBufferIndex >= 0) {
            ByteBuffer inputBuffer = decoder.getInputBuffer(inputBufferIndex);
            // MediaExtractor读取媒体文件的数据，存储到缓冲区中。并返回大小。结束为-1
            int sampleSize = extractor.readSampleData(inputBuffer, 0);
            if (sampleSize < 0) {
                decoder.queueInputBuffer(inputBufferIndex, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
                isMediaEOS = true;
                Log.v(TAG, "media eos");
            } else {
                // 在输入缓冲区添加数据之后，把它告诉 MediaCodec （解码）
                decoder.queueInputBuffer(inputBufferIndex, 0, sampleSize, extractor.getSampleTime(), 0);
                // MediaExtractor 准备下一个 单位的数据
                extractor.advance();
            }
        }
        return isMediaEOS;
    }

    private void sleepRender(MediaCodec.BufferInfo audioBufferInfo, long startMs) {
        // 这里的时间是 毫秒  presentationTimeUs 的时间是累加的 以微秒进行一帧一帧的累加
        // audioBufferInfo 是改变的
        while (audioBufferInfo.presentationTimeUs / 1000 > System.currentTimeMillis() - startMs) {
            try {
                // 10 毫秒
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
                break;
            }
        }
    }
}

```
## 滤镜类（BaseFilter）

