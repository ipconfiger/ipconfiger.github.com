--- 
layout: post
title: 用Redis作为后端来实现高性能的Longpolling消息系统
---

Redis是一个高性能的内存KV数据库，但是由于其支持了各种数据结构，所以我们还可以视其为一个高性能的数据结构，由此可以扩展出很多其他用途，比如用于实现一个可扩展的高性能消息系统。消息系统的前端可以用TCP也可以用HTTP来实现，TCP长连接在移动端受到很多限制，比如很多移动网关会自动关闭长时间空闲的TCP连接，所以还需要实现心跳，心跳频率也需要小心设计。而HTTP协议则是在互联网领域畅通无阻的协议，可以用在各种地方，但是HTTP是一个由客户端发起的无会话协议，轮询获取消息对服务端压力是很大的考验而且实时性也不是很好，要从服务端主动推送数据也是不可能的。但是Longpulling概念的推出解决了这个问题，服务器压力适中，实时性不输TCP长连接，在移动互联的时代也具备意义的事情是“长”轮询，正好规避了心跳包的问题，所以我们用Longpulling的http服务来作为消息服务的前端。

基本思路是这样子的。我们用tornado作为前端，手机App每两分钟发起一个http请求到tornado，请求会带一个客户端标识，比如user_id什么的，tornado收到这个ID后，对Redis发起一个blpop的请求，用这个ID作为key，这样子会阻塞这个请求直到超时，或者有一个值被push到这个key。

由于需要非阻塞的访问Redis，需要一个叫tornado-redis的库。

我们先 pip install tornado tornado-redis 安装需要的库

tornado的代码如下：

    from tornado import ioloop
    from tornado import web
    from tornado import gen
    import tornadoredis
    import logging
    import settings
    
    class MainHandler(web.RequestHandler):
        @web.asynchronous
        @gen.engine
        def get(self):
            tokens = self.get_argument("token").split(",")
            c = tornadoredis.Client(host=’127.0.0.1‘, port=6379, selected_db=1)
                message = yield gen.Task(c.blpop, tokens, 120)
            if message:
                self.finish(message.values()[0])
            else:
                self.finish("{}")
                
    class SendHandler(web.RequestHandler):
        @web.asynchronous
        @gen.engine
        def post(self):
            to = self.get_argument("target")
            message = self.get_argument("message")
            c = tornadoredis.Client(host=’127.0.0.1‘, port=6379, selected_db=1)
                message = yield gen.Task(c.blpop, tokens, 120)
            rs = yield gen.Task(c.rpush, to, message)
            self.finish(json.dumps(dict(rs=True)))
        

    application = web.Application([
        (r"/", MainHandler),
        (r"/send", SendHandler)
    ], debug=True)

    if __name__ == "__main__":
        application.listen(int(sys.argv[1]))
        loop = ioloop.IOLoop.instance()
        loop.start()
        
上面的代码实现了一个基于redis的消息服务，支持消息持久化，支持点对点。

然后我们可以写一个IOS的App来测试这个broker：

核心Objective-C代码大致是下面这样子的：

    -(void) startListen:(void (^)(BOOL, NSString *))handler{
        NSMutableURLRequest* request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:self.url]];
        request.timeoutInterval = 120;
        if(queue==nil){
            queue = [[NSOperationQueue alloc] init];
        }
        NSLog(@"start listen");
        [NSURLConnection sendAsynchronousRequest:request queue:queue completionHandler:^(NSURLResponse *response, NSData *data, NSError *connectionError) {
        //
            if(connectionError){
                NSLog(@"%@",connectionError);
            }else{
                NSString* txt = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
                NSLog(@"%@",txt);
                handler(YES, txt);
            }
            [self startListen:handler];
        }];
    }

如果觉得tornado的性能还不够的话还可以换node.js来。下面是一个参考的实现方式：


    var cluster = require('cluster');
    var http = require('http');
    var url = require('url');
    var util = require('util');
    var querystring = require('querystring');
    var redis = require('redis')
    var numCPUs = require('os').cpus().length;

    function processResp(res, err, item){
        if(!err && item){
            var data = JSON.parse(item[1]);
            res.end(JSON.stringify(data));
        }else{
            res.end(JSON.stringify(JSON.parse("{}")));
        }
    }

    function startServer(port){
        http.createServer(
            function (req, res) {        
                var arg = url.parse(req.url).query;
                var params = querystring.parse(arg);
                res.writeHead(200,{'Content-Type':'application/json'});
                var queue = params.token;
                var r = redis.createClient(6379, '127.0.0.1');
                r.blpop(queue, 10, function(err, item) {
                    processResp(res, err, item);
                });
            }
        ).listen(port, "127.0.0.1");
    }

    if (cluster.isMaster){
        console.log('master stared');
        for (var i=0;i<numCPUs;i++){
            cluster.fork();
        }
        cluster.on('listening', function(worker, address){
        });
    }else if (cluster.isWorker){
        startServer(8888);
    }

发送没写，需要的自己去实现。

扩展的方法：

用多组Redis组成多个区域，然后各自用一个域名来标识，比如 msg0到msg10，每个域一个redis实例，登录的时候给客户端分配一个区域就ok了。

如果想支持群组消息，就用pubsub来实现

