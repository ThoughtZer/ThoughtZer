---
title: wavesurfer 处理大音频文件波形渲染(二)
index_img: /img/wavesurfer/wavesurfer.jpg
tags: JavaScript三方库
---

wavesurfer 是一个利用Web Audio API和Canvas进行交互式导航音频可视化三方库。
<!-- more -->

在[上一篇](/2019/08/02/wavesurfer大音频(一)/)文章中，我们已经获取到所有的"完整"音频段落，接下来就要利用这些"完整"的音频段落，进行音频分段加载的**最后两步操作**:

4. wavesurfer 处理之前获取到的每一小段的 buffer 产生每一小段的波形信息
5. 当wav资源的所有字节都被请求到，并且 buffer 也都被 wavesurfer 处理完毕成波形信息，拼接所有请求段的波形信息，交给 wavesurfer 进行渲染，在渲染的同时，生成波形信息文件上传到服务端保存，下次再获取相同的wav资源就直接获取波形信息文件，避免重复的 decode

#### 如何让 wavesurfer 拥有只产生波形信息的能力

wavesurfer 是没有想外提供一个产出波形信息 Peaks 的方法的，所以就需要一点点技巧~

```js
/**
 * Get the correct peaks for current wave view-port and render wave
 *
 * @private
 * @emits WaveSurfer#redraw
 */
drawBuffer() {
    const nominalWidth = Math.round(
        this.getDuration() *
            this.params.minPxPerSec *
            this.params.pixelRatio
    );
    const parentWidth = this.drawer.getWidth();
    let width = nominalWidth;
    // always start at 0 after zooming for scrolling : issue redraw left part
    let start = 0;
    let end = Math.max(start + parentWidth, width);
    // Fill container
    if (
        this.params.fillParent &&
        (!this.params.scrollParent || nominalWidth < parentWidth)
    ) {
        width = parentWidth;
        start = 0;
        end = width;
    }

    let peaks;
    if (this.params.partialRender) {
        /* something */
    } else {
        peaks = this.backend.getPeaks(width, start, end);
        this.drawer.drawPeaks(peaks, width, start, end);
    }
    this.fireEvent('redraw', peaks, width);
}
```

以上代码来自 [wavesurfer github 源码](https://github.com/katspaugh/wavesurfer.js/blob/832e114b7be6436458fc351a57699ba169d08676/src/wavesurfer.js#L1130-L1183)，可以看到的是 wavesurfer 在 draweBuffer 过程中得到当前正确的 peaks 信息，是根据当前的渲染容器宽度、minPxPerSec、pixelRatio和资源时长来控制 getPeaks 方法的 width、start、end参数，然后调用 drawer 的 drawPeaks 方法绘制。

根据以上分析以及我们的需求仅仅是得到 peaks 波形信息，所以我们需要借用上面的代码扩展一下 wavesurfer 方法:

```js
// 扩展到 WaveSurfer 构造器中方法
// 每一段音频 buffer 产生 peaks 方法
getPeaks(arraybuffer, callback) {
    this.backend.decodeArrayBuffer(
        arraybuffer,
        buffer => {
            if (!this.isDestroyed) {
                // https://github.com/katspaugh/wavesurfer.js/blob/832e114b7be6436458fc351a57699ba169d08676/src/wavesurfer.js#L1395-L1396
                // decodeArrayBuffer 之后的一个赋值、置空操作。完全模仿
                this.backend.buffer = buffer;
                this.backend.setPeaks(null);
                const nominalWidth = Math.round(
                    this.getDuration() *
                        this.params.minPxPerSec *
                        this.params.pixelRatio
                );
                const parentWidth = this.drawer.getWidth();
                let width = nominalWidth;
                let start = 0;
                // 此处谨记 end 一定要赋值为 width
                // 原本的 let end = Math.max(start + parentWidth, width) 是比较了容器宽度和根据音频时长等计算出的长度，取最大值。
                // 那么会在当前的音频分段时长大小(例子是2M音频的时长)所能产生的波形长度小于容器的宽度时
                // 出现为了充满容器下面的 this.backend.getPeaks 方法在实际产生的波形信息后面添加不等位数的 0，从而充满容器。
                // 但是整个大音频的时长是固定的，根据大音频时长设定的canvas的个数和宽度已经固定
                // 如果分段加载之后最后一段如果出现被补0的情况，在最终合并的完整的波形信息就会超过原本设定的预值，导致挤压最终产生的波形
                let end = width;

                if (
                    this.params.fillParent
                    && (!this.params.scrollParent || nominalWidth < parentWidth)
                ) {
                    width = parentWidth;
                }

                const peaks = this.backend.getPeaks(width, start, end);
                // 通过回调函数的方式把 peaks 传递出去
                callback(peaks);
                // 清空 arraybuffer 避免占用过多内存
                this.arraybuffer = null;
                this.backend.buffer = null;
            }
        },
        () => this.fireEvent('error', 'Error decoding audiobuffer')
    );
}

// 加载所有的波形信息产生可视化canvas
loadPeaks(peaks) {
    this.backend.buffer = null;
    this.backend.setPeaks(peaks);
    this.drawBuffer();
    this.fireEvent('waveform-ready');
    this.isReady = true;
}
```

增强了 wavesurfer 的能力之后就需要在业务中调用了~~

#### 调用扩展能力，整合波形信息，渲染并上传保存

```js
import _ from 'lodash';
import pako from 'pako'; // JS压缩以及解压缩三方库
import WaveSurfer from 'wavesurfer.js';
import requestWav from 'requestWav';

const waveSurfer = null;
const peaksList = [];
const texture = null;

function initWaveSurfer() {
    const options = {
        container: '#waveform',
        backend: 'MediaElement',
        fillParent: false, // 重要
        height: 200,
        barHeight: 10,
        normalize: true,
        minPxPerSec: 100,
    }
    waveSurfer = WaveSurfer.create(options);
    renderWaveSurfer();
}

function renderWaveSurfer() {
    waveSurfer.load(source, [], 'none');
    if (!texture) {
        decodePeaks();
    }
}

function decodePeaks() {
    const that = this;
    requestWav.loadBlocks('音频Url', {
        loadRangeSucess(data, index) {
            // 每一段加载完成之后的回调
            peaksList[index - 1] = [];
            // 调用扩展的 waveSurfer 方法获取每一段音频的 peaks
            waveSurfer.getPeaks(data, (peaks) => {
                peaksList[index - 1] = peaks;
            });
        },
        loadAllSucess() {
            // 所有都加载完之后的回调
            let texture = _.flatten(peaksList); // peaksList 降维
            if (!texture) {
                return;
            }
            // 按照一定等级进行压缩 (减少传输时间，但是同时需要之后在下载使用波形信息的时候解压)
            waveSurfer.texture = pako.deflate(JSON.stringify(texture), { level: 9, to: 'string' });
            // 解压和压缩的方法是相反的
            // const texture = pako.deflate(JSON.stringify(waveSurfer.texture), { level: 9, to: 'string' });

            // 手动置空变量, 避免占用内存过大
            texture = null;
            // 创建上传 FormData
            const peaksFile = new FormData();
            peaksFile.append('sourceUrl', 音频的URL地址);
            // 创建上传文件 Blob
            const blob = new Blob([waveSurfer.texture], { type: 'application/json' });
            // 赋值文件名、文件内容
            peaksFile.append('sourcePeaks', blob, 'sourcePeaks');
            axios({
                method: 'post',
                url: '上传地址',
                data: peaksFile,
                headers: {
                    'Content-Type': 'multipart/form-data',
                },
                timeout: 1000000, // 防止文件过大上传超时，当然不设置也可
            });
        },
    });
}
```

至此 5个步骤全部完成，剩下的只有在二次请求相同资源的时候判断如果已经存储了当前wav资源的波形信息就不用再一次执行一次 decode 产生波形的操作。
