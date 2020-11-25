---
title: 『底层探索』14 - KVO 底层探索
toc: true
donate: false
tags: []
date: 2020-11-11 16:14:55
categories: [底层探索]
---

在 Objective-C 中，我们经常用 KVO 来观察对象的属性，当属性发生变化的时候，我们做做一些 UI 更新事情。那么在底层，它是怎么实现的呢？

<!-- more -->

KVO (Key-value observing) 是一个将对象属性的变化直接通知另一个对象的机制。本文的主要探索如下：

- 1、KVO 如何观察普通对象
- 2、KVO 如何观察可变数组
- 3、KVO 的使用注意事项
- 4、KVO 如何一对多观察
- 5、KVO 在底层是如何实现的？
- 6、自定义简易版 KVO

## 基本用法

### 观察普通对象

- 我们可以通过 `addObserver:forKeyPath:options:context:` 方法添加一个观察者。
- 我们可以通过 `removeObserver:forKeyPath` 或者 `removeObserver:forKeyPath:context` 方法来移除观察者。
- 我们可以在 `observeValueForKeyPath:ofObject:change:context:` 这个方法中做一些任务处理，当被观察对象的属性发生改变时，这个方法会被调用。

我们在 `NSObject` 的子类中能够直接调用添加或删除方法，是因为 Apple 在 `NSObject` 的 `NSKeyValueObserverRegistration` 类别中实现了这三个方法。在 `NSKeyValueObserving` 类别中实现了 `observeValueForKeyPath` 方法。

代码示例：

```objc
{
    @public
    NSString *openDate;
}

@property (nonatomic, assign) NSInteger  hour;
@property (nonatomic, assign) NSInteger  minute;
@property (nonatomic, assign) NSInteger  second;
@property (nonatomic, strong) NSString *clockName;

@property (nonatomic, strong) NSMutableArray *openedClocks;

@end
```

```objc
// 添加
[self.user addObserver:self forKeyPath:@"nickname" options:NSKeyValueObservingOptionNew context:NULL];
// 移除 
[self.user removeObserver:self forKeyPath:@"nickname"];
```

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSLog(@"%@", change);
}
```

如果我们给对象的成员变量添加观察者，是否会触发通知回调呢？

```objc
[self.clock addObserver:self forKeyPath:@"openDate" options:NSKeyValueObservingOptionNew context:NULL];

// 设置值
self.clock->openDate = @"2020-12-20";
```

答案是**不会触发**通知回调的，因为给成员变量赋值不是通过 `setter` 方法来完成的。

### 观察可变数组

当我们对一个可变数组添加观察者后:

- 问题1：给可变数组重新赋值，`observeValueForKeyPath` 会回调么？

```objc
self.clock.openedClocks = [NSMutableArray array];
```

- 问题2：调用数组的 `addObject` 方法后，`observeValueForKeyPath` 会回调么？

```objc
 [self.clock addObserver:self forKeyPath:@"openedClocks" options:NSKeyValueObservingOptionNew context:NULL];
```

问题 1 的答案是**会**触发通知回调的，因为会触发 `setter` 方法。

```shell
2020-11-23 11:08:15.418674+0800 KVC&KVO[84975:13905549] {
    kind = 1;
    new =     (
    );
}
```

问题 2 答案是直接调用 `addObject` 方法是**不会**触发通知回调的，因为这不会触发 `setter` 方法。

如果我们在改变数组的内容（add、remove、replace）的时候，也触发通知回调，我们需要用到上文 KVC 中的知识，在 [Accessing Collection Properties](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/AccessingCollectionProperties.html#//apple_ref/doc/uid/10000107i-CH4-SW1) 章节有介绍，我们需要通过 `mutable proxy method` 来获取一个代理对象，然后再调用改变内容的方法，这样才会触发通知回调。

上面的代码，我们可以修改为：

```
[[self.clock mutableArrayValueForKey:@"openedClocks"] addObject:@"morning call"];
```

通知回调打印如下：

```shell
2020-11-23 11:17:16.952374+0800 KVC&KVO[85084:13910871] {
    indexes = "<_NSCachedIndexSet: 0x282c0c980>[number of indexes: 1 (in 1 ranges), indexes: (0)]";
    kind = 2;
    new =     (
        "morning call"
    );
}
```

也许你对打印的 `kind` 比较感兴趣，其实 `kind` 表示内容操作类型。源码定义如下，不然得知它代表的意思。

```objc
typedef NS_ENUM(NSUInteger, NSKeyValueChange) {
    NSKeyValueChangeSetting = 1,
    NSKeyValueChangeInsertion = 2,
    NSKeyValueChangeRemoval = 3,
    NSKeyValueChangeReplacement = 4,
};
```

### 使用注意事项

#### 移除观察者

当我们使用 `KVO` 添加观察者后，需要添加对象的 `dealloc` 的方法中移除观察者，当对象的声明周期不一致的时候，`KVO` 会调用已经释放对象中的方法，从而导致崩溃。

```objc
- (void)dealloc {
    NSLog(@"KVOViewController dealloc");
    
    [self.clock removeObserver:self forKeyPath:@"clockName"];
}
```

#### KVO 开关

如果你想要控制 `KVO` 通知的开关，属性不同，是否触发通知不同。我们可以在要观察的 `class` 中实现 [automaticallyNotifiesObserversForKey](https://developer.apple.com/documentation/objectivec/nsobject/1409370-automaticallynotifiesobserversfo) 方法来实现, 该方法的默认实现是返回 `YES`。 比如在 `Clock` 中：

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"second"]) {
        return NO;
    }
    return YES;
}
```

当然，我们也可以通过手动的方式来触发通知回调。调用 `willChangeValueForKey` 和 `didChangeValueForKey` 可以达到这个目的。比如在 `Clock` 中：

```objc
- (void)setSecond:(NSInteger)second {
    [self willChangeValueForKey:@"second"];
    _second = second;
    [self didChangeValueForKey:@"second"];
}
```

### 一次观察多个属性

如果我们的对象中某一个属性依赖多个属性的结果，想要使用一个 `key` 同时观察多个属性，多个属性中的任意一个属性发生变化，都会触发这个通知回调。我们可以通过 [keyPathsForValuesAffectingValueForKey](https://developer.apple.com/documentation/objectivec/nsobject/1414299-keypathsforvaluesaffectingvaluef) 来返回一个集合来达到这个目的。

比如在 `clock` 中我们需要要观察 `clockTime` 属性，`clockTime` 依赖时分秒的变化：

```objc
[self.clock addObserver:self forKeyPath:@"clockTime" options:NSKeyValueObservingOptionNew context: NULL];
```

在 `Clock.m` 中实现如下方法：

```objc
+ (NSSet<NSString *> *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
    if ([key isEqualToString:@"clockTime"]) {
        NSArray *affectingKeys = @[@"hour", @"minute", @"second"];
        keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
    }
    return keyPaths;
}
```

## 底层探索

在 KVO 官方文档 [Key-Value Observing Implementation Details](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOImplementation.html#//apple_ref/doc/uid/20002307-BAJEAIEE)中，给出了底层实现策略，是采用一种叫 `isa-swizzling` 的技术。

- 1、我们知道 `isa` 指向了对象所属的 class， class 中有方法列表。
- 2、当对象添加观察者后，`isa` 不再指向之前的 class，而是指向一个中间类。所以 `isa` 的值不一定反映对象的实际类。

所以我们应该通过调用 `class` 方法来获得对象所属的类，而不是通过 isa 指针。

### 验证实现机制

#### 探究 isa 指向？

01、添加观察者前后 `isa` 的指向是啥？移除观察者呢？

通过一个简单的打印，就可以验证上面的结论。

```objc
NSLog(@"Before: class -> %@, className -> %s",NSStringFromClass([self.clock class]), object_getClassName(self.clock));
[self.clock addObserver:self forKeyPath:@"second" options:NSKeyValueObservingOptionNew context: NULL];
NSLog(@"After: class -> %@, className -> %s",NSStringFromClass([self.clock class]), object_getClassName(self.clock));
[self.clock removeObserver:self forKeyPath:@"second"];
NSLog(@"Removed: class -> %@, className -> %s",NSStringFromClass([self.clock class]), object_getClassName(self.clock));
```

打印结果如下：

```shell
// 添加观察者前
Before: class -> Clock, className -> Clock
// 添加观察者后
After: class -> Clock, className -> NSKVONotifying_Clock
// 移除观察者后
Removed: class -> Clock, className -> Clock
```

根据打印结果我们可以得出结论，当添加观察者后，`isa` 指向的类是派生类 `NSKVONotifying_Clock`, 添加观察者前和删除观察者后，`isa` 指向的类是相同的，都是 `Clock`。

#### 探究中间类关系

02、`Clock` 和 `NSKVONotifying_Clock` 是什么关系？

我们可以打印出添加观察者前后，`Clock` 的子类来判断，是否是子类的关系。如下的是打印方法，用于打印指定类对象的所有子类。

```objc
- (void)printClasses:(Class)cls {
    /// 获取已注册的 class 数量
    int count = objc_getClassList(NULL, 0);
    /// 第一个元素为 cls
    NSMutableArray *array = [NSMutableArray arrayWithObject:cls];
    Class *classes = (Class *)malloc(sizeof(Class)*count);
    /// 获取已注册的 class
    objc_getClassList(classes, count);
    
    /// 获取 cls 的子类
    for (int i = 0; i < count; i++) {
        if (class_getSuperclass(classes[i]) == cls) {
            [array addObject:classes[i]];
        }
    }
    free(classes);
    
    NSLog(@"classes -> %@", array);
}
```

测试代码：

```objc
[self printClasses:[self.clock class]];
[self.clock addObserver:self forKeyPath:@"clockName" options:NSKeyValueObservingOptionNew context: NULL];
[self printClasses:[self.clock class]];
```

打印结果证明了我们的前面的猜测。`NSKVONotifying_Clock` 是 `Clock` 的子类。

```shell
// 添加观察者前，只有本类
2020-11-23 15:55:27.029416+0800 KVC&KVO[88165:14023728] classes -> (
    Clock
)
// 添加观察者后，除了本类，还有子类
2020-11-23 15:55:27.030822+0800 KVC&KVO[88165:14023728] classes -> (
    Clock,
    "NSKVONotifying_Clock"
)
```

#### 探究中间类的组成

03、 `NSKVONotifying_Clock` 类中，有哪些方法呢？

我们通过 `printAllMethodInClass` 方法，可打印出指定类的所有方法。

```objc
- (void)printAllMethodInClass:(Class)cls {
    unsigned int count = 0;
    Method *methodList = class_copyMethodList(cls, &count);
    
    for (int i = 0; i < count; i++) {
        Method method = methodList[i];
        SEL sel = method_getName(method);
        IMP imp = method_getImplementation(method);
        
        NSLog(@" %@ -> %p", NSStringFromSelector(sel), imp);
    }
    free(methodList);
}
```

测试代码：

```objc
[self printAllMethodInClass:NSClassFromString(@"NSKVONotifying_Clock")];
```

打印结果：

```shell
setClockName: -> 0x1b2c2d520
class -> 0x1b2c2bfd4
dealloc -> 0x1b2c2bd58
_isKVOA -> 0x1b2c2bd50
```

根据打印结果，可以看出，派生的 `NSKVONotifying_Clock` 子类主要实现了 `setClockName`, `class`, `dealloc`, `_isKVOA` 这 4 个方法。

在 [class_copyMethodList](https://developer.apple.com/documentation/objectivec/1418490-class_copymethodlist?language=objc) 的官方文档中，我们找到了如下说明：

> An array of pointers of type Method describing the instance methods implemented by the class—any instance methods implemented by superclasses are not included. 

就是说，这个方法获取到的仅仅是当前类实现的方法，如果继承了父类，是不包含父类实现的方法。

因此，我们可以得出如下结论：

- `class`, `dealloc` 是 override 了 `NSObject` 中的方法。
- `setClockName` 是 override 了父类 `Clock` 中的方法。
- `_isKVOA` 是当前类的实现方法，用于判断是否是 KVO 机制生成的类。

#### 中间类会销毁么？

04、 当我们移除观察者后，`NSKVONotifying_Clock` 类对象是否会被销毁呢？

我们给 `Clock` 增加一个父类 `BaseClock`, 移除观察者后，确保 clock dealloc 了，然后我们在其他 controller 页面调用 `printClasses` 方法来打印，看内存中是否还存在 `NSKVONotifying_Clock` 类。

测试代码：

```objc
[self printClasses:[Clock class]];
```

打印结果如下：

```shell
Clock dealloc 释放了

classes -> (
    Clock,
    "NSKVONotifying_Clock"
)
```

所以，当添加观察者的 clock 被销毁后，内存中注册的中间类 `NSKVONotifying_Clock` 还是存在的。Apple 这么做的原因应该是为了复用，提高效率。


## 自定义 KVO

### 实现策略和步骤

了解了 KVO 的机制后，我们能否自己实现一套 KVO 呢？我们的目标是：

- 优化系统 KVO 的使用步骤，实现 KVO 自动销毁机制。
- 通过函数式编程，将注册和响应绑定在一起。

主要有 3 个主要步骤，如下所示：

步骤 1：注册观察者

- 验证 KeyPath 是否存在对应的 setter 方法
- 动态生成子类，并注册到内存中
- 重写 class 方法，将方法添加到子类中
- 修改 isa 指向，指向中间类
- 添加 setter 方法到中间类
- 保存观察者到列表中

步骤 2：移除观察者

- 删除观察对象列表中的对象
- 重写 dealloc，在方法中，移除观察者和还原对象的 isa 指向

步骤 3：KVO 响应，也就是通知事件回调

- 通过 msgSendSuper 调用父类的方法
- 调用 ViewController 设置的 block

### 参考代码

以下是主要流程的参考代码。完整代码见 [mock-KVC-KVO](https://github.com/muhlenXi/mock-KVC-KVO)。

```objc
// 添加观察者
- (void)mx_addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath handleBlock:(MXBlock)block {
    // 1、检查 setter
    [self checkSetterForKeyPath:keyPath];
    // 2、动态添加子类
    Class newClass = [self createChildClassWithKeyPath:keyPath];
    // 3、修改 isa
    object_setClass(self, newClass);
    // 4、保存观察者
    [self as_saveObserver:observer block:block keyPath:keyPath];
}

// 移除观察者
- (void)mx_removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath {
    NSInteger remain = [self as_removeObserver:observer keyPath:keyPath];
    if (remain <= 0) {
        Class class = [self class];
        object_setClass(self, class);
    }
}
```

```objc
// 检查是否实现 setter 方法
- (void)checkSetterForKeyPath:(NSString *)keyPath {
    Class class = object_getClass(self);
    SEL setter = NSSelectorFromString([self setterForKeyPath:keyPath]);
    Method setterMethod = class_getInstanceMethod(class, setter);
    if (!setterMethod) {
        @throw [NSException exceptionWithName:NSInvalidArgumentException reason:[NSString stringWithFormat:@" %@ 没有 %@ setter 方法", NSStringFromClass(class), keyPath] userInfo:nil];
    }
}
```

```objc
// 生成中间类
- (Class)createChildClassWithKeyPath:(NSString *)keyPath {
    NSString *oldClassName = NSStringFromClass([self class]);
    NSString *newClassName = [NSString stringWithFormat:@"MXKVONotifying_%@", oldClassName];
    
    Class newClass = NSClassFromString(newClassName);
    if (!newClass) {
        // 1、注册 new class
        newClass = objc_allocateClassPair([self class], newClassName.UTF8String, 0);
        objc_registerClassPair(newClass);
        
        // 2、添加 class 方法
        SEL classSEL = NSSelectorFromString(@"class");
        Method classMethod = class_getInstanceMethod([self class], classSEL);
        const char *classType = method_getTypeEncoding(classMethod);
        class_addMethod(newClass, classSEL, (IMP) mxClass, classType);
        
        // 3、添加 dealloc 方法
        SEL deallocSEL = NSSelectorFromString(@"dealloc");
        Method deallocMethod = class_getInstanceMethod([self class], deallocSEL);
        const char *deallocType = method_getTypeEncoding(deallocMethod);
        class_addMethod(newClass, deallocSEL, (IMP) mxDealloc, deallocType);
    }
    
    // 0、添加 setter
    SEL setterSEL = NSSelectorFromString([self setterForKeyPath:keyPath]);
    Method setterMethod = class_getInstanceMethod([self class], setterSEL);
    const char *setterType = method_getTypeEncoding(setterMethod);
    class_addMethod(newClass, setterSEL, (IMP) mxSetter, setterType);
    
    return newClass;
}
```

```objc
// 重写 dealloc 方法
static void mxDealloc(id self, SEL _cmd) {
    Class class = [self class];
    object_setClass(self, class);
    
    objc_removeAssociatedObjects(self);
    
    NSLog(@"%@", [NSString stringWithFormat:@"%@ 释放了", NSStringFromClass([self class])]);
}

// 重写 class 方法
static Class mxClass(id self, SEL _cmd) {
    return class_getSuperclass(object_getClass(self));
}
```

```objc
// 重写 setter 方法
static void mxSetter(id self, SEL _cmd, id newValue) {
    // 转发消息给父类，赋值
    Class superClass = class_getSuperclass(object_getClass(self));
    void (*mx_msgSendSuper)(void *, SEL, id) = (void *)objc_msgSendSuper;
    struct objc_super father;
    father.receiver = self;
    father.super_class = superClass;
    mx_msgSendSuper(&father, _cmd, newValue);
    
    NSDictionary *info = objc_getAssociatedObject(self,  (__bridge const void * _Nonnull)(kMXBlockOAssiociateKey));
    NSString *setterName = [self keyPathFromSelector:_cmd];
    
    // block 回调
    for (NSString *key in info) {
        if ([key containsString:setterName]) {
            MXBlock block = (MXBlock) info[key];
            block(newValue);
        }
    }
}
```

## 后记

我是穆哥，卖码维生的一朵浪花。我们下期见。

## 参考资料

- [1、Key-Value Observing Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)
- [2、facebookarchive/KVOController](https://github.com/facebookarchive/KVOController)