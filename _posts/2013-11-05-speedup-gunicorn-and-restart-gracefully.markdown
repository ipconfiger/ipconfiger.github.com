--- 
layout: post
title: 用meinheld加速gunicorn与优雅的重启gunicorn的worker
---
废话不用多说，异步的worker就是快，不要问为什么，gunicorn支持的多种worker，经过测试meinheld超级快，网上扒的测试结果，见图：
![image](http://www.360ito.com/misc/photo/13.jpg)
没时间自己测了，有空的自己测试吧，下面用最简洁的步骤来实现用meinheld加速gunicorn。

step1:

    #pip install meinheld
    
step2:
修改supervisor的配置，添加

    --worker-class="egg:meinheld#gunicorn_worker”
    
这个参数。

重启，搞定

gunicorn还有一个问题是按照官网的文档用supervisor来管理后，一旦重启就会等很长时间来让mastor进程去boot子进程。官网文档还说了可以通过发送sighub给mastor进程可以“优雅”的重启子进程：[任意门](http://docs.gunicorn.org/en/latest/signals.html)

所以关键步骤就是发送SIGHUP信号给mastor进程就可以了。


    #!/bin/bash
    if [ x$1 == x ]; then
       echo 'which app?'
       exit 1;
    fi
    pid=`ps -ef|grep $1|awk '{ print $3 }'|uniq -c|sort -k1,1nr|head -1|awk '{ print $2 }'`
    rs=`kill -1 -$pid`
    pc=`ps aux|grep $1|wc -l`
    if [ $pc > 1 ]; then
        echo 'restart done'
        exit 0;
    else
        echo 'Error!Not start'
    fi

保存为 restart.sh,chmod +x restart.sh  ,最后如果你的gunicorn是这样启动的：
    
    /usr/bin/gunicorn -w 8 -b 0.0.0.0:8000 --worker-class=egg:meinheld#gunicorn_worker app:app
    
就只需要   ./restart.sh app:app  就可以了

ps bash shell 不识别 -SIGHUP所以会报错 kill -l 找到对应的信号编号，换成 kill -1 就好了

祝玩得开心