---
title: KSCrash-源码阅读-5 Dead lock
date: 2022-04-11 20:47:08
tags:
---
## 检测原理
主线程死锁的检测和 ANR 的检测有些类似

创建一个线程，在线程运行方法中用 do...while... 循环处理逻辑，加了 autorelease 避免内存过高

有一个 awaitingResponse 属性和 watchdogPulse 方法。watchdogPulse 主要逻辑为设置 awaitingResponse 为 YES，切换到主线程中，设置 awaitingResponse 为 NO，

线程的执行方法里面不断循环，等待设置的 g_watchdogInterval 后判断 awaitingResponse 的属性值是不是初始状态的值，否则判断为死锁

```
- (id) init
{
    if((self = [super init]))
    {
        // target (self) is retained until selector (runMonitor) exits.
        self.monitorThread = [[NSThread alloc] initWithTarget:self selector:@selector(runMonitor) object:nil];
        self.monitorThread.name = @"KSCrash Deadlock Detection Thread";
        [self.monitorThread start];
    }
    return self;
}

- (void) runMonitor
{
    BOOL cancelled = NO;
    do
    {
        // Only do a watchdog check if the watchdog interval is > 0.
        // If the interval is <= 0, just idle until the user changes it.
        @autoreleasepool {
            NSTimeInterval sleepInterval = g_watchdogInterval;
            BOOL runWatchdogCheck = sleepInterval > 0;
            if(!runWatchdogCheck)
            {
                sleepInterval = kIdleInterval;
            }
            [NSThread sleepForTimeInterval:sleepInterval];
            cancelled = self.monitorThread.isCancelled;
            if(!cancelled && runWatchdogCheck)
            {
                if(self.awaitingResponse)
                {
                    [self handleDeadlock];
                }
                else
                {
                    [self watchdogPulse];
                }
            }
        }
    } while (!cancelled);
}
- (void) watchdogPulse
{
    __block id blockSelf = self;
    self.awaitingResponse = YES;
    dispatch_async(dispatch_get_main_queue(), ^
                   {
                       [blockSelf watchdogAnswer];
                   });
}
```

## 处理异常
```
- (void) handleDeadlock
{
    thread_act_array_t threads = NULL;
    mach_msg_type_number_t numThreads = 0;
    ksmc_suspendEnvironment(&threads, &numThreads);
    kscm_notifyFatalExceptionCaptured(false);

    KSMC_NEW_CONTEXT(machineContext);
    ksmc_getContextForThread(g_mainQueueThread, machineContext, false);
    KSStackCursor stackCursor;
    kssc_initWithMachineContext(&stackCursor, KSSC_MAX_STACK_DEPTH, machineContext);
    char eventID[37];
    ksid_generate(eventID);

    KSLOG_DEBUG(@"Filling out context.");
    KSCrash_MonitorContext* crashContext = &g_monitorContext;
    memset(crashContext, 0, sizeof(*crashContext));
    crashContext->crashType = KSCrashMonitorTypeMainThreadDeadlock;
    crashContext->eventID = eventID;
    crashContext->registersAreValid = false;
    crashContext->offendingMachineContext = machineContext;
    crashContext->stackCursor = &stackCursor;
    
    kscm_handleException(crashContext);
    ksmc_resumeEnvironment(threads, numThreads);

    KSLOG_DEBUG(@"Calling abort()");
    abort();
}
```