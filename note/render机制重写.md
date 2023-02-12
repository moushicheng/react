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

这和他的具体渲染流程相关，最终的效果可以提前剧透，实际上就是在内存中把rootFiber的dom创建好，并将子元素插入到内存中的dom上，最后干完所有事情后，再把rootFiber插入到页面上。为什么这样干能达到这种效果呢？后续会讲。

总结一下:
有两个问题:
1. 新 alternate 如何形成？
2. 具体的渲染挂载流程如何？

## render的详细流程

