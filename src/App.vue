<template>
    <div id="verify">
        <div class="paperWrap">
            <div class="canvas-wrap">
                <div class="mask" v-show="!canRefresh && !slideDiff"><img src="./assets/verify/icons/load.svg" alt=""
                        @click="fresh"></div>
                <canvas class="paper" ref="paperRef" :width="canvasWidth" :height="canvasHeight"></canvas>
                <canvas class="block" ref="blockRef" :width="canvasWidth" :height="canvasHeight"></canvas>
                <div class="buttonWrap">
                    <img src="./assets/verify/icons/question.svg" alt="">
                    <img src="./assets/verify/icons/reload.svg" alt="" @click="fresh">
                </div>
            </div>
        </div>
        <div class="sliderWrap">
            <div class="progress-wrap">
                <p class="placeHolder" v-show="!slideDiff">向右拖动滑块完成拼图</p>
                <div ref="innerRef" :class="['inner-bar', status]"></div>
                <div ref="buttonRef" :class="['button', status]" @mousedown="down">
                    <img :src="pathList['icon'][status]" alt="" draggable="false">
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

const canvasWidth = 350; // 验证背景图片和canvas的大小
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
const currentIdx = ref(0) // 当前背景index 防止刷新多次出现重复背景

const isDown = ref(false); // 是否按下拖动按钮
const slideX = ref(0);   // 按下滑块时鼠标距离左侧的距离
const curX = ref(0); // 鼠标距离左侧的距离
const slideDiff = ref(0); // 鼠标移动的距离
const innerRef = ref(null); // 滑块左侧已划过区域的Ref引用
const buttonRef = ref(null); // 滑块实例

const canRefresh = ref(true); // 背景图片加载是否完成
const status = ref('default') // 校验状态 默认 -> default | 成功 -> success | 失败 -> error 

// 拼图块的位移样式 用户移动的距离 + 随机初始偏移的距离 + 拼图块初始距离背景左侧的距离 - 背景总宽度
const diffStyle = computed(() => `translateX(${slideDiff.value + paperDeviation.value - canvasWidth + transformLeft}px)`);

// 默认 成功 失败 icon、背景图片的路径
const pathList = {
    icon: {
        default: new URL(`./assets/verify/icons/arrow.svg`, import.meta.url).href,
        success: new URL(`./assets/verify/icons/success.svg`, import.meta.url).href,
        error: new URL(`./assets/verify/icons/fail.svg`, import.meta.url).href,
    },
    bg: [
        new URL(`./assets/verify/bg/1.jpg`, import.meta.url).href,
        new URL(`./assets/verify/bg/2.jpg`, import.meta.url).href,
        new URL(`./assets/verify/bg/3.jpg`, import.meta.url).href,
        new URL(`./assets/verify/bg/4.jpg`, import.meta.url).href,
        new URL(`./assets/verify/bg/5.jpg`, import.meta.url).href,
        new URL(`./assets/verify/bg/6.jpg`, import.meta.url).href,
        new URL(`./assets/verify/bg/7.jpg`, import.meta.url).href,
        new URL(`./assets/verify/bg/8.jpg`, import.meta.url).href,
    ]
}

/**
* 绘制拼图块的形状 并根据填充方式opt选择 填充渲染 或 剪切渲染
* @param {CanvasRenderingContext2D} ctx 绘制图形的canvas的上下文对象
* @param {'fill'|'clip'} opt 填充方式 fill -> 渲染拼图块形状外背景 | clip -> 只渲染拼图块内背景
*/
const draw = (ctx, opt) => {
    ctx.beginPath();
    ctx.moveTo(paperX.value, paperY.value);
    ctx.lineTo(paperX.value + blockSize.value, paperY.value);

    //绘制上方弧形
    ctx.arc(paperX.value + (2.5 * blockSize.value), paperY.value, blockSize.value, deg2arc(-180), deg2arc(0));
    ctx.lineTo(paperX.value + (5 * blockSize.value), paperY.value);
    ctx.lineTo(paperX.value + (5 * blockSize.value), paperY.value + blockSize.value);
    //绘制右侧弧形
    ctx.arc(paperX.value + (5 * blockSize.value), paperY.value + (2.5 * blockSize.value), blockSize.value, deg2arc(-90), deg2arc(90));
    ctx.lineTo(paperX.value + (5 * blockSize.value), paperY.value + (5 * blockSize.value));
    ctx.lineTo(paperX.value, paperY.value + (5 * blockSize.value));
    ctx.lineTo(paperX.value, paperY.value + (4 * blockSize.value));
    //绘制左侧弧形,true表示逆时针绘制
    ctx.arc(paperX.value, paperY.value + (2.5 * blockSize.value), blockSize.value, deg2arc(90), deg2arc(-90), true);
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
    return Math.random() * (max - min + 1) + min;
}

//按下按钮处理
const down = (e) => {
    if (!canRefresh.value) return;
    isDown.value = true;
    slideX.value = e.pageX;
    e.preventDefault();
    e.stopPropagation();
    document.addEventListener("mousemove", move);
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
    // 滑块位移后才进行校验判断
    if (!slideDiff.value) {
        return document.removeEventListener('mousemove', move);
    }
    canRefresh.value = false;

    const verifySize = 4; // 校验误差范围
    const transformDistance = canvasWidth - transformLeft - paperDeviation.value;
    if (slideDiff.value > (transformDistance - verifySize) && slideDiff.value < (transformDistance + verifySize)) {
        status.value = 'success';
        // 校验成功
    } else {
        status.value = 'error';
        // 校验失败
    }
    document.removeEventListener('mousemove', move);

    setTimeout(() => {
        status.value = 'default';
        // 重置滑块的距离
        isDown.value = false;
        slideDiff.value = 0;
        innerRef.value.style.width = 20 + 'px';
        buttonRef.value.style.transform = `translateX(0)`;
        fresh();
    }, 500);
}

const fresh = () => {
    canRefresh.value = false;

    paperDeviation.value = getRandom(0, 150); // 拼图块随机的x轴偏移量
    paperX.value = canvasWidth - transformLeft - paperDeviation.value + 20;  // 拼图块的初始x轴位置 (减偏移量则拼图缺口偏移， 不减偏移量则拼图块偏移)
    paperY.value = getRandom(20, 80); // 拼图块的初始y轴位置

    const img = document.createElement('img');
    let newIdx = parseInt(getRandom(0, pathList.bg.length - 1));
    while (newIdx === currentIdx.value) {
        newIdx = parseInt(getRandom(0, pathList.bg.length - 1));
    }
    currentIdx.value = newIdx;
    img.src = pathList['bg'][newIdx];
    img.onload = () => {
        // 重新设置canvas的宽高来刷新canvas (clearRect清除有图形残留)
        paperRef.value.width = canvasWidth;
        paperRef.value.height = canvasHeight;
        blockRef.value.width = canvasWidth;
        blockRef.value.height = canvasHeight;
        const ctx = paperRef.value.getContext('2d');
        const block_ctx = blockRef.value.getContext('2d');
        //绘制碎片形状 先画路径，之后再填充图片
        draw(ctx, 'fill');
        ctx.drawImage(img, 0, 0, canvasWidth, canvasHeight);
        draw(block_ctx, 'clip');
        block_ctx.drawImage(img, 0, 0, canvasWidth, canvasHeight);

        canRefresh.value = true;
    }
    img.onerror = () => fresh();
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
    width: v-bind(canvasWidthStyle);
    height: 230px;
    padding: 10px 15px;
    margin: 0 auto;
    border: 1px solid #ccc;
    border-radius: 5px;
    color: #2c3e50;
    text-align: center;

    .sliderWrap {
        padding: 20px 0 10px;
    }

    .canvas-wrap {
        width: v-bind(canvasWidthStyle);
        height: v-bind(canvasHeightStyle);
        margin: 0 auto;
        border-radius: 5px;
        position: relative;
        overflow: hidden;

        %absolute {
            position: absolute;
            top: 0;
            left: 0;
        }

        canvas {
            @extend %absolute;
        }

        .block {
            transform: v-bind(diffStyle)
        }

        .buttonWrap {
            @extend %absolute;
            left: auto;
            top: 0;
            right: 0;
            padding: 2px 5px;
            display: flex;
            justify-content: flex-end;
            background-color: rgba(0, 0, 0, .2);

            &:hover {
                background-color: rgba(0, 0, 0, .3);
            }

            img {
                width: 30px;
                height: 30px;
                margin-right: 5px;
                user-select: none;
                opacity: .5;
                transition: .3s;
                cursor: pointer;

                &:hover {
                    opacity: 1;
                }
            }
        }

        .mask {
            @extend %absolute;
            width: 100%;
            height: 100%;
            display: flex;
            z-index: 6;
            background-color: rgba(255, 255, 255, .7);

            @keyframes turn {
                0% {
                    transform: rotate(0deg);
                }

                25% {
                    transform: rotate(90deg);
                }

                50% {
                    transform: rotate(180deg);
                }

                75% {
                    transform: rotate(270deg);
                }

                100% {
                    transform: rotate(360deg);
                }
            }

            img {
                width: 30px;
                height: 30px;
                margin: auto;
                color: #9ba4ab;
                user-select: none;
                animation: turn 2s linear infinite;
            }
        }
    }

    .progress-wrap {
        position: relative;
        height: 40px;
        border: 1px solid #e4e7eb;
        border-radius: 3px;
        box-sizing: border-box;
        background-color: #f7f9fa;

        .placeHolder {
            position: absolute;
            top: 50%;
            left: 50%;
            height: 14px;
            width: 100%;
            margin: 0;
            font-size: 14px;
            font-weight: lighter;
            line-height: 1;
            transform: translate(-50%, -50%);
            user-select: none;
        }

        .inner-bar {
            height: 40px;
            width: 0;
            border: 1px solid #1991fa;
            border-radius: 3px;
            box-sizing: border-box;
            background-color: #d1e9fe;
            z-index: 1;

            &.success {
                background-color: #d2f4ef;
                border-color: #4dc7c0;
            }

            &.error {
                background-color: #fce1e1;
                border-color: #f57a7a;
            }
        }

        .button {
            position: absolute;
            top: 0;
            left: 0;
            height: 40px;
            width: 60px;
            box-sizing: border-box;
            border-radius: 3px;
            z-index: 6;
            display: flex;
            transform: translateX(0);
            background: #2a6cfe;
            cursor: pointer;

            img {
                width: 80%;
                height: 60%;
                margin: auto;
            }

            &.success {
                background-color: #4dc7c0;
            }

            &.error {
                background-color: #f57a7a;
            }
        }
    }
}
</style>