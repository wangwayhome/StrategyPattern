# 高效扩展 : 单例模式(懒汉式) + 策略模式 实战  

##问题思考：

在App 接入第三方SDK的工作中，我们一般 不会只使用一家播放器，或者不会只使用一家地图，亦或者 一种支付方式。

更多的时候，我们接入的是2家以及以上的SDK。

接入分析：

1. 他们 都包含一些实现方式不同，但 结果相同的功能。比如， 百度 & 高德 都 含有 开始定位，结束定位，暂停定位 等等。

2. 这些 结果相同的功能  实现的 开发思想不通，造成了其提供的接口和功能实现也不尽相同。

3. 假设 提出了一个项目开发中需求变更的情况。就是在地图开发中，需要替换原有地图的场景。

   并且需要保证项目需求变更时代码扩展性。

**综上所述，单例模式 & 策略模式 可以很好的解决这一问题。**



##策略模式

> 策略模式定义了一系列的算法，并将每一个算法封装起来，而且使它们还可以相互替换。策略模式让算法独立于使用它的客户而独立变化。
>
> （原文：The Strategy Pattern defines a family of algorithms,encapsulates each one,and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.）
>
> Context(应用场景):
>
> 1、需要使用ConcreteStrategy提供的算法。
>
> 2、内部维护一个Strategy的实例。
>
> 3、负责动态设置运行时Strategy具体的实现算法。
>
> 4、负责跟Strategy之间的交互和数据传递。
>
> Strategy(抽象策略类)：
>
> 1、 定义了一个公共接口，各种不同的算法以不同的方式实现这个接口，Context使用这个接口调用不同的算法，一般使用接口或抽象类实现。
>
> ConcreteStrategy(具体策略类)：
>
> 2、 实现了Strategy定义的接口，提供具体的算法实现。



一句话总结：

**策略模式的核心就是对算法变化的封装。**

实践核心：

**定义一个通用算法协议，让每个算法遵守其规则。**

仔细想想场景，我们则可以把高德地图，百度地图，共有的方法，抽象出来，放在一个通用的算法协议中去。

让百度 高德 的算法 都需要遵守 协议中的规则。

### 部分代码实现

#### UCarLocationImpl 协议

```objective-c
@protocol UCarLocationImpl <NSObject>

- (void)startUpdatingLocation;
- (void)stopUpdatingLocation;

@end
```

####UCARLocationBaiduImpl 百度算法 封装

```objective-c
#import "UCarLocationImpl.h"

@interface UCARLocationBaiduImpl : NSObject <UCarLocationImpl>

@end


@implementation UCARLocationBaiduImpl

- (instancetype)init
{
    self = [super init];
    if ( self )
    {
        locationManager = [[CLLocationManager alloc] init];
    }
    return self;
}


- (void)startUpdatingLocation
{
    [locationService startUpdatingLocation];
}

- (void)stopUpdatingLocation
{
    [locationService stopUpdatingLocation];
}

```

####UCARLocationAMapImpl 高德算法 封装

```objective-c
#import "UCarLocationImpl.h"

@interface UCARLocationAMapImpl : NSObject <UCarLocationImpl>

@end
 

@implementation UCARLocationAMapImpl

- (instancetype)init
{
    self = [super init];
    if ( self )
    {
        locationManager = [[CLLocationManager alloc] init];
    }
    return self;
}

- (void)startUpdatingLocation
{
    [amapLocationManager startUpdatingLocation];
}

- (void)stopUpdatingLocation
{
    [amapLocationManager stopUpdatingLocation];
}
```



## 单例模式

####主要核心类 UCARLocationService 的实现

根据不同的策略选择不同的算法。

但现在出现了一个问题 ，UCARLocationService究竟 以什么方式 来实现更好，

需要能够保证这个类仅有唯一的实例，并提供一个全局访问点。防止别人实例化多个对象，导致百度和高德出现冲突。显然，我们发现用 单例 模式 来实现，核心类的 策略调度 是比较好的选择。

```objective-c
typedef NS_ENUM(NSInteger ,UCarMapImplementType)
{
    UCarMapImplementType_None = 0
    , UCarMapImplementType_AMap = 1
    , UCarMapImplementType_Baidu = 2
};


@interface UCARLocationService : NSObject

@property(nonatomic, readwrite) UCarMapImplementType    implementType;

+ (instancetype)sharedInstance;

- (void)startUpdatingLocation;
- (void)stopUpdatingLocation;

@end
    
@implementation UCARLocationService

+ (instancetype)sharedInstance
{
    static dispatch_once_t onceToken;
    static UCARLocationService *instance = nil;
    dispatch_once( &onceToken, ^{
        instance = [[UCARLocationService alloc] init];
    });
    return instance;
}

- (instancetype)init
{
    self = [super init];
    if ( self )
    {
        implementType = UCarMapImplementType_None;
    }
    return self;
}

//通过枚举 实现 策略 切换
- (void)setImplementType:(UCarMapImplementType)type
{
    if ( type != implementType )
    {
        implementType = type;
        if ( impl )
        {
            [impl stopUpdatingLocation];
            impl = nil;
        }
        switch (implementType)
        {
            case UCarMapImplementType_AMap:
                impl = [[UCARLocationAMapImpl alloc] init];
                break;
            case UCarMapImplementType_Baidu:
                impl = [[UCARLocationBaiduImpl alloc] init];
                break;
            default:
                break;
        }
    }
}

//此处 策略 的 各自实现
- (void)startUpdatingLocation
{
    [impl startUpdatingLocation];
}

- (void)stopUpdatingLocation
{
    [impl stopUpdatingLocation];
}


```

### **大家可以看到，APP 切换地图只要替换一下枚举值就可以了轻松切换了，而且最新的哪个地图火了，扩展新的地图SDK也是轻而易举，不对客户端造成任何影响。这就是策略模式的好处所在。**



**课后 思考题：策略模式 有哪些缺点，不适用于哪些场景？**

