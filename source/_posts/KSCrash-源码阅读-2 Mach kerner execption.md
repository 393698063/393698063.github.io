---
title: KSCrash 源码阅读-2 Mach kerner exception
date: 2021-12-29 19:02:57
tags:
---
## mach crash 处理流程
![h5 load time](./KSCrash-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-2%20Mach%20kerner%20execption/mach_crash_flow.png)
{% asset_img mach_crash_flow.png This is an example image %}
## 1. 注册mach异常处理回调
1. 保存原先的端口，用来处理完后回调给原来的端口
2. mach_port_allocate 已receive rights新建一个端口
3. 为新建端口添加发送消息权限
4. task_set_exception_ports 将新建端口作为异常接收端口
5. pthread_create创建一个备用线程
6. 将备用线程将pthread_t的类型转换成mach_port_t类型
7. pthread_create 创建初始处理异常的线程
```
static bool installExceptionHandler()
{
    KSLOG_DEBUG("Installing mach exception handler.");

    bool attributes_created = false;
    pthread_attr_t attr;

    kern_return_t kr;
    int error;
/**
 任务（task）是一种容器（container）对象，
 虚拟内存空间和其他资源都是通过这个容器对象管理的，
 这些资源包括设备和其他句柄。资源进一步被抽象为端口。
 因而资源的共享实际上相当于允许对对应端口的访问

 链接：https://www.jianshu.com/p/cc655bfdac13
 */
    const task_t thisTask = mach_task_self(); //获得任务的端口，带有发送权限的名称
    exception_mask_t mask = EXC_MASK_BAD_ACCESS |
    EXC_MASK_BAD_INSTRUCTION |
    EXC_MASK_ARITHMETIC |
    EXC_MASK_SOFTWARE |
    EXC_MASK_BREAKPOINT;
    // 保存原先的端口，用来处理完后回调给原来的端口
    KSLOG_DEBUG("Backing up original exception ports.");
    kr = task_get_exception_ports(thisTask,
                                  mask,
                                  g_previousExceptionPorts.masks,
                                  &g_previousExceptionPorts.count,
                                  g_previousExceptionPorts.ports,
                                  g_previousExceptionPorts.behaviors,
                                  g_previousExceptionPorts.flavors);
    if(kr != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_get_exception_ports: %s", mach_error_string(kr));
        goto failed;
    }

    if(g_exceptionPort == MACH_PORT_NULL)
    {
        KSLOG_DEBUG("Allocating new port with receive rights.");
        kr = mach_port_allocate(thisTask,
                                MACH_PORT_RIGHT_RECEIVE,
                                &g_exceptionPort);
        if(kr != KERN_SUCCESS)
        {
            KSLOG_ERROR("mach_port_allocate: %s", mach_error_string(kr));
            goto failed;
        }

        KSLOG_DEBUG("Adding send rights to port.");
        kr = mach_port_insert_right(thisTask,
                                    g_exceptionPort,
                                    g_exceptionPort,
                                    MACH_MSG_TYPE_MAKE_SEND);
        if(kr != KERN_SUCCESS)
        {
            KSLOG_ERROR("mach_port_insert_right: %s", mach_error_string(kr));
            goto failed;
        }
    }

    KSLOG_DEBUG("Installing port as exception handler.");
    kr = task_set_exception_ports(thisTask,
                                  mask,
                                  g_exceptionPort,
                                  (int)(EXCEPTION_DEFAULT | MACH_EXCEPTION_CODES),
                                  THREAD_STATE_NONE);
    if(kr != KERN_SUCCESS)
    {
        KSLOG_ERROR("task_set_exception_ports: %s", mach_error_string(kr));
        goto failed;
    }

    KSLOG_DEBUG("Creating secondary exception thread (suspended).");
    pthread_attr_init(&attr);
    attributes_created = true;
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    error = pthread_create(&g_secondaryPThread,
                           &attr,
                           &handleExceptions,
                           (void*)kThreadSecondary);
    if(error != 0)
    {
        KSLOG_ERROR("pthread_create_suspended_np: %s", strerror(error));
        goto failed;
    }
    g_secondaryMachThread = pthread_mach_thread_np(g_secondaryPThread);
    ksmc_addReservedThread(g_secondaryMachThread);

    KSLOG_DEBUG("Creating primary exception thread.");
    error = pthread_create(&g_primaryPThread,
                           &attr,
                           &handleExceptions,
                           (void*)kThreadPrimary);
    if(error != 0)
    {
        KSLOG_ERROR("pthread_create: %s", strerror(error));
        goto failed;
    }
    pthread_attr_destroy(&attr);
    g_primaryMachThread = pthread_mach_thread_np(g_primaryPThread);
    ksmc_addReservedThread(g_primaryMachThread);

    KSLOG_DEBUG("Mach exception handler installed.");
    return true;


failed:
    KSLOG_DEBUG("Failed to install mach exception handler.");
    if(attributes_created)
    {
        pthread_attr_destroy(&attr);
    }
    uninstallExceptionHandler();
    return false;
}
```
## 2. 处理异常
1. 在for(;;)循环中等待mach消息，处理异常的线程不会一直执行 mach_msg会等待消息
2. 抓取异常
2. ksmc_suspendEnvironment 挂起所有线程除异常处理线程之外
```
/** Our exception handler thread routine.
 * Wait for an exception message, uninstall our exception port, record the
 * exception information, and write a report.
 */
 / ============================================================================

/** A mach exception message (according to ux_exception.c, xnu-1699.22.81).
 */
#pragma pack(4)
typedef struct
{
    /** Mach header. */
    mach_msg_header_t          header;

    // Start of the kernel processed data.

    /** Basic message body data. */
    mach_msg_body_t            body;

    /** The thread that raised the exception. */
    mach_msg_port_descriptor_t thread;

    /** The task that raised the exception. */
    mach_msg_port_descriptor_t task;

    // End of the kernel processed data.

    /** Network Data Representation. */
    NDR_record_t               NDR;

    /** The exception that was raised. */
    exception_type_t           exception;

    /** The number of codes. */
    mach_msg_type_number_t     codeCount;

    /** Exception code and subcode. */
    // ux_exception.c defines this as mach_exception_data_t for some reason.
    // But it's not actually a pointer; it's an embedded array.
    // On 32-bit systems, only the lower 32 bits of the code and subcode
    // are valid.
    mach_exception_data_type_t code[0];

    /** Padding to avoid RCV_TOO_LARGE. */
    char                       padding[512];
} MachExceptionMessage;
// 处理异常
static void* handleExceptions(void* const userData)
{
    MachExceptionMessage exceptionMessage = {{0}};
    MachReplyMessage replyMessage = {{0}};
    char* eventID = g_primaryEventID;

    const char* threadName = (const char*) userData;
    pthread_setname_np(threadName);
    if(threadName == kThreadSecondary)
    {
        KSLOG_DEBUG("This is the secondary thread. Suspending.");
        thread_suspend((thread_t)ksthread_self());
        eventID = g_secondaryEventID;
    }

    for(;;)
    {
        KSLOG_DEBUG("Waiting for mach exception");

        // Wait for a message.
        kern_return_t kr = mach_msg(&exceptionMessage.header,
                                    MACH_RCV_MSG,
                                    0,
                                    sizeof(exceptionMessage),
                                    g_exceptionPort,
                                    MACH_MSG_TIMEOUT_NONE,
                                    MACH_PORT_NULL);
        if(kr == KERN_SUCCESS)
        {
            break;
        }

        // Loop and try again on failure.
        KSLOG_ERROR("mach_msg: %s", mach_error_string(kr));
    }

    KSLOG_DEBUG("Trapped mach exception code 0x%llx, subcode 0x%llx",
                exceptionMessage.code[0], exceptionMessage.code[1]);
    if(g_isEnabled)
    {
        thread_act_array_t threads = NULL;
        mach_msg_type_number_t numThreads = 0;
        ///挂起所有的线程
        ksmc_suspendEnvironment(&threads, &numThreads);
        g_isHandlingCrash = true;
        kscm_notifyFatalExceptionCaptured(true);

        KSLOG_DEBUG("Exception handler is installed. Continuing exception handling.");


        // Switch to the secondary thread if necessary, or uninstall the handler
        // to avoid a death loop.
        if(ksthread_self() == g_primaryMachThread)
        {
            KSLOG_DEBUG("This is the primary exception thread. Activating secondary thread.");
// TODO: This was put here to avoid a freeze. Does secondary thread ever fire?
            restoreExceptionPorts();
            if(thread_resume(g_secondaryMachThread) != KERN_SUCCESS)
            {
                KSLOG_DEBUG("Could not activate secondary thread. Restoring original exception ports.");
            }
        }
        else
        {
            KSLOG_DEBUG("This is the secondary exception thread.");// Restoring original exception ports.");
//            restoreExceptionPorts();
        }

        // Fill out crash information
        KSLOG_DEBUG("Fetching machine state.");
        KSMC_NEW_CONTEXT(machineContext);
        KSCrash_MonitorContext* crashContext = &g_monitorContext;
        crashContext->offendingMachineContext = machineContext;
        kssc_initCursor(&g_stackCursor, NULL, NULL);
        if(ksmc_getContextForThread(exceptionMessage.thread.name, machineContext, true))
        {
            kssc_initWithMachineContext(&g_stackCursor, KSSC_MAX_STACK_DEPTH, machineContext);
            KSLOG_TRACE("Fault address %p, instruction address %p",
                        kscpu_faultAddress(machineContext), kscpu_instructionAddress(machineContext));
            if(exceptionMessage.exception == EXC_BAD_ACCESS)
            {
                crashContext->faultAddress = kscpu_faultAddress(machineContext);
            }
            else
            {
                crashContext->faultAddress = kscpu_instructionAddress(machineContext);
            }
        }

        KSLOG_DEBUG("Filling out context.");
        crashContext->crashType = KSCrashMonitorTypeMachException;
        crashContext->eventID = eventID;
        crashContext->registersAreValid = true;
        crashContext->mach.type = exceptionMessage.exception;
        crashContext->mach.code = exceptionMessage.code[0] & (int64_t)MACH_ERROR_CODE_MASK;
        crashContext->mach.subcode = exceptionMessage.code[1] & (int64_t)MACH_ERROR_CODE_MASK;
        if(crashContext->mach.code == KERN_PROTECTION_FAILURE && crashContext->isStackOverflow)
        {
            // A stack overflow should return KERN_INVALID_ADDRESS, but
            // when a stack blasts through the guard pages at the top of the stack,
            // it generates KERN_PROTECTION_FAILURE. Correct for this.
            crashContext->mach.code = KERN_INVALID_ADDRESS;
        }
        crashContext->signal.signum = signalForMachException(crashContext->mach.type, crashContext->mach.code);
        crashContext->stackCursor = &g_stackCursor;

        kscm_handleException(crashContext);

        KSLOG_DEBUG("Crash handling complete. Restoring original handlers.");
        g_isHandlingCrash = false;
        ksmc_resumeEnvironment(threads, numThreads);
    }

    KSLOG_DEBUG("Replying to mach exception message.");
    // Send a reply saying "I didn't handle this exception".
    replyMessage.header = exceptionMessage.header;
    replyMessage.NDR = exceptionMessage.NDR;
    replyMessage.returnCode = KERN_FAILURE;

    mach_msg(&replyMessage.header,
             MACH_SEND_MSG,
             sizeof(replyMessage),
             0,
             MACH_PORT_NULL,
             MACH_MSG_TIMEOUT_NONE,
             MACH_PORT_NULL);

    return NULL;
}
```

##  LLDB Debugger
__unsafe_unretained NSObject *objc = [[NSObject alloc] init];
    NSLog(@"✳️✳️✳️ objc: %@", objc)
 ARC 下会发生 EXC_BAD_ACCESS 异常，这里要注意一下，只有我们关闭 xcode 的 Debug executable 选项才能收到 exc_handler 回调


## Q&A
1. Q:添加mac异常自定义处理是如何将线程和端口绑定
在异常处理线程等待异常端口的消息
```
KSLOG_DEBUG("Installing port as exception handler.");
    kr = task_set_exception_ports(thisTask,
                                  mask,
                                  g_exceptionPort,
                                  (int)(EXCEPTION_DEFAULT | MACH_EXCEPTION_CODES),
                                  THREAD_STATE_NONE);

KSLOG_DEBUG("Creating primary exception thread.");
    error = pthread_create(&g_primaryPThread,git
                           &attr,
                           &handleExceptions,
                           (void*)kThreadPrimary); -->
 ```


## 参考资料
[MAC OS 的mach_port_t和pthread_self()](https://blog.csdn.net/yxccc_914/article/details/79854603) <br>
[iOS Mach异常和signal信号](https://developer.aliyun.com/article/499180)<br>
[获取线程ID有更好的方式吗](https://blog.csdn.net/killer1989/article/details/106674973)<br>
