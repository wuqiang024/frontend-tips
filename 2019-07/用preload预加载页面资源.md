# 用 preload 预加载页面资源
> 作者简介 felix 蚂蚁金服·数据体验技术团队

本文主要介绍preload的使用，以及与prefetch的区别。然后会聊聊浏览器的加载优先级。

preload 提供了一种声明式的命令，让浏览器提前加载指定资源(加载后并不执行)，在需要执行的时候再执行。提供的好处主要是
 - 将加载和执行分离开，可不阻塞渲染和 document 的 onload 事件
 - 提前加载指定资源，不再出现依赖的font字体隔了一段时间才刷出

## 如何使用 preload
### 使用 link 标签创建
```html
<!-- 使用 link 标签静态标记需要预加载的资源 -->
<link rel="preload" href="/path/to/style.css" as="style">

<!-- 或使用脚本动态创建一个 link 标签后插入到 head 头部 -->
<script>
const link = document.createElement('link');
link.rel = 'preload';
link.as = 'style';
link.href = '/path/to/style.css';
document.head.appendChild(link);
</script>
```
### 使用 HTTP 响应头的 Link 字段创建
```plain
Link: <https://example.com/other/styles.css>; rel=preload; as=style
```

如我们常用到的 antd 会依赖一个 CDN 上的 font.js 字体文件，我们可以设置为提前加载，以及有一些模块虽然是按需异步加载，但在某些场景下知道其必定会加载的，则可以设置 preload 进行预加载，如：
```html
<link rel="preload" as="font"   href="https://at.alicdn.com/t/font_zck90zmlh7hf47vi.woff">
<link rel="preload" as="script" href="https://a.xxx.com/xxx/PcCommon.js">
<link rel="preload" as="script" href="https://a.xxx.com/xxx/TabsPc.js">
```

## 如何判断浏览器是否支持 preload
目前我们支持的浏览器主要为高版本 Chrome，所以可放心使用 preload 技术。
其他环境在 caniuse.com 上查到的支持情况如下：
![image.png | center | 830x372](https://user-gold-cdn.xitu.io/2018/2/11/16182c9cfb2a97ba?w=932&h=418&f=png&s=66423 "")
在不支持 preload 的浏览器环境中，会忽略对应的 link 标签，而若需要做特征检测的话，则：
```javascript
const isPreloadSupported = () => {
  const link = document.createElement('link');
  const relList = link.relList;

  if (!relList || !relList.supports) {
    return false;
  }

  return relList.supports('preload');
};
```

## 如何区分 preload 和 prefetch
* preload   是告诉浏览器页面**必定**需要的资源，浏览器**一定会**加载这些资源；
* prefetch 是告诉浏览器页面**可能**需要的资源，浏览器**不一定会**加载这些资源。


preload 是确认会加载指定资源，如在我们的场景中，x-report.js 初始化后一定会加载 PcCommon.js 和 TabsPc.js, 则可以预先 preload 这些资源；

prefetch 是预测会加载指定资源，如在我们的场景中，我们在页面加载后会初始化首屏组件，当用户滚动页面时，会拉取第二屏的组件，若能预测用户行为，则可以 prefetch 下一屏的组件。

## preload 将提升资源加载的优先级
使用 preload 前，在遇到资源依赖时进行加载：
![image.png | left | 483x75](https://user-gold-cdn.xitu.io/2018/2/11/16182c9cfc08d33e?w=483&h=75&f=png&s=13650 "")
使用 preload 后，不管资源是否使用都将提前加载：
![image.png | left | 496x75](https://user-gold-cdn.xitu.io/2018/2/11/16182c9cfc3795d0?w=496&h=75&f=png&s=13377 "")
可以看到，preload 的资源加载顺序将被提前：
![image.png | left | 522x193](https://user-gold-cdn.xitu.io/2018/2/11/16182c9cfd983ba1?w=522&h=193&f=png&s=40482 "")

## 避免滥用 preload
使用 preload 后，Chrome 会有一个警告：
![image.png | left | 782x34](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d0099ff27?w=782&h=34&f=png&s=18757 "")

如上文所言，若不确定资源是必定会加载的，则不要错误使用 preload，以免本末倒置，给页面带来更沉重的负担。

当然，可以在 PC 中使用 preload 来刷新资源的缓存，但在移动端则需要特别慎重，因为可能会浪费用户的带宽。

## 避免混用 preload 和 prefetch
preload 和 prefetch 混用的话，并不会复用资源，而是会重复加载。
```html
<link rel="preload"   href="https://at.alicdn.com/t/font_zck90zmlh7hf47vi.woff" as="font">
<link rel="prefetch"  href="https://at.alicdn.com/t/font_zck90zmlh7hf47vi.woff" as="font">
```

使用 preload 和 prefetch 的逻辑可能不是写到一起，但一旦发生对用一资源 preload 或 prefetch 的话，会带来双倍的网络请求，这点通过 Chrome 控制台的网络面板就能甄别：
![image.png | left | 649x111](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d024a861b?w=649&h=111&f=png&s=21597 "")

## 避免错用 preload 加载跨域资源
若 css 中有应用于已渲染到 DOM 树的元素的选择器，且设置了 @font-face 规则时，会触发字体文件的加载。
而字体文件加载中时，DOM 中的这些元素，是处于不可见的状态。对已知必加载的 font 文件进行预加载，除了有性能提升外，更有体验优化的效果。

在我们的场景中，已知 antd.css 会依赖 font 文件，所以我们可以对这个字体文件进行 preload:
```html
<link rel="preload" as="font" href="https://at.alicdn.com/t/font_zck90zmlh7hf47vi.woff">
```

然而我发现这个文件加载了两次：
![image.png | left | 712x111](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d39e6a355?w=712&h=111&f=png&s=21798 "")
![image.png | center | 830x59](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d3aa4a0bd?w=1039&h=75&f=png&s=21425 "")
![image.png | center | 830x59](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d3abb2f7e?w=1047&h=75&f=png&s=20508 "")

原因是对跨域的文件进行 preload 的时候，我们必须加上 crossorigin 属性：
```html
<link rel="preload" as="font" crossorigin href="https://at.alicdn.com/t/font_zck90zmlh7hf47vi.woff">
```
再看一下网络请求，就变成一条了。

W3 规范是这么解释的：

> Preload links for CORS enabled resources, such as fonts or images with a crossorigin attribute, must also include a crossorigin attribute, in order for the resource to be properly used.


那为何会有两条请求，且优先级不一致，又没有命中缓存呢？这就得引出下一个话题来解释了。

## 不同资源加载的优先级规则
我们先来看一张图：
![image.png | left | 687x601](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d3ff9f3c2?w=687&h=601&f=png&s=69267 "")
> 这张表详见：Chrome Resource Priorities and Scheduling

这张图表示的是，在 Chrome 46 以后的版本中，不同的资源在浏览器渲染的不同阶段进行加载的优先级。
在这里，我们只需要关注 DevTools Priority 体现的优先级，一共分成五个级别：
* Highest 最高
* Hight 高
* Medium 中等
* Low 低
* Lowest 最低

![image.png | left | 689x136](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d3adc6ac4?w=689&h=136&f=png&s=22667 "")

### html 主要资源，其优先级是最高的
![image.png | left | 686x165](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d481a1d53?w=686&h=165&f=png&s=25805 "")
![image.png | left | 501x69](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d66475b20?w=501&h=69&f=png&s=13875 "")

### css 样式资源，其优先级也是最高的
![image.png | left | 684x182](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d6655ca9e?w=684&h=182&f=png&s=28812 "")
CSS(match) 指的是对已有的 DOM 具备规则的有效的样式文件。
![image.png | left | 475x191](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d668e2d05?w=475&h=191&f=png&s=42007 "")
### script 脚本资源，优先级不一
![image.png | left | 686x200](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d6d350cd4?w=686&h=200&f=png&s=35097 "")
![image.png | left | 476x247](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d714e2b5d?w=476&h=247&f=png&s=49422 "")
前三个 js 文件是写死在 html 中的静态资源依赖，后三个 js 文件是根据首屏按需异步加载的组件资源依赖，这正验证了这个规则。

### font 字体资源，优先级不一
![image.png | left | 686x164](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d75e89bad?w=686&h=164&f=png&s=26721 "")
![image.png | left | 472x114](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d92882579?w=472&h=114&f=png&s=20800 "")
css 样式文件中有一个 @font-face 依赖一个 font 文件，样式文件中依赖的字体文件加载的优先级是 Highest；
在使用 preload 预加载这个 font 文件时，若不指定 crossorigin 属性(即使同源)，则会采用匿名模式的 CORS 去加载，优先级是 High，看下图对比：
第一条 High 优先级也就是 preload 的请求：
![image.png | left | 830x411](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d959d176e?w=2396&h=1188&f=png&s=657709 "")

第二条 Highest 也就是样式引入的请求：
![image.png | left | 830x415](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d9b47ac32?w=2322&h=1162&f=png&s=644778 "")

可以看到，在 preload 的请求中，缺少了一个 origin 的请求头字段，表示这个请求是匿名的请求。
让这两个请求能共用缓存的话，目前的解法是给 preload 加上 crossorigin 属性，这样请求头会带上 origin, 且与样式引入的请求同源，从而做到命中缓存：
```html
<link rel="preload" as="font" crossorigin href="https://at.alicdn.com/t/font_zck90zmlh7hf47vi.woff">
```
这么请求就只剩一个：
![image.png | left | 475x81](https://user-gold-cdn.xitu.io/2018/2/11/16182c9d9b434455?w=475&h=81&f=png&s=11924 "")
![image.png | left | 830x424](https://user-gold-cdn.xitu.io/2018/2/11/16182c9da3d6f330?w=2298&h=1176&f=png&s=599582 "")
在网络瀑布流图中，也显示成功预加载且后续命中缓存不再二次加载：
![image.png | left | 662x86](https://user-gold-cdn.xitu.io/2018/2/11/16182c9dab663c25?w=662&h=86&f=png&s=11789 "")

## 总结
preload 是个好东西，能告诉浏览器提前加载当前页面必须的资源，将加载与解析执行分离开，做得好可以对首次渲染带来不小的提升，但要避免滥用，区分其与 prefetch 的关系，且需要知道 preload 不同资源时的网络优先级差异。

preload 加载页面必需的资源如 CDN 上的字体文件，与 prefetch 预测加载下一屏数据，兴许是个不错的组合。

参考资料：
* https://www.w3.org/TR/preload/
* https://www.w3.org/TR/resource-hints/
* [Prioritizing Your Resources with link rel='preload'](https://developers.google.com/web/updates/2016/03/link-rel-preload)
* [Preload, Prefetch And Priorities in Chrome](https://medium.com/reloading/preload-prefetch-and-priorities-in-chrome-776165961bbf)
* [A Link: rel=preload Analysis From the Chrome Data Saver Team](https://medium.com/reloading/a-link-rel-preload-analysis-from-the-chrome-data-saver-team-5edf54b08715)