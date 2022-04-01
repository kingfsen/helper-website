---
title: "网站一键分享插件Share.js"
date: 2019-03-27T23:02:17+08:00
description: "超级易用的一键分享插件share.js,轻松将您的博客分享至微信,QQ空间,微博,豆瓣"
Tags:
- 分享插件
Categories:
- 博客组件
---

一键分享功能是网站社交化的一个重要组件，当前发现一款使用非常简单的js插件，就是share.js，项目地址: https://github.com/overtrue/share.js 。
share.js使用非常简单，它可以通过参数配置自由控制展示哪些分享图标，同时它还可以自定义分享时的title以及icon。

![share](/blog/share_js/001.png)

如此简单的一个js组件，对于我这样的一个后端开发人员，那也是分分钟上手，用法这里不细说，
请参考作者github。现在结合share.js，给自己的静态博客实现一键分享功能，同时给自己的博客实现通过参数控制分享图标的显示位置。

要使用share.js功能，首先要把share.js的css文件、js文件都引入到html中。本博客是利用Hugo生成的，从share.js项目的src目录，直接将子目录css、font、js文件夹
复制到博客项目的static目录中，然后将js、css文件引入主模板baseof.html中即可，js文件只需要qrcode.js以及social-share.js文件即可。

```HTML
	<link rel="stylesheet" href="{{ "css/share.min.css" | relURL }}">
	<script src="/js/social-share.min.js"></script>
	<script src="/js/qrcode.js"></script>
```

博客文章一键分享功能常见的是出现的文章的结尾部分，另外就是固定在屏幕中间位置。如此我们也来实现两种展示，通过网站参数进行控制。
在网站的配置文件config.toml中增加一个参数share_style，ver表示纵向排列在屏幕中间，hor表示横向展示在文章尾部。

```Text
[Params]
  description = "这里是孙金福的个人技术博客"
  toc = true # Enable Table of Contents
  authorbox = true # Show authorbox at bottom of pages if true
  readmore = true # Show "Read more" button in list if true
  dateformat = "2006-01-02" # change the format of dates
  post_navigation = true
  share_style = "ver"
```

新建一个分享模板share.html，里面内容如下

```
<div class="social-share"></div>
```

你完全可以把上面这个div添加到你博客任何需要展示分享的地方，由于我这博客采用的模板，通过在baseof.html中定义一个block share。

```HTML
<body class="body">
  {{ block share . }}
  
  {{ end }}
	<div class="container container--outer">
		{{ partial "header" . }}
		<div class="wrapper flex">
			<div class="primary">
			...
```

Hugo中single.html模板才是展示文章具体内容的版本，分享功能就是应该添加在这个模板中，如果参数设置的是横向展示，则直接展示在文章尾部，本博客当前采用的就是这种方法。

```
{{ if eq .Site.Params.share_style "hor" }}
{{ partial "share.html" . }}
{{ end }}
```

有时候根据个人喜好，喜欢全屏正中间展示，有点像开源中国pc端blog，分享按钮就是固定的。我这里也简单实现一个，作为一个后端开发者，感觉基本的css3以及js知识还是需要具备的,
给它设置一些基本的控制样式即可，这种方式需要注意微信二维码的展示位置以及浮动层优先级，没有什么东西一成不变就能完美使用，直接按F12打开浏览器调试，结合自己的网站，适当对
share.js相关代码或者css样式文件修改即可。注意这种方式在手机上的展示，最好多做一个按钮，点击分享的时候再弹出分享图标。

```HTML
{{ define "share" }}
  {{ if eq .Site.Params.share_style "ver" }}
	<div style="width: 40px; position: fixed; top: 50%; z-index: 999; left: 25px; transform: translateY(-50%)"> 
		{{ partial "share.html" . }}
	</div>
	{{ end }}
{{ end }}
```

![share](/blog/share_js/002.png)

分享功能在本地用localhost的时候测试会报错或者失败，部署到服务器通过域名分享则功能正常。
