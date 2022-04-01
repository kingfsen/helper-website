---
title: "深入理解Go语言的基础概念"
date: 2019-03-30T13:48:07+08:00
description: "介绍go语言开发入门的一些基础知识，包括go中的一些基本概念、环境配置等"
thumbnail: "img/go.png"
Tags:
- golang
Categories:
- golang
---

这篇文章主要讲解一些go语言开发的入门知识，希望对准备入门学习Go语言的开发者有点帮助。我之前也一直用Java语言进行开发，后来迫于公司的相关容器项目，
就自学Go语言，一边学习一边完成工作。Go语言是Google开源的编程语言，众多开源项目kubernetes、docker、prometheus等都是使用go开发的，个人涉及的项目
如goharbor、clair、chartmuseum、helm、docker registry等。由于Java的相关思维概念先入为主，刚接触Go的时候，有些东西没理解清楚，到了具体用idea开发的时候
挺折腾的。

### 环境配置

访问[Go语言中文网](https://studygolang.com/dl)，下载对应系统版本的Go SDK，以本机64位windows为例，选择go1.x.x.windows-amd64.msi下载，然后将Go安装在目录C:\Go。
安装之后添加一个系统环境变量GOROOT，值即为C:\GO，然后在Path环境变量中添加Go的可执行命令路径，即C:\Go\bin，打开cmd执行go version验证是否安装成功。

 ```Bash
C:\Users\kingfsen>go version
go version go1.11.1 windows/amd64
 ```

Go安装成功之后，则开始配置GOPATH，有点类似Java中的Classpath概念。注意当前go版本依赖GOPATH这个东西，官方说go后面会消除这个东西，不再依赖GOPATH。
选择在E盘新建目录/code/go_workspace，然后把GOPATH添加到系统环境变量，值即为E:\code\go_workspace，然后在GOPATH对应的目录下新建子目录src、bin、pkg，
这三个目录是Go默认约定的，名字不能修改，然后执行go env查看Go相关环境变量是否已经正确设置，同时执行 go env查看与Go有关的所有环境变量，大部分是Go安装时内置的，不要轻易修改。

```Bash
>go env
set GOARCH=amd64
set GOBIN=
set GOCACHE=C:\Users\kingfsen\AppData\Local\go-build
set GOEXE=.exe
set GOFLAGS=
set GOHOSTARCH=amd64
set GOHOSTOS=windows
set GOOS=windows
set GOPATH=E:\code\go_workspace
set GOPROXY=
set GORACE=
set GOROOT=C:\Go
set GOTMPDIR=
set GOTOOLDIR=C:\Go\pkg\tool\windows_amd64
set GCCGO=gccgo
set CC=gcc
set CXX=g++
set CGO_ENABLED=1
set GOMOD=
set CGO_CFLAGS=-g -O2
set CGO_CPPFLAGS=
set CGO_CXXFLAGS=-g -O2
set CGO_FFLAGS=-g -O2
set CGO_LDFLAGS=-g -O2
set PKG_CONFIG=pkg-config
set GOGCCFLAGS=-m64 -mthreads -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=C:\Users\kingfsen\AppData\Local\Temp\go-build872742366=/tmp/go-build -gno-record-gcc-switches
```


- src <br/>
  存放项目源代码，go run等命令执行时的搜索目录，在idea新建go project时，一般选择存放在src目录下，src目录下每一个文件夹代表了一个项目。
- bin <br/>
  存放编译后生成的可执行文件，为了方便，应该把这个目录即$GOPATH/src/bin也添加到系统环境变量Path中，这样目录下的命令在机器上都可直接执行。
- pkg <br/>
  存放Go编译包时生成的.a文件。


Go语言是以包来组织内容的，包即文件夹，每个文件夹下的子文件夹也都是一个包，子文件夹与父文件夹是两个不同的包，如下面某个go文件中同时import了父子两个包，包名
并没有强制要与文件夹名字相同，但一般按照规范package名与文件夹名字一一对应，这样项目结构更清晰，类似log下的.go文件中声明的包名应该都是log，出了main.go文件。

```Go
import "github.com/vmware/harbor/src/common/utils"
import "github.com/vmware/harbor/src/common/utils/log"
```


### 编译命令

Go语言的入口函数是main包下的main方法，在$GOPATH/src下新增项目wechat-backend作为例子，项目结构如下。

```Bash
│  main.go
├─common
│      const.go
│
├─conf
│  │  config.go
│  │
│  ├─env
│  │      loader.go
│  │
│  └─file
│          loader.go
│          loader_test.go
│
├─db
│      db.go
│
├─model
│      user.go
│
├─util
│      file.go
│      net.go
│      str.go
```

其中main.go是声明了main包，定义了main方法，则该文件是项目的入口方法，main包是不能重复定义的。

```Go
package main

import (
	"flag"
	"github.com/gin-gonic/gin"
	"wechat-backend/middleware"
	"wechat-backend/controller"
	"wechat-backend/conf"
)

func main() {
	conf.Init()
	controller.Init()
	router := gin.New()
	router.Use(middleware.Logger())
	router.Run(":" + conf.GetServerPort())
}
```

Go语言中最基本的命令是go build、go run、go install三个命令，也是初学者比较困惑的地方，这里详细讲解一下这三个命令的区别与作用，GOPATH为E:\code\go_workspace。

- go run<br/>
  windows下进入$GOPATH的src目录，执行 
  
    ```Bash
    go run wechat-backend/main.go
    ```
    
    需要显示go run具体的运行过程，可以加上flag参数-n，同时可以通过执行命令 go run -work查看临时目录。
    
    ```Bash
    go run -n wechat-backend/main.go
    ```
    
    Go在windows下虽然设置了GOPATH，但go run在运行时文件参数仍必须是完整的绝对路径或者相对路径，直接在$GOPATH下执行上面go run命令会报错 `The system cannot find the path specified`，必须进入$GOPATH下的src下执行。
    Linux下的Go可以在GOPATH之外的任何地方直接执行`go run -n wechat-backend/main.go` 或者`go run -n wechat-backend`。go run不能运行非main包，比如运行model目录会直接报错。
    
    ```Bash
    > go run -n wechat-backend/model
    go run: cannot run non-main package
    ```
    go run在运行过程中会把项目中的依赖的每个包(包括Go系统包)都编译成一个.a文件，都由于带上-n会输出很多信息，主要是Go系统包以及其他相关import包产生的输出信息，
    这里抽取本项目主要的输出信息，分析一下go run运行的流程，上面的main方法依赖了conf包，首先看conf包的处理。
    
      ```Bash
      #
      # wechat-backend/conf
      #

      mkdir -p $WORK\b120\
      cat >$WORK\b120\importcfg << 'EOF' # internal
      # import config
      packagefile errors=C:\Go\pkg\windows_amd64\errors.a
      packagefile flag=C:\Go\pkg\windows_amd64\flag.a
      packagefile fmt=C:\Go\pkg\windows_amd64\fmt.a
      packagefile log=C:\Go\pkg\windows_amd64\log.a
      packagefile os=C:\Go\pkg\windows_amd64\os.a
      packagefile wechat-backend/common=$WORK\b121\_pkg_.a
      EOF
      cd E:\code\go_workspace\src\wechat-backend\conf
      "C:\\Go\\pkg\\tool\\windows_amd64\\compile.exe" -o "$WORK\\b120\\_pkg_.a" -trimpath "$WORK\\b120" -p wechat-backend/conf -complete -buildid csWsuyfFJATx81ewh3i-/csWsuyfFJATx81ewh3i- -goversion go1.11.1 -D "" -importcfg "$WORK\\b120\\importcfg" -pack -c=4 "E:\\code\\go_workspace\\src\\wechat-backend\\conf\\config.go"
      "C:\\Go\\pkg\\tool\\windows_amd64\\buildid.exe" -w "$WORK\\b120\\_pkg_.a" # internal
      ```
      这个$WORK就是GOCACHE环境变量，一般是用户临时目录，通过go env命令或者go run -work即可获取。
      conf包依赖了Go的errors、flag、fmt、log、os、以及项目自身wechat-backend的common包，因为go的加载规则总是先编译当前包的依赖包，再编译自身的，
      此时conf包的依赖包早已被编译成对应的.a文件，conf包编译步骤。
      
       - 通过mkdir命令在$WORK目录中创建了b120子目录
       - 在b120目录中生成conf包的依赖包声明文件importcfg
       - 执行compile命令编译confg包，编译成功之后输出文件为b120文件夹下的\_pkg_.a
       - 执行buildid命令给b120目录下的.a文件写入一个id，这个id应该就是conf，之后main包编译的时候直接通过这个即可引入这个b120目录下的.a文件
       
      .a文件编译之后，一般某个包下所有源文件(.go)没有发生任何变化则Go默认不会再次编译，除非在go run运行的时候通过参数-a强制编译。main包肯定是最后编译的，
      此时main包依赖的所有包均已编译完成，main包编译逻辑和其它包没什么变化，只不过main包在编译之后还要将其链接成一个可执行程序。
      
      ```Bash
      mkdir -p $WORK\b001\exe\
      cd .
      "C:\\Go\\pkg\\tool\\windows_amd64\\link.exe" -o "$WORK\\b001\\exe\\main.exe" -importcfg "$WORK\\b001\\importcfg.link" -s -w -buildmode=exe -buildid=p-o7yJByJbDCaiuhn0gt/BnjKSB2BhbPjHCHgvSnF/BnjKSB2BhbPjHCHgvSnF/p-o7yJByJbDCaiuhn0gt -extld=gcc "$WORK\\b001\\_pkg_.a"
      ```
      
      b001是main编译后的目录，在链接main程序的时候，在b001下新建了exe子目录，Go通过调用link命令将main编译后的.a文件链接成了可执行程序main.exe，存放在b001/exe目录中，
      此时编译项目工作全部结束，最后直接执行该程序。
      
      ```
      $WORK\b001\exe\main.exe
      ```
      
- go build<br/>
    go build命令如果不带任何参数，则默认编译当前文件夹所有的文件，如果不在$GOPATH目录下执行，直接报错`no Go files`，带上参数则是指定编译$GOPATH目录下的某个包，指定目录下
    包含了main包，则编译之后会在当前目录下生成一个默认与包名一致的可执行文件，如果指定文件夹下无main包，则不会有任何输出结果，它只会检查源文件有效性以及依赖关系。
    
    下面进入$GOPATH/src执行go build命令，观察一下 go build 与 go run 命令的执行区别，通过增加-a参数强制编译。
    
    ```
    go build -a -n wechat-backend
    ```
    
    从命令输出结果信息分析，两个命令执行流程基本一致，只有一点不一样，go build会把可执行文件通过mv命令移动到当前执行命令的路径下，并不会去执行该文件。
    
    ```
    mv $WORK\b001\exe\a.out.exe wechat-backend.exe
    ```
    
    直接go run运行一个非main包，如下面这个命令，强制编译model包，则Go重新编译指定的model包，并不会重新编译其依赖的包。
    
    ```
    go build -a -n wechat-backend/model
    ```
    
    在go build的时候可以通过参数-v显示编译后的文件，通过参数-o指定编译后的文件路径与名字，进入$GOPATH/src/wechat-backend，执行如下命令，则会在当前目录下生成可执行文件chatMe。
    
    ```
    go build -v -o chatMe
    ```
    
- go install<br/>
    该命令是用来编译并安代码包的，它与go build的执行方式基本一致，只不过它会把编译后的文件输出到指定目录。编译后的.a文件会安装在$GOPATH/pkg对应的目录下，可执行文件会移动到
    $GOPATH/bin目录中，执行go install命令查看执行流程。
    
    ```
    mv $WORK\b001\exe\a.out.exe E:\code\go_workspace\bin\wechat-backend.exe
    ```

Go生成的可执行文件之后可以像系统脚本文件一样执行，比如Linux上，通过给可执行程序赋予执行权限即可。

```
chmod u+x wechat-backend
./wechat-backend
```

### 依赖包管理

在项目中，我们不可避免会使用外部的开源依赖包，go在包的依赖管理方面还不成熟，没有Java中使用Maven或者Gradle那样方便。
Go是直接通过设置必须的GOPATH环境变量来管理项目外部依赖包，所以$GOPATH缺失或者目录不存在时，执行go命令的时候会提示错误。
在Go中，可以直接通过go get命令来获取不同代码库的包，比如github.com、golang.org等，执行`go get github.com/gin-gonic/gin`成功之后，
gin-gonic包则会存放在$GOPATH/src/github.com目录下，其他需要依赖该报包的文件直接通过`"github.com/gin-gonic/gin"`引入即可。

Go这种通过GOPATH统一解决依赖包的方式，存在很大的问题，如果一个机器上部署了多个人的开发的项目，如果某个项目开发者修改或者删除了在GOPATH下的某个包，那么
依赖该包的其他项目则会出现编译错误。Go为了解决这个问题，从1.6开始默认开启vendor属性来弥补上述缺陷，实际上这就类似Java项目在maven等工具还未开源流行的时候，直接
在每个项目下新建一个lib文件夹，然后把项目依赖的所有包都拷贝到lib中的方式。Go编译时，优先从当前项目根目录下的vendor目录下查找依赖包，如果vendor下未发现，再去GOPATH
中寻找。这种方式的好处就是各项目依赖包独立管理，互不干扰，缺点就是依赖包很多的时候，项目占用空间变得更大。

vendor同时也带来了新问题，在vendor中的包脱离了版本管理，Go社区为了解决这个问题，在vendor的基础上开发出了godep、govendor、glide、gvt，Go官方也开发了dep。
每个人根据喜好，自由选择，不过大部分开源项目由于历史原因，用的是godep，我个人经常选择使用gvt，下面简单讲讲这两个工具。

- godep<br/>
    docker、k8s、coreos项目早起都用godep，新项目可能会逐渐采用Go的go moudles方式来处理。godep这种方式一般根go get结合使用，
    一般在项目中通过go get命令获取了项目依赖包，正确编译之后，通过执行godep save命令之后，godep会把项目依赖的包以及版本(commit id)写入到
    项目根目录下的Godep/Godeps.json文件中，同时把这些依赖包复制到项目根目录中的vendor目录中。
    
    > * 增加新的依赖包，先执行go get，在代码中通过import导入之后，再次执行godep save命令
    > * 更新依赖包，先执行go get -u github.com/gin-gonic/gin， 再执行godep update github.com/gin-gonic/gin命令
    <br/>
    
- gvt<br/>
  我个人用的更多的是gvt，项目依赖某个包A时，通常这个A包又依赖了其他很多包，不想每次通过go get去获取这些包，因为gvt会递归去把这些包以及依赖包的依赖全都
  下载，并在存放在当前执行gvt命令的路径下的vendor目录，如果vendor存在，会报错，需要手动先删除。不过gvt也有缺陷，当在递归下载依赖包的过程中，某个依赖包下载失败了，
  gvt整个下载程序则停止了，还是需要手动下载包。下载依赖包失败的原因，大部分时候是由于国内网络环境导致，所以找一台能FQ的机器很重要。
  
  - 安装gvt
  
        执行go get，go get成功下载包之后，会自动执行go install命令，通过前面的介绍应该知道，go install会把可执行程序移动到$GOPATH下的bin目录中。
        
        ```
        go get -u github.com/FiloSottile/gvt
        ```
        
        命令执行成功之后，进入$GOPATH/bin目录，则会多了一个gvt程序，之前已经把$GOPATH/bin添加到环境变量Path中了，此时直接执行gvt查看命令帮助即可。
        
        ```Bash
        >gvt
        2019/03/30 21:43:40 WARNING: for go vendoring to work your project needs to be somewhere under $GOPATH/src/
        gvt, a simple go vendoring tool based on gb-vendor.

        Usage:
                gvt command [arguments]

        The commands are:

                fetch       fetch a remote dependency
                restore     restore dependencies from manifest
                update      update a local dependency
                list        list dependencies one per line
                delete      delete a local dependency

        Use "gvt help [command]" for more information about a command.
        ```
        
        gvt给了一个WARNING，告诉我们gvt的工作目录应该是在$GOPATH/src目录下，可以忽略，因为这个gvt fetch命令可以在任何不存在vendor的目录下执行，然后把生成的vendor目录拷贝到
        任何需要的项目下。
        
    - 拉取新包
    
        通过执行gvt fetch xx即可获取对应的包以及依赖包。
        
        ```Bash
          E:\code\go>gvt fetch github.com/astaxie/beego
          2019/03/30 21:57:27 WARNING: for go vendoring to work your project needs to be somewhere under $GOPATH/src/
          2019/03/30 21:57:27 Fetching: github.com/astaxie/beego
          2019/03/30 21:58:08 · Fetching recursive dependency: github.com/siddontang/ledisdb/ledis
        ```
        命令执行之后，会在E:\code\go目录下生成vendor目录，依赖包都在vendor下对应的代码库文件夹中，同时vendor目录下会生成manifest文件，记录了全部下载的依赖包。
        
        ```JSON
        {
          "version": 0,
          "dependencies": [
            {
              "importpath": "github.com/astaxie/beego",
              "repository": "https://github.com/astaxie/beego",
              "vcs": "git",
              "revision": "610f27d6849e227d2b5ff24c5573e071207cc730",
              "branch": "develop",
              "notests": true
            },
            {
              "importpath": "github.com/siddontang/ledisdb/\\vendor\\github.com\\cupcake\\rdb",
              "repository": "https://github.com/siddontang/ledisdb",
              "vcs": "git",
              "revision": "8ceb77e66a9234dc460f28234390a5896a96a584",
              "branch": "master",
              "path": "\\vendor\\github.com\\cupcake\\rdb",
              "notests": true
            }
          }
        ```
        其它命令用法很简单，请自行尝试。
        
### 包加载顺序

Java中Class的加载有双亲委托机制，而Go中虽然没有这个概念，但是Go的文件加载也是有组织顺序的。Go程序的初始化和执行起点是main包，在执行main包的main方法时，其依赖包中的
常量、变量以及init函数必定已经执行完成。当一个包被导入时，如果该包还导入了其它的包，那么会先将其它包导入进来，然后再对这些包中的包级常量和变量进行初始化，接着执行init函数，
下面这个图片表述的更清晰。

![init](/blog/go_base/001.jpg)

package3包中的常量、变量初始化，然后执行init方法，接着package2、package1，所有的包都初始化完成之后，才开始初始化main包中的常量、变量，然后执行init方法，最后执行main方法。
Go中导入包也有会有冲突，通过给某个包定义别名即可解决，例如下面这个文件中导入了beego的context包，同时又导入了Go的context包。

```Go
import (
	"context"
	"fmt"
	"net/http"
	"regexp"
	beegoctx "github.com/astaxie/beego/context"
	)
```

在go中有时仅仅需要执行某个包的init方法，做一些初始化的工作，而实际代码中并未引用到，可以通过 `_`解决，否则编译会报错。

```
_ "github.com/astaxie/beego/session/mysql"
```
