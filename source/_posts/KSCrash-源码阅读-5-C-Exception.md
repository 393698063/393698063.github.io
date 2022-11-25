---
title: KSCrash-源码阅读-5 C++ Exception
date: 2022-04-11 19:44:49
tags:
---
## 异常监听
c++ 异常处理的实现是依靠了标准库的 std::set_terminate(CPPExceptionTerminate) 函数。

iOS 工程中某些功能的实现可能使用了C、C++等。假如抛出 C++ 异常，如果该异常可以被转换为 NSException，则走 OC 异常捕获机制，如果不能转换，则继续走 C++ 异常流程，也就是 default_terminate_handler。这个 C++ 异常的默认 terminate 函数内部调用 abort_message 函数，最后触发了一个 abort 调用，系统产生一个 SIGABRT 信号。

在系统抛出 C++ 异常后，加一层 try...catch... 来判断该异常是否可以转换为 NSException，再重新抛出的C++异常。此时异常的现场堆栈已经消失，所以上层通过捕获 SIGABRT 信号是无法还原发生异常时的场景，即异常堆栈缺失。
```
static void setEnabled(bool isEnabled)
{
    if(isEnabled != g_isEnabled)
    {
        g_isEnabled = isEnabled;
        if(isEnabled)
        {
            initialize();

            ksid_generate(g_eventID);
            g_originalTerminateHandler = std::set_terminate(CPPExceptionTerminate);
        }
        else
        {
            std::set_terminate(g_originalTerminateHandler);
        }
        g_captureNextStackTrace = isEnabled;
    }
}
static void initialize()
{
    static bool isInitialized = false;
    if(!isInitialized)
    {
        isInitialized = true;
        kssc_initCursor(&g_stackCursor, NULL, NULL);
    }
}
void kssc_initCursor(KSStackCursor *cursor,
                     void (*resetCursor)(KSStackCursor*),
                     bool (*advanceCursor)(KSStackCursor*))
{
    cursor->symbolicate = kssymbolicator_symbolicate;
    cursor->advanceCursor = advanceCursor != NULL ? advanceCursor : g_advanceCursor;
    cursor->resetCursor = resetCursor != NULL ? resetCursor : kssc_resetCursor;
    cursor->resetCursor(cursor);
}
```

## 处理异常
```
static void CPPExceptionTerminate(void)
{
    thread_act_array_t threads = NULL;
    mach_msg_type_number_t numThreads = 0;
    ksmc_suspendEnvironment(&threads, &numThreads);
    KSLOG_DEBUG("Trapped c++ exception");
    const char* name = NULL;
    std::type_info* tinfo = __cxxabiv1::__cxa_current_exception_type();
    if(tinfo != NULL)
    {
        name = tinfo->name();
    }
    
    if(name == NULL || strcmp(name, "NSException") != 0)
    {
        kscm_notifyFatalExceptionCaptured(false);
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));

        char descriptionBuff[DESCRIPTION_BUFFER_LENGTH];
        const char* description = descriptionBuff;
        descriptionBuff[0] = 0;

        KSLOG_DEBUG("Discovering what kind of exception was thrown.");
        g_captureNextStackTrace = false;
        try
        {
            throw;
        }
        catch(std::exception& exc)
        {
            strncpy(descriptionBuff, exc.what(), sizeof(descriptionBuff));
        }
#define CATCH_VALUE(TYPE, PRINTFTYPE) \
catch(TYPE value)\
{ \
    snprintf(descriptionBuff, sizeof(descriptionBuff), "%" #PRINTFTYPE, value); \
}
        CATCH_VALUE(char,                 d)
        CATCH_VALUE(short,                d)
        CATCH_VALUE(int,                  d)
        CATCH_VALUE(long,                ld)
        CATCH_VALUE(long long,          lld)
        CATCH_VALUE(unsigned char,        u)
        CATCH_VALUE(unsigned short,       u)
        CATCH_VALUE(unsigned int,         u)
        CATCH_VALUE(unsigned long,       lu)
        CATCH_VALUE(unsigned long long, llu)
        CATCH_VALUE(float,                f)
        CATCH_VALUE(double,               f)
        CATCH_VALUE(long double,         Lf)
        CATCH_VALUE(char*,                s)
        catch(...)
        {
            description = NULL;
        }
        g_captureNextStackTrace = g_isEnabled;

        // TODO: Should this be done here? Maybe better in the exception handler?
        KSMC_NEW_CONTEXT(machineContext);
        ksmc_getContextForThread(ksthread_self(), machineContext, true);

        KSLOG_DEBUG("Filling out context.");
        crashContext->crashType = KSCrashMonitorTypeCPPException;
        crashContext->eventID = g_eventID;
        crashContext->registersAreValid = false;
        crashContext->stackCursor = &g_stackCursor;
        crashContext->CPPException.name = name;
        crashContext->exceptionName = name;
        crashContext->crashReason = description;
        crashContext->offendingMachineContext = machineContext;

        kscm_handleException(crashContext);
    }
    else
    {
        KSLOG_DEBUG("Detected NSException. Letting the current NSException handler deal with it.");
    }
    ksmc_resumeEnvironment(threads, numThreads);

    KSLOG_DEBUG("Calling original terminate handler.");
    g_originalTerminateHandler();
}
```
