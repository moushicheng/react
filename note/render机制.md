## 入口
children是一个Jsx Element，主要逻辑还是在于updateContainer
```js
ReactDOMHydrationRoot.prototype.render = ReactDOMRoot.prototype.render = function(
  children: ReactNodeList,
): void {
  const root = this._internalRoot;
  if (root === null) {
    throw new Error('Cannot update an unmounted root.');
  }

  if (__DEV__) {
    //...
  }
  updateContainer(children, root, null, null);
};
```
## updateContainer
两个主要逻辑点
1.enqueueUpdate
2.scheduleUpdateOnFiber
```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  if (__DEV__) {
    onScheduleRoot(container, element);
  }
  const current = container.current;
  const eventTime = requestEventTime();
  //获取当前更新事件的优先级
  const lane = requestUpdateLane(current);

  if (enableSchedulingProfiler) {
    markRenderScheduled(lane);
  }
  //对于根组件来说，由于没有parentComponent，所以context为{}
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  const update = createUpdate(eventTime, lane);
  update.payload = {element};

  //Update更新逻辑入队，感觉就是跟react的异步更新有关
  const root = enqueueUpdate(current, update, lane);
  if (root !== null) {
    scheduleUpdateOnFiber(root, current, lane, eventTime);
    entangleTransitions(root, current, lane);
  }

  return lane;
}
```
1.lane 32
2.context {}
3.update 存储一些更新信息
```js
  const update: Update<mixed> = {
    eventTime,
    lane,

    tag: UpdateState,
    payload: null, //载荷，一般是jsxElement
    callback: null,

    next: null,
  };
```

## enqueueUpdate
逻辑流转有点多，最终入队，这里的concurrentQueues，最后在哪里用上，暂时还不知道
```js
function enqueueUpdate(
  fiber: Fiber,
  queue: ConcurrentQueue | null,
  update: ConcurrentUpdate | null,
  lane: Lane,
) {
  // Don't update the `childLanes` on the return path yet. If we already in
  // the middle of rendering, wait until after it has completed.
  concurrentQueues[concurrentQueuesIndex++] = fiber;
  concurrentQueues[concurrentQueuesIndex++] = queue;
  concurrentQueues[concurrentQueuesIndex++] = update;
  concurrentQueues[concurrentQueuesIndex++] = lane;

  concurrentlyUpdatedLanes = mergeLanes(concurrentlyUpdatedLanes, lane);

  // The fiber's `lane` field is used in some places to check if any work is
  // scheduled, to perform an eager bailout, so we need to update it immediately.
  // TODO: We should probably move this to the "shared" queue instead.
  fiber.lanes = mergeLanes(fiber.lanes, lane);
  const alternate = fiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
}
  ```

## 核心渲染逻辑 scheduleUpdateOnFiber
英文翻译：调度Fiber上的更新逻辑
```JS
//省略一大堆
ensureRootIsScheduled(root, eventTime);
```

```JS
// Use this function to schedule a task for a root. There's only one task per
// 使用这个函数去调度根节点上的任务,每个根节点只有一个任务
// root; if a task was already scheduled, we'll check to make sure the priority
// 如果一个任务已经被调度了，我们将检查以确保现有任务的优先级与根进程正在处理的下一层任务的优先级相同。
// of the existing task is the same as the priority of the next level that the
// root has work on. This function is called on every update, and right before
// 此函数在每次更新时，以及在退出任务之前调用。
// exiting a task.
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
    //各种处理优先级最终得到schedulerPriorityLevel
    //调度回调
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```
对于scheduleCallback实际上就是unstable_scheduleCallback的封装，它是React的Scheduler（调度器）的入口
毕竟render任务也是一个任务嘛，react要把他放入调度器中执行
但本文主要讲render而不是讲调度器，所以如何调度先按下不表，只需要知道
最终,它会被调度执行performConcurrentWorkOnRoot函数
```js
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
```

###  performConcurrentWorkOnRoot

其主要进行dom diff的核心逻辑在
```javascript
let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)
    : renderRootSync(root, lanes);
```

