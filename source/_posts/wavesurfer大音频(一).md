---
title: wavesurfer 处理大音频文件波形渲染(一)
index_img: /img/wavesurfer/wavesurfer.jpg
tags: JavaScript三方库
date: 2019-07-29
---

wavesurfer 是一个利用Web Audio API和Canvas进行交互式导航音频可视化三方库。
<!-- more -->

#### 背景: 

通常用 wavesurfer 来做音轨波形的渲染。笔者目前所做的项目中用到 wavesurfer 来实现了对音频截取段落的分析标注，实现过程中发现大音频（**100M以上wav文件**）的加载和渲染会占用过多的浏览器内存，导致浏览器崩溃。针对此现象，调研过目前能寻找到的解决方案，例如:

* [基于wavesurfer.js的超大音频的渐进式请求实现](https://www.cnblogs.com/webhmy/p/10175785.html)

由于服务端小伙伴业务繁忙不能支持原因没能够尝试此方案，所以没有进行验证是否可行。最终阅读了部分的 wavesurfer 源码之后，采用了一种类似的分段加载方式解决了目前的问题。

---

#### 采用音频分段加载方式的主要流程:

1. 主动发起请求wav资源的前100字节
2. 根据前100字节内容拼接出当前wav资源文件的头信息
3. 按照一定的字节范围顺序请求当前的wav资源，每请求一段就拼接上之前获取的头信息，处理成 wavesurfer 可以操作的 buffer
4. wavesurfer 处理之前获取到的每一小段的 buffer 产生每一小段的波形信息
5. 当wav资源的所有字节都被请求到，并且 buffer 也都被 wavesurfer 处理完毕成波形信息，拼接所有请求段的波形信息，交给 wavesurfer 进行渲染，在渲染的同时，生成波形信息文件上传到服务端保存，下次再获取相同的wav资源就直接获取波形信息文件，避免重复的 decode

---

### 以下对之前的流程步骤做出解释:

在此之前需要了解wav资源是什么(基础知识)  

* [参考1](https://blog.csdn.net/mlkiller/article/details/12567139)
* [参考2](https://baike.baidu.com/item/WAV#1)

#### 为什么第一次请求前100字节

注意以上参考文中提到一个wav资源文件的数据信息 ```从 50H 位开始就是真正的数据部分(16进制)```，所以获取资源的100字节就可以截取到一个完整的头部信息

由于第3步的时候需要调用 wavesurfer 方法进行 decode，在这个过程中必须要是一个完整的音频文件，否则会 decode 失败。

那么分段加载的音频，除了第一段请求结果会带有wav资源的头信息之外，之后其他的所有请求段落都不会自动带有头信息，即非完整音频文件。所以就需要拿到当前wav资源的头信息拼接到之后所有的请求段前形成一个"完整"的音频文件，让 wavesurfer 正常进行 decode 产生当前请求段的波形信息。

#### 如何从前100字节获取到头信息

[参考2](https://baike.baidu.com/item/WAV#1) 中文章介绍到 ```48H ~ 4BH 64 61 74 71 对应的 ACSCII 码是 data ``` 以及 ```从 50H 开始就是真正的数据部分```，并且 ```data``` 对应的数据信息永远都是 ```64 61 74 71 (对应的10进制是 100 97 116 97)```，所以就可以以此为目标点获取真正数据前面的数据信息就是完整的头部信息。

```js
import axios from 'axios';

function getWavHeaderInfo (buffer) {
    const firstBuffer = buffer;
    // 创建数据视图来操作数据
    const dv = new DataView(firstBuffer);
    // 从数据视图拿出数据
    const dvList = [];
    for (let i = 0, len = dv.byteLength; i < len; i += 1) {
        dvList.push(dv.getInt8(i));
    }
    // 找到头部信息中的data字段位置
    const findDataIndex = dvList.join().indexOf(',100,97,116,97');
    if (findDataIndex <= -1) {
        throw new Error('解析失败');
    }
    // data 字段之前所有的数据信息字符串
    const dataAheadString = dvList.join().slice(0, findDataIndex);
    //  data 字段之后 8位 是 头部信息的结尾
    const headerEndIndex = dataAheadString.split(',').length + 7;
    // 截取全部的头部信息
    const headerBuffer = firstBuffer.slice(0, headerEndIndex + 1);

    return headerBuffer;
}

const requestOptions = {
    url: '音频url地址',
    method: 'get',
    responseType: 'arraybuffer', // 请求wav信息格式
    headers: {
        Range: `bytes=0-99`, // 前100字节
    },
};

axios(requestOptions).then(response => {
    const headerBuffer = getWavHeaderInfo(response.data);
})
```

#### 拼接每一段的请求结果和已经得到的头部信息

之前已经通过第一段请求获取到了当前资源的头部信息，接下来就是按照顺序一段一段请求剩余的资源去拼接头部信息，然后交给 wavesurfer 去进行 decode

(PS: 以下代码示例采用 Class 写法)

```js
import axios from 'axios';
import _ from 'lodash';

class requestWav {
    constructor() {
    }

    defaultOptions = {
        responseType: 'arraybuffer',
        rangeFirstSize: 100,
        rangeSize: 1024000 * 2,
        requestCount: 1, // 请求个数
        loadRangeSucess: null, // 每一段请求之后完成的回调
        loadAllSucess: null, // 全部加载完成之后的回调
    }

    // 合并配置
    mergeOptions(options) {
        this.options = Object.assign({}, this.defaultOptions, options);
    }

    loadBlocks(url, options) {
        if (!url || !_.isString(url)) {
            throw new Error('Argument [url] should be supplied a string');
        }
        this.mergeOptions(options);
        this.url = url;
        this.rangeList = [];
        this.rangeLoadIndex = 0; // 当前请求的索引
        this.rangeLoadedCount = 0; // 当前已经请求个数
        // 先取 100 个字节长度的资源以便获取资源头信息
        this.rangeList.push({ start: 0, end: this.options.rangeFirstSize - 1 });
        this.loadFirstRange();
    }

    // 真正发起请求
    requestRange(rangeArgument) {
        const range = rangeArgument;
        const { CancelToken } = axios;
        const requestOptions = {
            url: this.url,
            method: 'get',
            responseType: this.options.responseType,
            headers: {
                Range: `bytes=${range.start}-${range.end}`,
            },
            // 配置一个可取消请求的扩展
            cancelToken: new CancelToken((c) => {
                range.cancel = c;
            }),
        };
        return axios.request(requestOptions);
    }

    fileBlock(fileSize) {
        // 根据文件头所带的文件大小 计算需要请求的每一段大小 范围队列
        let rangeStart = this.options.rangeFirstSize;
        for (let i = 0; rangeStart < fileSize; i += 1) {
            const rangeItem = {
                start: rangeStart,
                end: rangeStart + this.options.rangeSize - 1,
            };
            this.rangeList.push(rangeItem);
            rangeStart += this.options.rangeSize;
        }
    }

    // 请求第一段
    loadFirstRange() {
        this.requestRange(this.rangeList[this.rangeLoadIndex]).then((response) => {
            const fileRange = response.headers['content-range'].match(/\d+/g);
            // 获取文件大小
            const fileSize = parseInt(fileRange[2], 10);
            // 计算请求块
            this.fileBlock(fileSize);
            // 放置请求结果到请求队列中的data字段
            this.rangeList[0].data = response.data;
            // 获取资源头部信息 (方法同第二步)
            this.headerBuffer = this.getWavHeaderInfo(response.data);
            // 每一段加载完成之后处理回调
            this.afterRangeLoaded();
            // 加载剩下的
            this.loadOtherRanges();
        }).catch((error) => {
            throw new Error(error);
        });
        this.rangeLoadIndex += 1;
    }

    afterRangeLoaded() {
        this.rangeLoadedCount += 1;
        // 每一次请求道数据之后 判断当前请求索引和应当请求的所有数量
        // 触发 loadRangeSucess 和 loadAllSucess 回调
        if (this.rangeLoadedCount > 1
            && this.options.loadRangeSucess
            && typeof this.options.loadRangeSucess === 'function'
        ) {
            this.contactRangeData(this.options.loadRangeSucess);
        }
        if (this.rangeLoadedCount >= this.rangeList.length
            && this.options.loadAllSucess
            && typeof this.options.loadAllSucess === 'function'
        ) {
            this.options.loadAllSucess();
            this.rangeList = [];
        }
    }

    loadOtherRanges() {
        // 循环请求范围队列
        if (this.rangeLoadIndex < this.rangeList.length) {
            this.loadRange(this.rangeList[this.rangeLoadIndex]);
        }
    }

    loadRange(rangeArgument) {
        const range = rangeArgument;
        this.requestRange(range).then((response) => {
            // 放置请求结果到请求队列中的data字段
            range.data = response.data;
            this.afterRangeLoaded();
            this.loadOtherRanges();
        }).catch((error) => {
            throw new Error(error);
        });
        this.rangeLoadIndex += 1;
    }

    contactRangeData(callback) {
        const blobIndex = this.rangeLoadIndex - 1;
        if (!this.headerBuffer) {
            return;
        }
        // 从请求队列中的每一个data获取数据，拼接上已经有的header头信息，保存为 audio/wav blob文件
        const blob = new Blob(
            [this.headerBuffer,
            this.rangeList[blobIndex].data],
            { type: 'audio/wav' }
        );
        const reader = new FileReader();
        // 将blob读取为 buffer 交给 loadRangeSucess 回调
        reader.readAsArrayBuffer(blob);
        reader.onload = () => {
            callback(reader.result, blobIndex);
        };
        reader.onerror = () => {
            throw new Error(reader.error);
        };
    }

    destroyRequest() {
        // 销毁请求，使用场景是:
        // 如果当前的资源没有加载完成，此时更换了资源的URL地址，应该取消之前设定的请求,避免浪费请求资源
        if (!this.rangeList) {
            return;
        }
        this.rangeList.forEach((rang) => {
            if (rang.cancel) {
                rang.cancel('取消音频下载');
            }
        });
    }
}

export default new requestWav();
```

至此前三步已经完成，拿到了所有的音频段落，并且拼装成了"完整"的音频，读取到了相对应的buffer。[下一篇](/2019/08/02/wavesurfer大音频(二)/)将继续完善第四步和第五步分段将获取到的每一段的 buffer 去产生波形以及最终拼装所有波形文件。
