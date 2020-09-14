---
title: 底层探索 - 对象、类的本质和类结构
toc: true
donate: false
tags: []
date: 2020-09-13 10:27:24
categories: [底层探索]
---

一个对象中，可以有属性，有方法，那么对象的本质是什么呢？类的本质又是什么呢？属性和方法又是存储在哪里呢？这就是我们今天探索的主题。

<!-- more -->

## 必要知识

在前篇的探索 [揭开 isa 的神秘面纱](https://www.muhlenxi.com/2020/09/10/075-isa/) 中，我们了解到每个类都有一个 isa，里面存储了该对象所属的类的信息。并且找到了对象和类在 C/C++ 层面的实现。

```c
struct objc_object {
private:
    isa_t isa;

public:
    // 一些方法
}
```

```c
typedef struct objc_object *id;
```

上面是 `对象` 在底层的实现，也找到了 id 类型的声明，这也是为什么 id 可以指向任何对象的根本原因。

```c
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;              
    class_data_bits_t bits;   
    
    // 一些方法
}
```

这个是 `类` 在底层的实现，它继承于 objc_object，所以 objc_class 也是一个对象，被称为类对象。我们常用的 Class 就是 objc_class 的 typedef。

```c
typedef struct objc_class *Class;
```

## 探究 『继承』

在 objc_class 中有个属性 superclass, 我们先探究下 superclass 的继承体系。

如下，我们分别定义了两个类，RDPerson 继承于 NSObject，RDTeacher 继承于 RDPerson 。

```objc
@interface RDPerson : NSObject

@end

@implementation RDPerson

@end
```

```objc
@interface RDTeacher : RDPerson

@property (nonatomic,copy) NSString * name;
@property (nonatomic,copy) NSString * hobby;
@property (nonatomic,copy) NSString * address;

+ (void)sayStandUp;
- (void)sayByebye;

@end

@implementation RDTeacher

+ (void)sayStandUp {
    NSLog(@"Stand up");
}

- (void)sayByebye {
    NSLog(@"Good bye");
}

@end
```

如图，我们实例化一个对象，在 NSLog 的地方打个断点，在 Debug area 中就可以使用 lldb 指令进行探索了。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/imgclass.png)

使用 `x/8gx` 查看当前对象地址开始的 8 个 8 字节的数据。 对象 t 首地址中存储的是isa，后面的依次是对象的每个属性。看看是否是我们设置的数据。

```c
(lldb) po 0x001d800100003465 & 0x00007ffffffffff8ULL
RDTeacher

(lldb) po 0x0000000100002058
Lucy

(lldb) po 0x0000000100002078
Men

(lldb) po 0x0000000100002098
Earth
```

接下来探索 isa 中的类对象的信息。用 `p/x` 打印出当前对象归属的 RDTeacher 类的地址 A（16进制）, 也就是当前对象中的 isa 中 shiftcls 的数据。

```c
(lldb) p/x 0x001d800100003465 & 0x00007ffffffffff8ULL
(unsigned long long) $5 = 0x0000000100003460
```

用 `x/4gx` 读取地址 A 中的内容，并用 `po` 打印出当前类的 superclass。根据前面的 objc_class 的属性分析，第二个 8 字节存储的就是 superclass。

```c
(lldb) x/4gx 0x0000000100003460
0x100003460: 0x0000000100003438 0x0000000100003500
0x100003470: 0x00000001016123a0 0x0002802c00000007

(lldb) po 0x0000000100003438 & 0x00007ffffffffff8ULL  // 当前类
RDTeacher

(lldb) po 0x0000000100003500  // 当前类的父类
RDPerson
```

可以看出 RDTeacher 的父类是 RDPerson，它的地址是 `0x0000000100003500` 。用 `x/4gx` 读取地址 `0x0000000100003500` 中的内容，然后打印 RDPerson 的 superclass。

```c
(lldb) x/4gx 0x0000000100003500
0x100003500: 0x00000001000034d8 0x00000001003f1140
0x100003510: 0x00000001003eb450 0x0000801000000000

(lldb) po 0x00000001003f1140 
NSObject
```

可以看出 RDPerson 的父类是 NSObject ，它的地址是 `0x00000001003f1140`。 用 `x/4gx` 读取 `0x00000001003f1140` 中的内容，然后打印 NSObject 的 superclass。

```c
(lldb) x/4gx  0x00000001003f1140
0x1003f1140: 0x00000001003f10f0 0x0000000000000000
0x1003f1150: 0x00000001016120c0 0x0001801000000003

(lldb) po 0x0000000000000000
<nil>
```
可以看出 NSObject 的父类是 `nil`。

所以我们可以得出 RDTeacher 的继承关系。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/imgcjicheng.png)

## 探究 isa 指向

对象 t 的内存地址中的数据如下所示：

```c
(lldb) p t
(RDTeacher *) $81 = 0x0000000101612260

(lldb) x/4gx 0x0000000101612260
0x101612260: 0x001d800100003465 0x0000000100002058
0x101612270: 0x0000000100002078 0x0000000100002098
```

接下来探索对象 t 的 isa 指向的是啥？首先用 `p/x` 和 `0x00007ffffffffff8ULL` mask 出 isa 中的 shiftcls。然后用 po 打印 shiftcls 的内容。

```
(lldb) p/x 0x001d800100003465 & 0x00007ffffffffff8ULL
(unsigned long long) $104 = 0x0000000100003460

(lldb) po 0x0000000100003460
RDTeacher
```

所以对象 t 的 isa 指向的类是 RDTeacher 类。接下来我们探索 RDTeacher 的 isa 指向的是啥。

先用 `x/4gx` 读出内存 `0x0000000100003460` 内容，然后用 `p/x` 和 `0x00007ffffffffff8ULL` mask 出 RDTeacher 类对象所属的类的地址。

最后用 `x/4gx` 读取这个地址中的内容，用 po 打印当前 isa 中的类。类对象所属的类称为 `元类`（meta class）。

```c
(lldb) x/4gx 0x0000000100003460
0x100003460: 0x0000000100003438 0x0000000100003500
0x100003470: 0x0000000100668c20 0x0004802c0000000f

(lldb) po 0x0000000100003500  // RDTeacher 的父类
RDPerson

(lldb) p/x 0x0000000100003438 & 0x00007ffffffffff8ULL
(unsigned long long) $107 = 0x0000000100003438

(lldb) po 0x0000000100003438
RDTeacher
```

我们直接打印 RDTeacher 的元类的地址，看是否等于上面输出的 `0x0000000100003438`。

```c
(lldb) p/x object_getClass(RDTeacher.class)
(Class) $109 = 0x0000000100003438
```

可以看出他们是相等的。也就是说我们得到了类对象的元类。

继续探索 `元类 RDTeacher` 的 isa，看它指向了啥。用 `x/4gw` 读取这个地址中的内容。然后用 `p/x` 和 `po` 打印元类的 isa 指向的类的信息。

```c
(lldb) x/4gx 0x0000000100003438
0x100003438: 0x00000001003f10f0 0x00000001000034d8
0x100003448: 0x0000000101204690 0x0004e03500000007

(lldb) p/x 0x00000001003f10f0 & 0x00007ffffffffff8ULL
(unsigned long long) $112 = 0x00000001003f10f0

(lldb) po 0x00000001003f10f0
NSObject
```

我们得到的这 NSObject 是我们常用的那个 NSObject 么？用 `p/x` 打印下类对象 NSObject 的地址。

```c
(lldb) p/x [NSObject class]
(Class) $115 = 0x00000001003f1140 NSObject
```

这和 `0x00000001003f10f0` 是不相等的，我们再打印下 NSObject 类对象的元类的地址。

```c
(lldb) p/x object_getClass(NSObject.class)
(Class) $116 = 0x00000001003f10f0
```

此时得到的这个值是和 `0x00000001003f10f0` 是相等的。说明 `元类 RDTeacher` 的 isa 指向的是 NSObject 的元类，这个元类称之为根元类（root meta class）。

最后我们看看根源类的 superclass 和 isa 指向哪里？用 `x/4gx` 查看下 `0x00000001003f10f0` 地址的内容。

```c
(lldb) x/4gx 0x00000001003f10f0
0x1003f10f0: 0x00000001003f10f0 0x00000001003f1140
0x1003f1100: 0x0000000101018bd0 0x0005e03100000007
```

我们先看看 NSObject 根元类的 superclass 是啥？

```
(lldb) po 0x00000001003f1140
NSObject
```

这个 NSObject 的地址和我们之前的 NSObject 的类对象的地址一样的，都是 `0x00000001003f1140`, 所以根元类的父类是 NSObject。

再看一下根元类的 isa 指向啥？用 `p/x` 和 `0x00007ffffffffff8ULL` mask 出根源类所属的类的 isa。

```
(lldb) p/x 0x00000001003f10f0 & 0x00007ffffffffff8ULL
(unsigned long long) $118 = 0x00000001003f10f0

(lldb) po 0x00000001003f10f0
NSObject
```

`0x00000001003f10f0` 这个地址和根元类的地址是一样的，也就是说根元类的 isa 指向了自己本身。

根据以上的探索，我们可以得出一个完整的 isa 指向图。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/imgisaclass.png)

根据前面探索的继承关系和 isa 走向，我们可以将两张图合并成一个图。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/imgclassisa.png)

## class 的结构

class 中的属性和方法存储在什么地方呢？先回到源码中 `objc_class` 声明的地方。

```c
// objc_class 定义
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() const {
        return bits.data();
    }
    void setData(class_rw_t *newData) {
        bits.setData(newData);
    }
    
    // 其他方法
}
```

### class_rw_t

根据上面的源代码，可以推测 class 的方法和属性应该是在 bits 中，通过 `data()` 方法可以获取到 bits 中的数据，这个方法的返回值是 `class_rw_t` 类型的指针。 跳入 `class_rw_t` 的声明中看看这个是啥。

```c
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint16_t witness;
#if SUPPORT_INDEXED_ISA
    uint16_t index;
#endif

    explicit_atomic<uintptr_t> ro_or_rw_ext;

    Class firstSubclass;
    Class nextSiblingClass;
private:
    using ro_or_rw_ext_t = objc::PointerUnion<const class_ro_t *, class_rw_ext_t *>;
    
    // 一些方法
    
    // 获取方法列表
    const method_array_t methods() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>()->methods;
        } else {
            return method_array_t{v.get<const class_ro_t *>()->baseMethods()};
        }
    }

    // 获取属性列表
    const property_array_t properties() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>()->properties;
        } else {
            return property_array_t{v.get<const class_ro_t *>()->baseProperties};
        }
    }

    // 获取协议列表
    const protocol_array_t protocols() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>()->protocols;
        } else {
            return protocol_array_t{v.get<const class_ro_t *>()->baseProtocols};
        }
    }
```

在 `class_rw_t` 的声明中，找到了三个函数。

- `methods()` 用于获取方法列表，它的返回值是 `method_array_t`。
- `properties()` 用于获取属性列表，它的返回值是 `property_array_t`。
- `protocols()` 用于获取协议列表，它的返回值是 `protocol_array_t`。

接下来，我们分别看这个三个 array_t 中的元素是什么。

#### method_array_t

```c
class method_array_t : 
    public list_array_tt<method_t, method_list_t> 
{
    typedef list_array_tt<method_t, method_list_t> Super;

 public:
    method_array_t() : Super() { }
    method_array_t(method_list_t *l) : Super(l) { }

    method_list_t * const *beginCategoryMethodLists() const {
        return beginLists();
    }
    
    method_list_t * const *endCategoryMethodLists(Class cls) const;

    method_array_t duplicate() {
        return Super::duplicate<method_array_t>();
    }
};

// 方法定义
struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};

struct method_list_t : entsize_list_tt<method_t, method_list_t, 0x3> {
    bool isUniqued() const;
    bool isFixedUp() const;
    void setFixedUp();

    uint32_t indexOfMethod(const method_t *meth) const {
        uint32_t i = 
            (uint32_t)(((uintptr_t)meth - (uintptr_t)this) / entsize());
        ASSERT(i < count);
        return i;
    }
};
```

#### property_array_t

```c
class property_array_t : 
    public list_array_tt<property_t, property_list_t> 
{
    typedef list_array_tt<property_t, property_list_t> Super;

 public:
    property_array_t() : Super() { }
    property_array_t(property_list_t *l) : Super(l) { }

    property_array_t duplicate() {
        return Super::duplicate<property_array_t>();
    }
};

// 属性定义
struct property_t {
    const char *name;
    const char *attributes;
};

struct property_list_t : entsize_list_tt<property_t, property_list_t, 0> {
};
```

#### protocol_array_t

```c
class protocol_array_t : 
    public list_array_tt<protocol_ref_t, protocol_list_t> 
{
    typedef list_array_tt<protocol_ref_t, protocol_list_t> Super;

 public:
    protocol_array_t() : Super() { }
    protocol_array_t(protocol_list_t *l) : Super(l) { }

    protocol_array_t duplicate() {
        return Super::duplicate<protocol_array_t>();
    }
};

typedef uintptr_t protocol_ref_t;  // protocol_t *, but unremapped

struct protocol_list_t {
    // count is pointer-sized by accident.
    uintptr_t count;
    protocol_ref_t list[0]; // variable-size

    size_t byteSize() const {
        return sizeof(*this) + count*sizeof(list[0]);
    }

    protocol_list_t *duplicate() const {
        return (protocol_list_t *)memdup(this, this->byteSize());
    }

    typedef protocol_ref_t* iterator;
    typedef const protocol_ref_t* const_iterator;

    const_iterator begin() const {
        return list;
    }
    iterator begin() {
        return list;
    }
    const_iterator end() const {
        return list + count;
    }
    iterator end() {
        return list + count;
    }
};
```

通过上述源代码的分析，可以得到类的结构图为

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/imgobjcClassMindnode.png)

现在我们用 lldb 调试的方式，验证下 RDTeacher 类中的属性和方法的存储位置。让代码运行到图一的那个断点处。首先找到类对象的地址。

```c
(lldb) p/x [RDTeacher class]
(Class) $0 = 0x00000001000034f0 RDTeacher
```

 用 `x/8gx` 读出该地址起始的 8 段数据。
 
 ```c
 (lldb) x/8gx 0x00000001000034f0
0x1000034f0: 0x00000001000034c8 0x0000000100003590
0x100003500: 0x0000000101856a40 0x0002802c00000007
0x100003510: 0x0000000101856654 0x00000001003350f0
0x100003520: 0x00000001003350f0 0x000000010032f410
 ```

第一个 8 字节数据 `0x00000001000034c8` 是 isa，第二个 8 字节数据是 superclass，第三个数据存储的是 cache, `cache_t` 是一个 struct，通过对 struct 中属性的分析，可以得出 cache 占用 16 个字节大小。所以地址 `0x100003500` 起始的 16 个字节中存储的 cache。

第四个数据是 bits，它的类型是 `class_data_bits_t`，`class_data_bits_t` 是一个 struct，其内部只有一个 uintptr_t 类型的 bits 数据，所以 bits 占用 8 个字节大小。所以地址 `0x100003510` 起始的 8 个字节中存储的是 bits 数据。

用 `p/x` 打印出 bits 数据的地址。

```c
(lldb) p (class_data_bits_t*) 0x100003510
(class_data_bits_t *) $2 = 0x0000000100003510
```

调用 `data()` 方法得到 `class_rw_t` 类型的数据。

```c
(lldb) p $2 -> data()
(class_rw_t *) $3 = 0x0000000101856650
```

此时我们得到的是 `class_rw_t` 类型的指针，通过运算符 `*` 取出地址中的数据。

```c
(lldb) p *$3
(class_rw_t) $4 = {
  flags = 2148007936
  witness = 1
  ro_or_rw_ext = {
    std::__1::atomic<unsigned long> = 4294980136
  }
  firstSubclass = nil
  nextSiblingClass = nil
}
```

调用 `class_rw_t` 中的 `properties()` 方法得到属性列表。

```c
(lldb) p $4.properties()
(const property_array_t) $5 = {
  list_array_tt<property_t, property_list_t> = {
     = {
      list = 0x00000001000031f0
      arrayAndFlag = 4294980080
    }
  }
}
```

遍历属性列表中的元素，看看是否有 name，hobby，address 这三个属性。

```c
(lldb) p $5.list->get(0)
(property_t) $8 = (name = "name", attributes = "T@\"NSString\",C,N,V_name")

(lldb) p $5.list->get(1)
(property_t) $9 = (name = "hobby", attributes = "T@\"NSString\",C,N,V_hobby")

(lldb) p $5.list->get(2)
(property_t) $10 = (name = "address", attributes = "T@\"NSString\",C,N,V_address")

(lldb) p $5.list->get(3)
Assertion failed: (i < count), function get, file /Users/muhlenxi/xiyinjun/objc-runtime/objc4-781/runtime/objc-runtime-new.h, line 438.
```

当我们取第四个属性的时候，发生了数组越界操作，崩溃了。因为 RDTeacher 中一共定义了 3 个 属性。

调用 `class_rw_t` 中的 `methods()` 方法得到方法列表。

```c
(lldb) p $4.methods()
(const method_array_t) $11 = {
  list_array_tt<method_t, method_list_t> = {
     = {
      list = 0x00000001000030c0
      arrayAndFlag = 4294979776
    }
  }
}
```

遍历方法列表中的元素，看看有哪些方法呢？

```c
(lldb) p $11.list->get(0)
(method_t) $14 = {
  name = "sayByebye"
  types = 0x0000000100001f87 "v16@0:8"
  imp = 0x00000001000016c0 (HelloWorld`-[RDTeacher sayByebye] at RDTeacher.m:17)
}

(lldb) p $11.list->get(1)
(method_t) $15 = {
  name = "hobby"
  types = 0x0000000100001f8f "@16@0:8"
  imp = 0x0000000100001760 (HelloWorld`-[RDTeacher hobby] at RDTeacher.h:16)
}
(lldb) p $11.list->get(2)
(method_t) $16 = {
  name = "setHobby:"
  types = 0x0000000100001f97 "v24@0:8@16"
  imp = 0x0000000100001790 (HelloWorld`-[RDTeacher setHobby:] at RDTeacher.h:16)
  
(lldb) p $11.list->get(3)
(method_t) $17 = {
  name = ".cxx_destruct"
  types = 0x0000000100001f87 "v16@0:8"
  imp = 0x0000000100001840 (HelloWorld`-[RDTeacher .cxx_destruct] at RDTeacher.m:11)
}

(lldb) p $11.list->get(4)
(method_t) $18 = {
  name = "name"
  types = 0x0000000100001f8f "@16@0:8"
  imp = 0x00000001000016f0 (HelloWorld`-[RDTeacher name] at RDTeacher.h:15)
}
(lldb) p $11.list->get(5)
(method_t) $19 = {
  name = "setName:"
  types = 0x0000000100001f97 "v24@0:8@16"
  imp = 0x0000000100001720 (HelloWorld`-[RDTeacher setName:] at RDTeacher.h:15)
}

(lldb) p $11.list->get(6)
(method_t) $20 = {
  name = "address"
  types = 0x0000000100001f8f "@16@0:8"
  imp = 0x00000001000017d0 (HelloWorld`-[RDTeacher address] at RDTeacher.h:17)
}
(lldb) p $11.list->get(7)
(method_t) $21 = {
  name = "setAddress:"
  types = 0x0000000100001f97 "v24@0:8@16"
  imp = 0x0000000100001800 (HelloWorld`-[RDTeacher setAddress:] at RDTeacher.h:17)
}

(lldb) p $11.list->get(8)
Assertion failed: (i < count), function get, file /Users/muhlenxi/xiyinjun/objc-runtime/objc4-781/runtime/objc-runtime-new.h, line 438.
error: Execution was interrupted, reason: signal SIGABRT.
The process has been returned to the state before expression evaluation.  
```

通过上面的打印，可以看出方法列表中不仅有我们定义的 `sayByebye` 方法，还有编译器生成的 name，hobby，address 这三个属性的 setter 和 getter 方法，没有找到我们定义的 `sayStandUp` 方法，这是为什么呢？

`sayStandUp` 是类方法，存储在 `RDTeacher` 的元类中，同理，按照上面的思路，我们看看元类中的 methods 列表。

```c
(lldb) p object_getClass(RDTeacher.class)
(Class) $2 = 0x00000001000034c8

(lldb) x/8gx 0x00000001000034c8
0x1000034c8: 0x00000001003350f0 0x0000000100003568
0x1000034d8: 0x0000000103274dc0 0x0002e03500000007
0x1000034e8: 0x0000000103274724 0x00000001000034c8
0x1000034f8: 0x0000000100003590 0x0000000103274d40

(lldb) p (class_data_bits_t*) 0x1000034e8
(class_data_bits_t *) $3 = 0x00000001000034e8

(lldb) p $3 -> data()
(class_rw_t *) $4 = 0x0000000103274720

(lldb) p *$4
(class_rw_t) $5 = {
  flags = 2684878849
  witness = 1
  ro_or_rw_ext = {
    std::__1::atomic<unsigned long> = 4294979704
  }
  firstSubclass = nil
  nextSiblingClass = nil
}

(lldb) p $5.methods()
(const method_array_t) $6 = {
  list_array_tt<method_t, method_list_t> = {
     = {
      list = 0x0000000100003058
      arrayAndFlag = 4294979672
    }
  }
}

(lldb) p $6.list->get(0)
(method_t) $7 = {
  name = "sayStandUp"
  types = 0x0000000100001f87 "v16@0:8"
  imp = 0x0000000100001690 (HelloWorld`+[RDTeacher sayStandUp] at RDTeacher.m:13)
}

(lldb) p $6.list->get(1)
Assertion failed: (i < count), function get, file /Users/muhlenxi/xiyinjun/objc-runtime/objc4-781/runtime/objc-runtime-new.h, line 438.
error: Execution was interrupted, reason: signal SIGABRT.
The process has been returned to the state before expression evaluation.
```

methods 列表中第一个方法就是 `sayStandUp`，符合我们的预期。

今天的图，值得仔细揣摩，或许你能得出一些关于人生的大道理。

## 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。

