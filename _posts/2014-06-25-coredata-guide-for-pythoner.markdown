--- 
layout: post
title: 给Python程序员的CoreData简单指南
---

刚上手的时候都说CoreData学习曲线很陡，其实如果有Hibernate或者SqlAlchemy经验的开发人员来说，只要相应的几个概念对上号了后，学习起来还是感觉挺简单的了。

> 阅读本文需要Python编程经验，Objective－C编程经验以及使用SqlAlchemy的经验

首先我们从概念上来理清楚一个对应关系

数据映射层：

<table border="1">
    <tr>
    	<td width="50%">SqlAlchemy</td>
    	<td width="50%">CoreData</td>
    </tr>
    <tr>
        <td>
        SqlAlchemy的数据映射是由继承自sqlalchemy.ext.declarative下的declarative_base的类来定义的，我们必须手工指定字段的对应关系，写代码来完成映射的工作。
        </td>
        <td>
        CodeData的映射是由xcode维护的XML文件来定义，用工具可以生成对应的实体类文件，运行时通过NSManagedObjectModel的实例加载到程序里。
        </td>
    </tr>
</table>

持久化层：

<table border="1">
    <tr>
    	<td width="50%">SqlAlchemy</td>
    	<td width="50%">CoreData</td>
    </tr>
    <tr>
        <td width="50%">
        SqlAlchemy的持久化层是抽象了对数据库的连接和基本操作，比如：
        DB = create_engine(settings.DB_URI, encoding="utf-8", pool_recycle=settings.TIMEOUT, echo=False)
        所有的sql语句最终都是经过DB对象来执行的
        </td>
        <td width="50%">
        CodeData的持久话是通过NSPersistentStoreCoordinator类的实例来抽象的。如果就使用SqlLite底层持久话来看的化，等于是这个对象保持了对SqlLite的连接和通过他处理所有的SQL语句，事实上在构造的时候还传入了NSManagedObjectModel的实例，所以我猜生成SQL也是在这里完成的。
        </td>
    </tr>
</table>

会话层：

会话层保持了对象状态，对对象的操作都是通过这个层级完成的。
<table border="1">
    <tr>
    	<td width="50%">SqlAlchemy</td>
    	<td width="50%">CoreData</td>
    </tr>
    <tr>
        <td>
        SqlAlchemy的会话层是通过一系列的session对象来实现的，最经常的数据操作都是通过这些session的对象来完成的。比如Session = scoped_session(sessionmaker(bind=DB))，然后  session ＝ Session()。最后得到的session对象就是会话层的对象了。
        </td>
        <td>
        CodeData的会话是通过NSManagedObjectContext类的实例来实现的会话，insert，delete，update和query都是通过会话来操作的。
        </td>
    </tr>
</table>

> 所以基本上ORM的大部分概念是想通的，我们了解了各自概念上相同的，可类比的地方，然后再搞清楚具体每个点上差异，那么基本上就可以很快的掌握新的技术了。

下面从实际的代码上来感受一下相同点和不同点：

## Python Sqlalchemy版 初始化数据库环境：

### model定义：

```
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base, declared_attr

TABLEARGS = {
    'mysql_engine': 'InnoDB',
    'mysql_charset': 'utf8'
}

class DeclaredBase(object):
    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()
    id = Column(Integer, primary_key=True, autoincrement=True)

Base = declarative_base(cls=DeclaredBase)
GroupBase = declarative_base(cls=DeclaredBase)

class User(Base):
    user_id = Column(String(36), unique=True)
    user_name = Column(String(20))
    password = Column(String(36))
    gender = Column(Integer)
    online = Column(Boolean)
    last_update = Column(Integer)
    __table_args__ = TABLEARGS
```
    
### 初始化持久话和会话：

	from sqlalchemy import create_engine
	from sqlalchemy.orm import sessionmaker, scoped_session
	
	DB = create_engine("localhost", encoding="utf-8", pool_recycle=3600, echo=False)
	Session = scoped_session(sessionmaker(bind=DB))
	db = Session()


## CoreData版 初始化数据库环境：

### model定义

值得开心的是，Xcode定义模型只需要点点鼠标填名字，太现代化了
![image](http://ww4.sinaimg.cn/large/578b198bgw1ehqh5ifbwsj20bs0bhq3s.jpg)

工具自动生成子类：

#import <Foundation/Foundation.h>
#import <CoreData/CoreData.h>


	@interface User : NSManagedObject
	
	@property (nonatomic, retain) NSString * avatar;
	@property (nonatomic, retain) NSDecimalNumber * balance;
	@property (nonatomic, retain) NSString * birthday;
	@property (nonatomic, retain) NSString * desp;
	@property (nonatomic) int16_t gender;
	@property (nonatomic, retain) NSString * mobile;
	@property (nonatomic, retain) NSString * nick;
	@property (nonatomic) int32_t user_id;
	
	@end


### 初始化持久话和会话：

这一步顺序上和SqlAlchemy在顺序上有点区别，因为需要在持久话对象里传入数据模型对象，所以需要先实例化数据模型的对象

	//返回实例化后的模型对象
	- (NSManagedObjectModel *)managedObjectModel
	{
	    NSManagedObjectModel *managedObjectModel;
	    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"模型的名字" withExtension:@"momd"];
	    managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
	    return managedObjectModel;
	}
	
	//返回实例化的持久话对象,对应到create_engine
	- (NSPersistentStoreCoordinator *)persistentStoreCoordinator
	{
	    NSPersistentStoreCoordinator *persistentStoreCoordinator = nil;
	    NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"agent.sqlite"];
	
	    NSError *error = nil;
	    persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:[self managedObjectModel]];
	    if (![persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:storeURL options:nil error:&error]) {
	        NSLog(@"Unresolved error %@, %@", error, [error userInfo]);
	        abort();
	    }
	    return persistentStoreCoordinator;
	}

    //返回会话对象，对应到 Session()
    - (NSManagedObjectContext *)createObjectContext:(NSPersistentStoreCoordinator*)coordinator
	{
	    NSManagedObjectContext *ctx = [[NSManagedObjectContext alloc] init];
	    [ctx setPersistentStoreCoordinator:coordinator];
	    return ctx;
	}

相对来说Python的代码肯定短小很多了，Objective-C写起来比较啰嗦，但是其实简单的封装一下用起来也没有这么麻烦了。下回我们来说说封装的事情，这里就简单的丢到一个单例的类里就ok。Xcode的代码模版是把Coordinator和ObjectContext都放到了appDelegate对象里，但是必须

    [(YourAppDelegate*)[[UIApplication sharedApplication] delegate] doYourBusiness];
    
这样子才能访问，比较啰嗦，反正官方的例子都是放全局变量了，我们就用一个简单的单例代替：

	static mmDAO *onlyInstance;
	
	@implementation mmDAO
	+(mmDAO*)instance{
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	        onlyInstance = [[mmDAO alloc] init];
	    });
	    return onlyInstance;
	}
	
	- (instancetype)init
	{
	    self = [super init];
	    if (self) {
	        [self initCoreDataStack];
	    }
	    return self;
	}
	
	- (void)initCoreDataStack
	{
	    NSPersistentStoreCoordinator *coordinator = [self persistentStoreCoordinator];
	    if (coordinator != nil) {
	        _ObjectContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
	        [_ObjectContext setPersistentStoreCoordinator:coordinator];
	    }
	
	}
	
	- (NSManagedObjectModel *)managedObjectModel
	{
	    NSManagedObjectModel *managedObjectModel;
	    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"XXX" withExtension:@"momd"];
	    managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
	    return managedObjectModel;
	}
	
	- (NSPersistentStoreCoordinator *)persistentStoreCoordinator
	{
	    NSPersistentStoreCoordinator *persistentStoreCoordinator = nil;
	    NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"agent.sqlite"];
	
	    NSError *error = nil;
	    persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:[self managedObjectModel]];
	    if (![persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:storeURL options:nil error:&error]) {
	        NSLog(@"Unresolved error %@, %@", error, [error userInfo]);
	        abort();
	    }
	    return persistentStoreCoordinator;
	}
	
	- (NSURL *)applicationDocumentsDirectory
	{
	    return [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask] lastObject];
	}
	
	@end

这样子只需要通过

    [mmDAO instance].objectContext 
    
就可以访问到会话对象来操作数据了

##各自版本的增删改

SqlAlchemy的想法是创建一个对象，放入Context：

    user = User()
    user.user_name = 'Alex'
    ...
    session.add(user)
    
CoreData的想法是从Context里拿一个出来就不用放进去了：

    User *user = [NSEntityDescription insertNewObjectForEntityForName:@"User" inManagedObjectContext:[mmDAO instance].objectContext];
    user.user_name = @"Alex"
    ...

SqlAlchemy删除对象的方法是，把对象丢进context的delete方法

    session.delete(user)
    
CoreData也是这样子

    [[mmDAO instance].objectContext delete:user];

SqlAlchemy插入了对象后，修改了对象的属性后，或者删除了对象后，只需要执行commit，就会将改动全部一次性提交给数据库，其实执行flush就可以，执行commit会提交事务。而CoreData也是通过Context的save方法将数据一次性提交给持久化设施，比如SqlLite。

> 对比到这里其实我们可以发现，ORM的基本实现都趋同化了，在掌握一门ORM的操作后再学习其他的ORM其实也并不存在很抖的学习曲线。学过Hibernate的同学也可以通过相同的方式来类比一下，相信对于掌握CoreData的开发能够起到意想不到的作用



