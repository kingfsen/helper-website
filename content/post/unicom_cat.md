---
title: "破解联通HG8347R光猫"
date: 2018-10-26T08:41:19+08:00
description: "破解联通宽带自带的HG8347R光猫，使用自购路由拨号上网"
keywords: "HG8347R,联通光猫破解"
Tags:
- 光猫
Categories:
- 生活技巧
---

安装的联通宽带，自带光猫设备，型号为HG8347R，光猫后面可以清楚的看见这个设备是华为生产的，联通定制版本。
后台管理页面基本只有设备重启这个功能了，其他所有的功能都被屏蔽，无法操作，这个自带的光猫有一些限制。

- 不支持5G
- 只有LAN1口是千兆口
- 穿透能力、信号很弱
- 强制路由拨号，联通限制一个宽带账号只能有一个设备拨号上网，这样我们便不能使用自己的路由器拨号上网
- 限制了无线网络的连接设备数目，归根结底还是他们的光猫能力不能，他们不可能送你一个超级贵的路由

很多人应该都有自己的高档的路由器了，安装了联通的宽带之后肯定想用自己的路由器拨号上网，但是联通的师傅很明确的告诉你不行，还会告诉你不要去乱动光猫，
可能各地的情况不一样，我这坐标是北京昌平。很多人都会想把它破解了，其实我们本次破解目的就是将光猫还原到华为出厂模式，得到超级管理员账号，
然后登录后台管理页面，所有的设置都可以操作了，重要的就是把光猫的路由模式改为桥接模式。

连接wifi，浏览器访问`http://192.168.1.1`,用光猫后面的用户名密码登录，查看设备基本信息。

{{< figure src="/blog/route/001.jpg" width="640px" height="800px" >}}

光猫软件版本是`V3R017C`，这个版本应该是最新的，之前的应该是`15`,`16`版本，都可以进入维护模式，具体的没试过。看了网上一些文章，
说这个`R17`版本已经不提供维护模式了，要进入维护模式必须把固件刷到`R16`版本，刷成功了，这个地方的显示还是`R17`，并不会改变，
不要误以为没有刷成功，以下所有的操作都是在`windows7`，光猫版本`R17`进行的。

1. 下载工具
    如果你是17之前的版本那你没必要刷固件升级，直接维修使能就行，如果是17版本，直接维修使能操作，很快就能看见红色失败提示。这些必备工具，网上随便一搜就有，基本都能用。
    
    - R16固件**hg8347r16-rom.bin**，大概20多M
    - 华为光猫破解工具包，里面包含**华为ONT组播版本配置工具.exe**，**华为光猫配置文件加解密工具.exe**，**tftpd32.exe**，**telnet.exe**。

2. 刷固件
    首先，把光猫后面那个蓝色光纤拔掉，别让联通持续下发配置。然后重启一下光猫，用网线连接笔记本电脑，在电脑上打开**华为ONT组播版本配置工具.exe**，每次打开这个软件之前记得重启一下光猫，否则，软件右侧可能什么也看不见。
    {{< figure src="/blog/route/002.jpg" width="640px" height="800px" >}}
    
    注意网卡选择，这个时候用的网线连接的光猫，打开cmd执行`ipconfig -all`，查看本地连接，ip应该是192.168.1.*，不要选错其他的无线连接网卡。
    ONT版本包就是上面第一步下载的rom.bin文件。点击启动按钮，进度条会不停的来回滑动，右边窗口会有时间，这个进度条不准确，不用管，
    一会儿就可以看见光猫所有的指示灯全都开始闪烁。网上很多文章说8分半钟左右刷完，光猫所有的灯均会常亮，表示固件成功刷完。
    其实不然，我在刷的过程中，持续进行了10分半钟，然后光猫所有的指示灯全都熄灭了，是全都熄灭了，我以为光猫让我给整挂了，立马点击停止按钮，
    手动去重启一下光猫。接着打开win7的cmd，执行`telnet 192.168.1.1`，居然不通(如果telnet命令不存在，win7系统可以直接在控制面板->程序 ->打开功能中添加telnet)，
    这个时候查了一些文章，通过登录192.168.1.1，把防火墙关掉了，再次执行telnet居然通了，这表示成功刷入固件了。
    
3. 获取联通配置文件
    打开cmd，执行`telnet 192.168.1.1`，输入用户名`root`，密码`adminHW`, 有些人的密码可能是`admin`，很快进入了WAP，接着执行`su`，显示success之后，执行`shell`，
    接着就提示进入了简易版的Linux，当然很多命令都没有。首先我们就是要找到当前联通定制版本的配置文件`hw_ctree.xml`，我们可以用`find`命令查找。
    
    ```bash
    find . -name hw_ctree.xml
    ```
    
    命令执行完之后会出现2个文件，一个是`/etc/wap/hw_ctree.xml`，另外一个是`/mnt/jffs2/hw_ctree.xml`，这两个文件都是加密的，联通当前使用的就是mnt目录下的，
    不是etc下的，这个从后面解密出来的明文内容就知道了，我们就是要把这个`/mnt/jffs2/hw_ctree.xml`文件拷贝到本地。
    启动破解工具包中的`tftpd.exe`，不启动这个软件在cmd中执行tftp肯定报错timeout。
    
    启动tftpd服务之后，在cmd中执行
    
    ```bash
    tftp -l /mnt/jffs2/hw_ctree.xml -r hw_ctree.xml -p 192.168.1.*
    ```
    
    ip修改成本地连接ip，命令执行成功之后，hw_ctree.xml则存放在tftpd对应的目录下了。
    
4. 修改配置文件
    打开工具包中的加解密工具.exe，输入文件选择第3步中得到的hw_ctree.xml，输出文件设置为hw_ctree.xml.gz，点击解密。
    然后解压hw_ctree.xml.gz文件，最好用命令gzip进行解压，这时就得到了明文文件。如果你还想要用光猫后面那个用户登录web，则找到X_HW_WebUserInfoInstantce，把UserLevel设置为0，同时PassMode=0，Password弄成明文密码。
    修改完之后，保存文件内容，保证格式正确，再次用加解密工具进行加密，输出文件仍然命名为hw_ctree.xml。
    
5. 恢复光猫出厂模式
    在第3步打开的shell模式下，继续执行`restorehwmode.sh`，然后登待路由器重启，光猫重启之后，就恢复到了出厂模式。
    修改电脑本地连接属性，设置静态ip为`192.168.100.2`，子网掩码为`255.255.255.0`，`网关为192.168.100.1`，通过浏览器访问192.168.100.1，
    就看到了华为默认的登录web页面，用户名`telecomadmin`，密码`admintelecom`登录，登录成功之后，首先导出当前配置到本地进行备份，
    然后再导入第4中加密好的配置文件，如果格式错误，页面直接报错，如果配置正确，路由器会重启。
    
6. 设置wlan桥接
    光猫重启之后，再次把电脑的ip设置为dhcp，浏览器访问`http://192.168.1.1`，这个时候你光猫后面的这个普通管理员可以登录，但是没啥用，
    我们可以用超级管理员`telecomadmin`，密码`admintelecom`登录，进入之后找到WLAN设置，把`3961`那个直接设置为wlan桥接，然后关闭无限网络。
    
7. 路由拨号
    插上光纤，用自己的路由接在光猫的lan1口上，输入宽带账号密码拨号上网即可。