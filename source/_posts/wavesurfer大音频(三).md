---
title: wavesurfer 处理大音频文件波形渲染(三)
index_img: /img/wavesurfer/wavesurfer.jpg
tags: JavaScript三方库
date: 2020-01-01 21:12:00
---

wavesurfer 是一个利用Web Audio API和Canvas进行交互式导航音频可视化三方库。
<!-- more -->

在之前的文章中[wavesurfer大音频(一)](/2019/07/27/wavesurfer大音频(一)/)、[wavesurfer大音频(二)](/2019/08/02/wavesurfer大音频(二)/)，我们已经能够使用前端的 WebAudio API 对大音频进行分段的解析产生波形文件。但是目前这样的方式仍然不能够满足更大音频的解析，因此继续进行探索，有了这篇总结~

#### 前端 VS 后端

有的时候觉得前端很强大，JS什么都能做，像是要"一统天下"的样子，然而这次的大音频事件，真的是改变了这个看法，在浏览器运行的JS真的是受制于浏览器大多了~

后端运行于服务器，只要有足够大的算力，那么就会有无限的可能~

所以想到了既然浏览器由于内存的限制，那么就把这个消耗内存的操作放到服务端吧~

#### Node

前端工程师的首选后端语言，与其他各种语言的配合也很方便。

#### 三方库

[audiowaveform](https://github.com/bbc/audiowaveform)------一个基于 C++ 实现的可以产出音频波形数据的三方库，一条命令就能产生波形文件: 

`audiowaveform -i 音频文件 -o 要保存的波形文件.json`

上面这条命令也就是本篇文章的核心内容了，一切的 Node 代码都是围绕着它进行的。

#### 主要步骤

emmmmm....想了想，总结的话还是少写点代码，理一个顺序出来吧。

1、首先 audiowaveform 是一个 C++ 的库，产出波形使用的是命令行，所以需要服务器上需要安装 audiowaveform。

2、安装 audiowaveform 之后就可以调用之前的命令来生成 Json 文件。但是也要注意生成 Json 文件之前是要下载音频文件~

3、生成 Json 文件之后要进行一个保存，与音频进行一个一一对应，以便之后相同音频的波形产生，不再进行一次生成操作，再次节省时间

4、读取生成的Json返回给前端浏览器

#### 重要代码

- 由于要使用服务器环境中的 audiowaveform ，所以就要使用命令行工具，Node 中的 `child_process` 子进程。依靠 `child_process` 的 `exec` 方法，可以在 Node 代码中执行 command 命令。

```js
const exec = require('child_process').exec;

const AudioDecodePeaks = async (fileInfo, session) => {
  return new Promise((resolve, reject) => {
    const comand = `audiowaveform -i '${ 文件路径 + 文件名 }.${ 文件扩展名 }' -o ${ 输出路径 + 输出文件名 }.json`;
    child = exec(comand, (err, stdout, stderr) => {
      if (err) {
        reject(err);
      } else {
        resolve(`输出的文件地址`);
      }
    });
  });
};
```

需要注意的是一定要记得获取服务器文件的读写权限，因为 decode 就是一个读和写的操作。

- 读取Json

```js
const fs = require('fs');

fs.readFile(`输出的文件地址`, { encoding: 'utf8' }, (error, file) => {
  if (!error) {
    const fileContent = JSON.parse(file);
    // fileContent 即是可以发送给前端的JSON
  }
})
```

#### 结尾

好像啰嗦了一堆没用的废话...不过既然是使用型文章，就这样吧~ 中间的如何存储一个音频和波形文件关系呀、错误处理呀、日志呀，就不在多说了。这种方案是在线上奔跑着的最终版了，感兴趣的小伙伴可以尝试一下，针对于600M左右的音频毫无压力，当然压力在于服务端去下载音频那了，所以我们采用的是内网下载，用完即删除。
