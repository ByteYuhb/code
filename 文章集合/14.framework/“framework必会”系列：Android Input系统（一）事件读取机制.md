> 🔥 **Hi，我是小余。本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 浅谈

曾经在开发的很长一段时间内，笔者对点击事件的认知只存在于自定义View中的`onTouchEvent`等方法的处理。 后来慢慢的接触到`Android的事件分发机制`，但也只是在**Activity->ViewGroup->View**层面的分发逻辑。

**诚然在我们开发中也仅需要搞懂这个层面就够我们平时所用了**。

但笔者脑海里一直有个声音在问我：这些事件是怎么来的，根源在哪里？ 秉着追根溯源的精神，踏上了慢慢源码路。。。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77b9252b57964c9c9efc3dc40efe2ad4~tplv-k3u1fbpfcp-zoom-1.image)

**特别说明**:文章中部分知识或者图片摘自其他网络文章，如有侵权，可以联系博主进行删除。

## Android输入系统

说到屏幕点击事件，大部分同学最早想到的就是自定义中点击事件的处理过程，**屏幕滑动，Down事件，Up事件**等。

是的，笔者也是如此。

自定义View的事件处理其实在整个Android输入系统中只能算是最上层的。 输入系统如果按结构分层可以分为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1aaee5d36644f9f8a0e1769144aa0b3~tplv-k3u1fbpfcp-zoom-1.image)

-   **输入系统部分**

    包含输入子系统以及InputManagerService，用于事件的捕获以及分发给上一级

-   **WindowManagerService处理部分**

    输入系统部分将事件分发给对应的Window，而Window正是由WMS来管理的

-   **View处理部分**

    这里就是前面我们说的Activity->ViewGroup->View层面的逻辑

这里再提供一张更加详细的Android输入系统模型：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57b25c9c817a429786e21a83e53077f2~tplv-k3u1fbpfcp-zoom-1.image)

这张图涵盖了Android整个输入系统事件传递的大致模型，非常具有参考意义，本系列文章就是根据这张图展开的 [图片来自](https://blog.csdn.net/a734474820/article/details/125613810?spm=1001.2014.3001.5502)


输入系统说白了就是捕获事件，并将事件分发给WMS进行处理。关键字：**事件捕获，事件分发** 那系统是如何进行事件捕获和分发的呢？

输入系统在结构上又可以分为：

-   **输入子系统**
-   **InputManagerService（简称IMS）**

### 输入子系统

Android中的输入设备有很多种，如：**键盘，屏幕，鼠标**等，开发中最常见的就是屏幕和按键（如Home键等属于键盘）了。

这些设备对于核心处理器来说就是一个“即插即用”外设，和我们电脑插入鼠标或手机后，在“设备管理器”里面会新增一个输入设备节点一样（**前提是已经安装好对应的驱动**） ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af291cc2f80d462e9e22de64f5120f92~tplv-k3u1fbpfcp-zoom-1.image)

Android处理器在接入这些“外设”后，比如滑动屏幕，**设备驱动层就会接受到原始事件最终将事件传递到用户空间的设备节点（dev/input/）中**。

**Android提供了一些api可以让开发者在设备节点（dev/input/）中读取内核写入的事件**。

### IMS

**IMS的作用**：读取设备节点（dev/input/）中的输入事件，并对数据进行二次甚至三次加工后分发给合适的Window进行处理。

而我们**今天要介绍的就是关于底层输入系统IMS的处理部分**，关于WMS处理以及View处理部分，后续会出一些文章讲解。

文章将分两个阶段来对输入系统介绍

-   **1.输入事件获取**
-   **2.输入事件分发**

## 事件获取流程

我们以IMS为入口。在设备开机过程中，启动SystemServer部分初始了很多系统服务，这里面就包括了IMS的创建和启动过程。 注：**文章使用的源码版本为8.0**：

### 1.IMS的创建与启动

在SystemServer的run方法中调用了startOtherServices。

```
private void startOtherServices() {
    ...
    inputManager = new InputManagerService(context);//1
    ...
    wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore, new PhoneWindowManager());//2
    ...
    inputManager.start();//3
}
```

注释1：调用了IMS的构造方法 注释2：将IMS作为参数传递给WMS 注释3：调用了IMS的启动方法start

**我们来具体分析下注释1和注释3**： 注释1：

```
public InputManagerService(Context context) {
    ...
    mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
    ...
}
```

重点看nativeInit方法，这是一个native方法

```
//frameworks\base\services\core\jni\com_android_server_input_InputManagerService.cpp
static jlong nativeInit(JNIEnv* env, jclass /* clazz */,
        jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
    ...
    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
            messageQueue->getLooper());
    im->incStrong(0);
    return reinterpret_cast<jlong>(im);
}
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
    JNIEnv* env = jniEnv();

    mContextObj = env->NewGlobalRef(contextObj);//1
    mServiceObj = env->NewGlobalRef(serviceObj);//2
    ...
    sp<EventHub> eventHub = new EventHub();//3
    mInputManager = new InputManager(eventHub, this, this);//4
}
```

nativeInit方法中创建了一个NativeInputManager对象，并将该对象指针返回给了java framework层。 这就是为了打通java和native层，下次需要使用native层的NativeInputManager对象的时候，直接传递这个指针就可以访问了。

继续看NativeInputManager构造方法：

-   **注释**1.将java层的传下来的Context上下文保存在mContextObj中
-   **注释**2：将java层传递下来的InputManagerService对象保存在mServiceObj中。

如果你对源码比较熟悉，可以知道大部分native层和java层的交互都是通过这个模式

`调用模型图`如下:

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f14282c9dfc453fb3ee6e9df05b804a~tplv-k3u1fbpfcp-zoom-1.image)

**NativeInputManager构造方法注释3处**：创建一个EventHub，EventHub通过Linux内核的INotify和epoll机制监听设备节点，**使用它的getEvent函数就可以读取设备节点的原始事件以及设备节点增删事件**。 这里说明下原始事件和设备增删事件：

-   **原始事件**：比如键盘节点的输入事件或者屏幕的触屏DOWN或者UP事件等，都属于原始事件。
-   **设备增删事件**：值键盘或者屏幕等节点的增删，EventHub在获取这类事件后，会在native层创建对应的节点处理Mapper对象。

**NativeInputManager构造方法注释4处**：创建一个InputManager对象并将注释3处的EventHub对象作为参数传入。 进入InputManager构造方法：

```
InputManager::InputManager(
        const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    mDispatcher = new InputDispatcher(dispatcherPolicy);//1
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);//2
    initialize();//3
}
void InputManager::initialize() {
    mReaderThread = new InputReaderThread(mReader);
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}

```

InputManager构造方法中：

-   1.创建InputDispatcher类对象mDispatcher，InputDispatcher类主要用来对原始事件进行分发，传递给WMS
-   2.创建InputReader类对象mReader，并传入1中mDispatcher对象以及eventHub对象， 为什么要传入这2个对象呢？因为InputReader机制就是：eventHub对象用来读取事件数据，mDispatcher对象用来将读取到事件数据分发。
-   3.initialize方法中创建了InputReaderThread对象和InputDispatcherThread对象 因为事件读取机制是一个耗时过程，不能在主线程中进行，所以使用InputReaderThread线程来读取事件，用InputDispatcherThread线程来分发事件

**关于IMS的构造方法就讲了这么多，先来小结下：**

-   1.**IMS构造方法中**：创建了一个NativeInputManager的native对象，并将java层的Context上下文保存在native层的mContextObj，将java层的IMS对象保存在native层的mServiceObj中 创建InputManager对象并传入一个新建的EventHub对象，用于读取设备节点事件。
-   2.**InputManager构造方法中**：创建了一个InputDispatcher和InputReader对象，以及用于读取事件的InputReaderThread线程和分发事件的InputDispatcherThread线程

下面我们继续看IMS的启动方法，在startOtherServices方法的调用inputManager.start

```
private void startOtherServices() {
    ...
    inputManager = new InputManagerService(context);//1
    ...
    wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore, new PhoneWindowManager());//2
    ...
    inputManager.start();//3
}
```

start方法中主要调用了nativeStart方法，参数为初始化时，native返回的NativeInputManager对象地址

```
static void nativeStart(JNIEnv* env, jclass /* clazz */, jlong ptr) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    status_t result = im->getInputManager()->start();
    if (result) {
        jniThrowRuntimeException(env, "Input manager could not be started.");
    }

}
```

nativeStart方法调用了NativeInputManager的InputManager的start方法

```
status_t InputManager::start() {
    status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
    ...
    result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
    if (result) {
        ALOGE("Could not start InputReader thread due to error %d.", result);

        mDispatcherThread->requestExit();
        return result;
    }
    
    return OK;

}
```

start方法主要作用就是启动初始化时创建的两个线程：mDispatcherThread和mReaderThread 这里注意先后顺序先启动事件分发线程，再启动事件读取线程。这是为了在事件读取后可以立即对事件进行分发。

**IMS启动时序图如下：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1e9ab3f010b48b497a2c0a2bcbcfaff~tplv-k3u1fbpfcp-zoom-1.image)

### 2.线程的创建与启动

在分析两个线程启动过程之前，我们先来讲解下Thread的run方法

```
system\core\libutils\Threads.h
status_t Thread::run(const char* name, int32_t priority, size_t stack)
{
    ...
    res = createThreadEtc(_threadLoop,
                this, name, priority, stack, &mThread);
    ...
}
Thread的run方法中调用了createThreadEtc，这个方法第一个参数_
```

threadLoop是一个方法指针，第二个参数是自己,最终会调用到_threadLoop方法并传入this指针、

```
int Thread::_threadLoop(void* user)
    Thread* const self = static_cast<Thread*>(user);
    ...
    do {
        bool result;
        result = self->threadLoop();
        ...
        if (result == false || self->mExitPending) {
            ...
            break;
        }
        ...
    } while(strong != 0);
}
```

_threadLoop方法内部会调用self->threadLoop()，这个self就是当前Thread的this指针，除了result返回false或者调用了requestExit才会退出。 不然会一直循环调用self->threadLoop()函数，threadLoop在Thread中是一个纯虚函数，在其子类中实现。

所以后面只要分析子线程的threadLoop方法即可。

好，下面我们先来分析InputDispatcherThread：

```
bool InputDispatcherThread::threadLoop() {
    mDispatcher->dispatchOnce();
    return true;
}
```

这里的mDispatcher是InputDispatcher类对象。

```
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { // acquire lock
        ..
        if (!haveCommandsLocked()) {//1
            dispatchOnceInnerLocked(&nextWakeupTime);//2
        }
        

        if (runCommandsLockedInterruptible()) {//3
            nextWakeupTime = LONG_LONG_MIN;//4
        }
    } // release lock
    ...
    mLooper->pollOnce(timeoutMillis);//5

}
```

注释1：判断有没有命令需要执行，如果没有命令，则调用注释2的dispatchOnceInnerLocked 如果有命令，则在注释3处执行完所有命令后，将nextWakeupTime置为LONG_LONG_MIN，这样就可以让下一次线程可以被立即唤醒。 注释5处调用mLooper->pollOnce，让线程进入休眠。第一次启动的时候是直接进入休眠，等待事件的到来。

**InputReader/InputReaderThread**

下面再来分析事件读取线程InputReaderThread：

```
bool InputReaderThread::threadLoop() {
    mReader->loopOnce();
    return true;
}
```

这里mReader是InputReader类对象：

```
void InputReader::loopOnce() {
    ...
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);//1

    { // acquire lock
        ...
        if (count) {
            processEventsLocked(mEventBuffer, count);//2
        }
        ...
    } // release lock
    
    ...
    mQueuedListener->flush();//3

}
```
### 3.事件的获取与赋值
注释1处调用EventHub的getEvents方法读取设备节点中的输入事件，

注意这个方法内部在没有输入事件的时候也是一个休眠的过程，并不是死循环耗时操作。

在注释2处如果count不为0，说明有事件，调用processEventsLocked方法。 
### 4.分件的处理与加工
进入processEventsLocked看看：

```
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        //1 原始事件类型
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            ...
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {//2.设备节点事件
            switch (rawEvent->type) {
            case EventHubInterface::DEVICE_ADDED:
                addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::DEVICE_REMOVED:
                removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::FINISHED_DEVICE_SCAN:
                handleConfigurationChangedLocked(rawEvent->when);
                break;
            default:
                ALOG_ASSERT(false); // can't happen
                break;
            }
        }
        count -= batchSize;
    }
}
```

processEventsLocked方法主要实现了：根据事件的type类型进行不同处理。

-   1.**原始事件**：调用processEventsForDeviceLocked处理
-   2.**设备节点事件**：调用节点的添加和删除等操作。

我们先来看原始事件：processEventsForDeviceLocked

```
void InputReader::processEventsForDeviceLocked(int32_t deviceId,
        const RawEvent* rawEvents, size_t count) {
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    ...
    InputDevice* device = mDevices.valueAt(deviceIndex);
    ...
    device->process(rawEvents, count);
}
```

根据deviceId去获取设备在mDevices中的索引，根据索引获取InputDevice对象 调用InputDevice的process方法继续处理。

```
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    ...
    size_t numMappers = mMappers.size();
    for (const RawEvent* rawEvent = rawEvents; count--; rawEvent++) {
        ...
            ...
            for (size_t i = 0; i < numMappers; i++) {
                InputMapper* mapper = mMappers[i];
                mapper->process(rawEvent);
            }
            ...
        ... 
    }   
}
```

process方法最终是使用不同的InputMapper进行处理， 那这个InputMapper在哪里设置成员呢。

我们回到前面processEventsLocked方法：如果是节点处理事件，如添加则调用addDeviceLocked方法, addDeviceLocked方法中又调用了createDeviceLocked方法，并将返回的InputDevice放入到mDevices列表中。

```
void InputReader::addDeviceLocked(nsecs_t when, int32_t deviceId) {
    ...
    InputDevice* device = createDeviceLocked(deviceId, controllerNumber, identifier, classes);
    ...
    mDevices.add(deviceId, device);
    ...
}
InputDevice* InputReader::createDeviceLocked(int32_t deviceId, int32_t controllerNumber,
        const InputDeviceIdentifier& identifier, uint32_t classes) {
    InputDevice* device = new InputDevice(&mContext, deviceId, bumpGenerationLocked(),
            controllerNumber, identifier, classes);

    ...
    // Switch-like devices.
    if (classes & INPUT_DEVICE_CLASS_SWITCH) {
        device->addMapper(new SwitchInputMapper(device));
    }
    
    // Scroll wheel-like devices.
    if (classes & INPUT_DEVICE_CLASS_ROTARY_ENCODER) {
        device->addMapper(new RotaryEncoderInputMapper(device));
    }
    ...
    // Keyboard-like devices.
    ...
    if (keyboardSource != 0) {
        device->addMapper(new KeyboardInputMapper(device, keyboardSource, keyboardType));
    }
    ...
    // Touchscreens and touchpad devices.
    if (classes & INPUT_DEVICE_CLASS_TOUCH_MT) {
        device->addMapper(new MultiTouchInputMapper(device));
    } else if (classes & INPUT_DEVICE_CLASS_TOUCH) {
        device->addMapper(new SingleTouchInputMapper(device));
    }
    ...
    return device;

}
```

可以看到createDeviceLocked根据不同输入类型给Device设备添加了不同的InputMapper。

回到前面InputDevice::process方法中： 这里用KeyboardInputMapper来做例子。 这个方法调用了KeyboardInputMapper的process方法

```
void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent->type) {
        case EV_KEY: {
            ..
            if (isKeyboardOrGamepadKey(scanCode)) {
                processKey(rawEvent->when, rawEvent->value != 0, scanCode, usageCode);
            }
            break;
        }
    }
}
void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t scanCode,
        int32_t usageCode) {
    

    ...
    NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
    getListener()->notifyKey(&args);

}
```

最终在processKey方法中将原始事件封装为一个新的NotifyKeyArgs对象，并调用 getListener()->notifyKey方法作为参数传入,这里getListener是在InputReader构造方法中初始化

```
InputReader::InputReader(const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& policy,
        const sp<InputListenerInterface>& listener) :
        mContext(this), mEventHub(eventHub), mPolicy(policy),
        mGlobalMetaState(0), mGeneration(1),
        mDisableVirtualKeysTimeout(LLONG_MIN), mNextTimeout(LLONG_MAX),
        mConfigurationChangesToRefresh(0) {
    mQueuedListener = new QueuedInputListener(listener);
    ...
}
```

就是这个mQueuedListener，而这个listener是外部传入的InputDispatcher对象。

那进入QueuedInputListener的notifyKey看看：

```
void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
    mArgsQueue.push(new NotifyKeyArgs(*args));
}
```

这里只是将NotifyKeyArgs事件对象存储到mArgsQueue队列中，并没有真正对事件进行处理，那什么时候处理呢？ 观察仔细的同学应该看到在InputReader的loopOnce中调用了mQueuedListener->flush()

```
void InputReader::loopOnce() {
    ...
    mQueuedListener->flush();//3
}
void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i < count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        args->notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}
```

这个flush方法就是循环调用列表中的事件，并依次执行NotifyArgs的notify方法传入的参数mInnerListener为初始化时的InputDispatcher对象
### 5.事件的分发与通知
继续进入NotifyArgs的notify方法：

```
struct NotifyArgs {
    virtual void notify(const sp<InputListenerInterface>& listener) const = 0;
}
```

notify方法是一个纯虚函数，由其子类实现。这里子类就是NotifyKeyArgs对象。

```
void NotifyKeyArgs::notify(const sp<InputListenerInterface>& listener) const {
    listener->notifyKey(this);
}
```

这里listener是InputDispatcher的对象。

**InputDispatcher/InputDispatcherThread**

```
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    ...
    bool needWake;
    { // acquire lock
        ...
        KeyEntry* newEntry = new KeyEntry(args->eventTime,
                args->deviceId, args->source, policyFlags,
                args->action, flags, keyCode, args->scanCode,
                metaState, repeatCount, args->downTime);//1
    
        needWake = enqueueInboundEventLocked(newEntry); //2  
    } // release lock
    
    if (needWake) {
        mLooper->wake();//3
    }

}
```

-   **注释1处**：将NotifyKeyArgs事件重新封装为一个KeyEntry对象。
-   **注释2处**：enqueueInboundEventLocked将KeyEntry对象压入mInboundQueue中。
-   **注释3处**：唤醒InputDispatcherThread线程进行处理。

到这里InputReader的输入事件读取流程已经全部走完。

**事件读取时序图如下**：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a083a2323f4146e09d4e6236d61b5432~tplv-k3u1fbpfcp-zoom-1.image)

### 6.事件获取流程小结
**这里对输入事件的获取流程做个小结：**

-   1.SystemServer创建并启动InputManagerService
-   2.InputManagerService在native层创建一个NativeInputManager对象
-   3.NativeInputManager内部创建一个InputManager对象
-   4.InputManager启动InputReaderThread和InputDispatcherThread
-   5.在InputReaderThread线程中调用EventHub的getEvents获取设备节点中的输入事件。
-   6.并将输入事件封装为NotifyKeyArgs对象放入队列中。
-   7.之后再调用flush，依次将事件传递给InputDispatcher。
-   8.InputDispatcher在收到事件后，会重新封装为一个KeyEntry对象，并压力压入mInboundQueue列表中。
-   9.最后唤醒InputDispatcherThread线程。

## 总结
篇幅问题，为了防止大家迷失在源码中，此篇就讲到这里，**关于事件的分发过程将在下一篇中进行讲解**。

**如果笔者文章对你有帮助，希望您可以帮忙点个赞，关注下，这是对我最大的鼓励。**

笔者公众号：***小余的自习室***

**参考**

[Android输入系统（一）输入事件传递流程和InputManagerService的诞生](http://liuwangshu.cn/framework/ims/1-ims-produce.html)

[Android Input系统(一) 事件的获取流程](https://www.jianshu.com/p/ad476f199e39)

***本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")***