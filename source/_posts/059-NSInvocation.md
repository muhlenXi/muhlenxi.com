---
title: 通过 NSInvocation 调用对象的方法
toc: false
donate: false
tags: [Swift]
date: 2019-05-04 20:54:29
categories: Swift
---

在知道一个类的类名和方法的前提下，如何调用这个类的指定方法呢？

<!-- more -->

## 实现步骤

1. 创建 targetClassName （NSString）
2. 创建 targetClass (Class)
3. 创建 target （NSObject）
4. 创建 actionString （String）
5. 创建 selector （SEL）
6. 创建 methodSignature （NSMethodSignature）
7. 创建 returnType （char）
8. 创建 invocation （NSInvocation）

## 代码

```objc
NSString * const kMediatorParamsKeySwiftTargetModuleName = @"kMediatorParamsKeySwiftTargetModuleName";


@interface Mediator ()

@property (nonatomic, copy) NSMutableDictionary *cachedTarget;

@end

@implementation Mediator

+ (instancetype)sharedInstance {
    static Mediator *mediator;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        mediator = [[Mediator alloc] init];
    });
    
    return mediator;
}

- (id)performActionWithUrl:(NSURL *)url completion:(void (^)(NSDictionary *))completion {
    NSMutableDictionary *params = [NSMutableDictionary dictionary];
    NSString *urlString = [url query];
    
    for (NSString * params in [urlString componentsSeparatedByString:@"&"]) {
        NSArray *elements = [params componentsSeparatedByString:@"="];
        if (elements.count <2) {
            continue;
        }
        
        [params setValue:elements.lastObject forKey:elements.firstObject];
    }
    
    NSString *actionName = [url.path stringByReplacingOccurrencesOfString:@"/" withString:@""];
    if ([actionName hasPrefix:@"native"]) {
        return @(NO);
    }
    
    id result = [self performTarget:url.host action:actionName params:params shouldCacheTarget:NO];
    if (completion && result) {
        completion(@{@"reuslt": result});
    } else {
        completion(nil);
    }
    return result;
}

- (id)performTarget:(NSString *)targetName action:(NSString *)actionName params:(NSDictionary *)params shouldCacheTarget:(BOOL)shouldCacheTarget {
    
    NSString *swiftModuleName = params[kMediatorParamsKeySwiftTargetModuleName];
    
    // 1、创建 targetClassName
    NSString *targetClassName;
    if (swiftModuleName.length > 0) {
        targetClassName = [NSString stringWithFormat:@"%@.Target_%@", swiftModuleName, targetName];
    } else {
        targetClassName = [NSString stringWithFormat:@"Target_%@", targetName];
    }
    
    NSObject *target = self.cachedTarget[targetClassName];
    if (target == nil) {
        // 2、创建 targetClass
        Class targetClass = NSClassFromString(targetClassName);
        // 3、创建 target
        target = [[targetClass alloc] init];
    }
    
    if (shouldCacheTarget) {
        self.cachedTarget[targetClassName] = target;
    }
    
    // 4、创建 actionString
    NSString *actionString = [NSString stringWithFormat:@"Action_%@", actionName];
    // 5、创建 selector
    SEL selctor = NSSelectorFromString(actionString);
    
    if ([target respondsToSelector:selctor]) {
        return [self safePerformAction:target action:selctor params:params];
    } else {
        return nil;
    }
}



- (id)safePerformAction:(NSObject *)target action:(SEL)action params:(NSDictionary *) params {
    // 6、创建 methodSignature
    NSMethodSignature *methodSignature = [target methodSignatureForSelector:action];
    if (methodSignature == nil) {
        return nil;
    }
    
    // 7、创建 returnType
    const char* returnType = [methodSignature methodReturnType];
    
    if (strcmp(returnType, @encode(void)) == 0) {
        // 8、创建 invocation
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
        [invocation setArgument:&params atIndex:2];
        [invocation setTarget:target];
        [invocation setSelector:action];
       
        [invocation invoke];
        
        return nil;
    }
    
    if (strcmp(returnType, @encode(BOOL)) == 0) {
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
        [invocation setTarget:target];
        [invocation setSelector:action];
        [invocation setArgument:&params atIndex:2];
        [invocation invoke];
        
        BOOL result = 0;
        [invocation getReturnValue:&result];
        return @(result);
    }
    
    if (strcmp(returnType, @encode(CGFloat)) == 0) {
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
        [invocation setTarget:target];
        [invocation setSelector:action];
        [invocation setArgument:&params atIndex:2];
        [invocation invoke];
        
        CGFloat result = 0;
        [invocation getReturnValue:&result];
        return @(result);
    }
    
    if (strcmp(returnType, @encode(NSInteger)) == 0) {
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
        [invocation setTarget:target];
        [invocation setSelector:action];
        [invocation setArgument:&params atIndex:2];
        [invocation invoke];
        
        NSInteger result = 0;
        [invocation getReturnValue:&result];
        return @(result);
    }
    
    if (strcmp(returnType, @encode(NSUInteger)) == 0) {
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
        [invocation setTarget:target];
        [invocation setSelector:action];
        [invocation setArgument:&params atIndex:2];
        [invocation invoke];
        
        NSUInteger result = 0;
        [invocation getReturnValue:&result];
        return @(result);
    }

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    return [target performSelector:action withObject:params];
#pragma cland diagnostic pop
}

- (void)releaseCachedTargetWithTargetName:(NSString *)targetName {
    NSString *key = [NSString stringWithFormat:@"Target_%@",targetName];
    [self.cachedTarget removeObjectForKey:key];
}

- (NSMutableDictionary *)cachedTarget{
    if (_cachedTarget == nil) {
        _cachedTarget = [NSMutableDictionary dictionary];
    }
    return _cachedTarget;
}

@end
```


