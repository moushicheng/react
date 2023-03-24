# react18渲染流程
特殊原因，该笔记不继续维护，转而使用：
[飞书](https://bytedance.feishu.cn/docx/YUbLdElyKohMGHxlTf6c6liwncc)
## test case
```javascript
  it('renders children', () => {
    const root = ReactDOMClient.createRoot(container); //container是容器DOM
    root.render(<div>Hi</div>);
    Scheduler.unstable_flushAll();
    expect(container.textContent).toEqual('Hi');
  });
```
## babel转换
首先，对于jsx来说,他会经过一层转换，最终转换后的test case如下所示:
```javascript
  it("renders children", function () {
    var root = ReactDOMClient.createRoot(container);
    root.render( /*#__PURE__*/React.createElement("div", { __source: { fileName: _jsxFileName, lineNumber: 37, columnNumber: 17 } }, "Hi"));
    Scheduler.unstable_flushAll();
    expect(container.textContent).toEqual("Hi");
  });
```

## 细说root
root是fiberRootNode的一层封装对象，这个对象包含一些重要的属性和方法，

比如render方法，

比如_internalRoot属性，其指向了fiberRootNode
这个fiberRootNode是整个应用的起点，可以理解为指向根组件(rootFiber)的根元素，这么说可能有点绕，可以看看这个图

![rootFiber tree](/image/rootFiber tree.png)

rootFiber是组件的根Fiber，而alternate指向的两个fiber树互为新旧fiber tree，这将用于diff算法的使用。

但值得一提的是，rootFiber在第一次mount（首屏渲染）前就有新旧rootFiber tree，也就是说fiberRootNode.current.alternate非空

这和他的具体渲染流程相关，最终的效果可以提前剧透：
实际上就是在内存中把rootFiber的dom创建好，并将子元素插入到内存中的父dom上，最后，再把rootFiber插入到页面上，注意：这里只有一次插入页面（placement）操作。为什么这样能达到这种效果呢[1]？后续会讲。

总结一下:
有两个问题:
1. 新 alternate 如何形成？
2. 具体的渲染挂载流程如何？

## render的详细流程


## 引子-fiber上的flags属性
fiber上的flags属性决定其对应的dom要做何操作
比如
```js
// DOM需要插入到页面中
export const Placement = /*                */ 0b00000000000010;
// DOM需要更新
export const Update = /*                   */ 0b00000000000100;
// DOM需要插入到页面中并更新
export const PlacementAndUpdate = /*       */ 0b00000000000110;
// DOM需要删除
export const Deletion = /*                 */ 0b00000000001000;
```

理解了flags之后，我们再解决两个问题
1. flags标记如何做？
2. 回应前面的问题[1]，rootFiber渲染时时如何只进行一次placement操作的？
 - 子问题1：为什么只有rootfiber进行了placement？
 - 子问题2: 为什么其他子孙fiber只会插入到父dom中

对于问题1
### flags标记如何做？
在render流程之初的beginWork会根据不同的jsx元素的tag来创建不同的fiber,比如这里

```javascript
     case REACT_ELEMENT_TYPE:
          return placeSingleChild( //placeSingleChild会处理flags
            reconcileSingleElement( //这里会生成元素Fiber
              returnFiber,
              currentFirstChild,
              newChild,
              lanes,
            ),
          );
```
reconcileSingleElement会创建单一的元素Element
然后传参给placeSingleChild，它会处理flags
看看实现
```javascript
  function placeSingleChild(newFiber: Fiber): Fiber {
    // This is simpler for the single child case. We only need to do a
    // placement for inserting new children.
    if (shouldTrackSideEffects && newFiber.alternate === null) {
      newFiber.flags |= Placement | PlacementDEV; //注意到了吧，这里会附带Placement
    }
    return newFiber;
  }
```
可以重点关注 一下shouldTrackSideEffects这个全局变量，它会在uptate时被设置为true，mount时被设置为false
可以再回顾一下上面的root Fiber tree图，在最开始rootFiber是走update流程
所以它的child（rootFiber）的flags会被设置为Placement
而其他孙元素（也就是rootFiber的其他子元素）不会附上Placement。
这就意味着，最终打上“请把这个fiber对应的dom插入到页面上”的只有rootFiber

## render流程
渲染主要分为几个大流程
1.生成fiber -- beginWork
2.生成dom   -- completeWork
3.处理副作用 -- commit阶段

## beginWork

## completeWork

## commit

在beforeMutaion之前最重要的操作就是调度useEffect，flushPassiveEffects会执行useEffect cb，其执行时机在异步。
```javascript
    if ((effectTag & Passive) !== NoEffect) {
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
```
### beforemutation阶段
Dom渲染前的操作
主要做了一件事，递归式遍历整个EffectList的fiber节点树并执行一定的操作
并调用
```js
commitBeforeMutationEffectsOnFiber()
```
它会调用ClassComponent上的getSnapshotBeforeUpdate并传入当前节点的snapshot。
具体逻辑不必细聊，毕竟都是functionCompoent的时代了 ，且getSnapshotBeforeUpdate钩子只会在classcomponent调用。

### mutation阶段
这里可以大讲特讲，因为其是Dom渲染的核心，
其入口函数是
```javascript
export function commitMutationEffects(
  root: FiberRoot,
  finishedWork: Fiber,
  committedLanes: Lanes,
) {
  inProgressLanes = committedLanes;
  inProgressRoot = root;

  setCurrentDebugFiberInDEV(finishedWork);
  commitMutationEffectsOnFiber(finishedWork, root, committedLanes);
  setCurrentDebugFiberInDEV(finishedWork);

  inProgressLanes = null;
  inProgressRoot = null;
}
```
关键在于执行了commitMutationEffectsOnFiber，
记住xxxxOnfiber在react上的抽象含义是对具体的Fiber执行xxx操作，比如这里的含义就是“在具体的Fiber上执行commitMutationEffects操作”

### layout阶段
先看官方的解释
```javascript
    // The next phase is the layout phase, where we call effects that read
    // the host tree after it's been mutated. The idiomatic use case for this is
    // layout, but class component lifecycles also fire here for legacy reasons.
    commitLayoutEffects(finishedWork, root, lanes);
    //....
    export function commitLayoutEffects(
      finishedWork: Fiber,
      root: FiberRoot,
      committedLanes: Lanes,
    ): void {
      inProgressLanes = committedLanes;
      inProgressRoot = root;

      const current = finishedWork.alternate;
      commitLayoutEffectOnFiber(root, current, finishedWork, committedLanes);

      inProgressLanes = null;
      inProgressRoot = null;
    }    
```
翻译：下一个阶段是layout阶段，该阶段我们会调用在dom树发生变化之后的effects。这里常见触发的钩子是“布局”但是一些class组建的生命周期也会在此处因为一些原因而触发。
根据注释我们知道
该阶段
1.会处理effects
2.会处理一些生命周期钩子
从资料中我们也可以知道:该阶段触发的生命周期钩子和hoo·k可以直接访问到已经改变后的DOM，即该阶段是可以参与DOM layout的阶段。

#### commitLayoutEffectOnFiber
commitLayoutEffectOnFiber方法会根据fiber.tag对不同类型的节点分别处理。
举个🌰函数式组件的分支逻辑：
```javascript
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      recursivelyTraverseLayoutEffects(
        finishedRoot,
        finishedWork,
        committedLanes,
      );
      if (flags & Update) {
        commitHookLayoutEffects(finishedWork, HookLayout | HookHasEffect);//核心工作
      }
      break;
    }
```
#### recursivelyTraverseLayoutEffects
该函数名：递归地遍历LayoutEffects,我们也能略知一二
```javascript
function recursivelyTraverseLayoutEffects(
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes,
) {
  const prevDebugFiber = getCurrentDebugFiberInDEV();
  if (parentFiber.subtreeFlags & LayoutMask) {
    let child = parentFiber.child;
    while (child !== null) {
      setCurrentDebugFiberInDEV(child);
      const current = child.alternate;
      commitLayoutEffectOnFiber(root, current, child, lanes);//继续遍历之前的流程
      child = child.sibling;
    }
  }
  setCurrentDebugFiberInDEV(prevDebugFiber);
}
```
其实就是层序遍历子fiber，给每个fiber再走一遍layout阶段
所以，这里不是核心流程,他只负责起一个递归的作用，对于函数式组件，
其主要工作是位于commitLayoutEffectOnFiber的 commitHookLayoutEffects，如下：
```js
     commitHookLayoutEffects(finishedWork, HookLayout | HookHasEffect);
     //...
    function commitHookLayoutEffects(finishedWork: Fiber, hookFlags: HookFlags) {
      // At this point layout effects have already been destroyed (during mutation phase).
      // This is done to prevent sibling component effects from interfering with each other,
      // e.g. a destroy function in one component should never override a ref set
      // by a create function in another component during the same commit.
      if (shouldProfile(finishedWork)) {
        //...
      } else {
          // 执行useLayoutEffect的回调函数
          commitHookEffectListMount(hookFlags, finishedWork);
      }
    }
```
#### 总结
综上我们可以总结：对于函数试组件，react会递归·的遍历其fiber子节点并调用其上的useLayoutEffect
