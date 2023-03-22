---
title: iOS block调用为啥要判空
author: tuyw
date: 2023-03-18 10:19:00 +0800
categories: [block]
tags: [block, 汇编]
---

## 0x1 前言

在iOS中，使用nil指针调用OC的方法是安全的，但是使用nil指针调用block却会产生崩溃。本篇文章，将会从汇编的角度解释该现象。

## 0x2 block的结构

Block 的结构可以在 Runtime 的开源代码[Objc4-706](https://github.com/isaacselement/objc4-706) 中找到，它位于 Block-private.h 中：
```
struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};

```

在arm64中，一个指针占8字节，int32_t占4个字节，所以一个block的内存基本布局如下图：

![footprint](/assets/img/block/footprint.jpg)


## 0x3 测试代码

1.首先定义Helper类辅助测试，代码如下：

```
@interface Helper : NSObject

@property (nonatomic, copy) dispatch_block_t block;

@end

@implementation Helper

- (void)triger {}

@end
```

2.测试用例1：调用一个正常对象的block
```
- (void)testBlock {
    Helper *helper = [Helper new];
    helper.block = ^{
        NSLog(@"test");
    };
    helper.block();
}
```
在testBlock函数入口处打上断点
![breakpoint](/assets/img/block/breakpoint.png)


然后在Xcode菜单栏找到`Debug` -> `Debug Workflow`，勾选`Always Show Disassembly`

![disassembly](/assets/img/block/disassembly.png)

运行代码，触发断点，会自动进入Xcode汇编，如图所示。

![asm](/assets/img/block/asm.png)


3.分析汇编

```
TestBlock`-[ViewController testBlock]:
    0x1001a5d1c <+0>:   sub    sp, sp, #0x40
    0x1001a5d20 <+4>:   stp    x29, x30, [sp, #0x30]
    0x1001a5d24 <+8>:   add    x29, sp, #0x30
    0x1001a5d28 <+12>:  stur   x0, [x29, #-0x8]
    0x1001a5d2c <+16>:  stur   x1, [x29, #-0x10]
    0x1001a5d30 <+20>:  adrp   x8, 8
->  0x1001a5d34 <+24>:  ldr    x0, [x8, #0x428]
    0x1001a5d38 <+28>:  bl     0x1001a634c               ; symbol stub for: objc_opt_new
    0x1001a5d3c <+32>:  ldr    x1, [sp]
    0x1001a5d40 <+36>:  add    x8, sp, #0x18
    0x1001a5d44 <+40>:  str    x8, [sp, #0x10]
    0x1001a5d48 <+44>:  str    x0, [sp, #0x18]
    0x1001a5d4c <+48>:  ldr    x0, [sp, #0x18]
    0x1001a5d50 <+52>:  adrp   x2, 3
    0x1001a5d54 <+56>:  add    x2, x2, #0x50             ; __block_literal_global.13
    0x1001a5d58 <+60>:  bl     0x1001a64c0               ; objc_msgSend$setBlock:
    0x1001a5d5c <+64>:  ldr    x1, [sp]
    0x1001a5d60 <+68>:  ldr    x0, [sp, #0x18]
    0x1001a5d64 <+72>:  bl     0x1001a6460               ; objc_msgSend$block
    0x1001a5d68 <+76>:  mov    x29, x29
    0x1001a5d6c <+80>:  bl     0x1001a6364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1001a5d70 <+84>:  str    x0, [sp, #0x8]
    0x1001a5d74 <+88>:  ldr    x8, [x0, #0x10]
    0x1001a5d78 <+92>:  blr    x8
    0x1001a5d7c <+96>:  ldr    x0, [sp, #0x8]
    0x1001a5d80 <+100>: bl     0x1001a6358               ; symbol stub for: objc_release
    0x1001a5d84 <+104>: ldr    x0, [sp, #0x10]
    0x1001a5d88 <+108>: mov    x1, #0x0
    0x1001a5d8c <+112>: bl     0x1001a6388               ; symbol stub for: objc_storeStrong
    0x1001a5d90 <+116>: ldp    x29, x30, [sp, #0x30]
    0x1001a5d94 <+120>: add    sp, sp, #0x40
    0x1001a5d98 <+124>: ret    
```
打个断点在`0x1001a5d64 <+72>:  bl     0x1001a6460               ; objc_msgSend$block`指令处，此时x0为helper对象，bl指令调用的是helper对象的block属性的get方法，即`[helper block]`函数。
```
->  0x1001a5d64 <+72>:  bl     0x1001a6460               ; objc_msgSend$block
    0x1001a5d68 <+76>:  mov    x29, x29
    0x1001a5d6c <+80>:  bl     0x1001a6364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1001a5d70 <+84>:  str    x0, [sp, #0x8]
    0x1001a5d74 <+88>:  ldr    x8, [x0, #0x10]
    0x1001a5d78 <+92>:  blr    x8

(lldb) register read x0
      x0 = 0x0000000282b80a60
(lldb) po 0x0000000282b80a60
<Helper: 0x282b80a60>

```

单步断点下一个指令，断点到` 0x1001a5d68 <+76>:  mov    x29, x29`处，此时执行完`[helper block]`函数，返回了block的指针，放于寄存器x0中，可以简单的理解为x0 = [helper block]。
```
    0x1001a5d64 <+72>:  bl     0x1001a6460               ; objc_msgSend$block
->  0x1001a5d68 <+76>:  mov    x29, x29
    0x1001a5d6c <+80>:  bl     0x1001a6364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1001a5d70 <+84>:  str    x0, [sp, #0x8]
    0x1001a5d74 <+88>:  ldr    x8, [x0, #0x10]
    0x1001a5d78 <+92>:  blr    x8

(lldb) register read x0
      x0 = 0x00000001001a8050  TestBlock`__block_literal_global.13
```

断点打在`0x1001a5d70 <+84>:  str    x0, [sp, #0x8]`指令处，执行完`objc_retainAutoreleasedReturnValue`函数后，x0仍然是block指针。

```
    0x1001a5d64 <+72>:  bl     0x1001a6460               ; objc_msgSend$block
    0x1001a5d68 <+76>:  mov    x29, x29
    0x1001a5d6c <+80>:  bl     0x1001a6364               ; symbol stub for: objc_retainAutoreleasedReturnValue
->  0x1001a5d70 <+84>:  str    x0, [sp, #0x8]
    0x1001a5d74 <+88>:  ldr    x8, [x0, #0x10]
    0x1001a5d78 <+92>:  blr    x8

(lldb) register read x0
      x0 = 0x00000001001a8050  TestBlock`__block_literal_global.13
```

断点打在`0x1001a5d78 <+92>:  blr    x8`指令处，`0x1001a5d74 <+88>:  ldr    x8, [x0, #0x10]`这句指令的伪代码：x8 = x0 + 0x10, 即 0x00000001001a8060 = 0x00000001001a8050 + 0x10，在地址0x00000001001a8060处内存存放的地址就是block对象的invoke指针0x00000001001a5d9c。
```
    0x1001a5d64 <+72>:  bl     0x1001a6460               ; objc_msgSend$block
    0x1001a5d68 <+76>:  mov    x29, x29
    0x1001a5d6c <+80>:  bl     0x1001a6364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1001a5d70 <+84>:  str    x0, [sp, #0x8]
    0x1001a5d74 <+88>:  ldr    x8, [x0, #0x10]
->  0x1001a5d78 <+92>:  blr    x8

(lldb) register read x0
      x0 = 0x00000001001a8050  TestBlock`__block_literal_global.13
(lldb) memory read 0x00000001001a8060
0x1001a8060: 9c 5d 1a 00 01 00 00 00 10 80 1a 00 01 00 00 00  .]..............
0x1001a8070: b8 2f 97 f6 01 00 00 00 c8 07 00 00 00 00 00 00  ./..............
(lldb) register read x8
      x8 = 0x00000001001a5d9c  TestBlock`__27-[ViewController testBlock]_block_invoke at ViewController.m:71

```
根据block的内存布局图可以知道在block的isa + 0x10处的内存就是block的invoke指针地址。指令`0x1001a5d78 <+92>:  blr    x8`是调用block的invoke指针进行函数调用，即调用的是`[helper block]()`，执行block的调用。这是一个正常oc对象的block的调用汇编分析，现在来看一下下面两种测试用例。

4.测试用例2：调用一个对象的nil block，重复2步骤，进入Xcode汇编
```
- (void)testBlockNilBlock {
    Helper *helper = [Helper new];
    helper.block();
}
```

将断点打到`0x100d09c14 <+52>:  bl     0x100d0a460               ; objc_msgSend$block`指令处，获取block指针的指令调用之后。查看此时的x0，发现获取的值为0，也就是nil，取到一个为nil的block指针。
```
TestBlock`-[ViewController testBlockNilBlock]:
    0x100d09be0 <+0>:   sub    sp, sp, #0x40
    0x100d09be4 <+4>:   stp    x29, x30, [sp, #0x30]
    0x100d09be8 <+8>:   add    x29, sp, #0x30
    0x100d09bec <+12>:  stur   x0, [x29, #-0x8]
    0x100d09bf0 <+16>:  stur   x1, [x29, #-0x10]
    0x100d09bf4 <+20>:  adrp   x8, 8
    0x100d09bf8 <+24>:  ldr    x0, [x8, #0x428]
    0x100d09bfc <+28>:  bl     0x100d0a34c               ; symbol stub for: objc_opt_new
    0x100d09c00 <+32>:  ldr    x1, [sp]
    0x100d09c04 <+36>:  add    x8, sp, #0x18
    0x100d09c08 <+40>:  str    x8, [sp, #0x10]
    0x100d09c0c <+44>:  str    x0, [sp, #0x18]
    0x100d09c10 <+48>:  ldr    x0, [sp, #0x18]
    0x100d09c14 <+52>:  bl     0x100d0a460               ; objc_msgSend$block
->  0x100d09c18 <+56>:  mov    x29, x29
    0x100d09c1c <+60>:  bl     0x100d0a364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x100d09c20 <+64>:  str    x0, [sp, #0x8]
    0x100d09c24 <+68>:  ldr    x8, [x0, #0x10]
    0x100d09c28 <+72>:  blr    x8
    0x100d09c2c <+76>:  ldr    x0, [sp, #0x8]
    0x100d09c30 <+80>:  bl     0x100d0a358               ; symbol stub for: objc_release
    0x100d09c34 <+84>:  ldr    x0, [sp, #0x10]
    0x100d09c38 <+88>:  mov    x1, #0x0
    0x100d09c3c <+92>:  bl     0x100d0a388               ; symbol stub for: objc_storeStrong
    0x100d09c40 <+96>:  ldp    x29, x30, [sp, #0x30]
    0x100d09c44 <+100>: add    sp, sp, #0x40
    0x100d09c48 <+104>: ret      

(lldb) register read x0
      x0 = 0x0000000000000000
```
将断点打在`0x100d09c24 <+68>:  ldr    x8, [x0, #0x10]`指令处，该指令等价于x8 = x0 + 0x10，由于此时x0为0x0000000000000000，所以 0x0000000000000010 = 0x0000000000000000 + 0x10，该地址0x0000000000000010为非法地址，所以会触发非法地址异常。
```
    0x100d09c14 <+52>:  bl     0x100d0a460               ; objc_msgSend$block
    0x100d09c18 <+56>:  mov    x29, x29
    0x100d09c1c <+60>:  bl     0x100d0a364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x100d09c20 <+64>:  str    x0, [sp, #0x8]
->  0x100d09c24 <+68>:  ldr    x8, [x0, #0x10]
    0x100d09c28 <+72>:  blr    x8
```

放开断点，继续执行，触发`EXC_BAD_ACCESS`异常，异常信息中address=0x10，如下图：
![crash](/assets/img/block/crash.png)

从这个用例中可以得出结论，当对象的block为nil时，在汇编层，仍然会按照正常的block调用逻辑去取block的invoke指针去执行，当寄存器进行计算获取invoke指针时，由于block为nil，寄存器计算出的地址为0x10，触发非法地址异常。

5.测试用例3：调用一个nil对象的block，重复2步骤，进入Xcode汇编
```
- (void)testBlockNilObj {
    Helper *helper = nil;
    helper.block();
}
```

```
TestBlock`-[ViewController testBlockNilObj]:
    0x1025a5b78 <+0>:   sub    sp, sp, #0x40
    0x1025a5b7c <+4>:   stp    x29, x30, [sp, #0x30]
    0x1025a5b80 <+8>:   add    x29, sp, #0x30
    0x1025a5b84 <+12>:  mov    x8, x1
    0x1025a5b88 <+16>:  stur   x0, [x29, #-0x8]
    0x1025a5b8c <+20>:  stur   x8, [x29, #-0x10]
    0x1025a5b90 <+24>:  add    x8, sp, #0x18
    0x1025a5b94 <+28>:  str    x8, [sp, #0x8]
    0x1025a5b98 <+32>:  mov    x8, #0x0
    0x1025a5b9c <+36>:  str    x8, [sp, #0x10]
->  0x1025a5ba0 <+40>:  str    xzr, [sp, #0x18]
    0x1025a5ba4 <+44>:  ldr    x0, [sp, #0x18]
    0x1025a5ba8 <+48>:  bl     0x1025a6460               ; objc_msgSend$block
    0x1025a5bac <+52>:  mov    x29, x29
    0x1025a5bb0 <+56>:  bl     0x1025a6364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1025a5bb4 <+60>:  str    x0, [sp]
    0x1025a5bb8 <+64>:  ldr    x8, [x0, #0x10]
    0x1025a5bbc <+68>:  blr    x8
    0x1025a5bc0 <+72>:  ldr    x0, [sp]
    0x1025a5bc4 <+76>:  bl     0x1025a6358               ; symbol stub for: objc_release
    0x1025a5bc8 <+80>:  ldr    x0, [sp, #0x8]
    0x1025a5bcc <+84>:  ldr    x1, [sp, #0x10]
    0x1025a5bd0 <+88>:  bl     0x1025a6388               ; symbol stub for: objc_storeStrong
    0x1025a5bd4 <+92>:  ldp    x29, x30, [sp, #0x30]
    0x1025a5bd8 <+96>:  add    sp, sp, #0x40
    0x1025a5bdc <+100>: ret 
```

对比其获取block指针到取invoke指针去执行这一过程，与测试用例2并无区别：
```
    0x1025a5ba8 <+48>:  bl     0x1025a6460               ; objc_msgSend$block
    0x1025a5bac <+52>:  mov    x29, x29
    0x1025a5bb0 <+56>:  bl     0x1025a6364               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1025a5bb4 <+60>:  str    x0, [sp]
    0x1025a5bb8 <+64>:  ldr    x8, [x0, #0x10]
    0x1025a5bbc <+68>:  blr    x8
```
所以，不管是调用nil对象的block还是正常对象的一个为nil的block指针最终都会触发到非法地址异常上。


6.测试用例4: 调用一个nil对象的函数，重复2步骤，进入Xcode汇编

```
- (void)test {
    Helper *helper = nil;
    [helper triger];
}
```

```
TestBlock`-[ViewController test]:
    0x102635c4c <+0>:  sub    sp, sp, #0x40
    0x102635c50 <+4>:  stp    x29, x30, [sp, #0x30]
    0x102635c54 <+8>:  add    x29, sp, #0x30
    0x102635c58 <+12>: mov    x8, x1
    0x102635c5c <+16>: stur   x0, [x29, #-0x8]
    0x102635c60 <+20>: stur   x8, [x29, #-0x10]
    0x102635c64 <+24>: add    x8, sp, #0x18
    0x102635c68 <+28>: str    x8, [sp, #0x8]
    0x102635c6c <+32>: mov    x8, #0x0
    0x102635c70 <+36>: str    x8, [sp, #0x10]
->  0x102635c74 <+40>: str    xzr, [sp, #0x18]
    0x102635c78 <+44>: ldr    x0, [sp, #0x18]
    0x102635c7c <+48>: bl     0x102636500               ; objc_msgSend$triger
    0x102635c80 <+52>: ldr    x0, [sp, #0x8]
    0x102635c84 <+56>: ldr    x1, [sp, #0x10]
    0x102635c88 <+60>: bl     0x102636388               ; symbol stub for: objc_storeStrong
    0x102635c8c <+64>: ldp    x29, x30, [sp, #0x30]
    0x102635c90 <+68>: add    sp, sp, #0x40
    0x102635c94 <+72>: ret  
```
对于OC函数调用最终都会转换成objc_msgSend的调用
```
0x102635c7c <+48>: bl     0x102636500               ; objc_msgSend$triger
```

查看`objc_msgSend`的实现可知，指令`cbz	r0, LNilReceiver_f`先判断x0是否为nil，如果为nil，清空寄存器，消息发送返回nil。所以对于nil对象的方法调用，是安全的。并不会像block调用一样对寄存器中的内容（即使内存为0，没有作判空）进行偏移计算获取invoke指针进行调用，进而导致取到非法地址，触发异常。
![objc_msgSend](/assets/img/block/objc_msgSend.png)


## 0x4 总结
本篇文章通过分析以上几个测试用例的汇编代码，分析了OC对象函数与block调用在汇编层面上的区别，这种区别导致了对于block的调用需要进行判空后才能确保安全。
```
!block ?: block();
```
值得注意的是，调用多层对象的block时，也需要进行判空，即使d对象与其block必然存在，也可能因为a、b、c对象中任意一个为nil，导致出现测试用例3的场景，调用一个nil对象的block产生崩溃，比如：
```
//不安全调用
a.b.c.d.block();

//安全调用
!a.b.c.d.block ?: a.b.c.d.block();
```
对于这种情况，可以对将该block进行一层函数封装，可以避免过长的判断逻辑。
```
//d类
- (void)callBlock {
	!self.block ?: self.block();
}

//调用
[a.b.c.d callBlock];
```

总而言之，block调用之前需要进行判空。
