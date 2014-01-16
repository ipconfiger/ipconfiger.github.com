--- 
layout: post
title: 动态映射json数据到objective-c的对象
---

最近在实现IOS的app连接Restful Api的部分功能，发现从Api获取到json反序列化成字典后如果要映射到程序里的实体，需要写很多重复的代码，到处都是字符串，往往json结构比较复杂的时候会极其的蛋痛，所以实现了一个将字典对象映射到objective-c类实例属性的东东，顺便学习一下objective-c的一部分动态特性。

首先搭个架子先，因为要映射的目标很多，所以我们定义个基类，然后需要映射的类继承这个基类就ok了：

    @interface BaseModle : NSObject
    - (id)init:(NSDictionary *)data;
    @end

只需要暴露一个构造器就足够了，放入一个字典，然后一切基类自己搞定。

    #import "BaseModle.h"

    @implementation BaseModle
    -(id)init:(NSDictionary *)data{
        self = [super init];
        if (self){
            [self setAttributes:data];
        }
        return (self);
    }

    - (void)setAttributes:(NSDictionary *)data{
        //我们将在这里完成映射的功能，当然只是功能的起始点
    }
    @end

然后想一想，我们在子类里会定义很多属性，那么我们需要能够获取到这些属性的名称和类型。知道这些后再到字典里去取对应的键值赋值到属性上就ok啦。

要通过反射获取类里定义的属性列表，需要先引入 objc/message.h，这里定义了我们需要的class_copyPropertyList函数和objc_property_t类型。

    unsigned int outCount = 0;
    objc_property_t *properties = class_copyPropertyList(self.class, &outCount);
    for (int i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];
        NSString *propName = [NSString stringWithUTF8String:property_getName(property)];
    }
    
这个时候变量propName就是属性的名称了，这个时候只需要根据属性的名称从data里取对应的键值就ok了。得到值后需要给属性赋值，所以我们需要通过属性名称去构造一个给属性赋值的selector。

    - (SEL)getSetterAttributeName:(NSString *)attributeName
    {
        NSString *capital = [[attributeName substringToIndex:1] uppercaseString];
        NSString *setterSelStr = [NSString stringWithFormat:@"set%@%@:",capital,[attributeName substringFromIndex:1]];
        return NSSelectorFromString(setterSelStr);
    }

这个时候只需要 [self performSelectorOnMainThread:sel withObject:value waitUntilDone:YES]; 就可以给属性赋值了。

这个时候我们可以组织起第一个版本了，不过使用接口的一个喜欢呵呵的家伙说，如果json里有嵌套的字典怎么办呢？所以再进一步实现递归嵌套的功能。这个时候需要知道属性的类型，然后根据类型来动态创建一个新实例，然后把嵌套的字典丢进去继续映射。

先看看怎么获取属性的类型：

    - (NSDictionary *)attributeMapDictionary{
        NSMutableDictionary *propertyDict = [[NSMutableDictionary alloc] init];
        unsigned int outCount = 0;
        objc_property_t *properties = class_copyPropertyList(self.class, &outCount);
        for (int i = 0; i < outCount; i++) {
            objc_property_t property = properties[i];
            NSString *propName = [NSString stringWithUTF8String:property_getName(property)];
            const char *typeName =property_getAttributes(property);
            NSString *properyTypeName = [[NSString alloc] initWithCString:typeName encoding:NSUTF8StringEncoding];
            [propertyDict setObject:properyTypeName forKey:propName];
        }
        return propertyDict;
    }
        
我们在刚才获取属性名的基础上扩展一下，现在我们获取到了属性的名称和类型名，然后生成一个用属性名为Key，属性类型名称作为Value的字典返回。

不过这个时候返回的类型名称是类似：T@"NSString",C,N,V_test 这样子蛋痛的形式，所以还需要一个解析函数来辅助

    - (NSString*) className:(NSString *)propertyTypeName {
        NSLog(@"%@", propertyTypeName);
        NSString* name = [[propertyTypeName componentsSeparatedByString:@","] objectAtIndex:0];
        NSString* cName = [[name substringToIndex:[name length]-1] substringFromIndex:3];
        return cName;
    }
    
这个时侯我们可以得到一个干净的类名了，之后就可以通过NSClassFromString来获得类对象，然后实例化这个类型出来。整理一下第二个版本类似：

    - (void)setAttributes:(NSDictionary *)data{
        NSDictionary *propertys = [self attributeMapDictionary];
        for (NSString *attributeName in [propertys allKeys]){
            SEL sel = [self getSetterAttributeName:attributeName];
            if ([self respondsToSelector:sel]) {
                id value = [data objectForKey:attributeName];
                if (value) {
                    NSString* className = [self className:[propertys objectForKey:attributeName]];
                    if ([value isKindOfClass:[NSDictionary class]]) {
                        id subObject = [[NSClassFromString(className) alloc] init:value];
                        [self performSelectorOnMainThread:sel withObject: subObject waitUntilDone:YES];
                        continue;
                    }                
                    [self performSelectorOnMainThread:sel withObject:value waitUntilDone:YES];
                }
            }
        }
    }
    
这个时候嵌套的类只需要定义属性为：

    @property (nonatomic, strong) MyClass *subObject;

但是再继续想一下，如果嵌套的是数组呢，数组里再有嵌套的字典呢，这个时候的麻烦是需要知道数组里的对象是什么类型，也就是在定义属性的时候要传入一个类型的名称进去。How？最后试了试，在定义protocal的时候获取到的名称会包含protocal的名称，比如定义一个空协议和要嵌套进去的类同名： 
    
    @protocol MyClass
    @end
    
    @interface MyClass : BaseModle
    @property (nonatomic, copy) NSString* test;
    @property (nonatomic, strong) NSArray<MyClass>* subList;
    @end

这个时候获取到的类型名称是：T@"NSArray<MyClass>",C,N,V_mySelf

这个时候修改一下获取类型名称的辅助方法就可以获取到嵌套的类型了。

完整的实现代码：

<script src="https://gist.github.com/ipconfiger/8458066.js"></script>

使用的话：

    #import "BaseModle.h"
    @protocol mmMyObject
    @end

    @interface mmMyObject : BaseModle
    @property (nonatomic, copy) NSString* test;
    @property (nonatomic, copy) NSArray<mmMyObject>* mySelf;
    @end
    
先定义一个子类，然后：

    mmMyObject *myobj = [[mmMyObject alloc] init:@{@"test": @"hahahaha", @"mySelf": @[@{@"test": @"huohuohuohuo"}]}];
    mmMyObject *obj = [myobj.mySelf objectAtIndex:0];
    NSLog(@"%@", obj.test);
    
得到结果：huohuohuohuo

大功告成！！碎叫去也
----




    


        
        
        
        
    