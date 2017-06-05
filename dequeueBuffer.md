# dequeueBuffer
```sequence
Title : dequeueBuffer 时序图
Note right of Surface : 从OpenGL调用过来
Surface -> Surface : hook_dequeueBuffer()
Surface -> Surface : dequeueBuffer()
Surface -> BpGraphicBufferProducer : dequeueBuffer()
BpGraphicBufferProducer -> BnGraphicBufferProducer : onTransact()
Note right of BnGraphicBufferProducer : case DEQUEUE_BUFFER
BnGraphicBufferProducer -> BufferQueueProducer : dequeueBuffer()
Surface -> BpGraphicBufferProducer : requestBuffer()
BpGraphicBufferProducer -> BnGraphicBufferProducer : onTransact()
Note right of BnGraphicBufferProducer : case REQUEST_BUFFER
BnGraphicBufferProducer -> BufferQueueProducer : requestBuffer()
```
# 几个重要的变量
* `mCore` 是一个`BufferQueueCore对象指针`，在`BufferQueueProducer.h`中定义
    * `sp<BufferQueueCore> mCore;`
* `mAllowAllocation` 表示是否允许分配新的缓冲区，定义在`BufferQueueCore`中
* `mIsAllocating` 表示生产者目前是否正在尝试分配缓冲区
* `mIsAbandoned` 表示不再用于使用的图像缓冲区，定义在`BufferQueueCore` 中
* `mConnectedApi` 表示当前连接到BufferQueue上的生产者api
* `mActiveBuffers` 包含所有的不是Free状态的slot
* `mMaxDequeuedBufferCount` Producers一次可以出列（dequeue）的缓冲区数，默认为1，可以通过`setMaxDequeuedBufferCount()`进行修改
* `mFreeBuffers` 一个包括所有的状态为free的且有一个Buffer的slot
* `mFreeSlots`列表包含所有的状态为Free的且当前不含有Buffer的slot


## `BufferQueueProducer::dequeueBuffer()` 函数
1. 参数判断
2. 获取BufferSlot
3. 创建GraphicBuffer
```
status_t BufferQueueProducer::dequeueBuffer(int *outSlot,
        sp<android::Fence> *outFence, uint32_t width, uint32_t height,
        PixelFormat format, uint32_t usage) {
    ATRACE_CALL();
    {
        Mutex::Autolock lock(mCore->mMutex);
        mConsumerName = mCore->mConsumerName;
        if (mCore->mIsAbandoned) {
            return NO_INIT;
        }
        if (mCore->mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
            return NO_INIT;
        }
    }

    // 如果传入的宽和高中有一个为0，另一个不为0。返回错误
    if ((width && !height) || (!width && height)) {
        return BAD_VALUE;
    }

    status_t returnFlags = NO_ERROR;
    EGLDisplay eglDisplay = EGL_NO_DISPLAY;
    EGLSyncKHR eglFence = EGL_NO_SYNC_KHR;
    bool attachedByConsumer = false;

    {
        Mutex::Autolock lock(mCore->mMutex);
        // 调用BufferQueueCore的waitWhileAllocatingLocked()函数，判断是否正在分配Buffer，如果正在分配就block
        mCore->waitWhileAllocatingLocked();

        if (format == 0) {
            format = mCore->mDefaultBufferFormat;
        }
        usage |= mCore->mConsumerUsageBits;

        // 如果宽和高两个都为0，则useDefaultSize为true，使用默认的宽高
        const bool useDefaultSize = !width && !height;
        if (useDefaultSize) {
            width = mCore->mDefaultWidth;
            height = mCore->mDefaultHeight;
        }
        //创建一个slot的index
        int found = BufferItem::INVALID_BUFFER_SLOT;

        //如果found为错误，就一直循环
        while (found == BufferItem::INVALID_BUFFER_SLOT) {
            // 调用waitForFreeSlotThenRelock()函数，获取一个Free状态的BufferSlot
            status_t status = waitForFreeSlotThenRelock(FreeSlotCaller::Dequeue, &found);
            // 如果有错误返回错误状态
            if (status != NO_ERROR) {
                return status;
            }
            if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
                return -EBUSY;
            }
            // 到这里found已经被成功赋值
            const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);
            // mAllowAllocation定义在BufferQueueCore.h中，表示是否允许分配新的缓冲区
            // 如果不允许分配新的缓冲区，但申请出来的GraphicBuffer需要重新分配，继续
            if (!mCore->mAllowAllocation) {
                // 调用needsReallocation() 函数，如果需要重新分配
                if (buffer->needsReallocation(width, height, format, usage)) {
                    if (mCore->mSharedBufferSlot == found) {
                        return BAD_VALUE;
                    }
                    mCore->mFreeSlots.insert(found);
                    mCore->clearBufferSlotLocked(found);
                    found = BufferItem::INVALID_BUFFER_SLOT;
                    continue;
                }
            }
        }

        //存储获得的BufferSlot的GraphicBuffer
        const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);
        // 如果获得的BufferSlot是一个共享缓冲区且需要重新分配，返回错误
        if (mCore->mSharedBufferSlot == found &&
                buffer->needsReallocation(width,  height, format, usage)) {
            return BAD_VALUE;
        }

        // 如果不是共享缓冲区，将该BufferSlot加入到mActiveBuffers列表中
        if (mCore->mSharedBufferSlot != found) {
            mCore->mActiveBuffers.insert(found);
        }
        *outSlot = found;
        ATRACE_BUFFER_INDEX(found);

        attachedByConsumer = mSlots[found].mNeedsReallocation;
        mSlots[found].mNeedsReallocation = false;
        // 设置该BufferSlot的状态为DEQUEUED
        mSlots[found].mBufferState.dequeue();

        // 设置BufferSlot，如果GraphicBuffer为空或者需要重新申请
        if ((buffer == NULL) ||
                buffer->needsReallocation(width, height, format, usage))
        {
            mSlots[found].mAcquireCalled = false;
            mSlots[found].mGraphicBuffer = NULL;
            mSlots[found].mRequestBufferCalled = false;
            mSlots[found].mEglDisplay = EGL_NO_DISPLAY;
            mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
            mSlots[found].mFence = Fence::NO_FENCE;
            mCore->mBufferAge = 0;
            mCore->mIsAllocating = true;
            // 添加需要重新申请的flag
            returnFlags |= BUFFER_NEEDS_REALLOCATION;
        } else {
            mCore->mBufferAge =
                    mCore->mFrameCounter + 1 - mSlots[found].mFrameNumber;
        }

        eglDisplay = mSlots[found].mEglDisplay;
        eglFence = mSlots[found].mEglFence;
        *outFence = (mCore->mSharedBufferMode &&
                mCore->mSharedBufferSlot == found) ?
                Fence::NO_FENCE : mSlots[found].mFence;
        mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
        mSlots[found].mFence = Fence::NO_FENCE;

        if (mCore->mSharedBufferMode && mCore->mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore->mSharedBufferSlot = found;
            mSlots[found].mBufferState.mShared = true;
        }
    }

    // 如果带有需要重新申请的flag
    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        status_t error;
        // 调用BufferQueueCore的mAllocator指向的对象的createGraphicBuffer()函数，创建Buffer
        sp<GraphicBuffer> graphicBuffer(mCore->mAllocator->createGraphicBuffer(
                width, height, format, usage,
                {mConsumerName.string(), mConsumerName.size()}, &error));
        {
            Mutex::Autolock lock(mCore->mMutex);
            if (graphicBuffer != NULL && !mCore->mIsAbandoned) {
                graphicBuffer->setGenerationNumber(mCore->mGenerationNumber);
                // 将GraphicBuffer设置给BufferSlot
                mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
            }
            mCore->mIsAllocating = false;
            mCore->mIsAllocatingCondition.broadcast();
            if (graphicBuffer == NULL) {
                return error;
            }
            if (mCore->mIsAbandoned) {
                return NO_INIT;
            }
            VALIDATE_CONSISTENCY();
        }
    }

    if (attachedByConsumer) {
        returnFlags |= BUFFER_NEEDS_REALLOCATION;
    }

    if (eglFence != EGL_NO_SYNC_KHR) {
        EGLint result = eglClientWaitSyncKHR(eglDisplay, eglFence, 0,
                1000000000);
        eglDestroySyncKHR(eglDisplay, eglFence);
    }

    return returnFlags;
}
```

## `BufferQueueCore::waitWhileAllocatingLocked()` 函数
该方法会一直block，知道`mIsAllocating`为false，`mIsAllocating`表示正在分配Buffer
```
void BufferQueueCore::waitWhileAllocatingLocked() const {
    ATRACE_CALL();
    while (mIsAllocating) {
        mIsAllocatingCondition.wait(mMutex);
    }
}
```

## `BufferQueueProducer::waitForFreeSlotThenRelock()` 函数
获取一个状态为FREE的BufferSlot
* 如果是共享缓冲区模式或存在共享缓冲区，返回共享缓冲区
* 否则从`mFreeBuffers`或`mFreeSlots`列表中找一个
```
status_t BufferQueueProducer::waitForFreeSlotThenRelock(FreeSlotCaller caller,
        int* found) const {
    auto callerString = (caller == FreeSlotCaller::Dequeue) ?
            "dequeueBuffer" : "attachBuffer";
    bool tryAgain = true;
    while (tryAgain) {
        if (mCore->mIsAbandoned) {
            return NO_INIT;
        }

        // 统计所有的slot分别为DEQUEUEED和ACQUIRED状态的个数
        int dequeuedCount = 0;
        int acquiredCount = 0;
        for (int s : mCore->mActiveBuffers) {
            if (mSlots[s].mBufferState.isDequeued()) {
                ++dequeuedCount;
            }
            if (mSlots[s].mBufferState.isAcquired()) {
                ++acquiredCount;
            }
        }

        // 检查，dequeuedCount不能超过队列的最大值，直到有缓冲区出列
        if (mCore->mBufferHasBeenQueued &&
                dequeuedCount >= mCore->mMaxDequeuedBufferCount) {
            return INVALID_OPERATION;
        }

        *found = BufferQueueCore::INVALID_BUFFER_SLOT;

        // 如果Buffer的个数太多，超过队列允许的个数会造成OOM
        const int maxBufferCount = mCore->getMaxBufferCountLocked();
        bool tooManyBuffers = mCore->mQueue.size()
                            > static_cast<size_t>(maxBufferCount);
        if (tooManyBuffers) {
        } else {
            // 如果在共享缓冲区模式或存在共享缓冲区，返回共享缓冲区
            if (mCore->mSharedBufferMode && mCore->mSharedBufferSlot !=
                    BufferQueueCore::INVALID_BUFFER_SLOT) {
                *found = mCore->mSharedBufferSlot;
            } else {
                if (caller == FreeSlotCaller::Dequeue) { // 如果是dequeue操作
                    // 调用getFreeBufferLocked()函数
                    int slot = getFreeBufferLocked();
                    if (slot != BufferQueueCore::INVALID_BUFFER_SLOT) {
                        *found = slot;
                    } else if (mCore->mAllowAllocation) {
                        // 调用getFreeSlotLocked()函数
                        *found = getFreeSlotLocked();
                    }
                } else { // 如果是attach操作
                    int slot = getFreeSlotLocked();
                    if (slot != BufferQueueCore::INVALID_BUFFER_SLOT) {
                        *found = slot;
                    } else {
                        *found = getFreeBufferLocked();
                    }
                }
            }
        }

        // 如果没有找到缓冲区，或者队列中有很多缓冲区未完成，请等待获取或释放缓冲区，或者更改最大缓冲区数。
        tryAgain = (*found == BufferQueueCore::INVALID_BUFFER_SLOT) ||
                   tooManyBuffers;
        if (tryAgain) {
            if ((mCore->mDequeueBufferCannotBlock || mCore->mAsyncMode) &&
                    (acquiredCount <= mCore->mMaxAcquiredBufferCount)) {
                return WOULD_BLOCK;
            }
            if (mDequeueTimeout >= 0) {
                status_t result = mCore->mDequeueCondition.waitRelative(
                        mCore->mMutex, mDequeueTimeout);
                if (result == TIMED_OUT) {
                    return result;
                }
            } else {
                mCore->mDequeueCondition.wait(mCore->mMutex);
            }
        }
    }

    return NO_ERROR;
}
```

### `BufferQueueProducer::getFreeBufferLocked()` 函数
`mFreeBuffers`包括所有状态是`Free`的且有一个Buffer的slot
```
int BufferQueueProducer::getFreeBufferLocked() const {
    if (mCore->mFreeBuffers.empty()) {
        return BufferQueueCore::INVALID_BUFFER_SLOT;
    }
    int slot = mCore->mFreeBuffers.front();
    mCore->mFreeBuffers.pop_front();
    return slot;
}
```

### `BufferQueueProducer::getFreeSlotLocked()` 函数
`mFreeSlots`列表包含所有的状态为`Free`的且当前不含有`Buffer`的slot
```
int BufferQueueProducer::getFreeSlotLocked() const {
    if (mCore->mFreeSlots.empty()) {
        return BufferQueueCore::INVALID_BUFFER_SLOT;
    }
    int slot = *(mCore->mFreeSlots.begin());
    mCore->mFreeSlots.erase(slot);
    return slot;
}
```

### `GraphicBuffer::needsReallocation()` 函数
如果传入的参数与该GraphicBuffer本身的参数不同则返回true，表示需要重新分配
```
bool GraphicBuffer::needsReallocation(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inUsage)
{
    if (static_cast<int>(inWidth) != width) return true;
    if (static_cast<int>(inHeight) != height) return true;
    if (inFormat != format) return true;
    if ((static_cast<uint32_t>(usage) & inUsage) != inUsage) return true;
    return false;
}
```

## `BufferQueueCore::BufferQueueCore()` 构造函数
```
sp<IGraphicBufferAlloc> mAllocator;
//...
// 调用ComposerService::getComposerService()函数，获取ComposerService对象
sp<ISurfaceComposer> composer(ComposerService::getComposerService());
ISurfaceComposer 关联的是SurfaceFlinger对象,SurfaceFlinger继承BnSurfaceFlinger
// 调用SurfaceFlinger::createGraphicBufferAlloc()函数
mAllocator = composer->createGraphicBufferAlloc();
//...
```

### `SurfaceFlinger::createGraphicBufferAlloc()` 函数
创建GraphicBufferAlloc对象，GraphicBufferAlloc是缓冲区分配器
```
sp<IGraphicBufferAlloc> SurfaceFlinger::createGraphicBufferAlloc()
{
    sp<GraphicBufferAlloc> gba(new GraphicBufferAlloc());
    return gba;
}
```

### `GraphicBufferAlloc::GraphicBufferAlloc()` 构造函数
创建GraphicBuffer对象并返回
```
sp<GraphicBuffer> GraphicBufferAlloc::createGraphicBuffer(uint32_t width,
        uint32_t height, PixelFormat format, uint32_t usage,
        std::string requestorName, status_t* error) {
    sp<GraphicBuffer> graphicBuffer(new GraphicBuffer(
            width, height, format, usage, std::move(requestorName)));
    status_t err = graphicBuffer->initCheck();
    *error = err;
    if (err != 0 || graphicBuffer->handle == 0) {
        if (err == NO_MEMORY) {
            GraphicBuffer::dumpAllocationsToSystemLog();
        }
        return 0;
    }
    return graphicBuffer;
}
```

# requestBuffer
Surface通过Binder调用到BufferQueueProducer::requestBuffer()函数，获取指定的BufferSlot的GraphicBuffer对象

## `BufferQueueProducer::requestBuffer()` 函数
根据传入的slot，从mSlots数组中获取该BufferSlot的GraphicBuffer对象
```
status_t BufferQueueProducer::requestBuffer(int slot, sp<GraphicBuffer>* buf) {
    ATRACE_CALL();
    Mutex::Autolock lock(mCore->mMutex);

    if (mCore->mIsAbandoned) {
        return NO_INIT;
    }

    if (mCore->mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
        return NO_INIT;
    }

    if (slot < 0 || slot >= BufferQueueDefs::NUM_BUFFER_SLOTS) {
        return BAD_VALUE;
    } else if (!mSlots[slot].mBufferState.isDequeued()) {
        return BAD_VALUE;
    }

    mSlots[slot].mRequestBufferCalled = true;
    // 获取该BufferSlot的GraphicBuffer对象并赋值给传入的*buf
    *buf = mSlots[slot].mGraphicBuffer;
    return NO_ERROR;
}
```
