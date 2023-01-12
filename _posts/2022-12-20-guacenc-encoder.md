---
layout: post
title: guacenc 支持导出自定义格式的视频
date: 2022-12-20
categories: blog
tags: [guacamole,ffmpeg]
description: guacenc是官方提供的一个工具，通过修改其源代码实现导出指定编码格式的视频
---

## guacenc 支持导出自定义格式的视频
工作原因需要用到guacamole来录屏，并且需要转换成mp4格式导出，但是发现官方提供的导出工具仅支持m4v格式，虽然这个也是mp4的一种，但是并不是想要的。

官方的代码中可以看到, 初始化编码器的函数中，硬编码了mpeg4的编码器，这个编码器的支持列表可以在[ffmpeg官网](https://ffmpeg.org/ffmpeg-codecs.html)查到

```c
// from https://github.com/apache/guacamole-server/blob/be9041fefd9c1f7c647845aa0709caedcb54e812/src/guacenc/guacenc.c#L121

if (guacenc_encode(path, out_path, "mpeg4",width, height, bitrate, force)) {
    failures++;
    guacenc_log(GUAC_LOG_DEBUG,"%s was NOT successfully encoded.", path);
}

// from https://github.com/apache/guacamole-server/blob/be9041fefd9c1f7c647845aa0709caedcb54e812/src/guacenc/video.c#L66
/* Pull codec based on name */
    AVCodec* codec = avcodec_find_encoder_by_name(codec_name);
```

所以解决方案也很简单，只需要增加一个命令行参数，将`mpeg4`作为启动参数传进去即可。同时为了解决硬编码的后缀名，还需要增加一个-o参数指定输出文件的名字。

只需要修改guacenc.c文件即可，完整代码如下

```c
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

#include "config.h"

#include "encode.h"
#include "guacenc.h"
#include "log.h"
#include "parse.h"

#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

#include <getopt.h>
#include <stdbool.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char* argv[]) {

    int i;

    /* Load defaults */
    bool force = false;
    int width = GUACENC_DEFAULT_WIDTH;
    int height = GUACENC_DEFAULT_HEIGHT;
    int bitrate = GUACENC_DEFAULT_BITRATE;

    /* -o */
    char out_path[4096];
    char encoder_name[64] = {"mpeg4"}; // see https://ffmpeg.org/ffmpeg-codecs.html

    /* Parse arguments */
    int opt;
    while ((opt = getopt(argc, argv, "e:s:r:o:f")) != -1) {
        switch (opt)
        {
        case 'e':
            strncpy(encoder_name,optarg,sizeof(encoder_name)-1);
            break;
        case 's':
            if (guacenc_parse_dimensions(optarg, &width, &height)) {
                guacenc_log(GUAC_LOG_ERROR, "Invalid dimensions.");
                goto invalid_options;
            }
            break;
        case 'r':
            if (guacenc_parse_int(optarg, &bitrate)) {
                guacenc_log(GUAC_LOG_ERROR, "Invalid bitrate.");
                goto invalid_options;
            }
            break;     
        case 'o':
            strncpy(out_path,optarg,sizeof(out_path)-1);
            break;                   
        case 'f':
            force = true;
            break;            
        default:
            goto invalid_options;
        }

    }

    /* Log start */
    guacenc_log(GUAC_LOG_INFO, "Guacamole video encoder (guacenc) "
            "version " VERSION);

#if LIBAVCODEC_VERSION_INT < AV_VERSION_INT(58, 10, 100)
    /* Prepare libavcodec */
    avcodec_register_all();
#endif

#if LIBAVFORMAT_VERSION_INT < AV_VERSION_INT(58, 9, 100)
    av_register_all();
#endif

    /* Track number of overall failures */
    int total_files = argc - optind;
    int failures = 0;

    /* Abort if no files given */
    if (total_files <= 0) {
        guacenc_log(GUAC_LOG_INFO, "No input files specified. Nothing to do.");
        return 0;
    }

    guacenc_log(GUAC_LOG_INFO, "%i input file(s) provided.", total_files);

    guacenc_log(GUAC_LOG_INFO, "Video will be encoded at %ix%i "
            "and %i bps, with encoder %s.", width, height, bitrate,encoder_name);

    /* Encode all input files */
    for (i = optind; i < argc; i++) {

        /* Get current filename */
        const char* path = argv[i];

        /* Attempt encoding, log granular success/failure at debug level */
        if (guacenc_encode(path, out_path, encoder_name,
                    width, height, bitrate, force)) {
            failures++;
            guacenc_log(GUAC_LOG_DEBUG,
                    "%s was NOT successfully encoded.", path);
        }
        else
            guacenc_log(GUAC_LOG_DEBUG, "%s was successfully encoded.", path);

    }

    /* Warn if at least one file failed */
    if (failures != 0)
        guacenc_log(GUAC_LOG_WARNING, "Encoding failed for %i of %i file(s).",
                failures, total_files);

    /* Notify of success */
    else
        guacenc_log(GUAC_LOG_INFO, "All files encoded successfully.");

    /* Encoding complete */
    return 0;

    /* Display usage and exit with error if options are invalid */
invalid_options:

    fprintf(stderr, "USAGE: %s"
            " [-e FFMPEGENCODER]"
            " [-s WIDTHxHEIGHT]"
            " [-r BITRATE]"
            " [-o OUTPUT]"
            " [-f]"
            " [FILE]...\n", argv[0]);

    return 1;

}
```