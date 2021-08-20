
要将 React 元素渲染到页面中，分为两个阶段，render 阶段和 commit 阶段。 

**一、render 阶段负责创建 Fiber 数据结构并为 Fiber 节点打标记，标记当前 Fiber 节点要进行的 DOM 操作。** 

render阶段开始于packages/react-dom/src/client/ReactDOMLegacy.js，传入三个参数，渲染的ReactElement、渲染容器和回调函数，并将这些参数传入至legacyRenderSubtreeIntoContainer方法当中

```
/**
 * 渲染入口
 * element 要进行渲染的 ReactElement, createElement 方法的返回值
 * container 渲染容器 <div id="root"></div>
 * callback 渲染完成后执行的回调函数
 */
export function render(
  element: React$Element<any>,
  container: Container,
  callback: ?Function,
) {
  // 检测 container 是否是符合要求的渲染容器
  // 即检测 container 是否是真实的DOM对象
  // 如果不符合要求就报错
  invariant(
    isValidContainer(container),
    'Target container is not a DOM element.',
  );
  return legacyRenderSubtreeIntoContainer(
    // 父组件 初始渲染没有父组件 传递 null 占位
    null,
    element,
    container,
    // 是否为服务器端渲染 false 不是服务器端渲染 true 是服务器端渲染
    false,
    callback,
  );
}
```

在legacyRenderSubtreeIntoContainer方法当中，会通过判断container对象上的属性_reactRootContainer是否存在来判断是否为初次渲染，如果没有就调用legacyCreateRootFromDOMContainer方法创建，之后无论初次渲染还是更新都会使用updateContainer方法更新渲染容器，当然初次渲染会选择不执行批量更新，最后统一调用getPublicRootInstance方法返回真实的DOM对象

```
/**
 * 将子树渲染到容器中 (初始化 Fiber 数据结构: 创建 fiberRoot 及 rootFiber)
 * parentComponent: 父组件, 初始渲染传入了 null
 * children: render 方法中的第一个参数, 要渲染的 ReactElement
 * container: 渲染容器
 * forceHydrate: true 为服务端渲染, false 为客户端渲染
 * callback: 组件渲染完成后需要执行的回调函数
 **/
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function,
) {

  let root: RootType = (container._reactRootContainer: any);
  // 即将存储根 Fiber 对象
  let fiberRoot;
  if (!root) {
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    // 获取 Fiber Root 对象
    fiberRoot = root._internalRoot;

    if (typeof callback === 'function') {
      // 使用 originalCallback 存储 callback 函数
      const originalCallback = callback;
      // 为 callback 参数重新赋值
      callback = function () {
        const instance = getPublicRootInstance(fiberRoot);
        // 调用 callback 函数并改变函数内部 this 指向
        originalCallback.call(instance);
      };
    }
    // 初始化渲染不执行批量更新
    // 因为批量更新是异步的是可以被打断的, 但是初始化渲染应该尽快完成不能被打断
    // 所以不执行批量更新
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    // 非初始化渲染 即更新
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function () {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }

  return getPublicRootInstance(fiberRoot);
}
```

针对legacyCreateRootFromDOMContainer方法，其主要作用在不是服务端渲染的情况下删除container当中的子节点，然后调用createLegacyRoot方法。

createLegacyRoot方法中主要是使用ReactDOMBlockingRoot这个类来创建一个实例，这个实例最后会赋值给container._reactRootContainer

```
function ReactDOMBlockingRoot(
  container: Container,
  tag: RootTag,
  options: void | RootOptions,
) {
  // tag => 0 => legacyRoot
  // container => <div id="root"></div>
  // container._reactRootContainer = {_internalRoot: {}}
  this._internalRoot = createRootImpl(container, tag, options);
}
```
在这个实例当中会有一个属性_internalRoot，这是使用createRootImpl方法创建出来的，这个方法的核心就是createContainer方法

```
const root = createContainer(container, tag, hydrate, hydrationCallbacks);
```
createContainer方法中主要地就是调用了createFiberRoot方法，其主要功能就是创建根节点对应的fiber对象，其中通过FiberRootNode创建FiberRoot对象，使用createHostRootFiber来创建rootFiber，然后将fiberRoot的current属性指向rootFiber，将rootFiber的stateNode属性指向fiberRoot，再为初始化updateQueue 对象
用于存放 Update 对象，Update对象用于记录组件状态的改变

```
// 创建根节点对应的 fiber 对象
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): FiberRoot {
  // 
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
  // 创建根节点对应的 rootFiber
  const uninitializedFiber = createHostRootFiber(tag);
  // 为 fiberRoot 添加 current 属性 值为 rootFiber
  root.current = uninitializedFiber;
  // 为 rootFiber 添加 stateNode 属性 值为 fiberRoot
  uninitializedFiber.stateNode = root;
  // 为 fiber 对象添加 updateQueue 属性, 
  initializeUpdateQueue(uninitializedFiber);
  // 返回 root
  return root;
}
```

以上是legacyRenderSubtreeIntoContainer方法中为_reactRootContainer属性赋值的过程，在那之后会调用updateContainer方法进行容器的渲染

在updateContainer方法当中，主要是创建一个待执行的任务update，然后使用enqueueUpdate更新任务队列，最后调用scheduleWork方法

```
  const update = createUpdate(expirationTime, suspenseConfig);
  // 将 update 对象加入到当前 Fiber 的更新队列当中 (updateQueue)
  enqueueUpdate(current, update);
  // 调度和更新 current 对象
  scheduleWork(current, expirationTime);
```

在scheduleWork方法当中，首次渲染为同步非批量更新模式，会使用performSyncWorkOnRoot方法来构建workInProgress Fiber树

```
  if (expirationTime === Sync) {
    if (
      // 检查是否处于非批量更新模式
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // 检查是否没有处于正在进行渲染的任务
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // 同步任务入口点
      performSyncWorkOnRoot(root);
    }
  // 忽略了一些初始化渲染不会得到执行的代码
}
```

在performSyncWorkOnRoot方法中，首先是调用prepareFreshStack方法来创建一个新的Fiber对象，也就是新的workInProgress对象
```
// 进入 render 阶段, 构建 workInProgress Fiber 树
function performSyncWorkOnRoot(root) {
  // 参数 root 为 fiberRoot 对象
  // 检查是否有过期的任务
  // 如果没有过期的任务 值为 0
  // 初始化渲染没有过期的任务待执行
  const lastExpiredTime = root.lastExpiredTime;
  // NoWork 值为 0
  // 如果有过期的任务 将过期时间设置为 lastExpiredTime 否则将过期时间设置为 Sync
  // 初始渲染过期时间被设置成了 Sync
  const expirationTime = lastExpiredTime !== NoWork ? lastExpiredTime : Sync;

  if (root !== workInProgressRoot || expirationTime !== renderExpirationTime) {
    // 构建 workInProgressFiber 树及rootFiber
    prepareFreshStack(root, expirationTime);
  }
  // workInProgress 如果不为 null
  if (workInProgress !== null) {
    do {
      try {
        // 以同步的方式开始构建 Fiber 对象
        workLoopSync();
        // 跳出 while 循环
        break;
      } catch (thrownValue) {
        handleError(root, thrownValue);
      }
    } while (true);
    
    if (workInProgress !== null) {
      // 这是一个同步渲染, 所以我们应该完成整棵树.
      // 无法提交不完整的 root, 此错误可能是由于React中的错误所致. 请提出问题.
      invariant(
        false,
        'Cannot commit an incomplete root. This error is likely caused by a ' +
          'bug in React. Please file an issue.',
      );
    } else {
      // 将构建好的新 Fiber 对象存储在 finishedWork 属性中
      // 提交阶段使用
      root.finishedWork = (root.current.alternate: any);
      root.finishedExpirationTime = expirationTime;
      // 结束 render 阶段
      // 进入 commit 阶段
      finishSyncRender(root);
    }
  }
}
```

在该方法中，全局变量workInProgressRoot会被赋值为当前的FiberRoot,而rootFiber也就是当前root的current属性会被createWorkInProgress用来创建新的Fiber对象，然后将rootFiber的属性都复制一份给它，形成新的workInProgress Fiber对象，它与当前的rootFiber使用alternate相连接

```
  // 建构 workInProgress Fiber 树的 Fiber 对象
  workInProgressRoot = root;
  // 构建 workInProgress Fiber 树中的 rootFiber
  workInProgress = createWorkInProgress(root.current, null);
```

现在我们回到performSyncWorkOnRoot方法当中，调用workLoopSync方法开始更新workInProgress树的所有子节点Fiber，通过判断performUnitOfWork的返回是否为null来判断是否要继续构建fiber节点

```
function workLoopSync() {
  // workInProgress 是一个 fiber 对象
  // 它的值不为 null 意味着该 fiber 对象上仍然有更新要执行
  // while 方法支撑 render 阶段 所有 fiber 节点的构建
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```
在performUnitOfWork方法中，会首先调用beginWork从父到子构建子节点，会传入workInProgress对象和rootFiber对象

```
function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  // unitOfWork => workInProgress Fiber 树中的 rootFiber
  // current => currentFiber 树中的 rootFiber
  const current = unitOfWork.alternate;
  // 存储下一个要构建的子级 Fiber 对象
  let next;
  // false
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    // 初始渲染 不执行
  } else {
    // beginWork: 从父到子, 构建 Fiber 节点对象
    // 返回值 next 为当前节点的子节点
    next = beginWork(current, unitOfWork, renderExpirationTime);
  }
  // 为旧的 props 属性赋值
  // 此次更新后 pendingProps 变为 memoizedProps
  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  // 如果子节点不存在说明当前节点向下遍历子节点已经到底了
  // 继续向上返回 遇到兄弟节点 构建兄弟节点的子 Fiber 对象 直到返回到根 Fiber 对象
  if (next === null) {
    // 从子到父, 构建其余节点 Fiber 对象
    next = completeUnitOfWork(unitOfWork);
  }
  return next;
}
```

beginWork当中，因为是首次渲染且是根节点，所以使用updateHostRoot方法开始构建子节点，首先会把current的updateQueue复制给workInProgress，然后结合nextProps执行
该次更新，并把更新后的state赋值给workInProgress.memoizedState，其中state的属性stateNode就是nextChildren,在后面就是调用reconcileChildren方法来继续渲染子节点

```
// HostRoot => <div id="root"></div> 对应的 Fiber 对象
// 找出 HostRoot 的子 ReactElement 并为其构建 Fiber 对象
function updateHostRoot(current, workInProgress, renderExpirationTime) {
  // 获取更新队列
  const updateQueue = workInProgress.updateQueue;
  // 获取新的 props 对象 null
  const nextProps = workInProgress.pendingProps;
  // 获取上一次渲染使用的 state null
  const prevState = workInProgress.memoizedState;
  // 获取上一次渲染使用的 children null
  const prevChildren = prevState !== null ? prevState.element : null;

  // 浅复制更新队列, 防止引用属性互相影响
  // workInProgress.updateQueue 浅拷贝 current.updateQueue
  cloneUpdateQueue(current, workInProgress);
  // 获取 updateQueue.payload 并赋值到 workInProgress.memoizedState
  // 要更新的内容就是 element 就是 rootFiber 的子元素
  processUpdateQueue(workInProgress, nextProps, null, renderExpirationTime);
  // 获取 element 所在对象

  const nextState = workInProgress.memoizedState;
  // 从对象中获取 element
  const nextChildren = nextState.element;
  // 获取 fiberRoot 对象
  const root: FiberRoot = workInProgress.stateNode;
  // 服务器端渲染走 if
  if (root.hydrate && enterHydrationState(workInProgress)) {
    // 忽略
  } else {
    // 客户端渲染走 else
    // 构建子节点 fiber 对象
    reconcileChildren(
      current,
      workInProgress,
      nextChildren,
      renderExpirationTime,
    );
  }
  // 返回子节点 fiber 对象
  return workInProgress.child;
}
```



```
export function reconcileChildren(
  // 旧 Fiber
  current: Fiber | null,
  // 父级 Fiber
  workInProgress: Fiber,
  // 子级 vdom 对象
  nextChildren: any,
  // 初始渲染 整型最大值 代表同步任务
  renderExpirationTime: ExpirationTime,
) {
  /**
   * 为什么要传递 current ?
   * 如果不是初始渲染的情况, 要进行新旧 Fiber 对比
   * 初始渲染时则用不到 current
   */
  // 如果就 Fiber 为 null 表示初始渲染
  if (current === null) {
    // 为当前构建的 Fiber 对象添加子级 Fiber 对象
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderExpirationTime,
    );
  }
  // 忽略了 else 的情况
}
 ```

在完成子节点渲染之后，就回到了performUnitOfWork方法中，使用completeUnitOfWork从子到父构建每个层级对应的兄弟节点，并创建真实的dom对象，最后就进入了commit阶段。

**二、commit 阶段负责根据 Fiber 节点标记 ( effectTag ) 进行相应的 DOM 操作。**

