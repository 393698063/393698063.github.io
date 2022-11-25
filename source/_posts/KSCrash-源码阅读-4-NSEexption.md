---
title: KSCrash-源码阅读-4 NSEexption
date: 2022-04-11 10:49:33
tags:
---
## 添加NsException 自定义处理入口
1. NSGetUncaughtExceptionHandler() 获取已经添加的处理入口
2. NSSetUncaughtExceptionHandler(&handleUncaughtException) 添加自己的异常处理入口
```
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled)
        {
            KSLOG_DEBUG(@"Backing up original handler.");
            g_previousUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
            
            KSLOG_DEBUG(@"Setting new handler.");
            NSSetUncaughtExceptionHandler(&handleUncaughtException);
            KSCrash.sharedInstance.uncaughtExceptionHandler = &handleUncaughtException;
            KSCrash.sharedInstance.currentSnapshotUserReportedExceptionHandler = &handleCurrentSnapshotUserReportedException;
        }
        else
        {
            KSLOG_DEBUG(@"Restoring original handler.");
            NSSetUncaughtExceptionHandler(g_previousUncaughtExceptionHandler);
        }
    }
}
```
## 异常处理入口
``` 
// ============================================================================

/** Our custom excepetion handler.
 * Fetch the stack trace from the exception and write a report.
 *
 * @param exception The exception that was raised.
 */

static void handleException(NSException* exception, BOOL currentSnapshotUserReported) {
    KSLOG_DEBUG(@"Trapped exception %@", exception);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        ksmc_suspendEnvironment(&threads, &numThreads);
        kscm_notifyFatalExceptionCaptured(false);

        KSLOG_DEBUG(@"Filling out context.");
        NSArray* addresses = [exception callStackReturnAddresses];
        NSUInteger numFrames = addresses.count;
        uintptr_t* callstack = malloc(numFrames * sizeof(*callstack));
        for(NSUInteger i = 0; i < numFrames; i++)
        {
            callstack[i] = (uintptr_t)[addresses[i] unsignedLongLongValue];
        }

        char eventID[37];
        ksid_generate(eventID);
        KSMC_NEW_CONTEXT(machineContext);
        ksmc_getContextForThread(ksthread_self(), machineContext, true);
        KSStackCursor cursor;
        kssc_initWithBacktrace(&cursor, callstack, (int)numFrames, 0);

        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));
        crashContext->crashType = KSCrashMonitorTypeNSException;
        crashContext->eventID = eventID;
        crashContext->offendingMachineContext = machineContext;
        crashContext->registersAreValid = false;
        crashContext->NSException.name = [[exception name] UTF8String];
        crashContext->NSException.userInfo = [[NSString stringWithFormat:@"%@", exception.userInfo] UTF8String];
        crashContext->exceptionName = crashContext->NSException.name;
        crashContext->crashReason = [[exception reason] UTF8String];
        crashContext->stackCursor = &cursor;
        crashContext->currentSnapshotUserReported = currentSnapshotUserReported;

        KSLOG_DEBUG(@"Calling main crash handler.");
        kscm_handleException(crashContext);

        free(callstack);
        if (currentSnapshotUserReported) {
            ksmc_resumeEnvironment(threads, numThreads);
        }
        if (g_previousUncaughtExceptionHandler != NULL)
        {
            KSLOG_DEBUG(@"Calling original exception handler.");
            g_previousUncaughtExceptionHandler(exception);
        }
    }
}
```

