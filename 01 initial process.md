## react的初始化过程
### Concept scan
_概念速览_

**这是一个函数组件**

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

- **第三步同时，在fiber构建时，同时执行所属组件函数，创建dom**

```javascript

    // DOM 信息暂存在 fiber.stateNode
    // 但不会立即挂载到真实 DOM
    fiber.stateNode = createDOMElement(fiber);


```


**第四步，commit阶段，渲染ui**

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

