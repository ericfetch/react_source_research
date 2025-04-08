## react的初始化过程
### Concept scan
_概念速览_

#### **这是一个函数组件**

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
#### **第一步，经过babel 转译为如下代码结构：**

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

#### **第二步，生成react element树**

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

#### **第三步，在Reconciliation阶段构建fiber链表**

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


#### **第四步，commit阶段，渲染ui**

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

### 更多细节解析
将这一过程再向深展开，了解react是如何实现每个步骤的，有利于我们更好地理解react的工作原理，在写代码时，会更清晰地明白自己写的每一行业务代码在react内部有何影响。
如此，才能下笔如有神，更加灵活地运用react。

#### **第一步，jsx 转化 createElement**
jsx 是一种语法糖，它会被 babel 转译为 `createElement` 函数。
babel的实现细节，在工程化的进阶分享中，会详细介绍，此处与react无关。
createElement在转化时，对应不同的元素类型，会有不同的转化逻辑。
- 文本节点：直接返回文本字符串。
```javascript

// 文本节点
const textElement = React.createElement(
  'p',
  null,
  '这是一段文本'
);

```
- 元素节点：会转化为一个对象，包含元素的类型、属性、子元素等信息。
```javascript

const element = <input type="text" placeholder="Enter your name" />;

// 转化为
const element = React.createElement(
  'input',
  { type: "text", placeholder: "Enter your name" },
  null
);

```
- 组件节点：会转化为一个对象，包含组件的类型、属性、子元素等信息。
```javascript

function MyComponent() {
  return <h1>Hello from MyComponent!</h1>;
}

// 转化为
const element = React.createElement(MyComponent, null);

```

注意1：组件节点，myComponent 是一个组件函数的引用，而不是组件函数的执行结果。
注意2：要生成完整的 React 元素树，需要从最内层的， `React.reateElement` 调用开始依次向外执行。具体步骤如下：

- 最内层调用 ：先执行最内层的 React.createElement 调用，也就是创建 button 元素的调用。这个调用会返回一个表示 button 元素的 React 元素对象。
- 外层调用 ：接着执行外层的 React.createElement 调用，也就是创建 div 元素的调用。它会使用之前创建的 button 元素的 React 元素对象作为其子元素。
- 最终结果 ：当所有的 React.createElement 调用都执行完毕后，就会得到一个完整的 React 元素树，它是一个嵌套的 JavaScript 对象结构，代表了整个组件的虚拟 DOM。

#### **第二步，react element树**
react element 树，就是 `createElement` 执行之后，得到的实体对象，它是一个树状结构的虚拟 DOM。
react element 树，是react内部需要的数据，`fiber`就是在它的基础上构建的。
react element 树，是一个纯数据结构，它不包含任何与 DOM 相关的信息。

**数据结构如下：**

- `$$typeof` ：用于表明这是一个 React 元素，其值为` Symbol.for('react.element')` 。借助这个特殊的 `Symbol` ，可以防止一些安全问题，比如在反序列化时避免恶意代码注入。
- `type` ：表示元素的类型。它可以是字符串（代表原生 HTML 标签，像 'div' 、 'span' 等），也可以是 React 组件（函数组件或者类组件）的引用。
- `props` ：这是一个对象，包含了传递给元素的属性和子元素。 props 里有一个特殊的属性 children ，用于存放子元素。


所有正常的，参与UI渲染的元素，我们不必多说。

**特殊的element**
除了需要渲染的元素，react还会生成一些特殊的element，用于标识一些特殊的操作。
- `React.Fragment` ：用于包裹一组子元素，不生成对应的 DOM 节点。这是大家使用最多的特殊元素，根本原因是因为react不允许没有父元素的子元素。所有元素必须是树状结构，一个组件必须有且只有一个顶级元素，否则会报错。
```javascript
// 短语法
<>
  <p>第一个段落</p>
  <p>第二个段落</p>
</>

// 完整语法
<React.Fragment>
  <p>第一个段落</p>
  <p>第二个段落</p>
</React.Fragment>

// 转化为
{
  $$typeof: Symbol.for('react.element'),
  type: React.Fragment,
  props: {
    children: [
      {
        $$typeof: Symbol.for('react.element'),
        type: 'p',
        props: {
          children: '第一个段落'
        }
      },
      {
        $$typeof: Symbol.for('react.element'),
        type: 'p',
        props: {
          children: '第二个段落'
        }
      }
    ]
  }
}

```

- `React.StrictMode` ：只在开发环境才产生作用，会被强制渲染两次，用于检测应用程序中的问题，如不安全的生命周期方法使用等。
```javascript
[]: # ... existing code ...
- `React.StrictMode` ：用于检测应用程序中的问题，如不安全的生命周期方法使用、过时的 context API、意外的副作用、过时的 findDOMNode 使用等。
```javascript
// 使用 React.StrictMode
<React.StrictMode>
  <div>
    <p>这是一个在 StrictMode 下的段落</p>
  </div>
</React.StrictMode>

// 转化为
{
  $$typeof: Symbol.for('react.element'),
  type: React.StrictMode,
  props: {
    children: [
      {
        $$typeof: Symbol.for('react.element'),
        type: 'div',
        props: {
          children: [
            {
              $$typeof: Symbol.for('react.element'),
              type: 'p',
              props: {
                children: '这是一个在 StrictMode 下的段落'
              }
            }
          ]
        }
      }
    ]
  }
}
```
- `React.Provider` ：用于提供上下文数据，类似于 React 的 Context API。

```javascript
// 创建一个上下文对象
const MyContext = React.createContext();

// 定义一个父组件
function Parent() {
  const value = "这是上下文数据";

  return (
    <MyContext.Provider value={value}>
      <Child />
    </MyContext.Provider>
  );
}

// 定义一个子组件
function Child() {
  return (
    <div>
      {/* 使用上下文数据 */}
      <MyContext.Consumer>
        {value => <p>{value}</p>}
      </MyContext.Consumer>
    </div>
  );
}


// 转化为
{
  $$typeof: Symbol.for('react.element'),
  type: {
    // 这是 React.createContext 返回的对象，包含 Provider 和 Consumer
    Provider: [Function: Provider],
    Consumer: [Function: Consumer]
  },
  props: {
    value: "这是上下文数据",
    children: [
      {
        $$typeof: Symbol.for('react.element'),
        type: Child,
        props: {}
      }
    ]
  }
}

```

- `React.lazy` ：用于实现懒加载，只有在需要的时候才加载组件。

```javascript
// 模拟一个异步组件
const LazyComponent = React.lazy(() => import('./LazyComponent'));

function App() {
  return (
    <div>
      <React.Suspense fallback={<div>Loading...</div>}>
        <LazyComponent />
      </React.Suspense>
    </div>
  );
}

// 转化为
{
  $$typeof: Symbol.for('react.element'),
  type: 'div',
  props: {
    children: [
      {
        $$typeof: Symbol.for('react.element'),
        type: React.Suspense,
        props: {
          fallback: {
            $$typeof: Symbol.for('react.element'),
            type: 'div',
            props: {
              children: 'Loading...'
            }
          },
          children: [
            {
              $$typeof: Symbol.for('react.element'),
              type: LazyComponent,
              props: {}
            }
          ]
        }
      }
    ]
  }
}


```


#### **第三步，fiber链表**
fiber链表，是react内部构建的一个数据结构，用于描述组件的层级结构和更新过程。
fiber链表，是一个链表结构，每个节点都有一个指向父节点、子节点和兄弟节点的指针。
fiber链表，是一个纯数据结构，它不包含任何与DOM相关的信息。

```javascript

const fiber = {
  // 节点类型相关
  tag: 'FunctionComponent',
  type: MyFunctionComponent,

  // 层级关系相关
  return: parentFiber,
  child: firstChildFiber,
  sibling: nextSiblingFiber,

  // 状态和属性相关
  pendingProps: { prop1: 'value1' },
  memoizedProps: { prop1: 'oldValue1' },
  pendingState: { state1: 'newStateValue' },
  memoizedState: { state1: 'oldStateValue' },

  // 副作用相关
  effectTag: 'Update',
  nextEffect: nextEffectFiber,
  firstEffect: firstChildEffectFiber,
  lastEffect: lastChildEffectFiber,

  // 其他属性
  stateNode: domElement,
  alternate: oldFiber
};

```
每个属性都对渲染过程有一定的影响，下面我们来详细解释一下：
- tag ：表示 Fiber 节点的类型，例如 FunctionComponent 、 HostComponent 、 HostRoot 等。不同的类型对应不同的组件类型或节点类型。
- `type` ：对于 HostComponent 类型的 Fiber 节点， type 是 HTML 标签名（如 'div' 、 'span' ）；对于函数组件或类组件， type 是组件函数或类本身。 
- `return` ：指向父 Fiber 节点。
- `child` ：指向第一个子 Fiber 节点。
- `sibling` ：指向下一个兄弟 Fiber 节点。 
- `pendingProps` ：新的 props，在开始处理一个 Fiber 节点时使用。
- `memoizedProps` ：上一次渲染时的 props，用于比较前后的 props 是否有变化。
- `pendingState` ：新的 state，在处理过程中更新。
- `memoizedState` ：上一次渲染时的 state，用于比较前后的 state 是否有变化。 
- `effectTag` ：表示该 Fiber 节点需要执行的副作用类型，例如 Placement （插入）、 Update （更新）、 Deletion （删除）等。
- `nextEffect` ：指向下一个有副作用的 Fiber 节点，用于构建副作用链表。
- `firstEffect` ：指向第一个有副作用的子 Fiber 节点。
- `lastEffect` ：指向最后一个有副作用的子 Fiber 节点。 
- `stateNode` ：存储与该 Fiber 节点对应的真实 DOM 节点或组件实例。
- `alternate` ：指向当前 Fiber 节点的上一次渲染时的 Fiber 节点，用于双缓存机制，提高渲染性能。

需要注意的是，原生hooks也被存在fiber链表中，因为它们是函数组件的一部分。

##### 1. memoizedState
`memoizedState` 存储的是上一次渲染时的状态信息，在函数组件里，它存储的是 Hooks。

```javascript
Fiber节点
  └── memoizedState
      ├── 第一个链表节点（对应useState(0)）
      │   ├── state: 0
      │   ├── updateQueue: 用于存储更新函数（如 setCount）
      │   └── next: 指向下一个链表节点
      └── 第二个链表节点（对应useState('')）
          ├── state: ''
          ├── updateQueue: 用于存储更新函数（如 setText）
          └── next: null

```

对于不同的 Hooks ， `memoizedState` 存储的内容有所不同：

- useState ： memoizedState 存储的是当前状态值。例如，使用 useState(0) 时， memoizedState 初始值为 0 。当调用 setState 更新状态后， memoizedState 会更新为新的状态值。
- useReducer ： memoizedState 存储的是 reducer 返回的状态值。
- useRef ： memoizedState 存储的是 ref 对象，这个对象在组件的整个生命周期内保持不变。
- useEffect 、 useLayoutEffect ： memoizedState 存储的是副作用函数及其依赖项数组。


**自定义Hook**
本质上，不是hook，对react来说，它只是一个函数，只是其内部有原生Hook，原生hook会被react平铺出来，和组件内部的原生hook平级，共同加入hooks链表。

##### 2. pendingState
pendingState 存储的是新的状态信息，在函数组件里，当调用 setState 或者 dispatch 时，新的状态会先存储在 pendingState 中。在处理 Fiber 节点时， pendingState 会被应用到 memoizedState 上，从而更新组件的状态。



### **第四步，commit 渲染UI**
在fiber构建阶段，react会执行组件函数，生成dom，将dom暂存到fiber的stateNode属性中，且已经通过diff算法计算出组件是何类型：placement、update、deletion。
在commit阶段，只要根据这三个属性，将dom渲染到页面上。
```javascript


    //  执行当前节点的 DOM 操作
        switch (fiber.effectTag) {
            case 'Placement':
                // 插入新的 DOM 节点
                const parentDOM = fiber.return.stateNode;
                parentDOM.appendChild(fiber.stateNode);
                break;
            case 'Update':
                // 更新 DOM 节点的属性和内容
                updateDOMProperties(fiber.stateNode, fiber.alternate.props, fiber.props);
                break;
            case 'Deletion':
                // 删除 DOM 节点
                const parent = fiber.return.stateNode;
                parent.removeChild(fiber.stateNode);
                break;
        }
```

**commit的三个阶段：**
- `Before mutation 阶段` ：在真实 DOM 更新之前执行，可用于读取 DOM 更新前的信息。
- `Mutation 阶段` ：进行真实 DOM 的更新操作，像插入、更新、删除 DOM 节点。
- `Layout 阶段` ：在真实 DOM 更新之后执行，可用于读取 DOM 更新后的信息并进行一些布局相关操作。
  - useLayoutEffect 在 Layout 阶段同步执行，在 DOM 更新后但浏览器绘制前，会阻塞浏览器绘制。
  - useEffect 在 Layout 阶段之后异步执行，在浏览器完成绘制之后，不会阻塞浏览器绘制。
通常建议优先使用 useEffect ，因为它不会阻塞浏览器绘制，能提升页面性能。只有在需要同步读取 DOM 布局信息时，才使用 useLayoutEffect 。