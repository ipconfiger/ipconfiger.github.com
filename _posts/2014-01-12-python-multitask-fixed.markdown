--- 
layout: post
title: 关于Python并行任务技巧的几点补完
---

早上逛微博发现了SegmentFault上的这篇文章：[关于Python并行任务技巧](http://segmentfault.com/a/1190000000382873) 。看过之后大有裨益。顺手试了试后遇到几个小坑，记录下来作为补完（作者也有点语焉不详哦^_^）。

第一点是传入的function，只能接收一个传入参数，一开始以为在传入的序列里用tuple可以自动解包成多个参数传进去，可惜实践后是不行的：

    #coding=utf8
    from multiprocessing import Pool

    def do_add(n1, n2):
        return n1+n2

    pool = Pool(5)
    print pool.map(do_add, [(1,2),(3,4),(5,6)])
    pool.close()
    pool.join()
    
执行后结果就报错了：

    Traceback (most recent call last):
      File "mt.py", line 8, in <module>
        print pool.map(do_add, [(1,2),(3,4),(5,6)])
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/multiprocessing/pool.py", line 250, in map
        return self.map_async(func, iterable, chunksize).get()
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/multiprocessing/pool.py", line 554, in get
        raise self._value
    TypeError: do_add() takes exactly 2 arguments (1 given)


第二是传入的function如果要做长期执行，比如放一个死循环在里面长期执行的话，必须处理必要的异常，不然ctrl+c杀不掉进程，比如：

    #coding=utf8
    from multiprocessing import Pool
    import time

    def do_add(n1):
        while True:
            time.sleep(1)
            print n1

    pool = Pool(5)
    print pool.map(do_add, [1,2,3,4,5,6])
    pool.close()
    pool.join()

这段代码一跑起来是ctrl+c杀不掉的，最后只能把console整个关掉才行。
不过这么写就ok了：

    #coding=utf8
    from multiprocessing import Pool
    import time

    def do_add(n1):
        try:
            while True:
                time.sleep(1)
                print n1
        except:
            return n1

    pool = Pool(5)
    print pool.map(do_add, [1,2,3,4,5,6])
    pool.close()
    pool.join()

--------
补完的补完，有网友提供了解决办法，使用functools的partial可以解决，详见 [爆栈](http://stackoverflow.com/questions/5442910/python-multiprocessing-pool-map-for-multiple-arguments)

--------



第三点是为什么要在子进程里用死循环让其长期执行。窃以为作者的直接把上千个任务暴力丢给进程池的做法并不是最高效的方式，即便是正在执行的进程数和CPU数能匹配得切到好处，但是一大堆的进程切换的开销也会有相当的负担。但是创建几个长期运行的工作进程，每个工作进程处理多个任务，省略掉了大量开启关闭进程的开销，原理上来说会效率高一些。不过这个问题我没有实测过。再不过其实从原理上来说这个开销虽然有但是并不是有多么大，很多时候完全可以忽略，比如作者用的例子。
所以其实更确切一点的需求反而是用于实现生产者消费者模式。因为在作者的例子里，任务数是固定的，不可控的，更多的时候我们反而是需要用生产者创建任务，由worker进程去执行任务。举个例子，生产者监听一个redis的队列，有新url放进去的时候就通知worker进程去取。

代码如下：

    #coding=utf8
    from multiprocessing import Pool, Queue
    import redis
    import requests

    queue = Queue(20)

    def consumer():
        r = redis.Redis(host='127.0.0.1',port=6379,db=1)
        while True:
            k, url = r.blpop(['pool',])
            queue.put(url)

    def worker():
        while True:
            url = queue.get()
            print requests.get(url).text

    def process(ptype):
        try:
            if ptype:
                consumer()
            else:
                worker()
        except:
            pass

    pool = Pool(5)
    print pool.map(process, [1,0,0,0,0])
    pool.close()
    pool.join()

比起经典的方式来说简单很多，效率高，易懂，而且没什么死锁的陷阱。




