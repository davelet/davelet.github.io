---
layout: post
title: REACT 中如何禁止页面不必要的重新渲染
categories: [dev]
tags: [javascript]
---

react的项目中，默认情况下每次state有变化都会重新渲染页面。没错，是整个页面重新渲染：可能你见过某些材料说只有部分dom会重新渲染，但实际就是全部。
如何防止这种情况呢？

# 为什么要禁止
你既然在读这篇文章，很有可能已经遇到类似的问题，也就明白为什么要禁止那些不必要的渲染。
如果页面是有进度信息的，重新渲染页面会丢失进度。

进度为什么不保存在state呢？因为如果页面是复杂的效果，是持续的长时间渲染，记在state里并不能解决重新渲染时特效也要重头来过的问题。

# 解决方案
可能你已经搜索到一些方案。这里我推荐的是使用插件react-immutable-render-mixin：
```
npm install react-immutable-render-mixi
```

然后在对应的class上增加：
```
@immutableRenderDecorator
class CK extends PureComponent {
    ...
```

然后增加插件"transform-decorators-legacy"：redux是在.babelrc中，dvajs是在.roadhogrc中。
# experimentalDecorators警告
我使用的编辑器是vs code，增加上门的注解后竟然提示experimentalDecorators警告。
如果你也使用vscode，解决方法是安装babel-plugin-transform-decorators-legacy：
```
npm install babel-plugin-transform-decorators-legacy
```

创建（或打开）tsconfig.json文件，写入内容：
```
{
    "compilerOptions": {
        "experimentalDecorators": true,
        "allowJs": true
    }
}
```
即可。