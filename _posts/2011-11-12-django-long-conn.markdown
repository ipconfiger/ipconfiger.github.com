--- 
layout: post
title: 让Django支持数据库长连接
---

书接上回 

上回我们说到：[《在生产系统使用Tornado WebServer来代替FastCGI加速你的Django应用》](http://www.cnblogs.com/Alexander-Lee/archive/2011/05/02/tornado_host_django.html)

那么现在很流行用一些高性能的nonblock的app server来host Django的应用，这些Server可以看做是一个单进程单线程的程序，然后用nginx在前端反向代理并且负载均衡到N多个后端工作进城来充分利用多CPU的性能，当然这部分的配置工作在上回已经说得很清楚了。但是对于Django来说有一个问题。因为Django的数据库连接是在查询的时候实时创建的，用完就会关掉，这样就会频繁的开闭连接。但是对于Tornado这种Server来说这种方式是低效的。这种Server最高效的工作模式是每个进程开启一个连接，并长期保持不关闭。本文的目的就是尝试使Django改变一贯的作风，采用这种高效的工作模式。本文基于Django1.3的版本，如果是低版本可以稍加更改一样可以使用。

Django的数据库可以通过配置使用专门定制的Backend，我们就从这里入手。

首先我们看看Django自带的Backend是如何实现的。在Django官网上可以看到自带MySql的Package结构，可以点击 此处 前往瞻仰。

通观源码我们可以发现，Django基本上是封装了MySQLdb的Connection和Cursor这两个对象。而且重头实现整个Backend既不实际而且也不能从根本上解决问题。所以我们可以换一个思路。所有的数据库操作都是从获取Connection对象开始的，而获取Connection对象只有一个入口，就是MySQLdb.connect这个函数。所以我们只需要包装MySQLdb这个模块，用我们自己的connect方法替代原本的，这样就从根源上解决了问题。我们在包装器内部维护MySQLdb的Connection对象，使其保持长连接，每次connect被调用的时候判断一下，如果连接存在就返回现有连接，不就完美了吗？所以我们可以分分钟写下第一个解决方案：

<script src="https://gist.github.com/ipconfiger/6141949.js"></script>

把上面代码存到一个叫pool.py的文件里。然后把Django源码里的db/backend/mysql这个package拷贝出来，单独存到我们project目录里一个mysql_pool的目录里。然后修改其中的base.py，在顶上import的部分，找到 import MySQLdb as Database 这句，用下面代码替换之

<script src="https://gist.github.com/ipconfiger/6141966.js"></script>

这样我们就用自己的模块替换了MySQLdb的，当要connect的时候判断到有连接的时候就不重新创建连接了。

把站点跑起来看，结果如何？刷新几次后报错了。Why？看看日志可以看到如下的错误：

<script src="https://gist.github.com/ipconfiger/6142422.js"></script>

看来我们光是包装了MySQLdb本身还不行，在connect后Django获取了Connection的对象，之后就能为所欲为，他用完后很自觉的关掉了，因为他直觉的以为每次connect都拿到了新的Connection对象。所以我们必须把Connection对象也包装了了。所以升级后的解决方案代码如下：

<script src="https://gist.github.com/ipconfiger/6141996.js"></script>

　　我们增加了一个_ConnectionWrapper类来代理Connection对象，然后屏蔽掉close函数。把站点跑起来后发现不会出现之前的问题了，跑起来也顺畅不少。但是过了几个小时后问题又来了。因为MySQLdb的Connection有个很蛋痛的问题，就是连接闲置8小时后会自己断掉。不过要解决这个问题很简单，我们发现连接如果闲置了快8小时就close掉重新建立一个连接不就行了么？所以最后解决方案的代码如下：
　
<script src="https://gist.github.com/ipconfiger/6142002.js"></script>

就此问题解决，世界终于清净了
