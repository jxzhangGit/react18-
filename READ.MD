# react 18 新特性

## 1. Automatic batching(自动批处理)
### 批处理
批处理是指React将多个状态更新分组为一个并重新渲染，以获得更好的性能。
在react 17中， 如果在同一个点击事件中有两个状态更新，React总是会把它们批处理成一个再重新渲染。如果运行下面的代码，每次点击时，React只执行一次渲染，尽管设置了两次状态:
```JavaScript
    function App() {
    const [count, setCount] = useState(0);
    const [flag, setFlag] = useState(false);

    function handleClick() {
        setCount(c => c + 1); // Does not re-render yet
        setFlag(f => !f); // Does not re-render yet
        // React will only re-render once at the end (that's batching!)
    }

    return (
        <div>
        <button onClick={handleClick}>Next</button>
        <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
        </div>
    );
    }
```
这对性能很有好处，因为它避免了不必要的重新渲染。它还防止组件呈现只有一个状态变量被更新的“半成品”状态。
然而，React在批量更新的时间上并不一致。例如，如果你需要获取数据，然后更新上面的handleClick中的状态，那么React不会对更新进行批处理，而是执行两个独立的更新。
``` JavaScript
    function App() {
        const [count, setCount] = useState(0);
        const [flag, setFlag] = useState(false);

        function handleClick() {
            fetchSomething().then(() => {
                // React 17 and earlier does NOT batch these because
                // they run *after* the event in a callback, not *during* it
                setCount(c => c + 1); // Causes a re-render
                setFlag(f => !f); // Causes a re-render
            });
        }

        return (
            <div>
                <button onClick={handleClick}>Next</button>
                <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
            </div>
        );
    }
```
### 自动批处理(new)
    从React 18的createRoot开始，所有的更新都会自动批处理。
    这意味着timeouts, promises, native event handlers或任何其他事件中的更新将以与React事件中的更新相同的方式进行批处理。即react在任何时候都进行批处理，这会减少渲染工作量，从而提高应用程序的性能:
``` Javascript
    function App() {
    const [count, setCount] = useState(0);
    const [flag, setFlag] = useState(false);

    function handleClick() {
        fetchSomething().then(() => {
        // React 18 and later DOES batch these:
        setCount(c => c + 1);
        setFlag(f => !f);
        // React will only re-render once at the end (that's batching!)
        });
    }

    return (
        <div>
            <button onClick={handleClick}>Next</button>
            <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
        </div>
    );
    }
```

由于 `setTimeout` 中的所有更新都是批处理的，React不会同步地呈现第一个`setState`的结果——呈现将在下一次浏览器计时时发生。所以渲染还没有发生:
``` Javascript
    handleClick = () => {
        setTimeout(() => {
            this.setState(({ count }) => ({ count: count + 1 }));

            // { count: 0, flag: false }
            console.log(this.state);

            this.setState(({ flag }) => ({ flag: !flag }));
        });
    };
```

如果不希望自动批处理，可以使用 `ReactDOM.flushSync()` (不过官方不建议这么用):
``` Javascript
    import { flushSync } from 'react-dom'; // Note: react-dom, not react

    function handleClick() {
        flushSync(() => {
            setCounter(c => c + 1);
        });
        // React has updated the DOM by now
        flushSync(() => {
            setFlag(f => !f);
        });
        // React has updated the DOM by now
    }
```

## updating......
