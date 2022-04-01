---
title: "高效的静态网站生成器Hugo"
date: 2019-03-19T20:05:49+08:00
description: "Hugo是一款用Go编写的静态网站生成器，简单、快速、高效"
Tags:
- hugo
Categories:
- 博客组件
thumbnail: "img/hugo.png"
---

静态网站比较适用于博客、宣传、文档类的站点，比动态站点占用资源更小，访问速度更快。当前最流行的Wordpress博客系统功能很强大，插件也很丰富，
但是整体感觉很臃肿，占用资源也比较大，同时还需要部署一个MySQL数据库，国内有一些个人或者小团队研发了一些开源的轻量级博客，根本没什么吸引力。
越来越多的人更喜欢使用静态博客，甚至一些企业同样采用静态布局官网，因为静态网页可以托管在github等一些平台上，例如下面这些开源项目。

- kubernetes，https://kubernetes.io
- prometheus，https://prometheus.io
- hogo，https://gohugo.io

当前比较流行的静态网站生成器有hexo、jekyll、hugo等，这些软件有成熟的社区，社区中有很多人无私奉献的主题、插件等，本博客则是由hugo生成器构建的。
静态网站生成框架一般都支持Markdown渲染，Markdown是最简洁最容易上手的轻量级标记语言，可以利用Markdown快速编写具备统一格式的文章。
这里暂不比较hexo、jekyll、hugo优缺点，只介绍如何使用hugo构建独特风格的博客网站，选择hugo作为博客生成器绝对事半功倍，构建静态网站的速度很快。

hugo是由Go语言编写的开源项目，在国内基本没有中文资料，但是官网的英文文档手册很全面，并且有一些使用小案例描述。
工作之余，每天在hugo的官方网站阅读几篇文档，两周下来大致看完整个文档手册了，英文资料写的非常详细，就是缺少基本的使用demo。

- hugo官方网站: https://gohugo.io 
- github项目地址: https://github.com/gohugoio/hugo
- hugo原始文档: https://github.com/gohugoio/hugoDocs

要利用hugo构建网站，必须理解清楚hugo组织HTML内容的设计概念。看完文档，我马上想到了著名的前端开源框架Vue.js，每个可以页面定义Vue Object，
然后通过Vue把data渲染到HTML中的Element Object上，作为一个后端程序员，基本的css、html、js技能知识还是必备的，何况还准备搭建网站呢。

```JS
<div id="vue_div">
    <h1>site : {{site}}</h1>
    <h1>url : {{url}}</h1>
    <h1>{{details()}}</h1>
</div>
<script type="text/javascript">
    var vm = new Vue({
        el: '#vue_div',
        data: {
            site: "Vue教程",
            url: "www.younedless.com",
            alexa: "10000"
        },
        methods: {
            details: function() {
                return  this.site + " - 学的不仅是技术，更是梦想！";
            }
        }
    })
</script>

```

hugo的思想跟这个Vue.js相似，在每个markdown文件中，都可以定义一些前置数据，官方的专业词语是Front Matter，定义之后在hugo的Page(html)中则可使用。
hugo定义前置数据的格式有三种，分别为yaml、toml、json，下面展示了本篇文章的markdown文件开头一部分字符，Front Matter是由yaml格式定义的，即前后
使用---包裹住，如果采用toml格式请使用前后三个+++，json则是{}。

```
---
title: "高效的静态网站生成器Hugo"
date: 2019-03-19T20:05:49+08:00
description: "Hugo是一款用Go编写的静态网站生成器，简单、快速、高效"
Tags:
- hugo
Categories:
- 开源分享
thumbnail: "img/hugo.png"
---

静态网站比较适用于博客、宣传、文档类的站点，比动态站点占用资源更小，访问速度更快。当前最流行的Wordpress博客系统功能很强大，插件也很丰富，
但是整体感觉很臃肿，占用资源也比较大，同时还需要部署一个MySQL数据库，国内有一些个人或者小团队研发了一些开源的轻量级博客，根本没什么吸引力。
越来越多的人更喜欢使用静态博客，甚至一些企业同样采用静态布局官网，因为静态网页可以托管在github等一些平台上，例如下面这些开源项目。

```

toml格式

```Markdown
+++
title = "高效的静态网站生成器Hugo"
date = 2019-03-19T20:05:49+08:00
description = "Hugo是一款用Go编写的静态网站生成器，简单、快速、高效"
Tags = ["hugo"]
Categories = ["开源分享"]
thumbnail = "img/hugo.png"
+++
```

json格式就不用多说，大家非常熟悉。

```
{ 
	"title":"高效的静态网站生成器Hugo"
}
```

前置数据之后就是文档内容了，hugo会利用html模板将这些内容进行渲染，在模板中可以通过hugo内置的函数或者标签获取Front Matter，比如显示title的某个html模板。

```
{{ with .Params.title }}
	<span class="title">{{- . -}}</span>
{{ end }}

```

这篇文章就讲解到这里，后续会继续写一些hugo入门使用的文章，可以通过点击Tags中的hugo标签筛选hugo相关文章。