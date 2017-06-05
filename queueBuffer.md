# queueBuffer

# queueBuffer 时序图
```puml
Title : queueBuffer 时序图
Note right of Surface : 从OpenGL调用
Surface -> Surface : hook_queueBuffer()
Surface -> Surface : queueBuffer()
Surface -> BpGraphicBufferProducer : queueBuffer()
BpGraphicBufferProducer -> BnGraphicBufferProducer : onTransact()
Note right of BnGraphicBufferProducer : case QUEUE_BUFFER
BnGraphicBufferProducer -> BufferQueueProducer : queueBuffer()
```

## `BufferQueueProducer.queueBuffer()` 函数
1. 参数判断
2. 创建BufferItem
3. 将BufferItem插入到mQueue中
4. 调用回调接口
```
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &input, QueueBufferOutput *output) {
    ATRACE_CALL();
    ATRACE_BUFFER_INDEX(slot);

    int64_t timestamp;
    bool isAutoTimestamp;
    android_dataspace dataSpace;
    Rect crop(Rect::EMPTY_RECT);
    int scalingMode;
    uint32_t transform;
    uint32_t stickyTransform;
    sp<Fence> fence;
    input.deflate(&timestamp, &isAutoTimestamp, &dataSpace, &crop, &scalingMode,
            &transform, &fence, &stickyTransform);
    Region surfaceDamage = input.getSurfaceDamage();

    if (fence == NULL) {
        return BAD_VALUE;
    }

    switch (scalingMode) {
        case NATIVE_WINDOW_SCALING_MODE_FREEZE:
        case NATIVE_WINDOW_SCALING_MODE_SCALE_TO_WINDOW:
        case NATIVE_WINDOW_SCALING_MODE_SCALE_CROP:
        case NATIVE_WINDOW_SCALING_MODE_NO_SCALE_CROP:
            break;
        default:
            return BAD_VALUE;
    }

    sp<IConsumerListener> frameAvailableListener;
    sp<IConsumerListener> frameReplacedListener;
    int callbackTicket = 0;
    BufferItem item;
    {
        Mutex::Autolock lock(mCore->mMutex);
        // 参数判断
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
        } else if (!mSlots[slot].mRequestBufferCalled) {
            return BAD_VALUE;
        }

        if (mCore->mSharedBufferMode && mCore->mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore->mSharedBufferSlot = slot;
            mSlots[slot].mBufferState.mShared = true;
        }
        // 获取传入的slot对应的BufferSlot的GraphicBuffer
        const sp<GraphicBuffer>& graphicBuffer(mSlots[slot].mGraphicBuffer);
        // 根据GraphicBuffer的宽、高创建bufferRect
        Rect bufferRect(graphicBuffer->getWidth(), graphicBuffer->getHeight());
        Rect croppedRect(Rect::EMPTY_RECT);
        crop.intersect(bufferRect, &croppedRect);
        if (croppedRect != crop) {
            return BAD_VALUE;
        }

        if (dataSpace == HAL_DATASPACE_UNKNOWN) {
            dataSpace = mCore->mDefaultBufferDataSpace;
        }

        mSlots[slot].mFence = fence;
        // 调用BufferState的queue()函数，即mDequeueCount减1，mQueueCount加1
        mSlots[slot].mBufferState.queue();

        ++mCore->mFrameCounter;
        mSlots[slot].mFrameNumber = mCore->mFrameCounter;

        // 设置BufferItem对象
        item.mAcquireCalled = mSlots[slot].mAcquireCalled;
        item.mGraphicBuffer = mSlots[slot].mGraphicBuffer;
        item.mCrop = crop;
        item.mTransform = transform &
                ~static_cast<uint32_t>(NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY);
        item.mTransformToDisplayInverse =
                (transform & NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY) != 0;
        item.mScalingMode = static_cast<uint32_t>(scalingMode);
        item.mTimestamp = timestamp;
        item.mIsAutoTimestamp = isAutoTimestamp;
        item.mDataSpace = dataSpace;
        item.mFrameNumber = mCore->mFrameCounter;
        item.mSlot = slot;
        item.mFence = fence;
        item.mIsDroppable = mCore->mAsyncMode ||
                mCore->mDequeueBufferCannotBlock ||
                (mCore->mSharedBufferMode && mCore->mSharedBufferSlot == slot);
        item.mSurfaceDamage = surfaceDamage;
        item.mQueuedBuffer = true;
        item.mAutoRefresh = mCore->mSharedBufferMode && mCore->mAutoRefresh;

        mStickyTransform = stickyTransform;

        if (mCore->mSharedBufferMode) {
            mCore->mSharedBufferCache.crop = crop;
            mCore->mSharedBufferCache.transform = transform;
            mCore->mSharedBufferCache.scalingMode = static_cast<uint32_t>(
                    scalingMode);
            mCore->mSharedBufferCache.dataspace = dataSpace;
        }

        //将BufferItem插入到mQueue
        if (mCore->mQueue.empty()) {
            mCore->mQueue.push_back(item);
            frameAvailableListener = mCore->mConsumerListener;
        } else {
            const BufferItem& last = mCore->mQueue.itemAt(
                    mCore->mQueue.size() - 1);
            if (last.mIsDroppable) {
                if (!last.mIsStale) {
                    mSlots[last.mSlot].mBufferState.freeQueued();
                    if (!mCore->mSharedBufferMode &&
                            mSlots[last.mSlot].mBufferState.isFree()) {
                        mSlots[last.mSlot].mBufferState.mShared = false;
                    }
                    if (!mSlots[last.mSlot].mBufferState.isShared()) {
                        mCore->mActiveBuffers.erase(last.mSlot);
                        mCore->mFreeBuffers.push_back(last.mSlot);
                    }
                }
                mCore->mQueue.editItemAt(mCore->mQueue.size() - 1) = item;
                frameReplacedListener = mCore->mConsumerListener;
            } else {
                mCore->mQueue.push_back(item);
                frameAvailableListener = mCore->mConsumerListener;
            }
        }

        mCore->mBufferHasBeenQueued = true;
        mCore->mDequeueCondition.broadcast();
        mCore->mLastQueuedSlot = slot;

        output->inflate(mCore->mDefaultWidth, mCore->mDefaultHeight,
                mCore->mTransformHint,
                static_cast<uint32_t>(mCore->mQueue.size()));

        ATRACE_INT(mCore->mConsumerName.string(),
                static_cast<int32_t>(mCore->mQueue.size()));

        callbackTicket = mNextCallbackTicket++;

        VALIDATE_CONSISTENCY();
    }

    //将BufferItem的GraphicBuffer和mSlot重置，因为回调不需要这两个参数
    item.mGraphicBuffer.clear();
    item.mSlot = BufferItem::INVALID_BUFFER_SLOT;

    {
        Mutex::Autolock lock(mCallbackMutex);
        while (callbackTicket != mCurrentCallbackTicket) {
            mCallbackCondition.wait(mCallbackMutex);
        }

        // 根据不同的插入形式，调用不同的回调接口，并传入BufferItem对象
        if (frameAvailableListener != NULL) {
            frameAvailableListener->onFrameAvailable(item);
        } else if (frameReplacedListener != NULL) {
            frameReplacedListener->onFrameReplaced(item);
        }

        ++mCurrentCallbackTicket;
        mCallbackCondition.broadcast();
    }

    if (mCore->mConnectedApi == NATIVE_WINDOW_API_EGL) {
        mLastQueueBufferFence->waitForever("Throttling EGL Production");
    }
    mLastQueueBufferFence = fence;
    mLastQueuedCrop = item.mCrop;
    mLastQueuedTransform = item.mTransform;

    return NO_ERROR;
}
```
