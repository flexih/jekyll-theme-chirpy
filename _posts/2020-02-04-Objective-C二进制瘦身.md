---
title: Objective-C二进制瘦身
author: flexih
date: 2020-02-04 22:55:48 +0800
categories: [iOS, Mach-O]
tags: [瘦身, Mach-O]
---

先说结论：我写了个工具检测无用方法、无用类以及无用协议，只需要Mach-O文件，对Build Setting里的Strip Style无要求，[Snake](https://github.com/flexih/Snake)。

Objective-C是采用消息发送的方式来实现类方法的调用。消息发送使用“查表”的方式实现从方法名到方法实现的定位。因此在编译的时候编译器不能确知一个方法是否真的被调用，也就无法像C语言一样只编译使用到的方法。也因此造成了目标二进制里包含没有使用的类、没有使用的方法、以及没有使用的协议等。

为了实现消息发送，Objective-C的编译器会在编译的时候自动生成相关的结构体变量，来存储类的信息，这些信息也被称作ObjC元信息。

clang命令使用-rewrite-objc参数可以得到.m文件的CPP实现。比如使用 `xcrun -sdk iphonesimulator clang -rewrite-objc -F /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/Framework a.m`
截取类A的定义，ObjC代码是：
```
@protocol AProtocol<NSObject>
- (void)aMeth;
@end

@interface A: NSObject<AProtocol>
@end

@implementation A
- (void)aMeth {
}
@end
```
rewrite之后的代码是：
```
extern "C" __declspec(dllexport) struct _class_t OBJC_CLASS_$_A __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_A,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_A,
};
static struct _class_ro_t _OBJC_CLASS_RO_$_A __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	0, sizeof(struct A_IMPL), sizeof(struct A_IMPL),
	(unsigned int)0,
	0,
	"A",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_A,
	(const struct _objc_protocol_list *)&_OBJC_CLASS_PROTOCOLS_$_A,
	0,
	0,
	0,
};
static struct _class_t *L_OBJC_LABEL_CLASS_$ [1] __attribute__((used, section ("__DATA, __objc_classlist,regular,no_dead_strip")))= {
	&OBJC_CLASS_$_A,
};
```
CPP代码变量的声明处都出现了`__attribute__((used, section("xx")))`。这里section的意思是在Mach-O里对应名字的section，并把变量的内容放到该section下。

Mach-O是iOS/macOS平台下可执行二进制文件格式。Mach-O文件格式这里不详述了，可以参考[MachOOverview](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html)和[osx-abi-macho-file-format-reference](https://github.com/aidansteele/osx-abi-macho-file-format-reference)。

以下内容特指64位Arch。
######__objc_classlist
该section下存储的是指针，指针指向`struct objc_class`的内存，位于`__objc_const`section下。这里存储了代码里所有的ObjC类。

######__objc_classrefs, __objc_superrefs
该section下存储的是指针，指针指向`struct objc_class`的内存。这里存储了代码里使用到的ObjC类，也即是使用过消息发送方式调用过alloc或者new方法生成对象的类。__不包含NSClassFromString()方式生成的对象的类__。

######__objc_selrefs
该section下存储的是指针，指针指向以\0结尾的字符串，字符串的内容是方法名。这里存储了被调用过的方法名。__不包含NSSelectorFromString()返回的SEL__。

######__objc_protolist
该section下存储的是指针，指针指向`struct protocol_t`的内存。这里存储了代码里所有的protocol。

#####__objc_catlist
该section下存储的是指针，指针指向`struct category_t`的内存。这里存储了代码里所有的分类。

######Binding Info
对于某些非零字段的值却是0，比如`struct objc_class`的`isa`字段。这种情况是引用了外部lib里的符号。外部符号记录在dyld_info_command下的binding info里。此部分的格式可以参考[MachOView](https://github.com/gdbinit/MachOView/blob/3431351a76778f156d5a8227290ce5475bc2f528/DyldInfo.mm#L87)的处理。从Binding Info里得到地址到符号的对应关系。遇到非零字段为0的时候，去Bind Info里查找该内存地址对于的符号即可。

######实现
一般的无用方法获取方式，是利用otool、nm等命令获取。这里直接读取Mach-O文件，解析出ObjC信息。
无用的方法 = `__objc_classlist`的 (`instanceMethods` - `__objc_selrefs`) + `clasMethods` - `__objc_selrefs`
无用的类 = `__objc_classlist` - ((`__objc_classrefs `+`__objc_superrefs`)+(`__objc_classrefs `+`__objc_superrefs`)的superclass)
无用的协议 = `__objc_protolist` - (`__objc_classrefs `+`__objc_superrefs`) 的`protocol_list`

配合使用Mach-O对应的Linkmap，可以获得方法的大小和方法、类、协议所属的library。可以生成json格式数据，以供进一步处理。
代码使用C++编写，文件读取使用mmap，处理一个460.6M大小的Mach-O文件和134.3M的linkmap文件只需要1.62秒。

```
Usage:
  snake [-scp] [-l path] mach-o ...

  -s, --selector     Unused selectors
  -c, --class        Unused classes
  -p, --protocol     Unused protocoles
  -l, --linkmap arg  Linkmap file, which has selector size, library name
  -j, --json         Output json format
      --help         Print help
```

具体实现移步[Snake](https://github.com/flexih/Snake)和[SnakKit](https://github.com/flexih/SnakeKit)。

