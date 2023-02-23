> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
> Handler系列：

> [Android体系课--Handler—按方法进行源码解析](url)

> [Android体系课--Handler-Handler面试题](url)
### Handler源码解析
##### 1.构造函数

```java
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;1.传入的当前线程的looper对象
    mQueue = looper.mQueue;2.传入的当前线程的looper对象的MessageQueue
    mCallback = callback;3.传入的Handler.CallBack对象，在处理的时候会判断该对象是否存在还有返回值是否为true
    mAsynchronous = async;
}
```
总结：
	1.哪个线程执行消息处理请求，是根据传入的`looper`来确认。
##### 2.获取Message

```java
Handler.obtainMessage
    Message.obtain(this, what){
            Message m = obtain();分析1：
            m.target = h;
            m.what = what;
            return m;
    }
    分析1：
    public static Message obtain() {
    synchronized (sPoolSync) {
            if (sPool != null) {sPool指向消息池的头节点，如果不为空进入
                Message m = sPool; 使用一个临时变量m=sPool
                sPool = m.next; 让sPool指向m的next
                m.next = null; 打断m到sPool的指针，这样sPool还是指向链表的头结点，只是这个节点是之前sPool的next节点
                m.flags = 0; // clear in-use flag
                sPoolSize--消息池链表大小减1
                return m; 返回之前从消息池中取出的头结点，
            }
        }
        return new Message();如果消息池没有消息，则创建消息
    }
```
总结：Handler.`obtainMessage`方法可以从`Message`的消息池中获取消息，取出的是消息池的`头结点`消息，如果`没有消息则创建消`息，这个方法可以避免不必要的消息创建，`重用消息池的消息`，`减少内存开销`

##### 3.发送sendMessage

```java
sendMessageDelayed(msg, 0);
        sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);延迟时间加上系统时间组成when
                MessageQueue queue = mQueue;这个mQueue是在Handler构造函数中赋值
                enqueueMessage(queue, msg, uptimeMillis);
                        msg.target = this;将当前Handler赋值给msg.target
                        if (mAsynchronous) {如果是异步消息，则设置msg的异步标志
                                msg.setAsynchronous(true);
                        }
                        queue.enqueueMessage(msg, uptimeMillis){调用MessageQueue的enqueueMessage方法
                                if (msg.target == null) {判断handler是否为空
                                        throw new IllegalArgumentException("Message must have a target.");
                                }
                                if (msg.isInUse()) {判断msg是否被使用
                                        throw new IllegalStateException(msg + " This message is already in use.");
                                }
                                synchronized (this) {
                                        if (mQuitting) {判断是否调用了退出
                                                IllegalStateException e = new IllegalStateException(
                                                                msg.target + " sending message to a Handler on a dead thread");
                                                Log.w(TAG, e.getMessage(), e);
                                                msg.recycle();
                                                return false;
                                        }

                                        msg.markInUse();设置msg为使用状态，防止msg被重复使用
                                        msg.when = when;设置msg的延迟时间
                                        Message p = mMessages;获取消息池的头节点赋值给临时变量p
                                        boolean needWake; 是否唤醒looper的next
                                        if (p == null || when == 0 || when < p.when) { 消息池为空或者延迟时间为0或者延迟时间小余头节点的延迟时间，则将其插入消息池的头节点
                                                // New head, wake up the event queue if blocked.
                                                msg.next = p;
                                                mMessages = msg;
                                                needWake = mBlocked; mBlocked是在消息处理next中赋值，如果有消息正在处理则mBlocked=false，如果空闲状态则mBlocked=true，即需要唤醒
                                        } else {
                                                // Inserted within the middle of the queue.  Usually we don't have to wake
                                                // up the event queue unless there is a barrier at the head of the queue
                                                // and the message is the earliest asynchronous message in the queue.
                                                needWake = mBlocked && p.target == null && msg.isAsynchronous();如果是空闲状态且p.target == null和msg是异步消息，则需要唤醒
                                                Message prev;
                                                for (;;) {遍历链表取出当前msg延迟时间小余其延迟时间的msg
                                                        prev = p;
                                                        p = p.next;
                                                        if (p == null || when < p.when) {
                                                                break;
                                                        }
                                                        if (needWake && p.isAsynchronous()) {
                                                                needWake = false;
                                                        }
                                                }
                                                msg.next = p; // invariant: p == prev.next 将msg插入p的前面
                                                prev.next = msg;将msg插入prev后面，即插入prev和p的中间
                                        }

                                        // We can assume mPtr != 0 because mQuitting is false.
                                        if (needWake) {如果需要唤醒，则调用nativeWake唤醒next方法中的nativePoolOnce
                                                nativeWake(mPtr);
                                        }
                                }
                                return true;														
                        }
```
总结：

	1.消息插入机制：插入消息的顺序是延迟时间最小的放在消息池的头部
	2.消息唤醒时机：
		2.1：如果当前消息插入的是头节点，则判断消息处理是不是空闲状态，如果是空闲则唤醒
		2.2：如果消息出入中间节点，则首先判断是不是空闲状态还有`p.target == null`和`msg`是异步消息，这三个条件都成立都可以把needWake置为true，
			 之后还要判断当前消息是不是最早的异步消息，如果不是最早的，则needWake 置为 false即不需要唤醒，如果是最早的异步消息，则直接唤醒消息处理循环

##### 4.消息获取过程：
Looper的`loop`方法：

```java
public static void loop() {
	final Looper me = myLooper();获取当前线程的looper对象
	final MessageQueue queue = me.mQueue;获取MessageQueue对象
	for (;;) {
		Message msg = queue.next(); // might block获取msg
		if (msg == null) {没有消息的时候则退出循环，即主线程退出，应用退出
			// No message indicates that the message queue is quitting.
			return;
		}
		try {
			msg.target.dispatchMessage(msg);处理msg
		}
		msg.recycleUnchecked();回收msg
	}
}
void recycleUnchecked() {回收消息，并将消息放到消息池中前提是消息池没有满
	// Mark the message as in use while it remains in the recycled object pool.
	// Clear out all other details.
	flags = FLAG_IN_USE;
	what = 0;
	arg1 = 0;
	arg2 = 0;
	obj = null;
	replyTo = null;
	sendingUid = -1;
	when = 0;
	target = null;
	callback = null;
	data = null;

	synchronized (sPoolSync) {
		if (sPoolSize < MAX_POOL_SIZE) {
			next = sPool;
			sPool = this;
			sPoolSize++;
		}
	}
}
MessageQueue.java:
Message next() {
	// Return here if the message loop has already quit and been disposed.
	// This can happen if the application tries to restart a looper after quit
	// which is not supported.
	final long ptr = mPtr;
	if (ptr == 0) {
		return null;
	}

	int pendingIdleHandlerCount = -1; // -1 only during first iteration
	int nextPollTimeoutMillis = 0;
	for (;;) {
		if (nextPollTimeoutMillis != 0) {
			Binder.flushPendingCommands();
		}
		在这里休眠，如果有消息并唤醒
		nativePollOnce(ptr, nextPollTimeoutMillis);

		synchronized (this) {
			// Try to retrieve the next message.  Return if found.
			final long now = SystemClock.uptimeMillis();
			Message prevMsg = null;
			Message msg = mMessages;
			if (msg != null && msg.target == null) {同步屏障消息的msg.target == null，循环遍历取出第一个异步消息处理
				// Stalled by a barrier.  Find the next asynchronous message in the queue.
				do {
					prevMsg = msg;
					msg = msg.next;
				} while (msg != null && !msg.isAsynchronous());
			}
			if (msg != null) {msg不为空
				if (now < msg.when) {当前时间小余msg的延迟时间，则等待：msg.when - now
					// Next message is not ready.  Set a timeout to wake up when it is ready.
					nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
				} else {当前时间大于取出的msg的延迟时间
					// Got a message.
					mBlocked = false;将mBlocked空闲时间置为false
					if (prevMsg != null) {
						prevMsg.next = msg.next;
					} else {
						mMessages = msg.next;取出msg后，将msg的next节点置为头节点
					}
					msg.next = null;打断msg到msg next的链表链接
					if (DEBUG) Log.v(TAG, "Returning message: " + msg);
					msg.markInUse();将msg的使用标志置为true
					return msg;返回msg
				}
			} else {msg为空表示没有消息需要处理
				// No more messages.
				nextPollTimeoutMillis = -1;
			}

			// Process the quit message now that all pending messages have been handled.
			if (mQuitting) {当调用了退出方法则返回null给上层
				dispose();内部调用nativeDestroy(mPtr);
				return null;
			}

			// If first time idle, then get the number of idlers to run.
			// Idle handles only run if the queue is empty or if the first message
			// in the queue (possibly a barrier) is due to be handled in the future.
			下面这些信息是处理对于设置了空闲消息处理任务的流程，这个可以用来提高Ui性能，即将主线程空闲状态来处理一些其他事情，充分利用资源
			if (pendingIdleHandlerCount < 0
					&& (mMessages == null || now < mMessages.when)) {
				pendingIdleHandlerCount = mIdleHandlers.size();
			}
			if (pendingIdleHandlerCount <= 0) {
				// No idle handlers to run.  Loop and wait some more.
				mBlocked = true; 如果没有任何空闲状态事情处理后，将mBlocked置为true，表示是真正空闲状态，无任何处理事务包括空闲事务，这个值在消息插入的时候对是否唤醒消息处理有关系
				continue;
			}

			if (mPendingIdleHandlers == null) {
				mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
			}
			mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
		}

		// Run the idle handlers.
		// We only ever reach this code block during the first iteration.
		for (int i = 0; i < pendingIdleHandlerCount; i++) {
			final IdleHandler idler = mPendingIdleHandlers[i];
			mPendingIdleHandlers[i] = null; // release the reference to the handler

			boolean keep = false;
			try {
				keep = idler.queueIdle();
			} catch (Throwable t) {
				Log.wtf(TAG, "IdleHandler threw exception", t);
			}

			if (!keep) {
				synchronized (this) {
					mIdleHandlers.remove(idler);
				}
			}
		}

		// Reset the idle handler count to 0 so we do not run them again.
		pendingIdleHandlerCount = 0;

		// While calling an idle handler, a new message could have been delivered
		// so go back and look again for a pending message without waiting.
		nextPollTimeoutMillis = 0;
	}
}

```
总结：

    消息取出过程：
    首先判断消息头是否是一个同步屏障消息msg.traget=null，
    如果是取出链表中第一个异步消息进行处理，如果不是则直接取出消息池中第一个消息。
    如果没有任何消息需要处理，则判断是否有空闲任务需要处理ideHandler，有就去处理空闲任务，没有就将最终空闲状态置为true

##### 5.消息处理：

```js
Looper.loop方法中：
msg.target.dispatchMessage(msg);msg.target =Handler
Handler.dispatchMessage(msg){
	if (msg.callback != null) {如果msg在创建过程中msg.callback不为null，则直接调用handleCallback(msg)---> message.callback.run();
		handleCallback(msg);
	} else {
		if (mCallback != null) {
			if (mCallback.handleMessage(msg)) {如果Handler在创建的时候传入的Handler.CallBack不为空则调用CallBack的handleMessage方法
				return;如果返回值为true则不会回调Handler的handleMessage，这里可以做一个消息拦截的处理
			}
		}
		handleMessage(msg);调用Handler的handleMessage
	}	
}
```
##### 6.消息循环处理退出：调用Looper的quit方法

```java
void quit(boolean safe) {
	if (!mQuitAllowed) {
		throw new IllegalStateException("Main thread not allowed to quit.");
	}

	synchronized (this) {
		if (mQuitting) {
			return;
		}
		mQuitting = true;将mQuitting标志置为true

		if (safe) {
			removeAllFutureMessagesLocked();待处理消息执行完再清理
		} else {
			removeAllMessagesLocked();直接清理，可能会有内存泄露风险
		}

		// We can assume mPtr != 0 because mQuitting was previously false.
		nativeWake(mPtr);唤醒消息处理线程
	}
}

```
##### 7.View的绘制流程中：View绘制会走到scheduleTraversals中

```java
/**
  *：ViewRootImpl.scheduleTraversals()
  */
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();分析1

            // 通过mHandler.post（）发送一个runnable，在run()方法中去处理绘制流程
            // 与ActivityThread的Handler消息传递机制相似
            // ->>分析7
            mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);分析2
        }
    }
	final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
	void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);分析3

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
	分析1：MessageQueue->postSyncBarrier
	private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();去Message的消息池中获取中获取msg
            msg.markInUse();设置inuse
            msg.when = when;设置延迟时间
            msg.arg1 = token;设置token

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {这个判断内部其实是消息链表mMessages中取出延迟时间比当前msg的延迟时间更大的msg，
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // 这里面其实是把msg插入消息链表mMessages中
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;返回msg的token
        }
    }
```
总结：

> `postSyncBarrier`的作用是去消息池中获取一个msg，设置了`msg.arg1 = token`是同步屏障消息的token值，且`msg.target` = null;并将这个`msg`放入到消息池`mMessages`中,下次处理线程被唤醒时会判断消息池的`第一个msg的target是不是空`，`并去后面取第一个异步任务`，这个异步任务其实是一个`view的绘制流程`。`同步屏障实现了view优先绘制`


```java
分析2：mChoreographer.postCallback
    postCallbackDelayed(callbackType, action, token, 0);
            postCallbackDelayedInternal(callbackType, action, token, delayMillis);{
                    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
            scheduleFrameLocked(now);//分析8
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);获取一个msg
            msg.arg1 = callbackType;设置arg1参数类型
            msg.setAsynchronous(true);设置为异步消息
            mHandler.sendMessageAtTime(msg, dueTime);发送消息
        }
    }
    分析8：scheduleFrameLocked(now);
    private void scheduleFrameLocked(long now) {
            ...
            scheduleVsyncLocked();
            ...
    }
    private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();//这个mDisplayEventReceiver = FrameDisplayEventReceiver对象
}
    public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);//这里调用nativeScheduleVsync注册了一个Vsync事件接收器，接收者为前面的mDisplayEventReceiver
    }
}
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver{
            @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
                    Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);//这里发送了一个异步信号，每16ms接收到一次信号，并绘制ui
            }

    }
	
```
总结：
>
> `postCallback`内部主要实现的是`获取一个msg`，并设置`msg为异步消息`，最后发送消息给`MessageQueue`	
> 注册了`vsync`信号回调，每`16ms`获取到`vsync`信号，并更新`ui`，所以`ondraw`方法是在接收到vsync信号后才调用的，会每`16ms`回调一次`ondraw`方法


```java
分析3：mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
	public void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {其实是取出target==null且p.arg1=传入的token的值，即之前插入消息链表mMessages的msg
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
			
            if (prev != null) {这个if是将p在链表中去除，prev不为null说明p不在表头
                prev.next = p.next;
                needWake = false;
            } else {为null说明p在表头。
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;如果mMessages == null || mMessages.target != null;表头数据不是同步屏障消息，或者消息池数据为空，则唤醒消息处理线程
            }
            p.recycleUnchecked();回收消息到消息池sPool中

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
```
总结：根据`token`值移除消息链表中的`msg`并根据情况`唤醒消息处理线程`
> 由`分析1和分析2，3可知`：`View的绘制流程`其实就是在View绘制流程启动前，给消息池`发送一个msg.target为空的消息`，然后给`View的绘制任务的msg设置为异步消息`,下次在`Handler`取消息的过程中`优先判断消息池的msg.target`是不是空，如果是，则去消息池中`取出第一个异步消息执行`。执行前`先把同步屏障消息移除`。这就是`消息同步屏障机制`

##### 7.ThreadLocal机制：sThreadLocal.set(new Looper(quitAllowed));

```java
public void set(T value) {
        //(1)获取当前线程（调用者线程）
        Thread t = Thread.currentThread();
        //(2)以当前线程作为key值，去查找对应的线程变量，找到对应的map
        ThreadLocalMap map = getMap(t);
        //(3)如果map不为null，就直接添加本地变量，key为当前定义的ThreadLocal变量的this引用，值为添加的本地变量值
        if (map != null)
                map.set(this, value);
        //(4)如果map为null，说明首次添加，需要首先创建出对应的map
        else
                createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
        return t.threadLocals; //获取线程自己的变量threadLocals，并绑定到当前调用线程的成员变量threadLocals上
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}


public T get() {
        //(1)获取当前线程
        Thread t = Thread.currentThread();
        //(2)获取当前线程的threadLocals变量
        ThreadLocalMap map = getMap(t);
        //(3)如果threadLocals变量不为null，就可以在map中查找到本地变量的值
        if (map != null) {
                ThreadLocalMap.Entry e = map.getEntry(this);
                if (e != null) {
                        @SuppressWarnings("unchecked")
                        T result = (T)e.value;
                        return result;
                }
        }
        //(4)执行到此处，threadLocals为null，调用该更改初始化当前线程的threadLocals变量
        return setInitialValue();
}

private T setInitialValue() {
        //protected T initialValue() {return null;}
        T value = initialValue();
        //获取当前线程
        Thread t = Thread.currentThread();
        //以当前线程作为key值，去查找对应的线程变量，找到对应的map
        ThreadLocalMap map = getMap(t);
        //如果map不为null，就直接添加本地变量，key为当前线程，值为添加的本地变量值
        if (map != null)
                map.set(this, value);
        //如果map为null，说明首次添加，需要首先创建出对应的map
        else
                createMap(t, value);
        return value;
}
```
总结：

> `ThreadLocal`其实是一种用空间换时间的机制：
> 	`ThreadLocal`内部的其实都是针对当前线程的`ThreadLocalMap`做的操作，一个线程只有一个`Thread`，一个`Thread`只有一个`ThreadLocalMap`，所以其内部存储的数据都是线程隔离的。
> 而且在
> `static final ThreadLocal<Looper> sThreadLocal = newThreadLocal<Looper>()`;
> 	可以看到这个`sThreadLocal`在所有线程中只有一个，所以获取`value`的时候`key``都是同一个`，只是这个`ThreadLocalMap`是在`每个线程中有一份`，所以获取的值是不同线程中的`value`值
> 	`sThreadLocal`同一个对象中，相当于一个统一入口，内部操作获取`value`的和设置`value`都是`针对当前线程来操作`的，`所以在不用线程中获取的是当前线程的值`