# OpenGL ES：视频加滤镜后导出


## 视频加滤镜播放

MediaCodec解码——>OpenGL es——> GLSurfaceView

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190719111816383.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NDEwMjM2,size_16,color_FFFFFF,t_70)

## 视频滤镜合成导出
MediaCodec解码——>OpenGL es——> MediaCodec编码
![MediaCodec解码——>outputSurface——>OpenGL es——>inputSurface——>MediaCodec编码](https://img-blog.csdnimg.cn/20190719111604329.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NDEwMjM2,size_16,color_FFFFFF,t_70)


大体来说就是OpenGL渲染后的输出不一样，需要使用编码器编码保存到本地，由于OpenGL ES并不负责窗口管理以及上下文管理，该职责由各个平台自行完成，没有了GLSurfaceView为我们创建的上下文环境，因此我们需要使用相关的API初始化OpenGL的上下文环境以支持后续的操作。

主要流程还是上图所示，该功能的实现，需要对整体的流程有一定的了解同时也需要掌握MediaCodec的使用以及需要注意一些坑，否则可能会导致视频无法播放。



## 注意

1.	MediaMuxer写完数据需要手动停止并释放资源，否则写的视频文件没有文件尾会导致不能播放。同理解码器和编码器在用完以后也需要释放。
2.	 一个线程对应一个OpenGL es 上下文，因此我们需要初始化俩个上下文环境，一个用于OpenGL着色器、顶点数据等渲染配置的初始化，一个用于在编解码线程中渲染。编解码线程中的上下文需要设置与OpenGL的初始化的线程共享上下文资源，包括纹理、FrameBuffer以及其他的Buffer等资源。
3.	渲染一次完成后，需要切换一次egl的frame buffer，EGL的工作模式是双缓存模式，内部拥有俩个frame buffer（fb），当EGL将一个fb渲染到屏幕上的时候，另一个就在后台等待OpenGL进行交换。

```
	EGL14.eglSwapBuffers(mEglDisplay, mEglSurfaceEncoder);
```


