---
title: 你可能并不需要useEffect
---
最近冲浪🏄看到去年 [BeJS](https://www.bejs.io/) conf 中一个关于 react 相关的视频，标题是 **Goodbye, useEffect**。然后又联想到新版 react doc 中有一节标题是 **You Might Not Need an Effect**，确实有必要再来看看这个 `useEffect`。 


## 背景

`useEffect` 相信写 react 同学不会陌生几乎每天都会与它打交道，在项目中也会大量使用它但是过多的 `useEffect` 也会让我们的代码变得混乱和难以维护（心智模型）。

## 什么是 useEffect

我们看最新官方文档对于 `useEffect` 定义：[useEffect](https://beta.reactjs.org/reference/react/useEffect) 是一个 React Hook，它允许您将组件与外部系统同步。没错最新的文档已经重新定义了 `useEffect`。
> 旧版 `useEffect` 定义：[useEffect](https://zh-hans.reactjs.org/docs/hooks-overview.html#effect-hook) 就是一个 Effect Hook，给函数组件增加了操作副作用的能力。它跟 `class` 组件中的 `componentDidMount`、`componentDidUpdate` 和 `componentWillUnmount` 具有相同的用途，只不过被合并成了一个 API。

新版的定义不在强调生命周期的作用（虽然可以这么使用），而是重点在**组件与外部系统同步**。

![](https://zzh-cat.oss-cn-beijing.aliyuncs.com/assets/2023-02-19-01.jpg)

### 你以为你写的 useEffect

```jsx
useEffect(() => {
  doSomething()

  return () => cleanup()
}, [whenever, these, things, change])
```

### 实际的 useEffect

```jsx
useEffect(() => {
    if (foo && bar && (baz || quo)) {
        doSomething()
    } else {
        doSomethingElse()
    }

  // oops, forgot the cleanup
}, [foo, bar, baz, quo])
```

## 你可能不需要 useEffect

我们看看 dan 大佬是如何描述：

![](https://zzh-cat.oss-cn-beijing.aliyuncs.com/assets/2023-02-19-02.jpg)

`useEffect` 是一种将 React 与一些外部系统(网络、订阅、DOM)同步的方法。如果你没有任何外部系统，只是试图用 `useEffect` 管理数据流，你就会遇到问题。

而且在最新的 React 18 中如果开启 `Strict Mode`，[useEffect](https://beta.reactjs.org/reference/react/useEffect#my-effect-runs-twice-when-the-component-mounts) 会在组件挂在的时候运行 2 次。

### 你不需要 useEffect 来转换数据

> `useEffect` ➡️`useMemo`

```jsx
const Cart = () => {
  const [items, setItems] = useState([])
  const [total, setTotal] = useState(0)

  useEffect(() => {
    setTotal(
        items.reduce((currentTotal, item) => {
            return currentTotal + item.price
        }, 0)
    )
  }, [items])

  // ...
}
```

上面代码使用 `useEffect` 来进行数据的转化，效率很低。当某些东西可以从现有的道具或状态计算出来时，不要把它放在状态中。相反，应该在渲染过程中计算它。

```jsx
const Cart = () => {
  const [items, setItems] = useState([])

  const total = items.reduce((currentTotal, item) => {
      return currentTotal + item.price
  }, 0)

  // ...
}
```

如果是昂贵的计算数据，可以使用 `useMemo` 代替

```jsx
const Cart = () => {
  const [items, setItems] = useState([])
  const total = useMemo(
      () => items.reduce((currentTotal, item) => {
      return currentTotal + item.price
  }, 0), 
  [items])

  // ...
}
```

### 在事件处理程序之间共享逻辑

> `useEffect` ➡️`eventhandler` 

```jsx
const Product = ({ onOpen, onClose }) => {
  const [isOpen, setIsOpen] = useState(false)

  useEffect(() => {
    if (isOpen) {
      onOpen()
    } else {
      onClose()
    }
  }, [isOpen])

  return (
    <div>
      <button
        onClick={() => {
          setIsOpen(!isOpen)
        }}
      >
        Toggle quick view
      </button>
    </div>
  )
}
```

在事件处理程序之间共享逻辑，更好的方式，可以使用事件处理函数：

```jsx
const Product = ({ onOpen, onClose }) => {
  const [isOpen, setIsOpen] = useState(false)

const toggleView = () => {
  const nextIsOpen = !isOpen
  setIsOpen(nextIsOpen)
  if (nextIsOpen) {
    onOpen()
  } else {
    onClose()
  }
}

  return (
    <div>
      <button onClick={toggleView}>
        Toggle quick view
      </button>
    </div>
  )
}
```

### 订阅外部存储

> `useEffect` ➡️`useSyncExternalStore` 

在以前我们都是在 `useEffect` 中添加 `EventListener` 相关的逻辑：

```jsx
function useOnlineStatus() {
  // Not ideal: Manual store subscription in an Effect
  const [isOnline, setIsOnline] = useState(true);
  useEffect(() => {
    function updateState() {
      setIsOnline(navigator.onLine);
    }

    updateState();

    window.addEventListener('online', updateState);
    window.addEventListener('offline', updateState);
    return () => {
      window.removeEventListener('online', updateState);
      window.removeEventListener('offline', updateState);
    };
  }, []);
  return isOnline;
}
```

在 react18 中提供一个更好的方式 `useSyncExternalStore`，它专门处理类似 `EventListener` 相关的逻辑：

```jsx
function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  // ✅ Good: Subscribing to an external store with a built-in Hook
  return useSyncExternalStore(
    subscribe, // React won't resubscribe for as long as you pass the same function
    () => navigator.onLine, // How to get the value on the client
    () => true // How to get the value on the server
  );
}
```

### 获取数据

> `useEffect` ➡️`renderAsYouFetch`

通常我们直接通过 `useEffect` 来获取数据:

 ```jsx
function SearchResults({ query }) {
    const [results, setResults] = useState([]);
    const [page, setPage] = useState(1);
    useEffect(() => {
        let ignore = false;
        fetchResults(query, page).then(json => {
            if (!ignore) {
                setResults(json);
            }
        });
        return () => {
            ignore = true;
        };
    }, [query, page]);

    function handleNextPageClick() {
        setPage(page + 1);
    }
    // ...
}
```

这里我们提取出一个 hooks 然后在组件中调用这个 hooks 来达到获取数据的目的：

```jsx
function useData(url) {
    const [data, setData] = useState(null);
    useEffect(() => {
        let ignore = false;
        fetch(url)
            .then(response => response.json())
            .then(json => {
                if (!ignore) {
                    setData(json);
                }
            });
        return () => {
            ignore = true;
        };
    }, [url]);
    return data;
}

function SearchResults({ query }) {
  const [page, setPage] = useState(1);
  const params = new URLSearchParams({ query, page });
  const results = useData(`/api/search?${params}`);

  function handleNextPageClick() {
    setPage(page + 1);
  }
  // ...
}
```

虽然单独使用这种方法不会像使用框架内置的数据获取机制那样高效，但是将数据获取逻辑移动到自定义 hooks 中将使之更容易在以后采用高效的数据获取策略。

### 你不需要 useEffect 来初始化应用

> `useEffect` ➡️`justCallIt`

通常我们都会使用 `useEffect` 来初始化我们的应用：

```jsx
function App() {
  // 🔴 Avoid: Effects with logic that should only ever run once
  useEffect(() => {
    loadDataFromLocalStorage();
    checkAuthToken();
  }, []);
  // ...
}
```

然而 react18 会运行 2 次这个我们之前也说过，所以在模块初始化期间和应用程序呈现之前运行它

```jsx
if (typeof window !== 'undefined') { // Check if we're running in the browser.
   // ✅ Only runs once per app load
  checkAuthToken();
  loadDataFromLocalStorage();
}

function App() {
  // ...
}
```

以上所述内容我们可能不需要 `useEffect`，当然这里也只是简单说明更多内容可以查看官方文档 [You Might Not Need an Effect](https://beta.reactjs.org/learn/you-might-not-need-an-effect)

## 参考链接

- [Goodbye, useEffect](https://www.youtube.com/watch?v=bGzanfKVFeU)
- [You Might Not Need an Effect](https://beta.reactjs.org/learn/you-might-not-need-an-effect)


