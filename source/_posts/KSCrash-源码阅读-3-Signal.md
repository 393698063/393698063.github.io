---
title: KSCrash-源码阅读-3 Signal
date: 2022-04-11 09:53:15
tags:
---
## Signal异常处理流程
1. Restore the default signal handlers
2. 添加signal handler 处理异常
3. record the signal information，write a crash report
4. Once we're done, re-raise the signal and let the default handlers deal with it
## 添加 signal handler
1. signal异常
```
定义在 #include <machine/signal.h> 
    SIGABRT,/* abort() */
    SIGBUS,/* bus error */
    SIGFPE,/* floating point exception */
    SIGILL,/* illegal instruction (not reset when caught) */
    SIGPIPE, /* write on a pipe with no one to read it */
    SIGSEGV, /* segmentation violation */
    SIGSYS,/* bad argument to system call */
    SIGTRAP /* trace trap (not reset when caught) */
```
2. 添加handler

关键函数
> /**
> 一、函数原型：sigaction函数的功能是检查或修改与指定信号相关联的> 处理动作（可同时> 两种操作）
>
> int sigaction(int signum, const struct sigaction *act,
>                      struct sigaction *oldact);
> signum参数指出要捕获的信号类型，act参数指定新的信号处理方式，> > oldact参数输出> 先前信号的处理方式
>                 */

```
static bool installSignalHandler()
{
    KSLOG_DEBUG("Installing signal handler.");

#if KSCRASH_HAS_SIGNAL_STACK

    if(g_signalStack.ss_size == 0)
    {
        KSLOG_DEBUG("Allocating signal stack area.");
        g_signalStack.ss_size = SIGSTKSZ;
        g_signalStack.ss_sp = malloc(g_signalStack.ss_size);
    }

    KSLOG_DEBUG("Setting signal stack area.");
     /**
     信号处理函数的栈挪到堆中，而不和进程共用一块栈区
    sigaltstack() 函数，该函数的第 1 个参数 sigstack
     是一个 stack_t 结构的指针，该结构存储了一个“可替换信号栈”
     的位置及属性信息。第 2 个参数 old_sigstack
     也是一个 stack_t 类型指针，
     它用来返回上一次建立的“可替换信号栈”的信息(如果有的话)
     */
    if(sigaltstack(&g_signalStack, NULL) != 0)
    {
        KSLOG_ERROR("signalstack: %s", strerror(errno));
        goto failed;
    }
#endif

    const int* fatalSignals = kssignal_fatalSignals();
    int fatalSignalsCount = kssignal_numFatalSignals();

    if(g_previousSignalHandlers == NULL)
    {
        KSLOG_DEBUG("Allocating memory to store previous signal handlers.");
        g_previousSignalHandlers = malloc(sizeof(*g_previousSignalHandlers)
                                          * (unsigned)fatalSignalsCount);
    }

    struct sigaction action = {{0}};
    action.sa_flags = SA_SIGINFO | SA_ONSTACK;
#if KSCRASH_HOST_APPLE && defined(__LP64__)
    action.sa_flags |= SA_64REGSET;
#endif
    sigemptyset(&action.sa_mask);
    action.sa_sigaction = &handleSignal;

    for(int i = 0; i < fatalSignalsCount; i++)
    {
        KSLOG_DEBUG("Assigning handler for signal %d", fatalSignals[i]);
        if(sigaction(fatalSignals[i], &action, &g_previousSignalHandlers[i]) != 0)
        {
            char sigNameBuff[30];
            const char* sigName = kssignal_signalName(fatalSignals[i]);
            if(sigName == NULL)
            {
                snprintf(sigNameBuff, sizeof(sigNameBuff), "%d", fatalSignals[i]);
                sigName = sigNameBuff;
            }
            KSLOG_ERROR("sigaction (%s): %s", sigName, strerror(errno));
            // Try to reverse the damage
            for(i--;i >= 0; i--)
            {
                /**
                 一、函数原型：sigaction函数的功能是检查或修改与指定信号相关联的处理动作（可同时两种操作）

                 int sigaction(int signum, const struct sigaction *act,
                                      struct sigaction *oldact);
                 signum参数指出要捕获的信号类型，act参数指定新的信号处理方式，oldact参数输出先前信号的处理方式
                 */
                sigaction(fatalSignals[i], &g_previousSignalHandlers[i], NULL);
            }
            goto failed;
        }
    }
    KSLOG_DEBUG("Signal handlers installed.");
    return true;

failed:
    KSLOG_DEBUG("Failed to install signal handlers.");
    return false;
}
```

## 处理异常
```
/** Our custom signal handler.
 * Restore the default signal handlers, record the signal information, and
 * write a crash report.
 * Once we're done, re-raise the signal and let the default handlers deal with
 * it.
 *
 * @param sigNum The signal that was raised.
 *
 * @param signalInfo Information about the signal.
 *
 * @param userContext Other contextual information.
 */
static void handleSignal(int sigNum, siginfo_t* signalInfo, void* userContext)
{
    KSLOG_DEBUG("Trapped signal %d", sigNum);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        ksmc_suspendEnvironment(&threads, &numThreads);
        kscm_notifyFatalExceptionCaptured(false);

        KSLOG_DEBUG("Filling out context.");
        KSMC_NEW_CONTEXT(machineContext);
        ksmc_getContextForSignal(userContext, machineContext);
        kssc_initWithMachineContext(&g_stackCursor, KSSC_MAX_STACK_DEPTH, machineContext);

        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        memset(crashContext, 0, sizeof(*crashContext));
        crashContext->crashType = KSCrashMonitorTypeSignal;
        crashContext->eventID = g_eventID;
        crashContext->offendingMachineContext = machineContext;
        crashContext->registersAreValid = true;
        crashContext->faultAddress = (uintptr_t)signalInfo->si_addr;
        crashContext->signal.userContext = userContext;
        crashContext->signal.signum = signalInfo->si_signo;
        crashContext->signal.sigcode = signalInfo->si_code;
        crashContext->stackCursor = &g_stackCursor;

        kscm_handleException(crashContext);
        ksmc_resumeEnvironment(threads, numThreads);
    }

    KSLOG_DEBUG("Re-raising signal for regular handlers to catch.");
    // This is technically not allowed, but it works in OSX and iOS.
    raise(sigNum);
}
```

## 调试signal
Xcode Debug模式运行App时，App进程signal被LLDB Debugger调试器捕获；需要使用LLDB调试命令，将指定signal处理抛到用户层处理，方便调试。

查看全部信号传递配置：<br>
// process handle缩写 <br>
pro hand <br>
修改指定信号传递配置：<br>
// option: <br>
//   -P: PASS <br>
//   -S: STOP <br>
//   -N: NOTIFY<br>
pro hand -option false 信号名<br>

// 例：SIGABRT信号处理在LLDB不停止，可继续抛到用户层<br>
pro hand -s false SIGABRT<br>

