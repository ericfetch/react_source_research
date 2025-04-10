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
更新标记会沿着组件树向上传播，影响父组件和相关的渲染分支。这个过程称为"感染"，确保受影响的组件树部分都能得到更新。

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

