> 著有[《React18 设计原理》](https://juejin.cn/column/7557562157064339496)[《javascript地月星》](https://juejin.cn/column/7550585427833864255)等多个专栏。欢迎关注。
>
> 创作不易，有帮助别忘了点赞，收藏，评论 ～ 你的鼓励是我继续挖干货的动力。

> 本文全部都是原创内容，商业转载请联系作者获得授权，非商业转载需注明出处，感谢理解 ～

> ✅ 本文还需要一定的修改。但是已经可以先阅读了。

> 推荐指数（值得一读）：⭐️⭐️⭐️⭐️⭐️⭐️⭐️⭐️⭐️⭐️⭐️⭐️

# 简单的介绍

自动重试是React中非常重要的一个机制，例如因为网络加载未完成、发生了某些错误等导致没有完成最终的“视觉”目标，就需要在适当的时候重试一遍，保证最终显示的准确性。（而不是放弃，或者让人手动刷新。是自动的就完成了。）

重试可以说是懒加载组建、错误恢复、Suspense等特性实现的基础了。

单独的使用懒加载组件会从根开始整个页面的恢复（唤醒重试）。Suspense的fallback和primary内容的切换也需要依赖重试，顺带能给懒加载组件一个重试边界，不需要从根开始整个页面恢复（Suspense重试），只需要局部切换fallback和primary。

# 添加ping和retry的流程概要
attachPingListener 和 attachRetryListener
```js
首先添加ping：wakeable.then(ping,ping)    直接执行。ping等promise返回后执行。
其次收集Promise：updateQueue.add(wakeable)   队列收集请求。
最后根据需要添加retry：wakeable.then(retry,retry)  commit阶段是Update，表示还有任务，添加retry。retry等promise返回后执行。
```

# Q：重复调度问题：同一个 Promise即添加唤醒又添加重试，是不是重复调度渲染了 ？

推测 1:确实重复❌，那样其他地方是不是检测避免重复？ 推测 2:不会重复✅。那么要如何理解 ping+retry 的逻辑？

根据 attachPingListener 和 attachRetryListener 的源码中的英文注释，可以知道一些信息：在提交 fallback 前，Promise 就完成了，用 ping 来重新渲染。在提交了 fallback 后，我们需要另一种监听器（retry）来更新 Suspense 的渲染内容。

并且，我知道不会重复。ping和retry是独立的。ping的同时会收集请求。在这一轮的提交后发现还有任务，就会尝试给请求添加retry。


## 需要知道的几个事实

1.  目前讨论问题的模型：\
wakeable.then(ping, ping);\
wakeable.then(retry, retry);\
~~唤醒/重试原理：ping和retry中设置给Fiber设置lane，包含ensureRootIsScheduled，Promise完成后，自然重新渲染。~~ (需要修改)\
⚠️ 这是并列的 then， 不是链式的：wakeable.then(ping,ping).then(retry,retry)；所以 retry 不需要等待 ping 的状态，他们等的都是 wakeable 的状态。
2.  ping 一定会生效，只要 Promise 的状态改变，ping 一定会生效。
3.  attachRetryListener只是收集 retry，没有真正添加。**retry 的添加逻辑是：只看这一轮的调度结果**，attachSuspenseRetryListeners 的执行在这一轮调度后 Suspense 边界 flags 存在 Update。
4.  Suspense并不是只靠自身完成的fallback和primary切换，attachRetryListener 首先收集了这个Promise。Suspene再添加上重试。Suspense 依赖 Promise 唤醒 Suspense 重试。

**所以不会重复，只有“需要”，需要Suspense重试的情况就添加。** 1. 关键在于 attachRetryListener 只是收集，2. retry 只根据这一轮的调度结果决定是否添加。


# 代码
`throwException`中执行attachPingListener 、attachRetryListener添加ping 和 retry。他们都是重试，一个是唤醒重试（从根节点开始的普通的重试），一个是Suspense重试（只在Suspense范围内的重试）。

其中的分支`if (suspenseBoundary !== null) {} else {}`，
1. 如果有 Suspense，需要 `attachPingListener + attachRetryListener`，attachRetryListener中的队列是专门给 Suspense 收集 Promise retry 的。
2. 没有 Suspense 只要 `attachPingListener`，不需要 attachRetryListener，因为根本没有 Suspense，收集了也无法添加给不存在的Suspense。
```js
function throwException(root, returnFiber, sourceFiber, value, rootRenderLanes) {
  // The source fiber did not complete.
  sourceFiber.flags |= Incomplete;

  {
    if (isDevToolsPresent) {
      // If we have pending work still, restore the original updaters
      restorePendingUpdaters(root, rootRenderLanes);
    }
  }

  //{then:function(){}}
  if (value !== null && typeof value === 'object' && typeof value.then === 'function') {
    // This is a wakeable. The component suspended.
    var wakeable = value;
    resetSuspendedComponent(sourceFiber);

    {
      if (getIsHydrating() && sourceFiber.mode & ConcurrentMode) {
        markDidThrowWhileHydratingDEV();
      }
    }

    var suspenseBoundary = getNearestSuspenseBoundaryToCapture(returnFiber);

    //1/2：如果有suspense边界，promise请求到达后的重试
    if (suspenseBoundary !== null) {
      suspenseBoundary.flags &= ~ForceClientRender;
      markSuspenseBoundaryShouldCapture(suspenseBoundary, returnFiber, sourceFiber, root, rootRenderLanes);

      // We only attach ping listeners in concurrent mode. Legacy Suspense always
      // commits fallbacks synchronously, so there are no pings.
      if (suspenseBoundary.mode & ConcurrentMode) {
        attachPingListener(root, wakeable, rootRenderLanes);// 唤醒重试
      }
      // 1/2:有Suspense，给Suspense添加重试
      attachRetryListener(suspenseBoundary, root, wakeable); // Suspense重试
      return;
    } else {
      // 2/2:如果没有suspense边界，promise请求到达后的唤醒重试
      // 完全没有attachRetryListener。只有attachPingListener。
      // No boundary was found. Unless this is a sync update, this is OK.
      // We can suspend and wait for more data to arrive.
      if (!includesSyncLane(rootRenderLanes)) {
        // This is not a sync update. Suspend. Since we're not activating a
        // Suspense boundary, this will unwind all the way to the root without
        // performing a second pass to render a fallback. (This is arguably how
        // refresh transitions should work, too, since we're not going to commit
        // the fallbacks anyway.)
        //
        // This case also applies to initial hydration.
        attachPingListener(root, wakeable, rootRenderLanes);
        renderDidSuspendDelayIfPossible();
        return;
      } 
      // This is a sync/discrete update. We treat this case like an error
      // because discrete renders are expected to produce a complete tree
      // synchronously to maintain consistency with external state.


      var uncaughtSuspenseError = new Error('A component suspended while responding to synchronous input. This ' + 'will cause the UI to be replaced with a loading indicator. To ' + 'fix, updates that suspend should be wrapped ' + 'with startTransition.');
      // If we're outside a transition, fall through to the regular error path.
      // The error will be caught by the nearest suspense boundary.

      value = uncaughtSuspenseError;
    }
  } else {
    // This is a regular error, not a Suspense wakeable.
    也是重试，通过添加任务的方式，添加到“水合错误hydrationErrors”数组，在后续的渲染轮会轮到这个任务。我们暂时不看这个。
    //queueHydrationError(createCapturedValueAtFiber(value, sourceFiber));
    ...
  }


  也是重试，通过添加任务的方式,添加任务到workInProgressRootConcurrentErrors，在后续的渲染轮会轮到这个任务。我们暂时不看这个。
  ...


  // We didn't find a boundary that could handle this type of exception. Start
  // over and traverse parent path again, this time treating the exception
  // as an error.
  也是重试，通过添加任务的方式，添加到“捕获更新CapturedUpdate”队列，但是我主要想介绍promise后的重试。我们暂时不看这个。
  // var update = createRootErrorUpdate(workInProgress, _errorInfo, lane);
  // enqueueCapturedUpdate(workInProgress, update);
  //
  // var _update = createClassErrorUpdate(workInProgress, errorInfo, _lane);
  // enqueueCapturedUpdate(workInProgress, _update);
  ...
}

function includesSyncLane(lanes) {
  return (lanes & SyncLane) !== NoLanes;
}
```
### attachPingListener
`attachPingListener`: pingCache 只是一个“过滤器”，作用不是真的把 ping 保存到一个集合里面，真正的作用是保证 ping 的唯一性，不会有重复的 ping，导致重复的唤醒调度。其结构 WeakMap\<wakeable,Set\<lanes>>。对于 Map，key 是唯一的，对于 Set，value 是唯一的。保证了wakeable 不是重复的，保证了一个 wakeable的 lanes 不是重复的。

ping 在 Promise 请求成功/失败后自动调用，ensureRootIsScheduled 自然重新调度渲染。

`pingSuspendedRoot`: ping 执行时还删除 (delete)了被执行掉的 ping。ping 是从根节点整个的重新渲染（也就是从头开始重新遍历 Fiber，因为 prepareFreshStack（把所有状态重置清空），从头开始遍历的每一个节点都不 bailout 快速跳过/早退）。
```js
function attachPingListener(root, wakeable, lanes) {
  // Attach a ping listener
  //
  // The data might resolve before we have a chance to commit the fallback. Or,
  // in the case of a refresh, we'll never commit a fallback. So we need to
  // attach a listener now. When it resolves ("pings"), we can decide whether to
  // try rendering the tree again.
  //
  // Only attach a listener if one does not already exist for the lanes
  // we're currently rendering (which acts like a "thread ID" here).
  //
  // We only need to do this in concurrent mode. Legacy Suspense always
  // commits fallbacks synchronously, so there are no pings.
  var pingCache = root.pingCache;
  var threadIDs;

  if (pingCache === null) {
    pingCache = root.pingCache = new PossiblyWeakMap$1();
    threadIDs = new Set();
    pingCache.set(wakeable, threadIDs);
  } else {
    threadIDs = pingCache.get(wakeable);

    if (threadIDs === undefined) {
      threadIDs = new Set();
      pingCache.set(wakeable, threadIDs);
    }
  }

  if (!threadIDs.has(lanes)) {
    // Memoize using the thread ID to prevent redundant listeners.
    threadIDs.add(lanes);
    var ping = pingSuspendedRoot.bind(null, root, wakeable, lanes);

    {
      if (isDevToolsPresent) {
        // If we have pending work still, restore the original updaters
        restorePendingUpdaters(root, lanes);
      }
    }
    //
    wakeable.then(ping, ping);//ping是pingSuspendedRoot
  }
}
```
wakeable.then(ping, ping): 核心在于调用 ensureRootIsScheduled 、markRootPinged
```js
function pingSuspendedRoot(root, wakeable, pingedLanes) {
  var pingCache = root.pingCache;

  if (pingCache !== null) {
    // The wakeable resolved, so we no longer need to memoize, because it will
    // never be thrown again.
    pingCache.delete(wakeable);//删除已经执行的
  }

  var eventTime = requestEventTime();
  markRootPinged(root, pingedLanes);  //
  warnIfSuspenseResolutionNotWrappedWithActDEV(root);

  if (workInProgressRoot === root && isSubsetOfLanes(workInProgressRootRenderLanes, pingedLanes)) {
    // Received a ping at the same priority level at which we're currently
    // rendering. We might want to restart this render. This should mirror
    // the logic of whether or not a root suspends once it completes.
    // TODO: If we're rendering sync either due to Sync, Batched or expired,
    // we should probably never restart.
    // If we're suspended with delay, or if it's a retry, we'll always suspend
    // so we can always restart.
    if (workInProgressRootExitStatus === RootSuspendedWithDelay || workInProgressRootExitStatus === RootSuspended && includesOnlyRetries(workInProgressRootRenderLanes) && now() - globalMostRecentFallbackTime < FALLBACK_THROTTLE_MS) {
      // Restart from the root.
      prepareFreshStack(root, NoLanes); //
    } else {
      // Even though we can't restart right now, we might get an
      // opportunity later. So we mark this render as having a ping.
      workInProgressRootPingedLanes = mergeLanes(workInProgressRootPingedLanes, pingedLanes);
    }
  }

  ensureRootIsScheduled(root, eventTime);
}
```
markRootPinged 和 prepareFreshStack：
```js
//没什么特别的。ping重试对应一个pingedLanes级别的任务。添加到任务。
function markRootPinged(root2, pingedLanes, eventTime) {
  root2.pingedLanes |= root2.suspendedLanes & pingedLanes;
}

//没有什么特别的，把遍历Fiber树过程中的全部状态重置，丢掉这一次渲染的中间结果，为从头开始重新渲染做准备
//例如workInProgrerss被重置成了rootWorkInProgress，那么下一次beginWork就是从根节点了。
function prepareFreshStack(root, lanes) {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  var timeoutHandle = root.timeoutHandle;

  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout; // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above

    cancelTimeout(timeoutHandle);
  }

  if (workInProgress !== null) {
    var interruptedWork = workInProgress.return;

    while (interruptedWork !== null) {
      var current = interruptedWork.alternate;
      unwindInterruptedWork(current, interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }

  workInProgressRoot = root;
  var rootWorkInProgress = createWorkInProgress(root.current, null);
  workInProgress = rootWorkInProgress;
  workInProgressRootRenderLanes = subtreeRenderLanes = workInProgressRootIncludedLanes = lanes;
  workInProgressRootExitStatus = RootInProgress;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootInterleavedUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;
  workInProgressRootConcurrentErrors = null;
  workInProgressRootRecoverableErrors = null;
  finishQueueingConcurrentUpdates();

  {
    ReactStrictModeWarnings.discardPendingWarnings();
  }

  return rootWorkInProgress;
}
```

### attachRetryListener
`attachRetryListener`: 添加重试，但却并不是真正的添加 retry，这里只是把 Promise 收集到 suspense 更新队列(updateQueue)里面。
```js
function attachRetryListener(suspenseBoundary, root, wakeable, lanes) {
  // Retry listener
  //
  // If the fallback does commit, we need to attach a different type of
  // listener. This one schedules an update on the Suspense boundary to turn
  // the fallback state off.
  //
  // Stash the wakeable on the boundary fiber so we can access it in the
  // commit phase.
  //
  // When the wakeable resolves, we'll attempt to render the boundary
  // again ("retry").
  var wakeables = suspenseBoundary.updateQueue;

  if (wakeables === null) {
    var updateQueue = new Set();
    updateQueue.add(wakeable);
    suspenseBoundary.updateQueue = updateQueue; //收集到队列，还没有真正添加，发挥作用
  } else {
    wakeables.add(wakeable);
  }
}
```



### 真正添加 Retry：commitMutationEffectsOnFiber
提交阶段执行`commitMutationEffectsOnFiber`。

主要意图在`retryTimedOutBoundary`: retry在 Promise 请求成功/失败后自动调用，ensureRootIsScheduled 自然重新调度渲染。

retry 是从 Suspense 开始重新渲染（也就是从 Suspense 开始重新遍历），因为没有如同 ping 一样prepareFreshStack（把所有状态重置清空），workInProgress 还是 Suspense 所以从 Suspense 继续beginWork，发现flags === ShouldCaputre/DidCapture/Update 之类的，不会 bailout Suspense，而是向下深入到 Suspense 子节点。
```js
//提交阶段的代码
function commitMutationEffectsOnFiber(finishedWork, root, lanes) {
  ...
  case SuspenseComponent: {
    if (flags & Update) {
      try {
        commitSuspenseCallback(finishedWork);
      } catch (error2) {
        captureCommitPhaseError(finishedWork, finishedWork.return, error2);
      }
      attachSuspenseRetryListeners(finishedWork);
    }
    return;
  }
}
```
attachSuspenseRetryListeners：提交阶段 retry 才被真正添加。attachSuspenseRetryListeners遍历（forEach）前面队列收集的Promise添加retry 。
```js
function attachSuspenseRetryListeners(finishedWork) {
  // If this boundary just timed out, then it will have a set of wakeables.
  // For each wakeable, attach a listener so that when it resolves, React
  // attempts to re-render the boundary in the primary (pre-timeout) state.
  var wakeables = finishedWork.updateQueue;

  if (wakeables !== null) {
    finishedWork.updateQueue = null;
    var retryCache = finishedWork.stateNode;

    if (retryCache === null) {
      retryCache = finishedWork.stateNode = new PossiblyWeakSet();
    }
    
    //遍历
    wakeables.forEach(function (wakeable) {
      // Memoize using the boundary fiber to prevent redundant listeners.
      var retry = resolveRetryWakeable.bind(null, finishedWork, wakeable);

      if (!retryCache.has(wakeable)) {
        retryCache.add(wakeable);

        {
          if (isDevToolsPresent) {
            if (inProgressLanes !== null && inProgressRoot !== null) {
              // If we have pending work still, associate the original updaters with it.
              restorePendingUpdaters(inProgressRoot, inProgressLanes);
            } else {
              throw Error('Expected finished root and lanes to be set. This is a bug in React.');
            }
          }
        }
        //
        wakeable.then(retry, retry); //retry是resolveRetryWakeable
      }
    });
  }
} 
```
wakeable.then(retry, retry)：
```js
function resolveRetryWakeable(boundaryFiber, wakeable) {
  var retryLane = NoLane; // Default

  var retryCache;

  switch (boundaryFiber.tag) {
    case SuspenseComponent:
      retryCache = boundaryFiber.stateNode;
      var suspenseState = boundaryFiber.memoizedState;

      if (suspenseState !== null) {
        retryLane = suspenseState.retryLane;
      }

      break;

    case SuspenseListComponent:
      retryCache = boundaryFiber.stateNode;
      break;

    default:
      throw new Error('Pinged unknown suspense boundary type. ' + 'This is probably a bug in React.');
  }

  if (retryCache !== null) {
    // The wakeable resolved, so we no longer need to memoize, because it will
    // never be thrown again.
    retryCache.delete(wakeable);
  }

  retryTimedOutBoundary(boundaryFiber, retryLane);
} 
```
retryTimedOutBoundary：核心在于调用ensureRootIsScheduled 和 markRootUpdated
```js
function retryTimedOutBoundary(boundaryFiber, retryLane) {
  // The boundary fiber (a Suspense component or SuspenseList component)
  // previously was rendered in its fallback state. One of the promises that
  // suspended it has resolved, which means at least part of the tree was
  // likely unblocked. Try rendering again, at a new lanes.
  if (retryLane === NoLane) {
    // TODO: Assign this to `suspenseState.retryLane`? to avoid
    // unnecessary entanglement?
    retryLane = requestRetryLane(boundaryFiber);
  } 
  
  // TODO: Special case idle priority?

  var eventTime = requestEventTime();
  var root = enqueueConcurrentRenderForLane(boundaryFiber, retryLane);

  if (root !== null) {
    markRootUpdated(root, retryLane, eventTime);
    //开始了一轮新的调度
    ensureRootIsScheduled(root, eventTime);
  }
}
```
```js
//没什么特别的，把retryLane添加到root.pendingLanes上， 一个retryLane对应一个pendingLanes，retryLane只是一个临时变量。
function markRootUpdated(root2, updateLane, eventTime) {
  root2.pendingLanes |= updateLane;
  if (updateLane !== IdleLane) {
    root2.suspendedLanes = NoLanes;
    root2.pingedLanes = NoLanes;
  }
  var eventTimes = root2.eventTimes;
  var index2 = laneToIndex(updateLane);
  eventTimes[index2] = eventTime;
}
```
```js
function enqueueConcurrentRenderForLane(fiber, lane) {
  return markUpdateLaneFromFiberToRoot(fiber, lane);
}

//没什么特别的，把retryLane添加到Suspense上，和Suspense的父节点的childLanes上
function markUpdateLaneFromFiberToRoot(sourceFiber, lane) {
  // Update the source fiber's lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  var alternate = sourceFiber.alternate;

  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }

  {
    if (alternate === null && (sourceFiber.flags & (Placement | Hydrating)) !== NoFlags) {
      warnAboutUpdateOnNotYetMountedFiberInDEV(sourceFiber);
    }
  }
    
  // Walk the parent path to the root and update the child lanes.
  var node = sourceFiber;
  var parent = sourceFiber.return;

  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;

    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    } else {
      {
        if ((parent.flags & (Placement | Hydrating)) !== NoFlags) {
          warnAboutUpdateOnNotYetMountedFiberInDEV(sourceFiber);
        }
      }
    }

    node = parent;
    parent = parent.return;
  }

  if (node.tag === HostRoot) {
    var root = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```
# 总结
唤醒/重试原理：ping和retry中设置给Fiber设置lane，最后ensureRootIsScheduled()，Promise完成后，能够重新渲染。\
ping和retry的：ping 一定会生效执行一次，retry看需要。ping从头开始重新渲染，retry从Suspense边界开始重新渲染。\
同一个Promise，即唤醒又重试，但不会重复，因为Suspense的重试是看需要添加的 1. 关键在于 attachRetryListener 只是收集，2. retry 只根据这一轮的调度结果决定是否添加。\
Suspense并不是只靠自身完成的fallback和primary切换，Suspense 依赖 Promise 唤醒 Suspense 重试。

有了 Promise 的重试，懒加载组件的实现就简单了，只要在 Promise\<pending>的时候抛出错误，用throwException 接住，就可以了。同样，收集了懒加载组件的 Promise， Suspense 的实现也方便了。



