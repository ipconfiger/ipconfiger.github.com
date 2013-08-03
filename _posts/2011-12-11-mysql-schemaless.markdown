--- 
layout: post
title: MySQL数据库支持Schemaless的数据库存储方案
---

在PyCon上有童鞋提供了一个类似概念的分享，不过不大适合一般类型的互联网项目，感觉有点过于另类。不过我实现这个方案是在看到PyCon的分享之前。算是同样的诉求不同的实现方式吧。且我这里只是实现了一个数据访问的组件而不是Server。

首先本文的方法来自FriendFeed分享的如何使用MySQL数据库的分享。简而言之就是把Python对象直接dumps后zip压缩存储在MySQL一个字段里。这样不就Schemaless了么？存什么数据类型，类什么结构，MySQL都不需要知道，加个属性什么的都不需要修改数据库表结构，对于业务快速变更、快速增长的互联网业务来说再合适不过了。访问对象直接通过主键查询，快速直接。but，查询怎么办？有的童鞋可能会问。OK，查询这事得分两说，如果是简单的检索，可以通过建索引表的方式来解决，或者呢用外部的索引，比如lucent，还能全文检索哦。现在而今眼目下我实现了索引表索引的方式，因为外部的索引方式比较千奇百怪，所以如果需要可以根据具体情况自己来写一个，反正实现相应的几个方法就行。

直接上一个例子来说明。假设要实现一个blog，需要存blog的信息，先定义一个blog的模型类(需要import什么大家自动脑补)

<script src="https://gist.github.com/ipconfiger/6142119.js"></script>

这个connection是因为我还没想好如何能无缝结合到Django中又能兼顾脱离Django独立使用的暂时措施，完成版会去掉

如果在使用django的话只需要 python manage.py shell 然后 Blog.objects.create_table()

这个时候会自动创建模型定义的表和索引表

<script src="https://gist.github.com/ipconfiger/6142155.js"></script>
 

这个时候，可以通过 Blog.objects.create(title=u"标题",content=u"内容",post_date=datetime.datetime.now(),auther=user) 或者Blog.objects.create(title=u"标题",content=u"内容",post_date=datetime.datetime.now(),auther=user.id)

就能够创建一个Blog的对象。这个时候blogs表和两个索引表都会插入数据。不过blogs表中的object列是人类无法理解的火星文..........

通过id直接获取对象  Blog.objects.get(1),根据索引获取  Blog.objects.auther.query(auther=user.id) 或者  Blog.objects.auther.query(auther=user)

这个会生成SQL，SELECT `id` FROM  `blog_idx_auther` WHERE `auther`=%s 然后取出match到对象的id列表，然后遍历id列表，通过 Blog.objects.get(id)获得的对象列表。

Blog.objects.get(id)的时候是有对象缓存的（现阶段通过redis实现），所以经过测试，速度是靠谱的。而相对MangoDB来说，MySQL的数据存储也更加靠谱一点，所以相比换现在而今眼目下还不怎么靠谱的mangodb来作为主存储来说，基于MySQL的Schemaless方案还是相对靠谱的。

 

项目地址：[https://github.com/ipconfiger/free4my](https://github.com/ipconfiger/free4my) 内附demo的blog实现一枚
