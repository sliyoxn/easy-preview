<figure class="wp-block-table"><table class="">
在线演示： [点这里]( https://sliyoxn.github.io/easy-preview-demo.github.io/  "标题")



## 使用方法

### 注册

#### 全局注册

    1. npm install easy-preview
    2. 在main.js中加入下面两句
        import EasyPreview from "easy-preview";
        Vue.use(EasyPreview);
    3. 使用<EasyPreview>标签即可

#### 局部注册

    1. 使用npm install easy-preview安装插件
    2. 在组件中注册EasyPreview
    3. 使用<EasyPreview>;标签即可

```
import EasyPreview from "easy-preview";
export default {
  name: 'app',
  components: {EasyPreview},
  data () {
    return {}
  }
}
```

### 使用

#### 控制权交给组件的使用

这种使用方式比较简单，只要传入一个img标签给插槽使用并传入img-src属性即可

```
<EasyPreview :img-src="imgSrc">
    <img :src="imgSrc" width="500" style="border-radius: 10px" alt="">
</EasyPreview>
```

#### 控制权不交给组件的使用

这个时候要传入一个属性options，并将options.controlByUsers置为true，此时插槽会失效，需要传入一个额外的属性:show-preview控制显示和隐藏，此时点击右上角自带的关闭按钮改为触发自定义的clickCloseButton /  click-close-button  （两个都会触发）事件，可以监听事件并修改传入的show-preview的值。

```			
<img :src="imgSrc" alt="" width="500" style="border-radius: 10px" @click="onclick">
<EasyPreview :img-src="imgSrc" :options="options" :show-preview="showPreview"   @clickCloseButton="onClickCloseButton"></EasyPreview>



{
    methods : { 
        onclick() {
            this.showPreview = true
        }
        onClickCloseButton() {
            this.showPreview = false;
        }
    }
}
```

### 说明文档

#### 全部属性的含义

| 属性名               | 含义                             | 默认值 | 备注                                          |
| -------------------- | -------------------------------- | ------ | --------------------------------------------- |
| imgSrc               | 浏览时的图片链接                 | ""     |                                               |
| options              | 自定义选项                       | null   | 具体参数看下面                                |
| showPreview          | 是否展示预览图                   | false  | 仅控制权不是组件内部时生效                    |
| clickCloseButton     | 点击关闭按钮时会触发的自定义事件 |        | 仅控制权不是组件内部时生效 ，需要绑定回调函数 |
| click\-close\-button | 点击关闭按钮时会触发的自定义事件 |        | 仅控制权不是组件内部时生效 ，需要绑定回调函数 |

PS：clickCloseButton绑定的事件执行时会被传入一个函数，执行这个函数可以把图片恢复初始状态，调用时可以传入一个延迟执行的时间，这个时间默认是500ms（如果你没有修改transition的时间的话，最好不要修改它）

```
onClickCloseButton(reset) {
    this.showPreview = false;
    reset(500);
},
```

#### options的几个可选项

| 属性名                | 含义                            | 默认值 | 备注                                       |
| --------------------- | ------------------------------- | ------ | ------------------------------------------ |
| controlByUsers        | 控制权是否交给组件外部          | false  |                                            |
| showCloseButton       | 是否显示右上角的关闭按钮        | true   |                                            |
| showStatusExtraStyle  | 展示状态时额外的样式            | ""     | 可以传入对象或者字符串，样式优先级为内联级 |
| hideStatusExtraStyle  | 隐藏状态时额外的样式            | ""     | 可以传入对象或者字符串，样式优先级为内联级 |
| buttonExtraStyle      | 右上的按钮没有hover时的额外样式 | ""     | 可以传入对象或者字符串，样式优先级为内联级 |
| buttonHoverExtraStyle | 右上的按钮hover时的额外样式     | ""     | 可以传入对象或者字符串，样式优先级为内联级 |
|                       |                                 |        |                                            |

