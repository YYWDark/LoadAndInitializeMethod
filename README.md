细说Load和Initialize

![封面.jpg](http://upload-images.jianshu.io/upload_images/307963-0ceffaf791b0e7f5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###load方法：
当一个类或者该类的分类被加入`Objective-C`运行时的时候被调用。`load method`调用的时机是非常早的，所以你不应该在该方法中去引用其他你自定义的对象，因为你没办法去判断该对象是否已经进行 `load method`。当然，你可以使用该类所依赖的`frameworks`，比如`Foundation`，在你调用`load   method`方法时这些框架已经确保被完全加载成功。当然你的父类也会在这前加载成功。

###load调用时机：
1.类本身的`load method`在所有父类的`load method`调用后调用。
2.分类的`load method`在父类的`load method`调用前调用。
3.类本身`load method`在分类的`load method`前调用。
 
我们现在建立4个文件,在每个文件里面添加`load method`，并在`load method`里面打印一句话：

![1.png](http://upload-images.jianshu.io/upload_images/307963-eb11a2d733b20b6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不做任何操作直接`bulid`，我们来看打印台的`log`：

![2.png](http://upload-images.jianshu.io/upload_images/307963-85187e4465203baa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)。
这样的结果就证明了我们前面的结论。

### 源码证明调用时机：
你可以点击
[这里](https://opensource.apple.com/tarballs/objc4/)下载`runtime源码`。这里下载的是`objc4-706.tar.gz`版本。
dyld 是`the dynamic link editor`的缩写，动态链接器.主要的任务是为了生成可执行文件。
更多了解可以点击[这里](https://www.gitbook.com/book/leon_lizi/-/details)

1.引导程序初始化：
```
/***********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Called by libSystem BEFORE library initialization time
**********************************************************************/
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_2_images, load_images, unmap_image);
}
```

2.获取在类的列表和分类的列表，如果没有则return;如果获取到则上锁调用prepare_load_methods
```
void load_images(const char *path __unused, const struct mach_header *mh)
{
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        rwlock_writer_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
```

3.`prepare_load_methods`方法中你可以看到先遍历了类的列表而不是分类的，所以类的`load method`会先调用。`schedule_class_load`方法中你可以看到如果父类有`load method`，会递归的去遍历，`add_class_to_loadable_list`然后加入到待加载列表。所以父类的`load method`方法会比子类先调用。而`_getObjc2NonlazyCategoryList`直接遍历不做任何操作所以子类的在父类的前面。

```
 void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertWriting();

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
  //递归加入到到待加载列表
static void schedule_class_load(Class cls)
{
    if (cls->info & CLS_LOADED) return;
    if (cls->superclass) schedule_class_load(cls->superclass);
    add_class_to_loadable_list(cls);
    cls->info |= CLS_LOADED;
}
```



###Initialize方法
`runtime`会发送`initialize message`当**初始化一个类**。父类会比子类先接受到这个消息。这个操作是线程安全的，这就是说，当你在某一个线程A里面初始化这个类，这个线程A会去发送`Initialize message`给这个类对象。如果其他线程B这个时候要发送其他的消息给这个类，线程B会阻塞直到线程A处理Initialize消息完成。

###Initialize调用时机：
1.父类会比子类先接受到这个消息。
2.如果子类没有实现`initialize mothod`,而父类实现了。那么父类的`Initialize mothod`将会调用多次。

我们现在在'Person'和'Student'文件里面重写`initialize method`,让后在'viewDidLoad method'添加下面的代码：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
          NSLog(@"%@ %s 当前的线程：%@", [self class], __FUNCTION__,[NSThread currentThread]);
          Student *Student1 = [[Student alloc] init];
          Student *Student2 = [[Student alloc] init];
    });
}
```
`bulid`项目，我们来看打印台的`log`:
![3.png](http://upload-images.jianshu.io/upload_images/307963-40448be5b9d2b683.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
你会发现：
1.我们做了两次初始化但第二次初始化`Student`并没有触动`initialize method`。
2.父类的`initialize method`方法比子类的先调用。
3.`Student`初始化的线程和`initialize method`线程是相同的。

现在我们注释掉子类的`initialize method`，再次bulid：
![4.png](http://upload-images.jianshu.io/upload_images/307963-3968b7c8e524916b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个时候`Person`父类调用了两次。

### 源码证明调用时机:
```
void _class_initialize(Class cls)
{
    assert(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    
    // Try to atomically set CLS_INITIALIZING.
    {
        monitor_locker_t lock(classInitLock);
        if (!cls->isInitialized() && !cls->isInitializing()) {
            cls->setInitializing();
            reallyInitialize = YES;
        }
    }
    
    if (reallyInitialize) {
        // We successfully set the CLS_INITIALIZING bit. Initialize the class.
        
        // Record that we're initializing this class so we can message it.
        _setThisThreadIsInitializingClass(cls);
        
        // Send the +initialize message.
        // Note that +initialize is sent to the superclass (again) if 
        // this class doesn't implement +initialize. 2157218
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: calling +[%s initialize]",
                         cls->nameForLogging());
        }

        // Exceptions: A +initialize call that throws an exception 
        // is deemed to be a complete and successful +initialize.
        @try {
            callInitialize(cls);

            if (PrintInitializing) {
                _objc_inform("INITIALIZE: finished +[%s initialize]",
                             cls->nameForLogging());
            }
        }
        @catch (...) {
            if (PrintInitializing) {
                _objc_inform("INITIALIZE: +[%s initialize] threw an exception",
                             cls->nameForLogging());
            }
            @throw;
        }
        @finally {
            // Done initializing. 
            // If the superclass is also done initializing, then update 
            //   the info bits and notify waiting threads.
            // If not, update them later. (This can happen if this +initialize 
            //   was itself triggered from inside a superclass +initialize.)
            monitor_locker_t lock(classInitLock);
            if (!supercls  ||  supercls->isInitialized()) {
                _finishInitializing(cls, supercls);
            } else {
                _finishInitializingAfter(cls, supercls);
            }
        }
        return;
    }
    
    else if (cls->isInitializing()) {
        // We couldn't set INITIALIZING because INITIALIZING was already set.
        // If this thread set it earlier, continue normally.
        // If some other thread set it, block until initialize is done.
        // It's ok if INITIALIZING changes to INITIALIZED while we're here, 
        //   because we safely check for INITIALIZED inside the lock 
        //   before blocking.
        if (_thisThreadIsInitializingClass(cls)) {
            return;
        } else {
            waitForInitializeToComplete(cls);
            return;
        }
    }
    
    else if (cls->isInitialized()) {
        // Set CLS_INITIALIZING failed because someone else already 
        //   initialized the class. Continue normally.
        // NOTE this check must come AFTER the ISINITIALIZING case.
        // Otherwise: Another thread is initializing this class. ISINITIALIZED 
        //   is false. Skip this clause. Then the other thread finishes 
        //   initialization and sets INITIALIZING=no and INITIALIZED=yes. 
        //   Skip the ISINITIALIZING clause. Die horribly.
        return;
    }
    
    else {
        // We shouldn't be here. 
        _objc_fatal("thread-safe class init in objc runtime is buggy!");
    }
}
```

```
void _class_initialize(Class cls) 方法里面下面这段代码就是为了递归保证先执行父类的class_initialize method。
supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }

下面的代码就是为了确认当本类没有实现这个class_initialize method，会再次调用父类的    
  if (PrintInitializing) {
            _objc_inform("INITIALIZE: calling +[%s initialize]",
                         cls->nameForLogging());
        }    
    
```
