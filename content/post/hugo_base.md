---
title: "Hugo入门详细教程"
date: 2019-03-31T12:54:26+08:00
description: "介绍Hugo生成静态网站的基础知识，让你快速入门，轻松部署属于自己的静态站点"
thumbnail: "img/hugo.png"
Tags: 
- hugo
Categories:
- 博客组件
---

首先访问Github下载Hugo的应用程序，Hugo各版本release文件下载地址 https://github.com/gohugoio/hugo/releases ， windows请选择下载hugo_0.xx.0_Windows-64bit.zip。
下载完成止之后解压文件至D:\soft\hugo_0.54，然后把该路径添加到系统环境变量Path中，执行 hugo version 命令验证是否安装成功。

```
C:\Users\kingfsen>hugo version
Hugo Static Site Generator v0.54.0-B1A82C61 windows/amd64 BuildDate: 2019-02-01T09:42:02Z
```

通过执行 hugo --help 查看命令帮助

```Bash
C:\Users\kingfsen>hugo --help
hugo is the main command, used to build your Hugo site.

Hugo is a Fast and Flexible Static Site Generator
built with love by spf13 and friends in Go.

Complete documentation is available at http://gohugo.io/.

Usage:
  hugo [flags]
  hugo [command]

Available Commands:
  config      Print the site configuration
  convert     Convert your content to different formats
  env         Print Hugo version and environment info
  gen         A collection of several useful generators.
  help        Help about any command
  import      Import your site from others.
  list        Listing out various types of content
  new         Create new content for your site
  server      A high performance webserver
  version     Print the version number of Hugo
```

执行 `hugo config`查看hugo的默认配置信息，各配置参数可以通过在站点的全局配置文件config.toml中进行修改。


### 目录结构

在E盘新建目录website，然后通过Hugo把站点生成到E:/website下，比如我现在新建一个站点second-blog，执行如下命令

```Bash
kingfsen@LAPTOP-M85G5JUT MINGW64 /e/website
$ hugo new site E:/website/second-blog
Congratulations! Your new Hugo site is created in E:\website\second-blog.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/, or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>\<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
```
命令执行成功之后查看E:/website/second-blog目录，查看文件目录结构。

```Bash
│  config.toml
│
├─archetypes
│      default.md
│
├─content
├─data
├─layouts
├─static
└─themes
```

- config.toml<br/>
  站点全局的参数配置文件
- archetypes<br/>
  Hugo的markdown文件中前置数据Front Matter定义的结构，默认使用的是default.md文件，可以自定义，然后在config.toml中指定自定义的结构文件，打开default.md文件。
  
    ```Yaml
      ---
      title: "{{ replace .Name "-" " " | title }}"
      date: {{ .Date }}
      draft: true
      ---
    ```
    
    这是一种yaml格式，前面的文章说过，Front Matter支持三种格式，除了yaml，还支持toml，json方式。
    
    toml
    
    ```toml
    +++
    title = "{{ replace .Name "-" " " | title }}"
    date = {{ .Date }}
    draft = true
    +++
    ```
    
    json
    
    ```json
    {
      "title":"{{ replace .Name "-" " " | title }}",
      "date":{{ .Date }},
      "draft":true
    }
    ```
    
    至于喜欢哪种格式，可以在config.toml中进行配置，默认使用的是yaml格式。通过执行hugo new 命令生成的markdown文件，头部默认会有这段渲染之后的Front Matter，一般我们会把draft属性去掉，draft草稿的意思，有这个属性的md文件不会渲染输出，
    当然通过hugo --buildDrafts可以强制输出。
  
- content<br/>
  存放网页内容的目录，即我们编写的markdown文件都存放在此目录，此目录是Hugo的默认源目录，在E:/website/second-blog下执行命令 `hugo new post/hugo_introduce.md`之后，
  则会在content目录下生成子目录post，post中有一个hugo_introduce.md文件。
  
- data<br/>
  data目录用来存放数据文件，一般是json文件，Hugo提供了相关命令可以从data目录下读取相关的文件数据，然后渲染到HTML页面中，将业务数据与模板分离。
  
- layouts<br/>
  存放自定义的模板文件，Hugo优先使用layouts目录下的模板，未发现再去themes目录下查找。
  
- static<br/>
  存放静态文件，比如css、js、img等文件目录，Hugo在渲染时，会直接将static目录下的文件直接复制到public目录下，不会做任何渲染。
  
- themes<br/>
  存放网站主题，可以下载多个主题，themes目录下的每个子目录代表了一个主题，可以通过在config.toml中通过参数theme指定主题，即theme目录下的子目录名字，
  也可以在执行hugo命令渲染时通过增加flag参数--theme=xx指定。
  
### 配置主题

一个静态站点的布局外观离不开css样式，在Hugo中通过主题theme来管理样式，在Hugo的[官方网站](https://themes.gohugo.io/)即可预览下载社区提供的很多主题，当然我们也可以
通过github下载对应的主题，[点击这里](https://github.com/kingfsen/hugoThemes)可以获取Hugo的全部主题，大部分主题提供了图片预览或者Demo在线预览，自由选择下载即可。

这里我选择排在第一个的主题AllinOne，进入E:\website\second-blog\themes，执行git clone命令下载主题。

```
kingfsen@LAPTOP-M85G5JUT MINGW64 /e/website/second-blog/themes
$ git clone https://github.com/orianna-zzo/AllinOne.git
Cloning into 'AllinOne'...
remote: Enumerating objects: 962, done.
remote: Total 962 (delta 0), reused 0 (delta 0), pack-reused 962
Receiving objects: 100% (962/962), 24.18 MiB | 32.00 KiB/s, done.
Resolving deltas: 100% (357/357), done.
```

命令执行成功之后，在themes目录下则有主题目录AllinOne，这个主题中的Example中图片有点多，比较大。在github上可以查看该主题的基本介绍以及详细的参数设置，主题很多都是个人提供出来，可能参差不齐，自行判断。
我们也可以制作自己的主题，上传到github上，或者在github上fork一个主题分支，在别人的基础上进行开发定制。

### 站点调试

Hugo提供了liveload方式，在执行hugo命令时通过增加flag参数即可。服务启动之后，可以一边修改内容文件或者html模板，浏览器会马上刷新，实时展示最新结果，在本地调试开发非常方便。
进入站点根目录second-blog目录，新建一个md文件，就比如我当前这个页面hugo_introduce.md文件，markdown这种轻量型标记语言非常容易学会，花点时间看几遍其语法就能学会。

```
$ hugo new post/hugo_introduce.md
E:\website\second-blog\content\post\hugo_introduce.md created
```

文件创建成功之后，通过其他工具打开hugo_introduce.md文件丰富一下内容，我们可以在本地启动调试，这里我为了方便直接把md文件中的draft属性去掉了。

```
hugo server --watch --theme=AllinOne
```

命令执行之后，发现报了一堆错误，仔细一看就是在主题下的模板强制要求定义`Site.Params.slidesDirPath`等属性，打开config.toml配置文件增加参数即可，由于我这里没有准备图片，
暂时就用AllinOne自带的默认配置，后序准备好了图片直接放在站点根目录下的static即可，再替换路径即可。

```
baseURL = "http://example.org/"
languageCode = "en-us"
title = "Hugo入门详细教程"

[Params]
slidesDirPath = "themes/AllinOne/static/img/header-slides"  
slidesDirPathURL = "img/header-slides"
```

配置增加之后，记得点击保存，再次启动本地服务，命令执行成功之后，服务默认在localhost的1313端口，至于端口也可以自行在config.toml中配置，
不知道参数名是什么，直接执行hugo config命令查看。

```Bash
$ hugo server --watch --theme=AllinOne
Building sites …
                   | EN
+------------------+-----+
  Pages            |   7
  Paginator pages  |   0
  Non-page files   |   0
  Static files     | 108
  Processed images |   0
  Aliases          |   0
  Sitemaps         |   1
  Cleaned          |   0

Total in 53 ms
Watching for changes in E:\website\second-blog\{content,data,layouts,static,themes}
Watching for config changes in E:\website\second-blog\config.toml
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

服务启动之后，用浏览器访问 `http://localhost:1313` ，此时没有文章内容，丰富一下hugo_introduce.md文件，推荐使用Notepad2编写markdown文件，非常简洁，
当然你喜欢通过第三方专业的markdown编辑软件也可以，不过个人感觉真没那个必要。

```Markdown
---
title: "Hugo构建静态站点入门"
date: 2019-03-31T12:54:26+08:00
description: "介绍Hugo生成静态网站的基础知识，让你快速入门，轻松部署属于自己的静态站点"
thumbnail: "img/hugo.png"
Tags: 
- hugo
Categories:
- hugo
---

首先访问Github下载Hugo的应用程序，Hugo各版本release文件下载地址 https://github.com/gohugoio/hugo/releases ， windows请选择下载hugo_0.xx.0_Windows-64bit.zip。
下载完成止之后解压文件至D:\soft\hugo_0.54，然后把该路径添加到系统环境变量Path中，执行 hugo version 命令验证是否安装成功。
```

查看页面，HTML已经实时渲染了，一边编写文章，一边查看页面效果是否与预期一致，非常方便，速度很快。

![theme](/blog/hugo_base/001.png)

### 站点部署

通过直接执行hugo server命令在站点目录下不会生成输出目录public，这个目录是默认的输出目录，当然可以通过命令或者config.toml进行配置。

```Bash
$ hugo --theme=AllinOne
Building sites …
                   | EN
+------------------+-----+
  Pages            |  14
  Paginator pages  |   0
  Non-page files   |   0
  Static files     | 108
  Processed images |   0
  Aliases          |   1
  Sitemaps         |   1
  Cleaned          |   0

Total in 129 ms
 
```

站点调试没问题之后，则可以部署到服务器上了。通过执行hugo命令会将渲染后的站点文件全部输出到站点目录下的public目录中，
然后可以把public目录中的东西直接提交到github上，或者以Github Pages方式发布，或者部署到自己服务器上，由于站点文件均是静态文件，
只需一个Nginx即可将站点运行起来，项目每次有更新，只需先执行 git pull，然后通过命令nginx -s reload重新加载即可。

本篇文章讲解了Hugo的基本入门操作，后面会讲解Hugo的运行机制以及详细的其他站点配置，请通过点击Tag中的hugo或者Category中的hugo阅读Hugo相关博文。