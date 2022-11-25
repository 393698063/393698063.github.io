---
title: KSCrash-源码阅读-6 zoombie_class
date: 2022-04-11 21:35:59
tags:
---
## 检测原理
hook 对象的dealloc 方法，将释放的对象保存为僵尸对象
```
#define CREATE_ZOMBIE_HANDLER_INSTALLER(CLASS) \
static IMP g_originalDealloc_ ## CLASS; \
static void handleDealloc_ ## CLASS(id self, SEL _cmd) \
{ \
    handleDealloc(self); \
    typedef void (*fn)(id,SEL); \
    fn f = (fn)g_originalDealloc_ ## CLASS; \
    f(self, _cmd); \
} \
static void installDealloc_ ## CLASS() \
{ \
    Method method = class_getInstanceMethod(objc_getClass(#CLASS), sel_registerName("dealloc")); \
    g_originalDealloc_ ## CLASS = method_getImplementation(method); \
    method_setImplementation(method, (IMP)handleDealloc_ ## CLASS); \
}
// TODO: Uninstall doesn't work.
//static void uninstallDealloc_ ## CLASS() \
//{ \
//    method_setImplementation(class_getInstanceMethod(objc_getClass(#CLASS), sel_registerName("dealloc")), g_originalDealloc_ ## CLASS); \
//}

CREATE_ZOMBIE_HANDLER_INSTALLER(NSObject)
CREATE_ZOMBIE_HANDLER_INSTALLER(NSProxy)

static void install()
{
    unsigned cacheSize = CACHE_SIZE;
    g_zombieHashMask = cacheSize - 1;
    g_zombieCache = calloc(cacheSize, sizeof(*g_zombieCache));
    if(g_zombieCache == NULL)
    {
        KSLOG_ERROR("Error: Could not allocate %u bytes of memory. KSZombie NOT installed!",
              cacheSize * sizeof(*g_zombieCache));
        return;
    }

    g_lastDeallocedException.class = objc_getClass("NSException");
    g_lastDeallocedException.address = NULL;
    g_lastDeallocedException.name[0] = 0;
    g_lastDeallocedException.reason[0] = 0;

    installDealloc_NSObject();
    installDealloc_NSProxy();
}

static inline void handleDealloc(const void* self)
{
    volatile Zombie* cache = g_zombieCache;
    likely_if(cache != NULL)
    {
        Zombie* zombie = (Zombie*)cache + hashIndex(self);
        zombie->object = self;
        Class class = object_getClass((id)self);
        zombie->className = class_getName(class);
        for(; class != nil; class = class_getSuperclass(class))
        {
            unlikely_if(class == g_lastDeallocedException.class)
            {
                storeException(self);
            }
        }
    }
}
```

## 参考资料
[iOS使用Zombie Objects检测僵尸对象及其原理](https://www.jianshu.com/p/7bb92761f2f5)