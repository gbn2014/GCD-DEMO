# GCD-DEMO
常用法
1、dispatch_async
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(globalQueue, ^{
    // 一个异步的任务，例如网络请求，耗时的文件操作等等
    ...
    dispatch_async(dispatch_get_main_queue(), ^{
        // UI刷新
        ...
    });
});
2、dispatch_after
dispatch_queue_t queue= dispatch_get_main_queue();
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5.0 * NSEC_PER_SEC)), queue, ^{
    // 在queue里面延迟执行的一段代码
    ...
});
3、dispatch_once
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    // 只执行一次的任务
    ...
});
4、dispatch_group
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{
    // 异步任务1
});

dispatch_group_async(group, queue, ^{
    // 异步任务2
});

// 等待group中多个异步任务执行完毕，做一些事情，介绍两种方式

// 方式1（不好，会卡住当前线程）
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
...

// 方式2（比较好）
dispatch_group_notify(group, mainQueue, ^{
    // 任务完成后，在主队列中做一些操作
    ...
});
5、dispatch_barrier_async
// dispatch_barrier_async的作用可以用一个词概括－－承上启下，它保证此前的任务都先于自己执行，此后的任务也迟于自己执行。本例中，任务4会在任务1、2、3都执行完之后执行，而任务5、6会等待任务4执行完后执行。

dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(queue, ^{
    // 任务1
    ...
});
dispatch_async(queue, ^{
    // 任务2
    ...
});
dispatch_async(queue, ^{
    // 任务3
    ...
});
dispatch_barrier_async(queue, ^{
    // 任务4
    ...
});
dispatch_async(queue, ^{
    // 任务5
    ...
});
dispatch_async(queue, ^{
    // 任务6
    ...
});
6、dispatch_apply
// for循环做一些事情，输出0123456789
for (int i = 0; i < 10; i ++) {
    NSLog(@"%d", i);
}

// dispatch_apply替换（当且仅当处理顺序对处理结果无影响环境），输出顺序不定，比如1098673452
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
/*! dispatch_apply函数说明
*
*  @brief  dispatch_apply函数是dispatch_sync函数和Dispatch Group的关联API
*         该函数按指定的次数将指定的Block追加到指定的Dispatch Queue中,并等到全部的处理执行结束
*
*  @param 10    指定重复次数  指定10次
*  @param queue 追加对象的Dispatch Queue
*  @param index 带有参数的Block, index的作用是为了按执行的顺序区分各个Block
*
*/
dispatch_apply(10, queue, ^(size_t index) {
    NSLog(@"%zu", index);
});
应用场景
那么，dispatch_apply有什么用呢，因为dispatch_apply并行的运行机制，效率一般快于for循环的类串行机制（
在for一次循环中的处理任务很多时差距比较大）。比如这可以用来拉取网络数据后提前算出各个控件的大小，防止
绘制时计算，提高表单滑动流畅性，如果用for循环，耗时较多，并且每个表单的数据没有依赖关系，所以用dispatch_apply比较好。

7、dispatch_suspend/dispatch_resume
dispatch_queue_t queue = dispatch_get_main_gueue();
dispatch_suspend(queue);//暂停
dispatch_reusme(queue);//恢复

8、dispatch_semaphore
// dispatch_semaphore_signal有两类用法：a、解决同步问题；b、解决有限资源访问（资源为1，即互斥）问题。
// dispatch_semaphore_wait，若semaphore计数为0则等待，大于0则使其减1。
// dispatch_semaphore_signal使semaphore计数加1。
通过dispatch_semaphore_wait来等待确保线程同步，见例子
// a、同步问题：输出肯定为1、2、3。
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_semaphore_t semaphore1 = dispatch_semaphore_create(1);
dispatch_semaphore_t semaphore2 = dispatch_semaphore_create(0);
dispatch_semaphore_t semaphore3 = dispatch_semaphore_create(0);

dispatch_async(queue, ^{
    // 任务1
    dispatch_semaphore_wait(semaphore1, DISPATCH_TIME_FOREVER);
    NSLog(@"1\n");
    dispatch_semaphore_signal(semaphore2);
    dispatch_semaphore_signal(semaphore1);
});

dispatch_async(queue, ^{
    // 任务2
    dispatch_semaphore_wait(semaphore2, DISPATCH_TIME_FOREVER);
    NSLog(@"2\n");
    dispatch_semaphore_signal(semaphore3);
    dispatch_semaphore_signal(semaphore2);
});

dispatch_async(queue, ^{
    // 任务3
    dispatch_semaphore_wait(semaphore3, DISPATCH_TIME_FOREVER);
    NSLog(@"3\n");
    dispatch_semaphore_signal(semaphore3);
});

// b、有限资源访问问题:for循环看似能创建100个异步任务，实质由于信号限制，最多创建10个异步任务。
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_semaphore_t semaphore = dispatch_semaphore_create(10);
for (int i = 0; i < 100; i ++) {
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    dispatch_async(queue, ^{
    // 任务
    ...
    dispatch_semaphore_signal(semaphore);
    });
}
9、dispatch_set_context、dispatch_get_context和dispatch_set_finalizer_f
// dispatch_set_context、dispatch_get_context是为了向队列中传递上下文context服务的。
// dispatch_set_finalizer_f相当于dispatch_object_t的析构函数。
// 因为context的数据不是foundation对象，所以arc不会自动回收，一般在dispatch_set_finalizer_f中手动回收，所以一般讲上述三个方法绑定使用。

- (void)test
{
    // 几种创建context的方式
    // a、用C语言的malloc创建context数据。
    // b、用C++的new创建类对象。
    // c、用Objective-C的对象，但是要用__bridge等关键字转为Core Foundation对象。

    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_CONCURRENT);
    if (queue) {
        // "123"即为传入的context
        dispatch_set_context(queue, "123");
        dispatch_set_finalizer_f(queue, &xigou);
    }
    dispatch_async(queue, ^{
        char *string = dispatch_get_context(queue);
        NSLog(@"%s", string);
    });
}

// 该函数会在dispatch_object_t销毁时调用。
void xigou(void *context)
{
    // 释放context的内存（对应上述abc）

    // a、CFRelease(context);
    // b、free(context);
    // c、delete context;
}
应用场景
dispatch_set_context可以为队列添加上下文数据，但是因为GCD是C语言接口形式的，所以其context参数类型是“void *”。
需使用上述abc三种方式创建context，并且一般结合dispatch_set_finalizer_f使用，回收context内存。
