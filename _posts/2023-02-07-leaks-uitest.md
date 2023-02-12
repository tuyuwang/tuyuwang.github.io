---
title: iOS 内存优化之自动导出内存图
author: tuyw
date: 2023-02-07 22:07:00 +0800
categories: [性能优化]
tags: [性能优化, 内存优化, 自动化]
---

## 0x1 前言

在上一篇文章[iOS 内存优化之工具介绍](https://tuyuwang.github.io/posts/leaks/)中提到利用leaks工具进行排查内存问题的最佳实践方案，本篇文章是对其的补充，这里将讲述录制UITest和自动导出内存图过程的具体实践。

这里对导出内存图的方式提供两种方案

1.xcodebuild test + 性能测试（XCTMemoryMetric）

2.xcodebuild test + expect + lldb + leaks


简要的说一下这两种方案的区别：

第一种方案来自2021-WWDC中的[Detect and diagnose memory issues](https://developer.apple.com/videos/play/wwdc2021/10180/)的演示。其特点是对测试用例设置的平均内存值超出预设值时，触发测试用例执行失败，便会自动导出内存图。但是该方案使用xcodebuild运行时似乎存在bug，具体情况后面会细说。

第二种方案来自2021年时，刚研究leaks工具时产生的一个想法，是否可以通过录制UI测试用例，跑完业务场景后自动导出内存图？然后就一顿操作猛如虎，实现了方案2，但是当时遇到一个问题（导出的内存图无法包含堆栈）无法绕过，最近才找到解决方案。该方案特点是，可以任意时刻去导出内存图，不受测试用例执行结果影响。

> Xcode 14.2


以下操作将以[MemoryGraphDemo](https://github.com/tuyuwang/MemoryGraphDemo)为例讲述


## 0x2 录制测试用例
1.首先，打开`MemoryGraphDemo`项目，在`MemoryGraphDemoUITests.m`中编写内存性能测试代码，如下：
```
- (void)testPerformanceExample {
    XCUIApplication *app = [[XCUIApplication alloc] init];
    XCTMeasureOptions *options = [[XCTMeasureOptions alloc] init];
    options.invocationOptions = XCTMeasurementInvocationManuallyStart;
    
    [self measureWithMetrics:@[[[XCTMemoryMetric alloc] initWithApplication:app]] options:options block:^{
        [app launch];
        [self startMeasuring];
        //录制测试用例

    }];
}
```

2.将光标移动至`//录制测试用例`下方，然后点击Xcode的录制按钮

![record-begin](/assets/img/export-memgrap/record-begin.png)

3.等待模拟器启动后，进行开始对页面进行交互。在模拟器中，每做一次UI交互（点击、滑动）Xcode都会在`//录制测试用例`下方光标位置插入相应的操作代码。

4.点击模拟器中按钮：`Click Me` -> `Retain Cycles` -> `Indirect Retain Cycles` -> `Dynamic Indirect Retain Cycles` -> `Large Buffers`。最后再次点击`录制按钮`停止录制，最终结果如下图

![record-result](/assets/img/export-memgrap/record-result.png)

## 0x3 xcodebuild test

1.执行该测试用例，点击方法前面的执行按钮，如图

![execute-test](/assets/img/export-memgrap/execute-test.png)

2.第一次执行测试用例完成后是测试通过的状态，此时需要为其设置内存值的baseline。

![pre-set-baseline](/assets/img/export-memgrap/pre-set-baseline.png)

3.在本例中将baseline设置为32000kB。

![set-baseline](/assets/img/export-memgrap/set-baseline.png)

4.由于`Max STDDEV：10%`，所以Average的值在32000kB *（100% - 10%）到32000kB *（100% + 10%）之间测试用例才会执行成功。重新执行测试用例，结果如图，从图中可以看到Xcode中执行时可以读取到步骤7中设置的baseline的值。

![test-failure](/assets/img/export-memgrap/test-failure.png)

5.xcodebuild test 参数 `-enablePerformanceTestsDiagnostics YES`来自[Detect and diagnose memory issues](https://developer.apple.com/videos/play/wwdc2021/10180/)中的介绍，按照视频中演示，使用了该参数后，当测试用例执行失败，会在xcresult中给出内存图的导出路径。

<center>
	<img src="/assets/img/export-memgrap/export-mem-demo.png" width="49%" height="300"/>
	<img src="/assets/img/export-memgrap/export-memgraph-path.png" width="49%" height="300"/>
</center>


6.使用`xcodebuild test -workspace MemoryGraphDemo.xcworkspace -scheme MemoryGraphDemo -destination platform="iOS Simulator",name="iPhone 14" -enablePerformanceTestsDiagnostics YES`命令执行测试用例，结果如下图

![export-mem-failure](/assets/img/export-memgrap/export-mem-failure.png)

在命令的执行结果里，测试用例是执行成功的，原因是xcodebuild运行的测试用例没有读取到baseline，所以无法触发失败流程，导出内存图。

这里查到些相关案例:
- [setting-baselines-for-performance-tests-using-xcodebuild-and-measure](https://stackoverflow.com/questions/65221838/setting-baselines-for-performance-tests-using-xcodebuild-and-measure)
- [Is it possible to record baseline for Performance tests with xcodebuild?](https://developer.apple.com/forums/thread/723526)

并且查找xcodebuild相关使用教程，也没有找到设置baseline相关的参数，再对比[Detect and diagnose memory issues](https://developer.apple.com/videos/play/wwdc2021/10180/)使用的xcodebuild命令，猜测是xcodebuild存在bug。


## 0x4 xcodebuild test + expect + lldb

本方案的原理：在xcodebuild执行测试用例期间，通过lldb进行吸附项目进程，并在用例结束的函数上添加结束断点，然后在该断点中添加leaks命令，当用例将要执行时，会触发断点，进而调用leaks命令导出*.memgraph内存。

1.新加一个测试用例，具体代码如下
```
- (void)testLLDBExample {
    XCUIApplication *app = [[XCUIApplication alloc] init];
    app.launchEnvironment = @{@"MallocStackLogging": @"YES"};
    [app launch];
    [app.staticTexts[@"Click Me"] tap];

    XCUIElementQuery *tablesQuery = app.tables;
    [tablesQuery.staticTexts[@"Large Buffers"] tap];
    [tablesQuery.staticTexts[@"Retain Cycles"] tap];
    [tablesQuery.staticTexts[@"Indirect Retain Cycles"] tap];
    [tablesQuery.staticTexts[@"Dynamic Indirect Retain Cycles"] tap];

    //以这个点击操作对应的方法作为触发leaks命令的入口
    [app.tables.staticTexts[@"LLDB Export Memory Graph File"] tap];  
}
```
这里以UITest触发点击`LLDB Export Memory Graph File`这个cell时，作为触发导出内存图的条件。在真实业务场景中，可以以退出登陆的点击事件来作为一次内存回归的内存图导出节点，这样可以方便的观察到非登陆状态下，各个业务对象是否释放干净。

`app.launchEnvironment = @{@"MallocStackLogging": @"YES"};`这句启动环境的配置可以使导出的内存图包含堆栈。

2.`cd /path/to/Scripts`切换控制台工作路径到Scripts文件夹下

![scripts](/assets/img/export-memgrap/scripts.png)

3.给两个脚本执行权限`sudo chmod +x emg.sh uitest.sh`

4.根据项目修改脚本`uitest.sh`中的配置

```
EXECUTE_NAME="MemoryGraphDemo"
SCHEME="MemoryGraphDemoUITests"
ROOT_PATH="../"
TRIGER_CMD="trigerLLDBExportMemoryGraphFile"
OUT_PUT="./"
```

- EXECUTE_NAME: 项目进程名
- SCHEME: xcodebuild test -scheme 的参数
- ROOT_PATH: 项目根路径(.xcodeproj所在路径)
- TRIGER_CMD: 触发leaks命令的函数（项目中存在+UITest最后的点击事件）
- OUT_PUT: 内存图的输出路径

3.执行脚本`./uitest.sh`

![export-lldb](/assets/img/export-memgrap/export-lldb.png)

按`MemoryGraphDemo`的默认配置，最终会把内存图输出的Scripts路径下

![mem-path](/assets/img/export-memgrap/mem-path.png)


## 0x5 总结

本文介绍了两种自动导出内存图的方式，第一种方式目前存在问题，无法成功触发导出操作，可能是我使用姿势不对（缺少必要的参数配置）或者xcodebuild存在bug，功能正常的情况下，这种方式使用简单，仅在测试用例失败时才导出内存图进行分析，可以做到按需分析。第二种方式是利用expect命令给lldb吸附的进程发送交互式命令，实现自动调用leaks导出内存图的方式，该方式灵活性高，不仅仅可使用在这种场景。但是这种方式需要额外分配精力去分析每次回归的内存图，而第一种方式仅需关注测试用例失败的场景。

所以，对于刚开始关注内存问题的项目，使用第二种方式不断排查和修复相关问题，当项目中的内存问题趋于稳定时， 使用第一种方式来确保项目在迭代时的回归效率和避免再次劣化。
