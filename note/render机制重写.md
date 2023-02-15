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

![rootFiber tree](E:\work_space\Technology_related\Front-end\MVVM\react\react\note\image\rootFiber tree.png)

rootFiber是组件的根Fiber，而alternate指向的两个fiber树互为新旧fiber tree，这将用于diff算法的使用。

但值得一提的是，rootFiber在第一次mount（首屏渲染）前就有新旧rootFiber tree，也就是说fiberRootNode.current.alternate非空

这和他的具体渲染流程相关，最终的效果可以提前剧透：
实际上就是在内存中把rootFiber的dom创建好，并将子元素插入到内存中的父dom上，最后，再把rootFiber插入到页面上，注意：这里只有一次插入页面（placement）操作。为什么这样能达到这种效果呢[1]？后续会讲。

总结一下:
有两个问题:
1. 新 alternate 如何形成？
2. 具体的渲染挂载流程如何？

## render的详细流程


## flags
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
在beginWork会根据不同的jsx元素的tag来创建不同的fiber,比如这里

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
所以它的child的flags会被设置为Placement
而其他孙元素不会附上Placement。

