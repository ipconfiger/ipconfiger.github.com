---
layout: post
title: 通则不痛，远离阻塞-在多线程环境中使用CoreData，以及一个简单的封装
---

上回书说道，其实CoreData学起来也没有很复杂，我们其实增删改查都和别的ORM大同小异。但是世界总是很复杂的，一根筋的去考虑问题很容易卡到蛋，默认情况下我们的代码都在Main Thread中执行，数据库操作一旦量多了，频繁了，势必会阻塞住主线程的其他操作，俗话说，卡住了。

这个世界天然是多线程的，所以我们操作数据也必须多线程。CoreData对多线程的支持比较奇怪（按照一般的思路来说），CoreData的NSPersistentStoreCoordinator和NSManagedObjectContext对象都是不能跨线程使用的，NSManagedObject也不行，有人想加锁不就完了。No，作为一个处女座是不能忍受这么丑陋的解决方案的。其实NSManagedObjectContext已经对跨线程提供了内置的支持，只不过方式比较特殊，需要脑洞大开才行。

在创建NSManagedObject的时候有个构造器参数initWithConcurrencyType就是解决的关键所在，这个参数是一个枚举，有三个可选值：

1. NSConfinementConcurrencyType  (或者不加参数，默认就是这个)
2. NSMainQueueConcurrencyType    (表示只会在主线程中执行)
3. NSPrivateQueueConcurrencyType (表示可以在子线程中执行)

那么究竟应该怎么使用才能无阻塞无痛呢？

经过参考： http://www.cocoanetics.com/2012/07/multi-context-coredata/ 这篇文章，我们使用三层 NSManagedObjectContext 嵌套的方式。

NSManagedObjectContext是可以基于其他的 NSManagedObjectContext的，通过 setParentContext 方法，可以设置另外一个 NSManagedObjectContext 为自己的父级，这个时候子级可以访问父级下所有的对象，而且子级 NSManagedObjectContext 的内容变化后，如果执行save方法，会自动的 merge 到父级 NSManagedObjectContext 中，也就是子级save后，变动会同步到父级 NSManagedObjectContext。当然这个时候父级也必须再save一次，如果父级没有父级了，那么就会直接向NSPersistentStoreCoordinator中写入，如果有就会接着向再上一层的父级冒泡......

那么这里如同参考的文章一样，通过三个级别的 NSManagedObjectContext， 一个负责在background更新NSPersistentStoreCoordinator。一个用在主线程，主要执行插入，修改和删除操作，一些小的查询也可以在这里同步执行，如果有大的查询，就起一个新的 NSPrivateQueueConcurrencyType 类型的 NSManagedObjectContext，然后放在后台去执行查询，查询完成后将结果返回主线程。

NSManagedObjectContext在后台线程执行是通过 performBlock 方法来实现的，在传入的匿名block中执行的代码就是在子线程中了，比如


```objectivec
[_bgObjectContext performBlock:^{
    __block NSError *inner_error = nil;
    [_bgObjectContext save:&inner_error];
}];
```

那么如果是查询的话，因为 NSManagedObject 也不能跨线程访问，所以在block里获取到的NSManagedObject对象只能将objectid传到主线程，主线程再通过 objectWithID 恢复对象的方法，如下面的示例：

```objectivec
+(void)one:(NSString*)predicate on:(ObjectResult)handler{
    NSManagedObjectContext *ctx = [[mmDAO instance] createPrivateObjectContext];
    [ctx performBlock:^{
        NSFetchRequest *fetchRequest = [self makeRequest:ctx predicate:predicate orderby:nil offset:0 limit:1];
        NSError* error = nil;
        NSArray* results = [ctx executeFetchRequest:fetchRequest error:&error];
        if (error) {
            NSLog(@"error: %@", error);
            [[mmDAO instance].mainObjectContext performBlock:^{
                handler(@[], nil);
            }];
        }
        if ([results count]<1) {
            [[mmDAO instance].mainObjectContext performBlock:^{
                handler(@[], nil);
            }];
        }
        NSManagedObjectID *objId = ((NSManagedObject*)results[0]).objectID;
        [[mmDAO instance].mainObjectContext performBlock:^{
            handler([[mmDAO instance].mainObjectContext objectWithID:objId], nil);
        }];
    }];
}
```

最后我们将上述的要点结合起来，解决方案就呼之欲出了。


首先我们还是需要一个抽象存储和数据操作的核心

mmDAO.h

```objectivec
//
//  mmDAO.h
//  agent
//
//  Created by LiMing on 14-6-24.
//  Copyright (c) 2014年 Alexander. All rights reserved.
//

#import <Foundation/Foundation.h>

typedef void(^OperationResult)(NSError* error);

@interface mmDAO : NSObject
@property (readonly, strong, nonatomic) NSOperationQueue *queue;
@property (readonly ,strong, nonatomic) NSManagedObjectContext *bgObjectContext;
@property (readonly, strong, nonatomic) NSManagedObjectContext *mainObjectContext;

+(mmDAO*)instance;
-(void) setupEnvModel:(NSString *)model DbFile:(NSString*)filename;
- (NSManagedObjectContext *)createPrivateObjectContext;
-(NSError*)save:(OperationResult)handler;

@end

```
mmDAO.m

```objectivec
//
//  mmDAO.m
//  agent
//
//  Created by LiMing on 14-6-24.
//  Copyright (c) 2014年 Alexander. All rights reserved.
//

#import "mmDAO.h"
#import "mmAppDelegate.h"

static mmDAO *onlyInstance;

@interface mmDAO ()
@property (nonatomic, copy)NSString *modelName;
@property (nonatomic, copy)NSString *dbFileName;
@end

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
    }
    return self;
}

-(void) setupEnvModel:(NSString *)model DbFile:(NSString*)filename{
    _modelName = model;
    _dbFileName = filename;
    [self initCoreDataStack];
}

- (void)initCoreDataStack
{
    NSPersistentStoreCoordinator *coordinator = [self persistentStoreCoordinator];
    if (coordinator != nil) {
        _bgObjectContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
        [_bgObjectContext setPersistentStoreCoordinator:coordinator];

        _mainObjectContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
        [_mainObjectContext setParentContext:_bgObjectContext];
    }

}


- (NSManagedObjectContext *)createPrivateObjectContext
{
    NSManagedObjectContext *ctx = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    [ctx setParentContext:_mainObjectContext];

    return ctx;
}


- (NSManagedObjectModel *)managedObjectModel
{
    NSManagedObjectModel *managedObjectModel;
    NSURL *modelURL = [[NSBundle mainBundle] URLForResource:_modelName withExtension:@"momd"];
    managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
    return managedObjectModel;
}

- (NSPersistentStoreCoordinator *)persistentStoreCoordinator
{
    NSPersistentStoreCoordinator *persistentStoreCoordinator = nil;
    NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:_dbFileName];

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


-(NSError*)save:(OperationResult)handler{
    NSError *error;
    if ([_mainObjectContext hasChanges]) {
        [_mainObjectContext save:&error];
        [_bgObjectContext performBlock:^{
            __block NSError *inner_error = nil;
            [_bgObjectContext save:&inner_error];
            if (handler){
                handler(error);
            }
        }];
    }
    return error;
}


@end

```

然后我们扩展 NSManagedObject，把操作注入到类里，这样子有个好处是可以自动获取类名，然后就不用子级写字符串的类名了。


NSManagedObject+helper.h

```objectivec

//
//  NSManagedObject+helper.h
//  agent
//
//  Created by LiMing on 14-6-24.
//  Copyright (c) 2014年 bangban. All rights reserved.
//

typedef void(^ListResult)(NSArray* result, NSError *error);
typedef void(^ObjectResult)(id result, NSError *error);

#import <CoreData/CoreData.h>
#import "mmDAO.h"


@interface NSManagedObject (helper)

+(id)createNew;

+(NSError*)save:(OperationResult)handler;

+(NSArray*)filter:(NSString *)predicate orderby:(NSArray *)orders offset:(int)offset limit:(int)limit;

+(void)filter:(NSString *)predicate orderby:(NSArray *)orders offset:(int)offset limit:(int)limit on:(ListResult)handler;

+(id)one:(NSString*)predicate;

+(void)one:(NSString*)predicate on:(ObjectResult)handler;

+(void)delobject:(id)object;
@end

```


NSManagedObject+helper.m


```objectivec

//
//  NSManagedObject+helper.m
//  agent
//
//  Created by LiMing on 14-6-24.
//  Copyright (c) 2014年 bangban. All rights reserved.
//

#import "NSManagedObject+helper.h"

@implementation NSManagedObject (helper)
+(id)createNew{
    NSString *className = [NSString stringWithUTF8String:object_getClassName(self)];
    return [NSEntityDescription insertNewObjectForEntityForName:className inManagedObjectContext:[mmDAO instance].mainObjectContext];
}

+(NSError*)save:(OperationResult)handler{
    return [[mmDAO instance] save:handler];
}

+(NSArray*)filter:(NSString *)predicate orderby:(NSArray *)orders offset:(int)offset limit:(int)limit{
    
    NSManagedObjectContext *ctx = [mmDAO instance].mainObjectContext;
    NSFetchRequest *fetchRequest = [self makeRequest:ctx predicate:predicate orderby:orders offset:offset limit:limit];

    NSError* error = nil;
    NSArray* results = [ctx executeFetchRequest:fetchRequest error:&error];
    if (error) {
        NSLog(@"error: %@", error);
        return @[];
    }
    return results;
}


+(NSFetchRequest*)makeRequest:(NSManagedObjectContext*)ctx predicate:(NSString*)predicate orderby:(NSArray*)orders offset:(int)offset limit:(int)limit{
    NSString *className = [NSString stringWithUTF8String:object_getClassName(self)];
    NSFetchRequest *fetchRequest = [[NSFetchRequest alloc] init];
    [fetchRequest setEntity:[NSEntityDescription entityForName:className inManagedObjectContext:ctx]];
    [fetchRequest setPredicate:[NSPredicate predicateWithFormat:predicate]];
    NSMutableArray *orderArray = [[NSMutableArray alloc] init];
    if (orders!=nil) {
        for (NSString *order in orders) {
            NSSortDescriptor *orderDesc = nil;
            if ([[order substringToIndex:1] isEqualToString:@"-"]) {
                orderDesc = [[NSSortDescriptor alloc] initWithKey:[order substringFromIndex:1]
                                                        ascending:NO];
            }else{
                orderDesc = [[NSSortDescriptor alloc] initWithKey:order
                                                        ascending:YES];
            }
        }
        [fetchRequest setSortDescriptors:orderArray];
    }
    if (offset>0) {
        [fetchRequest setFetchOffset:offset];
    }
    if (limit>0) {
        [fetchRequest setFetchLimit:limit];
    }
    return fetchRequest;
}

+(void)filter:(NSString *)predicate orderby:(NSArray *)orders offset:(int)offset limit:(int)limit on:(ListResult)handler{
    
    NSManagedObjectContext *ctx = [[mmDAO instance] createPrivateObjectContext];
    [ctx performBlock:^{
        NSFetchRequest *fetchRequest = [self makeRequest:ctx predicate:predicate orderby:orders offset:offset limit:limit];
        NSError* error = nil;
        NSArray* results = [ctx executeFetchRequest:fetchRequest error:&error];
        if (error) {
            NSLog(@"error: %@", error);
            [[mmDAO instance].mainObjectContext performBlock:^{
                handler(@[], nil);
            }];
        }
        if ([results count]<1) {
            [[mmDAO instance].mainObjectContext performBlock:^{
                handler(@[], nil);
            }];
        }
        NSMutableArray *result_ids = [[NSMutableArray alloc] init];
        for (NSManagedObject *item  in results) {
            NSLog(@"id=%@", item.objectID);
            [result_ids addObject:item.objectID];
        }
        [[mmDAO instance].mainObjectContext performBlock:^{
            NSMutableArray *final_results = [[NSMutableArray alloc] init];
            for (NSManagedObjectID *oid in result_ids) {
                [final_results addObject:[[mmDAO instance].mainObjectContext objectWithID:oid]];
            }
            handler(final_results, nil);
        }];
    }];
}


+(id)one:(NSString*)predicate{
    NSManagedObjectContext *ctx = [mmDAO instance].mainObjectContext;
    NSFetchRequest *fetchRequest = [self makeRequest:ctx predicate:predicate orderby:nil offset:0 limit:1];
    NSError* error = nil;
    NSArray* results = [ctx executeFetchRequest:fetchRequest error:&error];
    if ([results count]!=1) {
        raise(1);
    }
    return results[0];
}

+(void)one:(NSString*)predicate on:(ObjectResult)handler{
    NSManagedObjectContext *ctx = [[mmDAO instance] createPrivateObjectContext];
    [ctx performBlock:^{
        NSFetchRequest *fetchRequest = [self makeRequest:ctx predicate:predicate orderby:nil offset:0 limit:1];
        NSError* error = nil;
        NSArray* results = [ctx executeFetchRequest:fetchRequest error:&error];
        if (error) {
            NSLog(@"error: %@", error);
            [[mmDAO instance].mainObjectContext performBlock:^{
                handler(@[], nil);
            }];
        }
        if ([results count]<1) {
            [[mmDAO instance].mainObjectContext performBlock:^{
                handler(@[], nil);
            }];
        }
        NSManagedObjectID *objId = ((NSManagedObject*)results[0]).objectID;
        [[mmDAO instance].mainObjectContext performBlock:^{
            handler([[mmDAO instance].mainObjectContext objectWithID:objId], nil);
        }];
    }];
}


+(void)delobject:(id)object{
    [[mmDAO instance].mainObjectContext delete:object];

}

@end


```




