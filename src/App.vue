<template>
    <div id="verify">
        <h2>请<span style="color:#f56a00">拖动</span>左侧滑块完成拼图</h2>
        <div class="paperWrap">
            <div class="canvas-wrap">
                <canvas class="paper" ref="paperRef" :width="canvasWidth" :height="canvasHeight"></canvas>
                <canvas class="block" ref="blockRef" :width="canvasWidth" :height="canvasHeight"></canvas>
            </div>
        </div>
        <div class="sliderWrap">
            <div class="progress-wrap">
                <div ref="innerRef" class="inner-bar"></div>
                <div ref="buttonRef" class="button" @mousedown="down">
                    <img src="./assets/verify/arrow.svg" alt="">
                </div>
            </div>
        </div>
        <div class="buttonWrap">
            <img src="./assets/verify/question.svg" alt="">
            <img src="./assets/verify/reload.svg" alt="" @click="fresh">
        </div>
    </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
const canvasWidth = 350; // 验证图片的大小
const canvasHeight = 160;
const canvasWidthStyle = `${canvasWidth}px`; // canvas的动态样式
const canvasHeightStyle = `${canvasHeight}px`;

const paperX = ref(0);  // 拼图块的初始x轴位置
const paperY = ref(0); // 拼图块的初始y轴位置
const transformLeft = 120; // 拼图块距离背景左侧的距离
const paperDeviation = ref(0); // 拼图块随机的x轴偏移量
const blockSize = ref(8); // 拼图块的尺寸
const paperRef = ref(null);  // 拼图背景的Ref引用
const blockRef = ref(null);  // 拼图块的Ref引用

const slideX = ref(0);   // 按下滑块时鼠标距离左侧的距离
const isDown = ref(false); // 是否按下拖动按钮
const curX = ref(0); // 鼠标距离左侧的距离
const slideDiff = ref(0); // 鼠标移动的距离
const innerRef = ref(null); // 滑块左侧黑色区域的Ref引用
const buttonRef = ref(null); // 滑块实例

// 拼图块的位置 用户移动的距离 + 初始偏移的距离 + 120
const diffStyle = computed(() => `translateX(${slideDiff.value + paperDeviation.value - canvasWidth + transformLeft}px)`);

const bgList = [
    new URL(`./assets/verify/bg/1.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/2.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/3.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/4.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/5.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/6.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/7.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/8.jpg`, import.meta.url).href,
    new URL(`./assets/verify/bg/9.jpg`, import.meta.url).href,
]

/**
* 绘制拼图块的形状 并根据填充方式opt选择 填充渲染 或 剪切渲染
* @param {CanvasRenderingContext2D} ctx 绘制图形的canvas的上下文对象
* @param {'fill'|'clip'} opt 填充方式 fill -> 渲染拼图块形状外背景 | clip -> 只渲染拼图块内背景
*/
const draw = (ctx, opt) => {
    const size = blockSize.value;
    ctx.beginPath();
    ctx.moveTo(paperX.value, paperY.value);
    ctx.lineTo(paperX.value + size, paperY.value);

    //绘制上方弧形
    ctx.arc(paperX.value + (2.5 * size), paperY.value, size, deg2arc(-180), deg2arc(0));
    ctx.lineTo(paperX.value + (5 * size), paperY.value);
    ctx.lineTo(paperX.value + (5 * size), paperY.value + size);
    //绘制右侧弧形
    ctx.arc(paperX.value + (5 * size), paperY.value + (2.5 * size), size, deg2arc(-90), deg2arc(90));
    ctx.lineTo(paperX.value + (5 * size), paperY.value + (5 * size));
    ctx.lineTo(paperX.value, paperY.value + (5 * size));
    ctx.lineTo(paperX.value, paperY.value + (4 * size));
    //绘制左侧弧形,true表示逆时针绘制
    ctx.arc(paperX.value, paperY.value + (2.5 * size), size, deg2arc(90), deg2arc(-90), true);
    ctx.lineTo(paperX.value, paperY.value);

    ctx.lineWidth = 2;
    ctx.fillStyle = "rgba(255, 255, 255, 1)";
    ctx.strokeStyle = "rgba(255, 255, 255, 1)";
    ctx.stroke();
    ctx[opt]();
    ctx.globalCompositeOperation = "xor";
}

//角度转弧度
const deg2arc = (deg) => {
    return deg / 180 * Math.PI;
}

/**
* 获得一个最大值和最小值之间的随机数 未取整
* @param {number} min 最小值
* @param {number} max 最大值
* @returns {number} 随机数
*/
const getRandom = (min, max) => {
    let c = max - min + 1;
    return Math.random() * c + min;
}

//按下按钮处理
const down = (e) => {
    isDown.value = true;
    slideX.value = e.pageX;
    document.addEventListener("mousemove", move);
    e.preventDefault();
    e.stopPropagation();
}

//移动按钮处理
const move = (e) => {
    const maxDiff = canvasWidth - blockSize.value * 6 - 20;
    if (isDown.value) {
        curX.value = e.pageX;
        slideDiff.value = (curX.value - slideX.value) < -2 ? 0 : (curX.value - slideX.value) > maxDiff ? maxDiff : curX.value - slideX.value;
        if (slideDiff.value > -2 && slideDiff.value < (maxDiff + 20)) {
            innerRef.value.style.width = (slideDiff.value + 20) + 'px';
            buttonRef.value.style.transform = `translateX(${slideDiff.value}px)`;
        }
    }
    document.addEventListener('mouseup', up);
}

//鼠标抬起处理
const up = () => {
    if (isDown.value) {
        const shouldVerify = slideDiff.value > 50; // 移动超过50像素才进行判断
        if (shouldVerify) {
            const verifySize = 4; // 校验误差
            const transformDistance = canvasWidth - transformLeft - paperDeviation.value;
            if (slideDiff.value > (transformDistance - verifySize) && slideDiff.value < (transformDistance + verifySize)) {
                props.success();
                message('success', '校验成功');
            }
        }

        // 重置滑块的距离
        isDown.value = false;
        slideDiff.value = 0;
        innerRef.value.style.width = 20 + 'px';
        buttonRef.value.style.transform = `translateX(0)`;
        shouldVerify && fresh();
    }
    document.removeEventListener('mousemove', move);
}

const fresh = () => {
    paperDeviation.value = getRandom(0, 150); // 拼图块随机的x轴偏移量
    paperX.value = canvasWidth - transformLeft - paperDeviation.value + 20;  // 拼图块的初始x轴位置 (-偏移量则拼图缺口偏移， 不-偏移量则拼图块偏移)
    paperY.value = getRandom(20, 80); // 拼图块的初始y轴位置

    let img = document.createElement('img');
    img.onload = () => {
        // 重新设置canvas的宽高来刷新canvas (clearRect清除有图形残留)
        paperRef.value.width = canvasWidth;
        paperRef.value.height = canvasHeight;
        blockRef.value.width = canvasWidth;
        blockRef.value.height = canvasHeight;

        let ctx = paperRef.value.getContext('2d');
        let block_ctx = blockRef.value.getContext('2d');

        draw(ctx, 'fill');
        ctx.drawImage(img, 0, 0, canvasWidth, canvasHeight);
        //绘制碎片形状 先画路径，之后再填充图片
        draw(block_ctx, 'clip');
        block_ctx.drawImage(img, 0, 0, canvasWidth, canvasHeight);
    }
    const index = parseInt(getRandom(1, 9));
    img.src = bgList[index];
}

onMounted(() => {
    fresh();
});

onUnmounted(() => {
    document.removeEventListener('mousedown', down);
    document.removeEventListener('mouseup', up);
});
</script>

<style lang="scss">
#verify {
    text-align: center;
    color: #2c3e50;
    width: v-bind(canvasWidthStyle);
    padding: 10px 15px;
    margin: 0 auto;
    border: 1px solid #ccc;
    border-radius: 5px;

    h2 {
        font-size: 16px;
        text-align: start;
        font-weight: lighter;
    }

    .sliderWrap {
        padding: 20px 0 10px;
    }

    .canvas-wrap {
        width: v-bind(canvasWidthStyle);
        height: v-bind(canvasHeightStyle);
        margin: 0 auto;
        position: relative;
        border-radius: 5px;
        overflow: hidden;

        canvas {
            position: absolute;
            top: 0;
            left: 0;
        }

        .block {
            transform: v-bind(diffStyle)
        }
    }

    .progress-wrap {
        height: 30px;
        background-color: #ccc;
        border-radius: 8px;
        position: relative;
        box-sizing: border-box;

        .inner-bar {
            height: 30px;
            background-color: #cffaed;
            border: 1px solid #7af4d0;
            width: 0;
            border-radius: 8px;
            z-index: 1;
            box-sizing: border-box;
        }

        .button {
            position: absolute;
            top: -5px;
            height: 40px;
            transform: translateX(0);
            left: 0;
            width: 60px;
            display: flex;
            z-index: 6;
            border-radius: 8px;
            box-sizing: border-box;
            background: #0da68c;
            cursor: pointer;

            img {
                width: 80%;
                height: 60%;
                margin: auto;
            }
        }
    }

    .buttonWrap {
        display: flex;
        justify-content: flex-end;

        img {
            width: 30px;
            height: 30px;
            margin-left: 5px;
            cursor: pointer;
        }
    }
}
</style>