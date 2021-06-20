# react18-notes
## react 18 新特性

### 1. Automatic batching(自动批处理)
#### 批处理
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
#### 自动批处理(new)
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

### new API (startTransition)
在react18中引入了一个新的api——`startTransition`，React将允许您在状态转换期间提供可视化反馈，并在转换发生时保持浏览器响应。
在React 18之前，所有更新都被紧急渲染。即像点击按钮或输入这样的小动作可以导致屏幕上发生很多事情。这可能导致在完成前所有工作时页面冻结或挂起，造成页面在渲染时出现延迟，使输入或其他交互感觉缓慢且反应迟钝。很多时候我们不希望出现这种情况。所以需要进行两种不同的更新。第一个更新是紧急更新，更改输入字段的值，可能还会更改它周围的一些UI。第二个是一个不那么紧急的更新，以显示搜索结果。
``` Javascript
// Urgent: Show what was typed
setInputValue(input);

// Not urgent: Show the results
setSearchQuery(input);
```


---
### new API (startTransition)
在react18中引入了一个新的api——`startTransition`，将允许在状态转换期间提供可视化反馈，并在转换发生时保持浏览器响应。
#### Problem
在React 18之前，所有更新都会被紧急渲染。即像点击按钮或输入这样的小动作可以导致屏幕上发生很多事情。这可能导致在完成前所有工作时页面冻结或挂起，造成页面在渲染时出现延迟，使输入或其他交互感觉缓慢且反应迟钝。很多时候我们不希望出现这种情况。所以需要进行两种不同的更新。第一个更新是紧急更新，更改它周围的一些UI。第二个是一个不那么紧急的更新，进行后续的UI渲染。
以搜索为例，每当用户键入字符时，我们就更新输入值，并使用新值搜索结果并显示结果。对于大屏幕更新，这可能会导致页面在渲染时出现延迟，使输入或其他交互感觉缓慢且反应迟钝。并且在每次输入时都是不同的，而且可能没有明确的方法来优化它们的呈现。所以希望能够有以下功能实现渲染:
``` Javascript
// Urgent: Show what was typed
setInputValue(input);

// Not urgent: Show the results
setSearchQuery(input);
```
#### solution
为了解决上述的问题，提出了一个新的新的API`startTransition`：
``` Javascript
    import { startTransition } from 'react';


    // Urgent: Show what was typed
    setInputValue(input);

    // Mark any state updates inside as transitions
    startTransition(() => {
        // Transition: Show the results
        setSearchQuery(input);
    });
```
在`startTransition`中封装的更新被处理为非紧急的，如果出现更紧急的更新，如点击或按键，将被中断。如果一个转换被用户中断(例如，在一行中输入多个字符)，React将扔掉未完成的陈旧的呈现工作，只呈现最新的更新。
Transition可以使大多数交互保持敏捷，即使它们导致显著的UI更改。它们还可以让您避免浪费时间呈现不再相关的内容。

这点看上去和setTimeout相似：
``` Javascript
    // Show what you typed
    setInputValue(input);

    // Show the results
    setTimeout(() => {
        setSearchQuery(input);
    }, 0);
```
`startTransition`和`setTimeout`的重要的区别是`startTransition`不像`setTimeout`那样被延时执行，它立即执行。传递给`startTransition`的函数同步运行，但其中的任何更新都标记为“transitions”。React将在稍后处理更新时使用这些信息来决定如何呈现更新。这意味着要比`setTimeout`在中包装更新更早地呈现更新。在一个快速的设备上，两次更新之间几乎没有延迟。在速度较慢的设备上，延迟会更大，但UI将保持响应。另一个重要的区别是，`setTimeout`内的大屏幕更新仍然会锁定页面，只是在超时之后。如果超时触发时用户仍在输入或与页面交互，那么他们仍将被阻止与页面交互。但是标记为startTransition的状态更新是可中断的，所以它们不会锁定页面。它们允许浏览器在呈现不同组件之间的小间隙中处理事件。如果用户输入发生了变化，React将不必继续呈现用户不再感兴趣的内容。最后，因为`setTimeout`只是延迟更新，所以显示加载指示符需要编写异步代码，鲁棒性很差。通过转换，React可以跟踪暂挂状态，基于当前转换状态更新它，并让您能够在用户等待时显示加载反馈。

## updating......
