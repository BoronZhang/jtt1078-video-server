## jtt1078-video-server
基于JT/T 1078协议实现的视频转播服务器，当车机服务器端主动下发**音视频实时传输控制**消息（0x9102）后，车载终端连接到此服务器后，发送指定摄像头所采集的视频流，此服务器部分负责将视频流推送到RTMP服务器端，完成转播的流程。

### 准备工具
1. 安装了`ffmpeg`的机器一台
2. 安装了`nginx-rtmp-module`或`nginx-http-flv-module`（推荐）的`Nginx`服务器，用于实时视频的转播
3. `PotPlayer`，可用于播放实时转播的`RTMP`视频
4. 项目里准备了一个测试程序（`src/main/java/cn.org.hentai.jtt1078.test.VideoPushTest.java`），以及一个数据文件（`src/main/resources/tcpdump.bin`），数据文件是通过工具采集的一段几分钟时长的车载终端发送上来的原始消息包，测试程序可以持续不断的、慢慢的发送数据文件里的内容，用来模拟车载终端发送视频流的过程。

### 协议所定义的实时码流数据报文格式：

|起始字节|字段|数据类型|描述及要求|
|---|---|---|---|
|0|帧头标识|DWORD|固定为0x30 0x31 0x63 0x64|
|4|V|2 BITS|固定为2|
||P|1 BIT|固定为0|
||X|1 BIT|RTP头是否需要扩展位，固定为0|
||CC|4 BITS|固定为1|
|5|M|1 BIT|标志位，确定是否完整数据帧的边界|
||PT|7 BITS|负载类型，见表19|
|6|包序号|WORD|初始为0，每发送一个RTP数据包，序列号加1|
|8|SIM卡号|BCD[6]|终端设备SIM卡号|
|14|逻辑通道号|BYTE|按照JT/T 1076-2016中的表2|
|15|数据类型|4 BITS|0000：数据I祯等|
||分包处理标记|4 BITS|0000：原子包，不可拆分等|
|16|时间戳|BYTE[8]|标识此RTP数据包当前祯的相对时间，单位毫秒（ms）。当数据类型为0100时，则没有该字段|
|24|Last I Frame Interval|WORD|该祯与上一个关键祯之间的时间间隔，单位毫秒（ms），当数据类型为非视频祯时，则没有该字段|
|26|Last Frame Interval|WORD|该祯与上一个关键祯之间的时间时间，单位毫秒(ms)，当数据类型为非视频祯时，则没有该字段|
|28|数据体长度|WORD|后续数据体长度，不含此字段|
|30|数据体|BYTE[n]|音视频数据或透传数据，长度不超过950 byte|

经过阅读文档，以及测试，有如下问题或发现：
1. 视频流发送是不需要回应的，车载终端将一直持续不断的发送。
2. 1078协议借鉴了RTP协议，但不是真正的RTP协议。
3. 数据包的第30字节（非视频类的消息包位置不一样）起为视频数据体，可以将每个消息包的这个部分保存到文件中，比如`xx.h264`，可以使用`PotPlayer`来播放。

一般来说，视频的推流首先想到的是使用`ffmpeg`，如果要集成`ffmpeg`的sdk或是使用`javacv`、`jcodec`等视频编解码类的库，开发与学习维护的成本很大。在这里我们直接创建`ffmpeg`子进程，通过它的参数设定，让`ffmpeg`子进程直接从**stdin**中读取数据进行转码推流，通过这种方法，可以很好的达到推流的目的，并且不需要整合其它任何第三方的sdk，如果有后续的进一步的扩展需求（比如添加OSD字幕），可以直接通过修改创建子进程时的命令行来达到目的，非常的简单与易维护。

ffmpeg的输入端，可以是stdin、网络流、文件（图片或视频）等几乎所有常见的数据来源，也可以是同时多个输入，功能非常强大，我之前一直使用的是FIFO命名管道，现在可以抛弃对FIFO命名管道的依赖了，并且可以真正的跨平台运行了。

### 测试步骤
1. 配置好服务器端，确定`ffmpeg`的完整路径，替换掉`app.properties`配置项。
2. 确保nginx服务器已经启动，同时配置文件里的`rtmp.format`已经设置为正确的RTMP地址格式。
3. 直接在IDE里运行`cn.org.hentai.jtt1078.app.VideoServerApp`，或对项目进行打包，执行`mvn package`，执行`java -jar jtt1078-video-server-1.0-SNAPSHOT.jar`来启动服务器端。
4. 运行`VideoPushTest.java`，开始模拟车载终端的视频推送。
5. 服务器端的日志里将会输出`start streaming to rtmp://....`的字样，后面即为实际推送的RTMP地址。
6. 打开`PotPlayer`，按下`Ctrl+U`，输入上面输出的RTMP地址，稍等片刻就能看到转播出来的视频了。

### 常见问题
1. HLS、HTTP-FLV、RTMP等实时视频都可以通过ffmpeg推流，在RTMP服务器端配置来实现。
2. 网页端的RTMP播放，之前使用`Aliplay`，效果很不理想，可以尝试使用`video.js`。
3. 测试使用`nginx-http-flv-module`+`flv.js`来做网页播放，效果很不错，延迟低，不依赖于flash，值得推荐。
4. 音频不关注也没有测试，如果有很强的音视频同步传输的需求的话，可以考虑修改程序，将RTP消息包里的音频单独拆分出来，处理后也通过`ffmpeg`子进程合并音轨到视频流中去（我瞎想的）。
5. RTMP测试工具：[live_test.swf](http://www.cutv.com/demo/live_test.swf)，注意：要使用IE浏览器打开。

### 交流讨论
QQ群：808432702

