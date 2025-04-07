## react的初始化过程
### Concept scan
_概念速览_

**这是一个函数组件**

它只有三层结构： `div.container` → `div.header` → `button` 

```javascript

function App() {
  const [count, setCount] = useState(0);

  return (
    <div className="container">
      <div className="header">
        <button onClick={() => setCount(count + 1)}>
          Count: {count}
        </button>
      </div>
    </div>
  );
}

```
**第一步，经过babel 转译为如下代码结构：**

jsx被转译成了`createElement` 函数，同样的三层结构，和jsx的三层结构一一对应。

```javascript

function App() {
  const [count, setCount] = useState(0);

  return React.createElement(
    'div', 
    { className: "container" },
    
    // 第一个子元素 - header div
    React.createElement(
      'div', 
      { className: "header" },
      
      // header 的子元素 - button
      React.createElement(
        'button', 
        { onClick: () => setCount(count + 1) },
        
        // button 的文本子元素
        'Count: ', 
        count
      )
    ),
  );
}
```

**第二步，生成react element树**

`react element` 树，就是`createElement`执行之后，得到的实体对象，它是一个树状结构的`virtual dom`，依然是三层结构。

***这是一份react内部需要的数据，`fiber`就是在它的基础上构建的。***

```javascript

{
  $$typeof: Symbol.for('react.element'),
  type: 'div',
  props: {
    className: "container",
    children: [
      // 第一个子元素 - header div
      {
        $$typeof: Symbol.for('react.element'),
        type: 'div',
        props: {
          className: "header",
          children: {
            $$typeof: Symbol.for('react.element'),
            type: 'button',
            props: {
              onClick: () => setCount(count + 1),
              children: ['Count: ', count]
            }
          }
        }
      }
    ]
  }
}


```

**第三步，在Reconciliation阶段构建fiber链表**

只要有更新需求，react就会重构`fiber`链表，`diff`算法就发生在这一阶段。
注意，`fiber`不再是树状结构，而是链表结构，这样的结构是为了提升效率和性能。


```javascript

// 根 Fiber 节点
rootFiber = {
  tag: HostRoot,
  type: null,
  child: appFiber
}

// App 组件 Fiber
appFiber = {
  tag: FunctionComponent,
  type: App,
  return: rootFiber,
  child: containerFiber
}

// 第一级 div (container)
containerFiber = {
  tag: HostComponent,
  type: 'div',
  return: appFiber,
  child: headerFiber,
  sibling: contentFiber
}

// header div
headerFiber = {
  tag: HostComponent,
  type: 'div',
  return: containerFiber,
  child: buttonFiber
}

// button
buttonFiber = {
  tag: HostComponent,
  type: 'button',
  return: headerFiber,
  // 文本节点
  child: textFiber 
}

```

 ***第三步同时，在fiber构建时，同时执行所属组件函数，创建dom***

我们写的组件，就是在这一阶段被执行，业务逻辑将在这一步推入js引擎执行栈内。组件内`return`的dom，会暂时存放在`stateNode`变量内。

此时，dom不会渲染。

```javascript

    // DOM 信息暂存在 fiber.stateNode
    // 但不会立即挂载到真实 DOM
    fiber.stateNode = createDOMElement(fiber);


```


**第四步，commit阶段，渲染ui**

当`fiber`链表全部遍历完成之后，才会进入这个阶段，本阶段完全为了渲染UI而服务。
`useEffect`会在commit 阶段结束后执行。

```javascript
function commitRoot(root) {
  // 深度优先遍历 Fiber 树
  function commitRecursive(fiber) {
    if (!fiber) return;

    // 1. 先处理子节点 (深度优先)
    if (fiber.child) {
      commitRecursive(fiber.child);
    }

    // 2. 执行当前节点的 DOM 操作
    commitDOMEffects(fiber);

    // 3. 处理兄弟节点
    if (fiber.sibling) {
      commitRecursive(fiber.sibling);
    }
  }

  // 从根 Fiber 开始遍历
  commitRecursive(root.current);
}

```