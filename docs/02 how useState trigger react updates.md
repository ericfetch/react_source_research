# useState 触发react更新的全过程

## 概念速览

**先看demo**

demo必须有多个组件引的引用关系，这样才能构建一颗承父子级的fiber链表。
虽然是链表，我们依然可以想象它是一颗树。
记住这颗树，会让我们更好的理解更新过程。

```javascript

                   BlogHomepage (Root)
                   /             \
                  /               \
                 /                 \
    ArticlePreview (Parent)    其它组件
           |
           |
    LikeButton (Child)

```

```javascript 
import React, { useState } from 'react';

// 底层组件：点赞按钮（包含自身状态）
function LikeButton() {
  const [likes, setLikes] = useState(0);
  const [isLiked, setIsLiked] = useState(false);

  const handleLike = () => {
    // 更新点赞数
    setLikes(prevLikes => prevLikes + 1);
    
    // 切换点赞状态
    setIsLiked(!isLiked);
  };

  return (
    <div style={{ 
      border: '1px solid red', 
      padding: '10px', 
      margin: '5px' 
    }}>
      <h3>点赞组件</h3>
      <button onClick={handleLike}>
        {isLiked ? '❤️' : '🤍'} 点赞 ({likes})
      </button>
    </div>
  );
}

// 中间组件：文章预览
function ArticlePreview() {
  return (
    <div style={{ 
      border: '1px solid green', 
      padding: '10px', 
      margin: '5px' 
    }}>
      <h2>深入理解React状态更新</h2>
      <p>React的状态更新机制是前端框架中的一大亮点...</p>
      <LikeButton />
    </div>
  );
}

// 顶层组件：博客主页
function BlogHomepage() {
  return (
    <div style={{ 
      border: '2px solid blue', 
      padding: '15px' 
    }}>
      <h1>技术博客</h1>
      <ArticlePreview />
    </div>
  );
}

export default BlogHomepage;

```

### 第一步，dispatch,添加标记
当调用`setState`时，React会为组件添加一个更新标记。这是状态更新的起点，标志着需要重新渲染的信号。

```javascript

  // 1. 标记fiber需要更新
  fiber.flags |= Update;

```
### 第二步，向上标记，感染枝干
更新标记会沿着组件树向上传播，影响父组件和相关的渲染分支。这个过程称为"感染"。

```javascript

  // 2. 向上冒泡更新标记
  let parent = fiber.return;
  while (parent !== null) {
    parent.flags |= ChildDeletion;
    parent = parent.return;
  }

```

### 第三步，时间切片，合并state

合并state,不是说在代码层面，主动合并多个state的数据，而是等待多个因useState变化，产生的分支感染。

注意，这个等待，也不是主动写出来的代码，而是时间切片。

在下一个执行时机到来的时候，所有感染的分支，才会开始统一被重新构建。

而React的时间切片和调度是由Scheduler模块负责的，它是一个综合调度策略。
这里暂不延展。

贴一段代码片断：


```javascript

// React调度简化实现
class ReactScheduler {
  constructor() {
    this.messageChannel = new MessageChannel();
  }

  // 调度更新任务
  scheduleWork(callback) {
    // 使用MessageChannel调度
    this.messageChannel.postMessage(() => {
      // 执行React更新逻辑
      callback();
    });
  }
}


```



### 第四步，reconciliation阶段，重构fiber

在第二步的时候，从变化组件开始，向上感染fiber树，相像一下这是一个复杂的项目，那么被感染的树从树叶到树根，皆被感染。

fiber的重构，就是从react的顶部开始重新构建的，对没错，不是你想的只重渲染被改变的组件，是整个分支。

但是因为有了标记，所以不是所有fiber，而是被感染的组件到应用顶部。

至上向下的重构fiber树。

```javascript

// Fiber节点是否需要更新的核心判断逻辑
function shouldUpdateFiber(current, workInProgress) {
  // 1. 首次渲染，必定更新
  if (current === null) {
    return true;
  }

  // 2. Context变化
  const hasContextChanged = hasContextDiff(current, workInProgress);
  if (hasContextChanged) {
    return true;
  }

  // 3. Props是否变化
  const nextProps = workInProgress.pendingProps;
  const prevProps = current.memoizedProps;
  
  if (nextProps !== prevProps) {
    // Props发生变化
    return true;
  }

  // 4. State是否变化
  const nextState = workInProgress.memoizedState;
  const prevState = current.memoizedState;
  
  if (nextState !== prevState) {
    // State发生变化
    return true;
  }
}


```



### 第五步，diff算法，优化性能

除了被标记的组件要重构外，props变化和context变化也会引起重构。

这里要强调的是，diff算法针对的是fiber节点，而非dom



## 更多细节

了解了大概的更新机制，我们接下来了解更多细节。

### 第一步，useState 更新

非常有趣，useState的更新，并非即时的，我们这样使用useState时：

```javascript
const [value,setValue]=useState(1)

```

要注意几个关键点：

- 一个组件使用多个`useState`时，是相互隔离的
- 所谓的`setValue` 只是数组解构赋值时的重命名，实际上在fiber节点是类似于reducer 那样的`dispatch`，所以这是一个更新函数。

在组件内调用dispatch更新函数时，它会被推进一个队列，这个队列就是更新队列，多个useState，就有多个dipatch函数。

既然如此，那在什么时机被更新了呢？

答案是fiber被重构的时候，dipatch更新队列会逐一执行，并运用到组件函数。


下面是更新的源码片断：

```javascript
 // 不直接更新，而是推入队列
    queue.push(
      typeof newState === 'function' 
        ? newState(state) 
        : newState
    );

    // 调度更新
    scheduleUpdate();
  }

  function scheduleUpdate() {
    // 在更新阶段，处理队列
    if (queue.length) {
      // 取最后一个更新
      state = queue[queue.length - 1];
      
      // 清空队列
      queue.length = 0;
    }
  }

```

所以，更新的时机要比我们想得晚多了。

而且，我们道听途说的`state合并`，并没有专门的代码，而是在任务调度机制下的被动合并。

这样的机制，不会导致`fiber`树的重复构建，标记感染过程也只有一次。

### 第二步，标记感染

我想这个过程不需要有更多理解。
唯一要记住的是，在标记向上感染的过程中，是有优先级传递的，react会将这一次更新是由什么事件引起的，所对应的渲染优先级向上传播。
这一数据存在`fiber.lanes`里，传播的过程是位运算`|`，这样任一fiber节点，都能通过lanes值知道，它的子级里是否有优先级高的更新任务，如果有，则优先被调度更新。


### 第三步 调度

更新任务会封装在一个函数，传递给调度器`Scheduler`,逻辑在源码`Scheduler.js`文件里。

**调度器的核心目的是：**

- 高任务优先级，先执行
- 按时间切片执行任务，每隔一小段时间就释放执行栈，让UI渲染及时响应，尽量保障UI流畅。
- 用微任务/宏任务回调任务函数

为了实现这个核心机制，react现在的版本，主要用了两个`API`

1、`MessageChannel`,这是一个消息通道，可以在window和worker之间发消息，也可以在同一个window下跨作用域传递消息，可以用来做`事件总线`。

react利用了其`微任务`的特性，执行时机要比setTimeout早。

片断如下：
```javascript
// 源码 Scheduler.js
function requestHostCallback(callback) {
  // 1. 调度消息通道
  const channel = new MessageChannel();
  
  channel.port1.onmessage = () => {
    // 时间切片的关键
    const currentTime = getCurrentTime();
    let hasMoreWork = true;

    try {
      // 执行回调，但限制执行时间
      hasMoreWork = callback(currentTime);
    } finally {
      if (hasMoreWork) {
        // 还有剩余任务，继续调度
        postMessage();
      }
    }
  };

  // 2. 触发第一次调度
  postMessage();
}


```

2、setTimeout,这个api都知道，它是宏任务，是用来给上一个api兜底的，遇到不兼容的情况就用这个参与调度，将下一个任务函数推入js执行栈。

3、老版本用过的api:`requestIdleCallback`，被动监听浏览器的空闲时机，时机不可控，现已废弃。

在我们真实的业务逻辑里，经常要在执行栈之后，立马执行我们的某段代码，最常用的就是`setTimeOut`，但经过对react的学习，我们有了更多对微任务的理解，这对我们构建业务代码有了更多的帮助。

### 第四步，重构fiber
这个阶段，在上一章已经讲过了，只是上一章是初始化，这一章是更新。

情况会略有不同。

既然是更新，那就必然不会全量重构，这涉及到如何判断需要更新的机制。


```javascript
function shouldUpdateFiber(current, workInProgress, renderLanes) {
  // 1. 首次渲染，必定更新
  if (current === null) {
    return true;
  }

  // 2. Lanes优先级判断
  if (!includesSomeLane(renderLanes, workInProgress.lanes)) {
    return false;
  }

  // 3. Props变化
  const nextProps = workInProgress.pendingProps;
  const prevProps = current.memoizedProps;
  if (nextProps !== prevProps) {
    return true;
  }

  // 4. State变化
  const nextState = workInProgress.memoizedState;
  const prevState = current.memoizedState;
  if (nextState !== prevState) {
    return true;
  }

  // 5. Context变化
  if (hasContextChanged()) {
    return true;
  }

  // 6. 子节点是否有更新
  if (workInProgress.childLanes !== NoLanes) {
    return true;
  }

  // 7. 组件类型相关的特殊判断
  switch (workInProgress.tag) {
    case ClassComponent:
      // 类组件特殊处理
      return checkClassComponentUpdate(current, workInProgress);
    
    case FunctionComponent:
      // 函数组件Hooks变化
      return checkHooksUpdate(current, workInProgress);
  }

  return false;
}

```

其它的还好理解，第二个通过`lanes`值来判断更新与否，这一点不太好理解。
实际上这里的判断只是延缓更新，将这一fiber分支延缓到下一次任务调度时执行。
不是说它不更新，更是等会更新。
这就又涉及到另一个概念：`优先级`。

在react的调度器和fiber里都存在优先级的概念，它们的目的都是让用户使用应用流畅。
那么，谁来决定优先级的呢？
当然是用户，react通过用户的交互事件，来预先设计好了优先级分配原则。

```javascript
// React源码 ReactEventPriorities.js
const DiscreteEventPriority = 1;
const ContinuousEventPriority = 2;
const DefaultEventPriority = 3;
const IdleEventPriority = 4;

// 事件类型到优先级的映射
function getEventPriority(domEventName) {
  switch (domEventName) {
    case 'click':
    case 'keydown':
    case 'keyup':
    case 'mousedown':
    case 'mouseup':
      return DiscreteEventPriority;

    case 'drag':
    case 'dragenter':
    case 'dragexit':
    case 'dragleave':
    case 'dragover':
    case 'mousemove':
    case 'mouseenter':
    case 'mouseleave':
    case 'pointermove':
    case 'pointerenter':
    case 'pointerleave':
      return ContinuousEventPriority;

    case 'scroll':
      return DefaultEventPriority;

    default:
      return IdleEventPriority;
  }
}


```

**规个类的话，就是这样：**
- 离散事件 → SyncLane（同步）
- 连续事件 → InputContinuousLane
- 默认事件 → DefaultLane
- 空闲事件 → IdleLane

之所以要这样做，都是因为js引擎执行，会阻塞UI渲染，显然react将用户流畅性放在了首位。

### 第五步 diff

这一步和上一步fiber更新判断有些类似，但也有很多不同。

fiber更新判断更多的是判断是否需要做重构处理，是现在处理，还是等会处理。

而diff是，一旦开始处理，如何处理的问题，高效的处理。

**它有几点原则：**

- 尽可能复用老节点
- 通过key进行匹配
  
  一般来讲，只要节点类型无变化、key无变化、优先级无变化，因props\state\context导致的更新，都会复用老Fiber节点。
- 支持节点的增、删、移动
- 时间复杂度O(n)

这里还有一点猜测，我未得到验证，就是react在内部维护着一个大对象，保存着fiber节点和其对应的dom结构快照。
react能通过这个对象快速找到每个fiber所对应的dom结构，diff算法在新版本中是否仍观察dom结构，我不清楚。

