# rtp_h264
基于rtp的h264数据的硬解码，有两种硬解码方式，一种是基于intel media sdk的硬解码，一种是基于cuda的硬解码。
第一种主要是使用intel的自带的硬解码，启用vpp之后，可以转换为rgba格式的图像数据，还可以对输出图像进行裁剪，缩放等操作，还可以进行opencl的编程，实现图像的处理。
第二种cuvid可以直接将h264转换为yuv图像，然后将图像通过cuda编程转换为rgba或者其他格式的图像，也可以进行其他自定义的图像操作（cuda编程）。
从图像获取到图像在pc上的显示，延时大概300多ms（程序中是将数据从硬解码资源出拷贝到cpu端，然后使用opencv的imshow进行显示的）
## 开发环境
win10 + vs2015 + jrtplib + cuda（/intel media sdk) + opencv
intel media sdk 是在intel i7-7700  intel HD Graphics 630 上测试的
cuda 在笔记本

## jrtplib 配置
下载jrtplib 源码，使用cmake构建vs2015的工程，编译x64版本的debug和release版本，不需要开启thread。

## intel media sdk
下载intel media sdk 2018 R2 版本的可执行文件，直接一路next安装。
## cuda安装
这个百度安装就行，目前我的机器是gtx610m，cuda8.0。另外：cuda8.0中的cudadecode的sample跟cuda9.0的不一样，且cuda10.0就把这个demo删掉了。我使用的是cuda9.0的demo修改得到的。

## intel media sdk 解码过程
1、```mfxStatus sts = decoder->init(&initParam);```
2、在一个线程里不断的执行以下语句，往解码器里不断的送数据。
```sts = decoder->decode(packet->GetPayloadData(), length, timestamp1);```
3、在另一个线程里，先查询图像数据格式，然后不断查询得到解码后的图像数据。
```sts = decoder->getDecodedParam(vParam);```
```sts = decoder->getDecodedData(bufIndex[dataFlag], buflen, &pts); ```

## cuda cuvid解码过程
1、初始化，需要指定要解码的图像的属性（宽高，码率，解码格式等等）
2、不断的往解码器里送数据。
 ```
pkt.flags = 0;
pkt.payload_size = (unsigned long)packet->GetPayloadLength();
pkt.payload = (unsigned char *)packet->GetPayloadData();
pkt.timestamp = packet->GetTimestamp();
CUresult result = cuvidParseVideoData(g_pVideoParser->hParser_, &pkt);
 ```
3、查询是否有数据解析完成。

 ```
    oVideoParserParameters.pfnSequenceCallback    = HandleVideoSequence;    // Called before decoding frames and/or whenever there is a format change
    oVideoParserParameters.pfnDecodePicture       = HandlePictureDecode;    // Called when a picture is ready to be decoded (decode order)
    oVideoParserParameters.pfnDisplayPicture      = HandlePictureDisplay;   // Called whenever a picture is ready to be displayed (display order)
 ```
 这三个就是回调函数，第一个是要解码的数据格式有变化时才调用的函数，第二个是解码图片前调用的函数，主要是解码图像前查询是否有空闲的queue来接收解码之后的图片，第三个就是解码之后的数据，解码完成之后就会自动调用此函数，函数中是将解码后的图片id放到空闲的queue中，需要调用videoDecoder->mapFrame函数之后才能将图片的数据放到device的内存中，此时图片是yuv数据格式的，需要再次调用cuda kernal函数将yuv数据转换为rgb数据，然后再将device端的rgb数据拷贝回主机端。当然，如果我们需要对图像进行其他处理，则可以写cuda函数，替换现有的cuda kernal函数，完成我们需要的处理。
