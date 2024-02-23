---
title: 云端图片、视频处理体系
tags: cloud process sys
---

# 一、背景

&ensp;&ensp;2008 ~ 2020年，在这十几年期间，PC互联网往移动互联网风风火火地大规模迁移后，产生了无数的变化，其中，单谈图片、视频处理这一细分领域，从专业化变得越来越简单化、傻瓜化，普通用户都能简单基于手机app进行个性化的图片、视频处理。<br>
&ensp;&ensp;2017年，抖音 + 快手短视频社区爆火背后的大功臣之一：是视频处理工具，例如：服务端转码 + 加水印等。<br>
&ensp;&ensp;2023年随着aigc的爆火：文生图、图生图、文生视频等等火出圈。<br>
&ensp;&ensp;这些爆火的背后，背后展示出来的系统架构规律之一是：客户端部署算法、模型逐步往服务端部署算法、模型靠。也就是：客户端端处理 -> 云处理方向靠。

# 二、云处理
&ensp;&ensp;所谓：云处理，顾名思义，云端图片、视频处理体系的简称，它是对存储在小文件系统里的图片、视频，发起相关的处理。用公式来表达：媒体s = fn (媒体s，参数).
   

# 三、图片处理功能

  对存储在小文件系统里的图片地址，进行相关的处理，注意：请确保相关的url地址能够正常下载 or 访问。

## 3.1、图片基本信息
   获取图片的基本信息，例如：图片格式、文件大小、色彩模型、宽、搞、帧个数，格式如下：
   ```json
{
          "format":  "JPEG",
          "width":  224,
          "height":  224,
          "orientation":  "1",
          "colorModel":  "sRGB",
          "frameNumber":  "1",
          "fileSize":  "6574B"
}
   ```
使用限制：20M以内

## 3.2、图片瘦身

## 3.3、图片缩略

## 3.4、图片裁剪

## 3.5、格式转换

## 3.6、文字水印

## 3.7、logo水印

## 3.8、暗水印

## 3.9、图片合并

# 四、音视频处理功能

## 4.1、音视频meta信息
   获取指定音频、视频资源的元信息，例如：宽、高、码率、帧率、文件大小、时长
```json
{
          "streams":  [
                    {
                              "index":  0,
                              "codec_name":  "aac",
                              "codec_long_name":  "AAC (Advanced Audio Coding)",
                              "profile":  "LC",
                              "codec_type":  "audio",
                              "codec_tag_string":  "mp4a",
                              "codec_tag":  "0x6134706d",
                              "sample_fmt":  "fltp",
                              "sample_rate":  "44100",
                              "channels":  1,
                              "channel_layout":  "mono",
                              "bits_per_sample":  0,
                              "id":  "0x1",
                              "r_frame_rate":  "0/0",
                              "avg_frame_rate":  "0/0",
                              "time_base":  "1/44100",
                              "start_pts":  0,
                              "start_time":  "0.000000",
                              "duration_ts":  2284491,
                              "duration":  "51.802517",
                              "bit_rate":  "128039",
                              "nb_frames":  "2231",
                              "extradata_size":  2,
                              "disposition":  {
                                        "default":  1,
                                        "dub":  0,
                                        "original":  0,
                                        "comment":  0,
                                        "lyrics":  0,
                                        "karaoke":  0,
                                        "forced":  0,
                                        "hearing_impaired":  0,
                                        "visual_impaired":  0,
                                        "clean_effects":  0,
                                        "attached_pic":  0,
                                        "timed_thumbnails":  0,
                                        "captions":  0,
                                        "descriptions":  0,
                                        "metadata":  0,
                                        "dependent":  0,
                                        "still_image":  0
                              },
                              "tags":  {
                                        "creation_time":  "2024-02-23T13:46:34.000000Z",
                                        "language":  "eng",
                                        "handler_name":  "SoundHandle",
                                        "vendor_id":  "[0][0][0][0]"
                              }
                    },
                    {
                              "index":  1,
                              "codec_name":  "h264",
                              "codec_long_name":  "H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10",
                              "profile":  "Constrained Baseline",
                              "codec_type":  "video",
                              "codec_tag_string":  "avc1",
                              "codec_tag":  "0x31637661",
                              "width":  1080,
                              "height":  1920,
                              "coded_width":  1080,
                              "coded_height":  1920,
                              "closed_captions":  0,
                              "film_grain":  0,
                              "has_b_frames":  0,
                              "pix_fmt":  "yuv420p",
                              "level":  40,
                              "color_range":  "tv",
                              "color_space":  "bt709",
                              "color_transfer":  "bt709",
                              "color_primaries":  "bt709",
                              "chroma_location":  "left",
                              "field_order":  "progressive",
                              "refs":  1,
                              "is_avc":  "true",
                              "nal_length_size":  "4",
                              "id":  "0x2",
                              "r_frame_rate":  "120/1",
                              "avg_frame_rate":  "27405000/2324909",
                              "time_base":  "1/90000",
                              "start_pts":  0,
                              "start_time":  "0.000000",
                              "duration_ts":  4649818,
                              "duration":  "51.664644",
                              "bit_rate":  "6153719",
                              "bits_per_raw_sample":  "8",
                              "nb_frames":  "609",
                              "extradata_size":  33,
                              "disposition":  {
                                        "default":  1,
                                        "dub":  0,
                                        "original":  0,
                                        "comment":  0,
                                        "lyrics":  0,
                                        "karaoke":  0,
                                        "forced":  0,
                                        "hearing_impaired":  0,
                                        "visual_impaired":  0,
                                        "clean_effects":  0,
                                        "attached_pic":  0,
                                        "timed_thumbnails":  0,
                                        "captions":  0,
                                        "descriptions":  0,
                                        "metadata":  0,
                                        "dependent":  0,
                                        "still_image":  0
                              },
                              "tags":  {
                                        "creation_time":  "2024-02-23T13:46:34.000000Z",
                                        "language":  "eng",
                                        "handler_name":  "VideoHandle",
                                        "vendor_id":  "[0][0][0][0]"
                              }
                    }
          ],
          "format":  {
                    "filename":  "1gO0elRuVp8xPw2wvIP3VwZx3Zo.mp4",
                    "nb_streams":  2,
                    "nb_programs":  0,
                    "format_name":  "mov,mp4,m4a,3gp,3g2,mj2",
                    "format_long_name":  "QuickTime / MOV",
                    "start_time":  "0.000000",
                    "duration":  "51.802500",
                    "size":  "40591710",
                    "bit_rate":  "6268687",
                    "probe_score":  100,
                    "tags":  {
                              "major_brand":  "mp42",
                              "minor_version":  "0",
                              "compatible_brands":  "isommp42",
                              "creation_time":  "2024-02-23T13:46:34.000000Z",
                              "com.android.version":  "12"
                    }
          }
}
```


## 4.2、视频转码

## 4.3、水印转码

## 4.4、超分

## 4.5、截帧

## 4.6、超清

## 4.7、人像增强
