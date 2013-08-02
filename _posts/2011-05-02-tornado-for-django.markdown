--- 
layout: post
title: 在生产系统使用Tornado WebServer来代替FastCGI加速你的Django应用
---

由于换了工作，所以之前的游戏引擎暂时放下，但是不会停止的，这个项目会在我的业余时间来完成。

---------------------------------------闷骚的分割线，下面是正文-------------------------------------------

tornado我在之前的文章里已经有过多次介绍，在此就不详细介绍了，

详细介绍可参见：

[玩蛇记-使用tornado构建高性能Web应用之一](http://www.cnblogs.com/Alexander-Lee/archive/2010/03/20/1690292.html) 

[玩蛇记-使用Tornado构建高性能Web之二-autoreload](http://www.cnblogs.com/Alexander-Lee/archive/2010/03/24/1693367.html)

由于官网被墙，讨论组也被墙（囧，万恶的墙）所以tornado的资料很少，官网的资料也语焉不详，所以很多童鞋对如何部署使用Tornado心里没底。所以本文的主要目的就是教会刚入门的新手如何在生产环境使用Tornado

Tornado是一个异步web框架和服务器，所以在开发longpulling的chat之类应用非常的合适，但是其实本身也是一个高性能的http服务器，也可以作为一个WSGIServer。所以即使你的网站没有使用Tornado的框架，而是用了web.py或者是Django来开发（傻瓜万岁），这个时候Tornado依然可以用来加速你的网站。使用Tornado来代替fastCGI可以大幅提高性能，且可以承载的并发能力也有了成倍的提高（大家可以自己Profile，本文只介绍如果做）。

下面我们开始来介绍如何配置。这里我们假设你的一个用Django写的网站在一台Linux的服务器上快乐地着（ubuntu or CentOS，没试过在其他发行版折腾过，windows？你在说笑吧），随着网站越来越红火，你越发感觉服务器不堪重负。这个时候Tornado出现了，他可以让你再苟延残喘好几个月，节约一大把的银子去把妹.............回到正题。根据官网的推荐部署方式，我们还是采用Nginx通过upstream来反向代理到N个Tornado的服务器实例上的部署方式。so

Setp1：安装supervisord

由于Tornado并没有自身提供Daemon的能力，所以我们需要用一个服务管理工具来管理Tornado的进程，supervisord是用Python实现的一款非常实用的进程管理工具。可以很方便的管理N过进程，且支持进程分组。Supervisord可以通过sudo easy_install supervisor安装，当然也可以通过Supervisord官网下载后setup.py install安装。

Step2: 给Django的站点增加一个Tornado的服务器文件（比如serv.py）

创建一个文件Serv.py在Django站点的根目录，内容如下：

<script src="https://gist.github.com/ipconfiger/6142227.js"></script>

我这里通过第一个参数来指定Tornado服务监听的端口。这样比较灵活，这点我们在后面的步骤会用到。这个时候我们可以通过

python Serv.py 8000

这个命令来启动服务器

Step3: 配置Supervisord

第一步安装的Supervisord还没有配置，所以我们需要先创建一个配置文件的样板。在root权限下执行

echo_supervisord_conf > /etc/supervisord.conf

这个时候在/etc/创建了配置文件，用vim打开这个文件，在配置文件的屁股后面加上以下这一段

<script src="https://gist.github.com/ipconfiger/6142234.js"></script>

这个配置会启动4个Tornado的服务进程分别监听 8001,8002,8003,8004 这四个端口

command这一行是要执行的命令，这里是用 python /var/www/site/Serv.py 端口号来启动Tornado的服务进程 80%(process_num)02d 的用途是通过进程编号来生成端口号。下面的process_name这个参数也会用到。这里要指定的文件名就是上一步我们创建那个Serv.py文件

process_name是进程的名字，由于这里要启动4个进程，所以要用process_num来区分

umask是程序执行的权限参数

startsecs这个参数是程序启动的等待时间

stopwaitsecs这个参数是程序停止的等待时间

redirect_stderr这个参数将错误流重定向到std的流输出，这样可以省去一个日志文件的配置，当然也可以不用这个参数分开配置日志文件

stdout_logfile 这个参数是STD流输出日志文件的路径，Tornado会输出所有的请求和错误信息，通过这个可以统一做日志处理，分隔什么的，在程序里就只需要print到std流就行了。

numprocs 这个参数指定了进程的数量，这里是4，表明要启动4个Tornado进程

numprocs_start 这个参数指定了进程号的起始编号，这里是1，这样前面的command和process_name里的%(process_num)02d部分就会在执行的时候被替换为01~05的字符串

配置修改完成后:wq保存退出，执行：

supervisorctl reload

重新加载配置后，这些进程就启动起来了

Step4：修改配置Nginx

首先找到在vhost目录里你的站点配置文件，打开后，在头上增加upstream的内容

<script src="https://gist.github.com/ipconfiger/6142241.js"></script>

然后在Server配置节里找到

 location 这个配置节

将内部清空，在其中加入upstream的配置，变成如下样子


<script src="https://gist.github.com/ipconfiger/6142265.js"></script>

保存配置文件后执行  让nginx重启的指令 nginx -s reload(注意 nginx文件在不同发行版中位置有差别)

然后你就能够通过域名看到你的网站了，试试是不是快多了

注意：生产系统下开启多少个Tornado进程比较好呢，这个见仁见智了，据我压力测试的结果看来，用CPU核数x2的数量最好，再多 就浪费了没有提升（为什么乘2？因为有种CPU上的技术叫超线程）。我的VPS上用的4个进程。如果是8核IntelCPU要挖尽CPU潜能的话需要开16个进程