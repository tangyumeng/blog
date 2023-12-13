###  从 NSObject 的初始化了解 isa



####  OC的类是一个结构体

所有的OC类其实都是一个c语言的结构体，可以通过 `clang -rewrite-objc xxx.m ` 命令重写OC类为C++类。来理解OC的类就是一个C语言的结构体。下面是重写的实例，类的定义如下：



```swift
// .h 实现
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface MyClass : NSObject

@property (nonatomic, strong) NSString *name;

- (void)printName;

@end
NS_ASSUME_NONNULL_END


// .m 实现
#import "MyClass.h"

@implementation MyClass

- (void)printName {
    NSLog(@"MyClass printName");
}

@end
```

执行 `clang -rewrite-objc MyClass.m` 后， 从 MyClass.cpp 中可以找到 `typedef struct objc_object MyClass;` 可以知道 OC 类是一个结构体。 

从 runtime 源码中可以找到 

```swift

struct objc_object {
    isa_t isa;
};
```

objc_object结构中只有一个 `isa_t`类型成员。所以我们可以说含有isa的结构体都是一个对象。即： 所有继承自 `NSObject` 的类实例化后的对象都会包含一个类型为 `isa_t` 的结构体。

虽然上面重写 MyClass 后的C语言结构体是一个 `objc_object` 对象，但实际上他是一个`objc_class`结构体，由于`objc_class`继承与`objc_object`， 所以本质上`objc_class` 也是一个`objc_object`结构。
`objc_class`定义如下：

```swfit
struct objc_class : objc_object {
    isa_t isa;
    Class superclass;
    cache_t cache;
    class_data_bits_t bits;
};
```



