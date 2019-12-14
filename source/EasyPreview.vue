<template>
    <div class="picPreview wrapper" style="display: inline-block; font-size: 0">
        <div class="img-wrapper" @click="openPreview" style="display: inline-block; font-size: 0" v-if="computedShowSlot">
            <slot></slot>
        </div>
        <div class="mask" @mousemove="onmousemove" ref="wrapper"
             onselectstart="return false;" unselectable="on" style="-moz-user-select:none;"
             :class="{show : computedStatus === 'show'}" :style="extraStyle">
            <img :src="imgSrc" alt="" draggable="false" class="img"
                 @mousewheel="handleScroll"
                 :style="transformStyle" ref="img"
                 @mousedown="onmousedown" @mouseup="onmouseup"
                 @mouseleave="onmouseleave" ondragstart="return false;"
            >
            <span class="close" @click="onClickCloseButton"
                  v-if="computedShowCloseButton" @mouseenter="hoverStart"
                  @mouseleave="hoverEnd" :style="buttonExtraStyle">×</span>
        </div>
    </div>
</template>

<script>

    /**
     * @author SakuraSnow
     */
    export default {
        name: "EasyPreview",
        props: {
            imgSrc: {
                type: String,
            },
            options: {
                type: Object,
                default() {
                    return null;
                }
            },
            showPreview : {
                type : Boolean,
                default() {
                    return false
                }
            }
        },
        data() {
            return {
                // 矩阵的六个值
                a: 1,
                b: 0,
                c: 0,
                d: 1,
                e: 0,
                f: 0,
                // transform-origin 的x和y
                originX: 0,
                originY: 0,
                // 缩放倍数
                magnification: 1,
                // 滚动一次滑轮放大 / 缩小的倍数
                rate: 1.05,
                // 前一个transform-origin的值
                prevOrigin: {
                    x: -1,
                    y: -1
                },
                // 是否处在拖拽状态
                onDragStatus: false,
                // 上次onmousemove鼠标的位置
                prevPos: {
                    x: -1,
                    y: -1
                },
                // 放大图片时会 +1, 缩小图片时会-1
                optionCount: 0,
                status: "hide",
                scrollLock : false,
                buttonHover : false
            }
        },
        computed: {
            // 计算得到的transform和transform-origin会被应用于到img上
            transformStyle() {
                let {a, b, c, d, e, f, originX, originY, magnification} = this;
                return `transform:matrix(${magnification},${b},${c},${magnification},${e},${f}) translateZ(1px);transform-origin:${originX}px ${originY}px;`;
            },
            extraStyle() {
                let {options} = this;
                let style = "";
                if (options != null) {
                    style = this.transformExtraStyle(this.computedStatus === "show" ?
                        options.showStatusExtraStyle : options.hideStatusExtraStyle);
                }
                return style;
            },
            buttonExtraStyle() {
                let {options} = this;
                let style = "";
                if (options != null) {
                    style = this.transformExtraStyle(this.buttonHover ?
                        options.buttonHoverExtraStyle : options.buttonExtraStyle);
                }
                return style;
            },
            computedStatus() {
                let options = this.options;
                if (options == null) return this.status;
                else return this.showPreview ? "show" : "hide";
            },
            computedShowCloseButton() {
                let options = this.options;
                if (options == null) return true;
                else return options.showCloseButton;
            },
            computedShowSlot() {
                let options = this.options;
                if (options == null) return true;
                else return !options.controlByUsers;
            }
        },
        methods: {
            /**
             * 滚动鼠标滚轮时调用
             * @param event
             */
            handleScroll(event) {
                const offsetX = event.offsetX;
                const offsetY = event.offsetY;
                const direction = (event.wheelDelta && (event.wheelDelta > 0 ? "up" : "down")) ||
                    (event.detail && (event.detail < 0 ? "up" : "down")) || "down";
                // wrapper的宽高, 相当于视口宽高
                const wrapperWidth = this.$refs.wrapper.offsetWidth;
                const wrapperHeight = this.$refs.wrapper.offsetHeight;
                // img的视觉宽高，随着放大而变大
                const imgVisualWidth = this.$refs.img.width * this.magnification;
                const imgVisualHeight = this.$refs.img.height * this.magnification;
                // img实际的宽高
                const imgWidth = this.$refs.img.width;
                const imgHeight = this.$refs.img.height;
                // 下次transform-origin的坐标
                let originX = offsetX;
                let originY = offsetY;

                // 如果没有放大过就不能缩小
                if (this.magnification <= 1 && direction === "down") return;

                // 如果图片的视觉大小小于wrapper的大小，就把transform-origin取到图片中央
                if (imgVisualHeight < wrapperHeight) {
                    originY = imgHeight / 2;
                }
                if (imgVisualWidth < wrapperWidth) {
                    originX = imgWidth / 2;
                }

                // 修改transform-origin
                this.originX = originX;
                this.originY = originY;

                if (this.prevOrigin.x !== -1) {
                    // 如果是放大图片
                    if (direction === "up") {
                        // 两次放大的中心不同需要修正位置, 否则会鼠标指的就不是一个地方了
                        let moveX = (1 - this.magnification) * (originX - this.prevOrigin.x);
                        let moveY = (1 - this.magnification) * (originY - this.prevOrigin.y);
                        this.e -= moveX;
                        this.f -= moveY;
                        this.optionCount++;
                    } else if (direction === "down") {

                        // 如果现在的缩放倍率已经大于1.5
                        if (this.magnification > 1.5) {
                            // 计算让鼠标能指向同个位置的修正X和Y
                            let moveX = (1 - this.magnification) * (originX - this.prevOrigin.x);
                            let moveY = (1 - this.magnification) * (originY - this.prevOrigin.y);

                            // 下面这一堆东西是在计算下次图片放缩后的位置
                            const imgOffsetLeft = this.$refs.img.offsetLeft;
                            const imgOffsetTop = this.$refs.img.offsetTop;
                            const magnification = this.magnification / this.rate;
                            const x = this.originX;
                            const y = this.originY;
                            const e = this.e - moveX;
                            const f = this.f - moveY;
                            const nextImgVisualWidth = imgWidth * magnification;
                            const nextImgVisualHeight = imgHeight * magnification;
                            // 图片的视觉左边 / 顶边 / 右边 / 底边离wrapper左边 / 顶边 / 右边 / 底边的距离
                            const left = (magnification - 1) * x - e - imgOffsetLeft;
                            const top = (magnification - 1) * y - imgOffsetTop - f;
                            const right = nextImgVisualWidth - left - wrapperWidth;
                            const bottom = nextImgVisualHeight - top - wrapperHeight;
                            // 如果缩放后的图片比wrapper的宽吗，同时图片左边没有顶到wrapper的左边
                            if (nextImgVisualWidth > wrapperWidth && left < 0) {
                                // 让图片的左边顶到wrapper的左边
                                moveX -= left;
                            } else if (nextImgVisualWidth > wrapperWidth && right < 0) {
                                moveX += right;
                            }
                            if (nextImgVisualHeight > wrapperHeight && top < 0) {
                                moveY -= top;
                            } else if (nextImgVisualHeight > wrapperHeight && bottom < 0) {
                                moveY += bottom;
                            }
                            // 如果图片的大小已经小于wrapper的大小
                            // 要让图片两边的空白相同
                            if (nextImgVisualWidth < wrapperWidth) {
                                // 计算要移动多少才能让两边的空白相同
                                let average = (left + right) / 2;
                                let diff = left - average;
                                // 移动过去
                                moveX -= diff;
                            }
                            if (nextImgVisualHeight < wrapperHeight) {
                                let average = (top + bottom) / 2;
                                let diff = top - average;
                                moveY -= diff;
                            }
                            // 修正位置
                            this.e -= moveX;
                            this.f -= moveY;

                        }
                        else {
                            // 如果缩放倍率小于1.5
                            // 要开始把图片回归原来的位置了
                            // 用需要移动的距离 / 一共还能缩放的次数来获取本次的移动距离
                            if (this.optionCount >= 2) {
                                let moveX = -this.e / this.optionCount;
                                let moveY = -this.f / this.optionCount;
                                // 还是计算下次的位置
                                const imgOffsetLeft = this.$refs.img.offsetLeft;
                                const imgOffsetTop = this.$refs.img.offsetTop;
                                const magnification = this.magnification / this.rate;
                                const x = this.originX;
                                const y = this.originY;
                                const e = this.e + moveX;
                                const f = this.f + moveY;
                                const nextImgVisualWidth = imgWidth * magnification;
                                const nextImgVisualHeight = imgHeight * magnification;
                                const left = (magnification - 1) * x - e - imgOffsetLeft;
                                const top = (magnification - 1) * y - imgOffsetTop - f;
                                const right = nextImgVisualWidth - left - wrapperWidth;
                                const bottom = nextImgVisualHeight - top - wrapperHeight;
                                // 同样没顶到就移动回去
                                if (nextImgVisualWidth > wrapperWidth && left < 0) {
                                    moveX -= -left;
                                } else if (nextImgVisualWidth > wrapperWidth && right < 0) {
                                    moveX += -right;
                                }
                                if (nextImgVisualHeight > wrapperHeight && top < 0) {
                                    moveY -= -top;
                                } else if (nextImgVisualHeight > wrapperHeight && bottom < 0) {
                                    moveY += -bottom;
                                }
                                // 如果图片的大小已经小于wrapper的大小
                                // 要让图片两边的空白相同
                                if (nextImgVisualWidth < wrapperWidth) {
                                    // 计算要移动多少才能让两边的空白相同
                                    let average = (left + right) / 2;
                                    let diff = left - average;
                                    // 移动过去
                                    moveX += diff;
                                }
                                if (nextImgVisualHeight < wrapperHeight) {
                                    let average = (top + bottom) / 2;
                                    let diff = top - average;
                                    moveY += diff;
                                }
                                // 开始把图片移回去
                                this.e += moveX;
                                this.f += moveY;
                            }
                            else {
                                this.e = 0;
                                this.f = 0;
                            }
                        }
                        this.optionCount--;
                    }
                } else {
                    this.optionCount++;
                }
                // 放大 / 缩小图片
                if (direction === "up") {
                    this.magnification *= this.rate;
                } else {
                    this.magnification *= 1 / this.rate;
                }
                // 把现在的transform-origin 存起来
                this.prevOrigin.x = originX;
                this.prevOrigin.y = originY;

            },
            onmousedown(event) {
                // 保存现在鼠标点击的位置
                this.prevPos.x = event.clientX;
                this.prevPos.y = event.clientY;
                this.onDragStatus = true;
            },
            onmousemove: function (event) {
                let clientX = event.clientX;
                let clientY = event.clientY;
                // 通过上次保存的位置来计算图片需要移动的X和Y
                let translateX = clientX - this.prevPos.x;
                let translateY = clientY - this.prevPos.y;

                if (this.onDragStatus) {

                    // 水平移动的方向
                    let hd = translateX > 0 ? "right" : "left";
                    // 垂直移动方向
                    let vd = translateY > 0 ? "down" : "up";

                    // wrapper的宽高
                    let wrapperWidth = this.$refs.wrapper.offsetWidth;
                    let wrapperHeight = this.$refs.wrapper.offsetHeight;
                    // img标签放缩后的视觉大小
                    let imgVisualWidth = this.$refs.img.width * this.magnification;
                    let imgVisualHeight = this.$refs.img.height * this.magnification;
                    // 还是计算移动后图片的位置
                    const imgOffsetLeft = this.$refs.img.offsetLeft;
                    const imgOffsetTop = this.$refs.img.offsetTop;
                    const magnification = this.magnification;
                    const x = this.originX;
                    const y = this.originY;
                    const e = this.e;
                    const f = this.f;
                    const nextImgVisualWidth = imgVisualWidth;
                    const nextImgVisualHeight = imgVisualHeight;
                    const left = (magnification - 1) * x - e - imgOffsetLeft;
                    const top = (magnification - 1) * y - imgOffsetTop - f;
                    const right = nextImgVisualWidth - left - wrapperWidth;
                    const bottom = nextImgVisualHeight - top - wrapperHeight;
                    // 判断图片能否向左 / 右 / 上 / 下 移动
                    // 判断的依据是往该方向移动后是否出现空隙
                    let leftAble = right > 0;
                    let rightAble = left > 0;
                    let upAble = bottom > 0;
                    let downAble = top > 0;

                    // 如果水平移动方向是左
                    if (hd === "left") {
                        if (leftAble) {
                            // 计算最多能偏移的大小, 超过这个大小会出现间隙
                            translateX = -Math.min(Math.abs(translateX), right);
                        } else {
                            // 如果不能就把偏移量置为0
                            translateX = 0;
                        }
                    }
                    // 下面同理
                    if (hd === "right") {
                        if (rightAble) {
                            translateX = Math.min(translateX, left);
                        } else {
                            translateX = 0;
                        }
                    }
                    if (vd === "up") {
                        if (upAble) {
                            translateY = -Math.min(Math.abs(translateY), bottom);
                        } else {
                            translateY = 0;
                        }
                    }
                    if (vd === "down") {
                        if (downAble) {
                            translateY = Math.min(Math.abs(translateY), top);
                        } else {
                            translateY = 0;
                        }
                    }
                    // 进行偏移
                    this.e += translateX;
                    this.f += translateY;

                    // 保存该次鼠标移动的位置
                    this.prevPos.x = clientX;
                    this.prevPos.y = clientY;
                }
            },
            onmouseup() {
                this.onDragStatus = false;
            },
            onmouseleave() {
                this.onDragStatus = false;
            },
            onClickCloseButton() {
                let options = this.options;
                if (options && options.controlByUsers) {
                    this.$emit("click-close-button", this.resetPosition);
                    this.$emit("clickCloseButton", this.resetPosition);
                } else {
                    this.updateStatus("hide");
                }
            },
            openPreview() {
                this.status = "show";
            },
            transformExtraStyle(style) {
                if (typeof style === "string") return style;
                else if (typeof style === "object") {
                    // console.log(style);
                    let str = "";
                    let keys = Object.keys(style);
                    // console.log(keys);
                    for (let i = 0, length = keys.length; i < length; i++) {
                        str += `${keys[i]} : ${style[keys[i]]};`;
                    }
                    return str;
                } else {
                    return "";
                }
            },
            updateStatus(newStatus) {
                this.status = newStatus;
                if (newStatus === "hide") {
                    this.resetPosition();
                }
            },
            resetPosition(time = 500) {
                setTimeout(() => {
                    this.resetData();
                }, time)
            },
            resetData() {
                this.magnification = 1;
                this.e = 0;
                this.f = 0;
                this.originX = 0;
                this.originY = 0;
                this.prevPos.x = -1;
                this.prevPos.y = -1;
                this.prevOrigin.x = -1;
                this.prevOrigin.y = -1;
                this.optionCount = 0;
            },
            hoverStart() {
                this.buttonHover = true;
            },
            hoverEnd() {
                this.buttonHover = false;
            }
        },
        mounted() {
            // 兼容FireFox的处理
            this.$refs.img.addEventListener("DOMMouseScroll", this.handleScroll, false)
        },
        watch : {
            imgSrc : {
                handler() {
                    this.resetData();
                }
            },
            options : {
                handler(newOptions) {
                    if (newOptions) {
                        newOptions.controlByUsers = newOptions.controlByUsers != null ? newOptions.controlByUsers :  false;
                        newOptions.showCloseButton = newOptions.showCloseButton != null ? newOptions.showCloseButton : true;
                        newOptions.showStatusExtraStyle = newOptions.showStatusExtraStyle != null ? newOptions.showStatusExtraStyle : "";
                        newOptions.hideStatusExtraStyle = newOptions.hideStatusExtraStyle != null ? newOptions.hideStatusExtraStyle : "";
                        newOptions.buttonExtraStyle = newOptions.buttonExtraStyle != null ? newOptions.buttonExtraStyle : "";
                        newOptions.buttonHoverExtraStyle = newOptions.buttonHoverExtraStyle != null ? newOptions.buttonHoverExtraStyle : "";
                    }
                },
                immediate : true
            },

        }
    }
</script>

<style scoped lang="less">
    .mask {
        transition: all 0.5s;
        will-change: transform, opacity;
        opacity: 1;
        position: fixed;
        width: 100%;
        height: 100%;
        left: 0;
        top: 0;
        box-sizing: border-box;
        background-color: rgba(0, 0, 0, 0.8);
        z-index: 1;
        display: flex;
        justify-content: center;
        align-items: center;
        overflow: hidden;

        .img {
            max-width: 100%;
            max-height: 100%;
            will-change: transform;
        }

        .close {
            height: 40px;
            width: 40px;
            font-size: 25px;
            position: absolute;
            text-align: center;
            line-height: 40px;
            top: 0;
            right: 0;
            color: #ffffff;
            opacity: 0;
            transition: all 0.4s;
            cursor: pointer;
        }

        .close:hover {
            background-color: #ff1231;
            opacity: 1;
        }
    }

    .mask:not(.show) {
        transform: scale(0);
        opacity: 0;
    }
</style>