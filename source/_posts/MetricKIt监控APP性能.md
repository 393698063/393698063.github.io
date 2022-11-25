---
title: MetricKIt监控APP性能
date: 2022-09-27 10:57:45
tags:
---
## 目前的性能框架

## MetricKit简介
MetricKit是苹果在iOS13（2020）年推出的性能监测框架。[session](https://developer.apple.com/videos/play/wwdc2020/10081/) 
[apple文档]（https://developer.apple.com/documentation/metrickit?language=objc）
iOS14进行了优化。
使用MetricKit，您可以接收系统捕获的设备应用程序诊断以及电源和性能指标。该系统每天最多向注册的应用程序发送一次关于前24小时的度量报告，
并在iOS 15及更高版本和macOS 12及更高版中立即发送诊断报告。

MetricKit 将系统收集的数据交给开发者，让我们自己去决定如何利用这些数据去打造一个更省电、性能更好的 App。

该框架包括以下内容：
管理器类和订户协议
报告数据的有效载荷类别
每类度量和诊断的类
测量单位的类别，如蜂窝连接条
用于表示累积数据（如直方图）的类
用于在诊断中捕获堆栈跟踪的类
{% asset_img MetricKit.png This is an example image %}

## 如何集成MetricKit
MetricKit 的接入基本属于无侵入，并且步骤十分简单
1. 获取 MXMetricManager 单例
2. 注册数据接收对象
3. 实现 delegate 回调
```
import UIKit
import MetricKit


@objc class QGMetricKitManager: NSObject {
    @objc public static let shared = QGMetricKitManager()
    private override init() {
        
    }
    
    @available(iOS 13.0, *)
    @objc func start() {
        let metricManager = MXMetricManager.shared
        metricManager.add(self);
    }
}

extension QGMetricKitManager: MXMetricManagerSubscriber {
    @available(iOS 13.0, *)
    func didReceive(_ payloads: [MXMetricPayload]) {
        guard let firtPayload = payloads.first else { return }
        print(firtPayload.dictionaryRepresentation())
    }
    @available(iOS 14.0, *)
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        guard let firtPayload = payloads.first else { return }
        print(firtPayload.dictionaryRepresentation())
    }
}
```
## MetricKit如何工作
{% asset_img metricKitpayload.png This is an example image %}
MetricKit 会在用户使用 App 的时候同步收集诊断信息，然后每天结束的时候将他们打包成 MXDiagnosticPayload 给到我们。这样我们就拥有了同一个时间段的性能数据和诊断数据。
{% asset_img MetricKitDiagnostic.png This is an example image %}
由于他们是一一对应的，所以我们在对性能数据产生疑问的时候就可以掏出对应的诊断数据来进行排查。由于有了这些一一的对应关系，所以 MetricKit 2.0 也随之来了一些新的基类
* MXDiagnostic ：所有诊断类集成的基类
* MXDiagnosticPayload ：诊断包，包含一天结束时的所有诊断
* MXCallStackTree ：新数据类，用于封装当前环境的调用栈
MXCallStackTree 封装的调用栈并没有经过符号化，旨在用于设备外处理，非 debug 用。转换成 JSON 后如下所示
{% asset_img MxCakkStackTree.png This is an example image %}
如果想要了解怎么利用这些调用栈数据，可以观看 WWDC 2020 - 10057 Identify trends with the Power and Performance API[4].[sessions](https://developer.apple.com/videos/play/wwdc2020/10057/)
### 简单介绍下诊断类型
{% asset_img MetricKit2.0.png This is an example image %}
挂起异常、CPU异常、磁盘写入异常、和崩溃
1. MXHangDiagnostic(App对用户输入长时间无反应)，包括以下诊断数据
Time spent hanging (app 无响应时间)
Backtrace of main thread（主线程回溯）
2. MXCPUExceptionDiagnostic(CPU 异常)包括以下诊断数据
* CPU time consumed(CPU 使用时间)
* T*otal sampled time（CPU 高利用率期间的总采样时间）
* Backtrack of threads spining CPU(占用CPU时间的线程回溯)
3. MXDiskWriteExceptionDiagnostic(磁盘写入异常诊断)包括以下诊断数据
* Total write caused (导致异常的写入总数)
* Backtrace of threads causing writes （异常的线程）
当App突破每天1GB的阀值时，系统就会生成这类诊断
4. MXCrashDiagnostic ,
* EXception type，code，and signal
* Termination reason
* VM In（for bad access crash）
* Backtrack 藕粉crashing thread

总而言之，MetricKit诊断，是一个强大的新工具，可以让你在真实的客户用例中，找到回归问题的根本原因，可以帮助我们将优化工作推向新的高度
## MetricKit优点
基于苹果原生能力，无额外的性能损耗
## MetricKit缺点
只支持iOS13，iOS14以上的系统，
可能对于目前成熟的Crash捕获框架，信息还不够充分


## 示例数据
```
// MXMetricPayload
[AnyHashable("applicationResponsivenessMetrics"): {
    histogrammedAppHangTime =     {
        histogramNumBuckets = 3;
        histogramValue =         {
            0 =             {
                bucketCount = 50;
                bucketEnd = "100 ms";
                bucketStart = "0 ms";
            };
            1 =             {
                bucketCount = 60;
                bucketEnd = "400 ms";
                bucketStart = "100 ms";
            };
            2 =             {
                bucketCount = 30;
                bucketEnd = "700 ms";
                bucketStart = "400 ms";
            };
        };
    };
}, AnyHashable("signpostMetrics"): <__NSArrayM 0x280119170>(
{
    signpostCategory = TestSignpostCategory1;
    signpostIntervalData =     {
        histogrammedSignpostDurations =         {
            histogramNumBuckets = 3;
            histogramValue =             {
                0 =                 {
                    bucketCount = 50;
                    bucketEnd = "100 ms";
                    bucketStart = "0 ms";
                };
                1 =                 {
                    bucketCount = 60;
                    bucketEnd = "400 ms";
                    bucketStart = "100 ms";
                };
                2 =                 {
                    bucketCount = 30;
                    bucketEnd = "700 ms";
                    bucketStart = "400 ms";
                };
            };
        };
        signpostAverageMemory = "100,000 kB";
        signpostCumulativeCPUTime = "30,000 ms";
        signpostCumulativeHitchTimeRatio = "50 ms per s";
        signpostCumulativeLogicalWrites = "600 kB";
    };
    signpostName = TestSignpostName1;
    totalSignpostCount = 30;
},
{
    signpostCategory = TestSignpostCategory2;
    signpostIntervalData =     {
        histogrammedSignpostDurations =         {
            histogramNumBuckets = 3;
            histogramValue =             {
                0 =                 {
                    bucketCount = 60;
                    bucketEnd = "200 ms";
                    bucketStart = "0 ms";
                };
                1 =                 {
                    bucketCount = 70;
                    bucketEnd = "300 ms";
                    bucketStart = "201 ms";
                };
                2 =                 {
                    bucketCount = 80;
                    bucketEnd = "500 ms";
                    bucketStart = "301 ms";
                };
            };
        };
        signpostAverageMemory = "60,000 kB";
        signpostCumulativeCPUTime = "50,000 ms";
        signpostCumulativeLogicalWrites = "700 kB";
    };
    signpostName = TestSignpostName2;
    totalSignpostCount = 40;
}
)
, AnyHashable("applicationLaunchMetrics"): {
    histogrammedResumeTime =     {
        histogramNumBuckets = 3;
        histogramValue =         {
            0 =             {
                bucketCount = 60;
                bucketEnd = "210 ms";
                bucketStart = "200 ms";
            };
            1 =             {
                bucketCount = 70;
                bucketEnd = "310 ms";
                bucketStart = "300 ms";
            };
            2 =             {
                bucketCount = 80;
                bucketEnd = "510 ms";
                bucketStart = "500 ms";
            };
        };
    };
    histogrammedTimeToFirstDrawKey =     {
        histogramNumBuckets = 3;
        histogramValue =         {
            0 =             {
                bucketCount = 50;
                bucketEnd = "1,010 ms";
                bucketStart = "1,000 ms";
            };
            1 =             {
                bucketCount = 60;
                bucketEnd = "2,010 ms";
                bucketStart = "2,000 ms";
            };
            2 =             {
                bucketCount = 30;
                bucketEnd = "3,010 ms";
                bucketStart = "3,000 ms";
            };
        };
    };
}, AnyHashable("cellularConditionMetrics"): {
    cellConditionTime =     {
        histogramNumBuckets = 3;
        histogramValue =         {
            0 =             {
                bucketCount = 20;
                bucketEnd = "1 bars";
                bucketStart = "1 bars";
            };
            1 =             {
                bucketCount = 30;
                bucketEnd = "2 bars";
                bucketStart = "2 bars";
            };
            2 =             {
                bucketCount = 50;
                bucketEnd = "3 bars";
                bucketStart = "3 bars";
            };
        };
    };
}, AnyHashable("timeStampEnd"): 2022-09-27 15:59:00 +0000, AnyHashable("metaData"): {
    appBuildVersion = 1;
    bundleIdentifier = "com.baidu.ALALiveSDKDebug";
    deviceType = "iPhone11,2";
    osVersion = "iPhone OS 15.1 (19B74)";
    platformArchitecture = arm64e;
    regionFormat = CN;
}, AnyHashable("displayMetrics"): {
    averagePixelLuminance =     {
        averageValue = "50 apl";
        sampleCount = 500;
        standardDeviation = 0;
    };
}, AnyHashable("locationActivityMetrics"): {
    cumulativeBestAccuracyForNavigationTime = "20 sec";
    cumulativeBestAccuracyTime = "30 sec";
    cumulativeHundredMetersAccuracyTime = "30 sec";
    cumulativeKilometerAccuracyTime = "20 sec";
    cumulativeNearestTenMetersAccuracyTime = "30 sec";
    cumulativeThreeKilometersAccuracyTime = "20 sec";
}, AnyHashable("gpuMetrics"): {
    cumulativeGPUTime = "20 sec";
}, AnyHashable("timeStampBegin"): 2022-09-26 16:00:00 +0000, AnyHashable("appVersion"): 1.0, AnyHashable("cpuMetrics"): {
    cumulativeCPUInstructions = "100 kiloinstructions";
    cumulativeCPUTime = "100 sec";
}, AnyHashable("applicationExitMetrics"): {
    backgroundExitData =     {
        cumulativeAbnormalExitCount = 1;
        cumulativeAppWatchdogExitCount = 1;
        cumulativeBackgroundFetchCompletionTimeoutExitCount = 1;
        cumulativeBackgroundTaskAssertionTimeoutExitCount = 1;
        cumulativeBackgroundURLSessionCompletionTimeoutExitCount = 1;
        cumulativeBadAccessExitCount = 1;
        cumulativeCPUResourceLimitExitCount = 1;
        cumulativeIllegalInstructionExitCount = 1;
        cumulativeMemoryPressureExitCount = 1;
        cumulativeMemoryResourceLimitExitCount = 1;
        cumulativeNormalAppExitCount = 1;
        cumulativeSuspendedWithLockedFileExitCount = 1;
    };
    foregroundExitData =     {
        cumulativeAbnormalExitCount = 1;
        cumulativeAppWatchdogExitCount = 1;
        cumulativeBadAccessExitCount = 1;
        cumulativeCPUResourceLimitExitCount = 1;
        cumulativeIllegalInstructionExitCount = 1;
        cumulativeMemoryResourceLimitExitCount = 1;
        cumulativeNormalAppExitCount = 1;
    };
}, AnyHashable("diskIOMetrics"): {
    cumulativeLogicalWrites = "1,300 kB";
}, AnyHashable("memoryMetrics"): {
    averageSuspendedMemory =     {
        averageValue = "100,000 kB";
        sampleCount = 500;
        standardDeviation = 0;
    };
    peakMemoryUsage = "200,000 kB";
}, AnyHashable("animationMetrics"): {
    scrollHitchTimeRatio = "1,000 ms per s";
}, AnyHashable("networkTransferMetrics"): {
    cumulativeCellularDownload = "80,000 kB";
    cumulativeCellularUpload = "70,000 kB";
    cumulativeWifiDownload = "60,000 kB";
    cumulativeWifiUpload = "50,000 kB";
}, AnyHashable("applicationTimeMetrics"): {
    cumulativeBackgroundAudioTime = "30 sec";
    cumulativeBackgroundLocationTime = "30 sec";
    cumulativeBackgroundTime = "40 sec";
    cumulativeForegroundTime = "700 sec";
}]
```
```
MXDiagnosticPayload
[AnyHashable("hangDiagnostics"): <__NSArrayM 0x280b68720>(
{
    callStackTree =     {
        callStackPerThread = 1;
        callStacks =         (
                        {
                callStackRootFrames =                 (
                                        {
                        address = 74565;
                        binaryName = testBinaryName;
                        binaryUUID = "DDD90555-7FB2-4CC5-861B-4E4E35370FD2";
                        offsetIntoBinaryTextSegment = 123;
                        sampleCount = 20;
                    }
                );
                threadAttributed = 1;
            }
        );
    };
    diagnosticMetaData =     {
        appBuildVersion = 1;
        appVersion = "1.0";
        bundleIdentifier = "com.baidu.ALALiveSDKDebug";
        deviceType = "iPhone11,2";
        hangDuration = "20 sec";
        osVersion = "iPhone OS 15.1 (19B74)";
        platformArchitecture = arm64e;
        regionFormat = CN;
    };
    version = "1.0.0";
}
)
, AnyHashable("cpuExceptionDiagnostics"): <__NSArrayM 0x280b686f0>(
{
    callStackTree =     {
        callStackPerThread = 0;
        callStacks =         (
                        {
                callStackRootFrames =                 (
                                        {
                        address = 74565;
                        binaryName = testBinaryName;
                        binaryUUID = "C55495DA-8EC5-473A-8127-435E932B0B4B";
                        offsetIntoBinaryTextSegment = 123;
                        sampleCount = 20;
                    }
                );
            }
        );
    };
    diagnosticMetaData =     {
        appBuildVersion = 1;
        appVersion = "1.0";
        bundleIdentifier = "com.baidu.ALALiveSDKDebug";
        deviceType = "iPhone11,2";
        osVersion = "iPhone OS 15.1 (19B74)";
        platformArchitecture = arm64e;
        regionFormat = CN;
        totalCPUTime = "20 sec";
        totalSampledTime = "20 sec";
    };
    version = "1.0.0";
}
)
, AnyHashable("crashDiagnostics"): <__NSArrayM 0x280b689c0>(
{
    callStackTree =     {
        callStackPerThread = 1;
        callStacks =         (
                        {
                callStackRootFrames =                 (
                                        {
                        address = 74565;
                        binaryName = testBinaryName;
                        binaryUUID = "DDD90555-7FB2-4CC5-861B-4E4E35370FD2";
                        offsetIntoBinaryTextSegment = 123;
                        sampleCount = 20;
                    }
                );
                threadAttributed = 1;
            }
        );
    };
    diagnosticMetaData =     {
        appBuildVersion = 1;
        appVersion = "1.0";
        bundleIdentifier = "com.baidu.ALALiveSDKDebug";
        deviceType = "iPhone11,2";
        exceptionCode = 0;
        exceptionType = 1;
        osVersion = "iPhone OS 15.1 (19B74)";
        platformArchitecture = arm64e;
        regionFormat = CN;
        signal = 11;
        terminationReason = "Namespace SIGNAL, Code 0xb";
        virtualMemoryRegionInfo = "0 is not in any region.  Bytes before following region: 4000000000 REGION TYPE                      START - END             [ VSIZE] PRT/MAX SHRMOD  REGION DETAIL UNUSED SPACE AT START ---> __TEXT                 0000000000000000-0000000000000000 [   32K] r-x/r-x SM=COW  ...pp/Test";
    };
    version = "1.0.0";
}
)
, AnyHashable("diskWriteExceptionDiagnostics"): <__NSArrayM 0x280b687b0>(
{
    callStackTree =     {
        callStackPerThread = 0;
        callStacks =         (
                        {
                callStackRootFrames =                 (
                                        {
                        address = 74565;
                        binaryName = testBinaryName;
                        binaryUUID = "C55495DA-8EC5-473A-8127-435E932B0B4B";
                        offsetIntoBinaryTextSegment = 123;
                        sampleCount = 20;
                    }
                );
            }
        );
    };
    diagnosticMetaData =     {
        appBuildVersion = 1;
        appVersion = "1.0";
        bundleIdentifier = "com.baidu.ALALiveSDKDebug";
        deviceType = "iPhone11,2";
        osVersion = "iPhone OS 15.1 (19B74)";
        platformArchitecture = arm64e;
        regionFormat = CN;
        writesCaused = "2,000 byte";
    };
    version = "1.0.0";
}
)
, AnyHashable("timeStampBegin"): 2022-09-28 01:41:59 +0000, AnyHashable("timeStampEnd"): 2022-09-28 01:41:59 +0000]
```
