---
category: 前端工具
tags:
  - js
  - nodejs
date: 2019-04-20
title: webpack4 splitChunkPlugin配置介绍
header-title: true
vssue-title: webpack4 splitChunkPlugin配置介绍
---
# webpack4 splitChunkPlugin配置介绍
## 前言
webpack4删除了CommonsChunkPlugin插件，它使用内置API optimization.splitChunks和optimization.runtimeChunk,这意味着webpack会默认为你生成共享的代码块，splitchunk的配置非常灵活，让我们一起看一下他的配置。

基本配置项与commonsChunkPlugin大同小异，下方是启用mode=production后，splitChunk的默认配置：
```javascript
splitChunks: {
    chunks: "async",
    minSize: 30000,
    minChunks: 1,
    maxAsyncRequests: 5,
    maxInitialRequests: 3,
    automaticNameDelimiter: '~',
    name: true,
    cacheGroups: {
        vendors: {
            test: /[\\/]node_modules[\\/]/,
            priority: -10
        },
        default: {
            minChunks: 2,
            priority: -20,
            reuseExistingChunk: true
        }
    }
}

```
cacheGroups中的配置默认会继承外部的配置，但是有3哥配置是每个cacheGroup独有的，分别为test, priority 和 reuseExistingChunk，test支持正则匹配路径，priority表示某个文件被命中多个规则时采用哪个cacheGroup,reuseExistingChunk表示如果当前块包含已经从主bundle中分离出来的模块，那么它将被重用，而不是生成一个新的模块，一般设置为 true。从默认配置可以看出会默认将代码中从node_modules中引用的的代码按规则抽离出来。可以对optimization.splitChunks.cacheGroups.default或者optimization.splitChunks.cacheGroups.verndor为false以放弃使用默认的配置。

实际上cacheGroups中的配置只是对我们文件分离的细化配置，我们主要关心的配置项目主要是chunks, minSize, minChunks, maxAsyncRequests, maxInitialRequests

## 开始分析
我们一个配置一个配置的过，首先来看chunks属性

### chunks
默认作用于异步chunk，值为all/initial/async/function(chunk){}几类，当值为function时，第一个参数是遍历所有入口chunk的chunk模块，我们可以通过判断chunk的名字来按需处理。我们这里主要讲前三个配置。为了我们后面容易分析，我们这里采取一个用例如下图所示：

![图片 1.png](https://upload-images.jianshu.io/upload_images/12504052-bbfb58e7918b2c7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们新建两个文件testA.js和testB.js，他们共同静态引用Vue，共同动态引用qs,testA静态引用lodash，testB静态引用lodash.
代码结构如下：
testA.js
```javascript
import Vue from 'vue'
import lodash from 'lodash'
import('query-string').then(component=>{
    console.log(component)
})
console.log(lodash.add(1))
new Vue({
    el: '#app',
    template: '<h1>qewqwe<h1/>',
})

```
testB.js
```javascript
import Vue from 'vue'
import('query-string').then(component=>{
    console.log(component)
})
import('lodash').then(component=>{
    console.log(component)
})
new Vue({
    el: '#app',
    template: '<h1>qewqwe<h1/>',
})

```
我们仅切换chunks的配置看看现象：首先是默认的async，我们通过Bundle Analyzer可以看到只有异步引入的包被提取了出来

![屏幕快照 2019-04-21 15.06.19.png](https://upload-images.jianshu.io/upload_images/12504052-bf54acc3ae6effdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接着我们设置chunks选项为all,我们可以看到不管是异步引用还是静态引用的包都被提取了出来,并且引用方式不同的lodash只被提取出了一次（实际上静态引用被提取出来还是要满足一些条件的，我们后面再讲）文件命名上，走的是默认配置，静态引用的包的名字为verdor加～符号跟引用的包名加chunkhash

![屏幕快照 2019-04-21 15.07.09.png](https://upload-images.jianshu.io/upload_images/12504052-1de7907c58b22397.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接着我们再看当设置为initial时的情况，可以看到qs被提取，vue被提取，而lodash被提取了两次，产生了重复的代码。

![屏幕快照 2019-04-21 15.08.14.png](https://upload-images.jianshu.io/upload_images/12504052-9fd07b8c98ccfe89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### minSize与maxSize
这两个配置成对出现，字面意思就可以看出来是满足相应大小的包才会被提取出来，单位是字节，从上面的默认配置中我们可以得知，minSize的值为30000，也就是说小于这个数的包不会被提取而是会被打到引用的文件中去，maxSize与之相反，但优先级小于minSize，另外值得注意的是，这两个值只对静态引入的公共包有影响，对于异步引入的包，不管大小多少哪怕minSize设置的很大，也同样是会被提取出来的，大家可以自行尝试。

### minChunks
与上面两个配置一样，同样只作用于静态引入的情况，控制包被引用的次数来决定是否提取公共包，从上面的默认配置中我们得知默认是只要被引用的node_modules中的包就会被提取，当我们放大为3时，我们可以看到vue并不会被提取出来了。

### maxInitialRequests
简单来讲就是每一个入口文件打包完成后最多能有多少个模块组成（1.都打到一起，1以上：可以拆分，这个数值不包括入口文件（entrys）本身）举个例子：像刚才我们的引用方式，当我们设置为1时，我们可以看到我们的html模版中的引用只引用了testA和testB，也就是模版最多引入这两个文件，其他都会被合并，当我们增加之后就会在这两个入口文件的基础上增加引入的数量，也就从testA与testB中抽离出了公共代码。具体现象大家可以设置完试试看。
### maxAsyncRequests
这个配置的现象是最难以琢磨的，我至现在也没能明白其具体含义，按字面意思理解，我理解为如果异步引用的包的数量超过了maxAsyncRequests的设置，那么异步的包就会被合并，然而事实上现象并非如此，即使设置的很小设置成1，异步引入的包还是会生成一个个独立的被提取出来，我尝试去github上提问，看到作者只回复了一句话：
> It limits the requests per import() 
> 

![图片d 1.png](https://upload-images.jianshu.io/upload_images/12504052-3cc6f950f0d6e4bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可能本人比较愚钝，尚不能参破其中奥妙

### 优先级
有时我们在配置时常常发生不生效或者与预想不同的结果，往往时由于以上的属性的优先级不同造成的，以上属性的优先级为：

> minSize > maxSize > maxInitialRequest/maxAsyncRequests

参考文章：
[SplitChunksPlugin \| webpack 中文网](https://www.webpackjs.com/plugins/split-chunks-plugin/)