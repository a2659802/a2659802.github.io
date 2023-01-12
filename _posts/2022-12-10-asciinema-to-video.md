
---
layout: post
title: asciinema录像转mp4方案
date: 2022-12-10
categories: blog
tags: [asciinema,ffmpeg]
description: 用官方提供的agg项目+ffmpeg即可实现
---

## asciinema录像转mp4脚本

因工作原因需要将asciinema的录像转为mp4，经搜索发现asciinema官方提供了一个agg项目可以将录像文件转为gif.

所以我们只需要将gif转为mp4就能达成目的，这里使用ffmpeg即可

转换脚本如下：将其保存为cast2video，然后执行`./cast2video xxx.cast xxx.mp4`即可

```bash
#!/bin/bash

#set -ex

# usage: cast2video <cast_file> <output>
function print_help()
{
	echo "usage: $0 <cast_file> <output>"
}
function error_usage()
{
	echo "missing required argument"
	print_help 
	exit 1 
}

SRC_FILE=$1
DST_FILE=$2

# verify args 
if [ ! $SRC_FILE ];then 
	error_usage 
fi
if [ ! $DST_FILE ];then 
	error_usage
fi 


tmp_gif="/tmp/$(basename $SRC_FILE).gif"

# convert cast to gif 
agg "${SRC_FILE}" "${tmp_gif}" --font-family "JetBrains Mono,Fira Code,SF Mono,Menlo,Consolas,DejaVu Sans Mono,Liberation Mono,FreeMono"
# convert gif to mp4 
ffmpeg -y -i "${tmp_gif}" -movflags faststart -pix_fmt yuv420p -vf "scale=trunc(iw*1.4/2)*2:trunc(ih*1.4/2)*2" "${DST_FILE}"
convert_result=$?

# delete tmp gif file 
rm -f "${tmp_gif}"

if [ $convert_result -ne 0 ];then 
	echo "convert failed" >&2
	exit $convert_result
fi

exit 0 
```

**注：不支持中文等双宽的字符，这是agg项目所限制的. 详见[issue](https://github.com/asciinema/agg/issues/10)**