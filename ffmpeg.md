# FFmpeg学习笔记

## 语法

- 常规

```shell
ffmpeg -i a.avi -preset ultrafast -crf 18 -c:v libx265 -c:a aac a.mp4
```

- 批处理

```shell
for f in `ls *.avi`; do ffmpeg -i f -preset ultrafast -crf 18 -c:v libx265 -c:a aac ${f%.*}.mp4; done;
```

- 合并

```shell
# 先建立filelist.txt文件，文件内每个`file "a.avi"`占一行，有多少填多少，然后执行
ffmpeg -f concat -i filelist.txt -c copy a.mp4
```