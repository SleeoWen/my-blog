---
layout: '[post]'
title: React18-Suspense和SuspenseList
categories: [-React]
date: 2023-05-12 17:52:23
tags:
index_img: /images/react.jpeg
---


本文介绍了 React 18 版本中 `Suspense` 组件和新增 `SuspenseList` 组件的使用以及相关属性的用法。并且和 18 之前的版本做了对比，介绍了新特性的一些优势。

一、回顾 Suspense 用法
----------------

早在 React 16 版本，就可以使用 `React.lazy` 配合 `Suspense` 来进行代码拆分，我们来回顾一下之前的用法。

1. 在编写 `User` 组件，在 `User` 组件中进行网络请求，获取数据

   `User.jsx`

   ```
   import React, { useState, useEffect } from 'react';
   
   // 网络请求，获取 user 数据
   const requestUser = id =>
       new Promise(resolve =>
           setTimeout(() => resolve({ id, name: `用户${id}`, age: 10 + id }), id * 1000)
       );
   
   const User = props => {
       const [user, setUser] = useState({});
   
       useEffect(() => {
           requestUser(props.id).then(res => setUser(res));
       }, [props.id]);
    
       return <div>当前用户是: {user.name}</div>;
   };
   
   export default User;
   ```

2. 在 App 组件中通过 `React.lazy` 的方式加载 `User` 组件（使用时需要用 `Suspense` 组件包裹起来哦）

   `App.jsx`

   ```
   import React from "react";
   import ReactDOM from "react-dom";
   
   const User = React.lazy(() => import("./User"));
   
   const App = () => {
       return (
           <>
               <React.Suspense fallback={<div>Loading...</div>}>
                   <User id={1} />
               </React.Suspense>
           </>
       );
   };
   
   ReactDOM.createRoot(document.getElementById("root")).render(<App />);
   ```

3. 效果图：

   ![][img-0]

4. 此时，可以看到 User 组件在加载出来之前会 `loading` 一下，虽然进行了代码拆分，但还是有两个美中不足的地方

    *   需要在 `User` 组件中进行一些列的操作：定义 `state` ，`effect` 中发请求，然后修改 `state`，触发 `render`

    *   虽然看到 `loading` 展示了出来，但是仅仅只是组件加载完成，内部的请求以及用户想要看到的真实数据还没有处理完成

   > Ok, 带着这两个问题，我们继续向下探索。

二、Suspense 的实现原理
----------------

### 内部流程

*   `Suspense` 让子组件在渲染之前进行等待，并在等待时显示 fallback 的内容

*   `Suspense` 内的组件子树比组件树的其他部分拥有更低的优先级

*   执行流程

    *   在 `render` 函数中可以使用异步请求数据

    *   `react` 会从我们的缓存中读取

    *   如果缓存命中，直接进行 `render`

    *   如果没有缓存，会抛出一个 `promise` 异常

    *   当 `promise` 完成后，`react` 会重新进行 `render`，把数据展示出来

    *   完全同步写法，没有任何异步 `callback`

### 简易版代码实现

*   子组件没有加载完成时，会抛出一个 `promise` 异常

*   监听 `promise`，状态变更后，更新 `state`，触发组件更新，重新渲染子组件

*   展示子组件内容

```
import React from "react";

class Suspense extends React.Component {
    state = {
        loading: false,
    };

    componentDidCatch(error) {
        if (error && typeof error.then === "function") {
            error.then(() => {
                this.setState({ loading: true });
            });
            this.setState({ loading: false });
        }
    }

    render() {
        const { fallback, children } = this.props;
        const { loading } = this.state;
        return loading ? fallback : children;
    }
}

export default Suspense;
```

三、新版 User 组件编写方式
----------------

针对上面我们说的两个问题，来修改一下我们的 `User` 组件

```
const User = async (props) => {
    const user = await requestUser(props.id);
    return <div>当前用户是: {user.name}</div>;
};
```

多希望 `User` 组件能这样写，省去了很多冗余的代码，并且能够在请求完成之前统一展示 `fallback`

但是我们又不能直接使用 `async`、`await` 去编写组件。这时候怎么办呢？

结合上面我们讲述的 `Suspense` 实现原理，那我们可以封装一层 `promise`，请求中，我们将 `promise` 作为异常抛出，请求完成展示结果。

`wrapPromise` 函数的含义：

*   接受一个 `promise` 作为参数

*   定义了 `promise` 状态和结果

*   返回一个包含 `read` 方法的对象

*   调用 `read` 方法时，会根据 `promise` 当前的状态去判断抛出异常还是返回结果。

```
function wrapPromise(promise) {
    let status = "pending";
    let result;
    let suspender = promise.then(
        (r) => {
            status = "success";
            result = r;
        },
        (e) => {
            status = "error";
            result = e;
        }
    );
    return {
        read() {
            if (status === "pending") {
                throw suspender;
            } else if (status === "error") {
                throw result;
            } else if (status === "success") {
                return result;
            }
        },
    };
}
```

使用 `wrapPromise` 重新改写一下 `User` 组件

```
// 网络请求，获取 user 数据
const requestUser = (id) =>
    new Promise((resolve) =>
        setTimeout(
            () => resolve({ id, name: `用户${id}`, age: 10 + id }),
            id * 1000
        )
    );

const resourceMap = {
    1: wrapPromise(requestUser(1)),
};

const User = (props) => {
    const resource = resourceMap[props.id];
    const user = resource.read();
    return <div>当前用户是: {user.name}</div>;
};
```

这时候可以看到界面首先展示 `loading`，请求结束后，直接将数据展示出来。不需要编写副作用代码，也不需要在组件内进行 `loading` 的判断。

![][img-1]

四、SuspenseList
--------------

上面我们讲述了 `Suspense` 的用法，那如果有多个 `Suspense` 同时存在时，我们想控制他们的展示顺序以及展示方式，应该怎么做呢？

React 中也提供了一个新的组件：`SuspenseList`

### SuspenseList 属性

`SuspenseList` 组件接受三个属性

*   `revealOrder`: 子 `Suspense` 的加载顺序

    *   forwards: 从前向后展示，无论请求的速度快慢都会等前面的先展示

    *   Backwards: 从后向前展示，无论请求的速度快慢都会等后面的先展示

    *   together: 所有的 Suspense 都准备好之后同时显示

*   tail: 指定如何显示 `SuspenseList` 中未准备好的 `Suspense`

    *   不设置：默认加载所有 Suspense 对应的 fallback

    *   collapsed：仅展示列表中下一个 Suspense 的 fallback

    *   hidden: 未准备好的项目不限时任何信息

*   children: 子元素

    *   子元素可以是任意 React 元素

    *   **当子元素中包含非 `Suspense` 组件时，且未设置 `tail` 属性，那么此时所有的 `Suspense` 元素必定是同时加载，设置 `revealOrder` 属性也无效。当设置 `tail` 属性后，无论是 `collapsed` 还是 `hidden`，`revealOrder` 属性即可生效**

    *   子元素中多个 `Suspense` 不会相互阻塞

### SuspenseList 使用

`User` 组件

```
import React from "react";

function wrapPromise(promise) {
    let status = "pending";
    let result;
    let suspender = promise.then(
        (r) => {
            status = "success";
            result = r;
        },
        (e) => {
            status = "error";
            result = e;
        }
    );
    return {
        read() {
            if (status === "pending") {
                throw suspender;
            } else if (status === "error") {
                throw result;
            } else if (status === "success") {
                return result;
            }
        },
    };
}

// 网络请求，获取 user 数据
const requestUser = (id) =>
    new Promise((resolve) =>
        setTimeout(
            () => resolve({ id, name: `用户${id}`, age: 10 + id }),
            id * 1000
        )
    );

const resourceMap = {
    1: wrapPromise(requestUser(1)),
    3: wrapPromise(requestUser(3)),
    5: wrapPromise(requestUser(5)),
};

const User = (props) => {
    const resource = resourceMap[props.id];
    const user = resource.read();
    return <div>当前用户是: {user.name}</div>;
};

export default User;
```

`App` 组件

```
import React from "react";
import ReactDOM from "react-dom";

const User = React.lazy(() => import("./User"));
// 此处亦可以不使用 React.lazy()，直接使用以下 import 方式引入也可以
// import User from "./User"

const App = () => {
    return (
        <React.SuspenseList revealOrder="forwards" tail="collapsed">
            <React.Suspense fallback={<div>Loading...</div>}>
                <User id={1} />
            </React.Suspense>
            <React.Suspense fallback={<div>Loading...</div>}>
                <User id={3} />
            </React.Suspense>
            <React.Suspense fallback={<div>Loading...</div>}>
                <User id={5} />
            </React.Suspense>
        </React.SuspenseList>
    );
};

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
```

### 使用 SuspenseList 后效果图

![][img-2]

相关链接
----

*   本文 `wrapPromise` 方法取自 [Dan Abramov](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fgaearon "https://github.com/gaearon") 的 **[frosty-hermann-bztrp](https://link.juejin.cn?target=https%3A%2F%2Fcodesandbox.io%2Fs%2Ffrosty-hermann-bztrp%3Ffile%3D%2Fsrc%2FfakeApi.js "https://codesandbox.io/s/frosty-hermann-bztrp?file=/src/fakeApi.js")**

后记
--

好了，关于 React 中 Suspense 以及 SuspenseList 组件的用法，就已经介绍完了，在 SuspenseList 使用章节，所有的代码均已贴出来了。有疑惑的地方可以说出来一起进行讨论。

文中有写的不对或不严谨的地方，欢迎大家能提出宝贵的意见，十分感谢。

如果喜欢或者有所帮助，欢迎 Star。

[img-0]:data:image/webp;base64,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

[img-1]:data:image/webp;base64,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

[img-2]:data:image/webp;base64,UklGRiIZAABXRUJQVlA4WAoAAAASAAAA/wEAfwEAQU5JTQYAAAAAAAAAAABBTk1GFgUAAAAAAAAAAP8BAH8BAGYDAAJWUDgg/gQAAPA7AJ0BKgACgAE+kUikTKWkpCIg0rgYsBIJaW7hd2Eb87zqzk3DcJU/0PCfKRLm+MjuNuI+oAfmj9WfXu/4v8P50/qP2CP1p/5fYDA/DKr6fUMaiVea6BZ06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp0dZi2AVmxc4hSdidjtTi62mxfhG9Euyq+n1DGolXmugWOzCW+HrVDgngUwEZ68QAd1cEBbLdV0Czp06dOnTp06dRZ9oVX0+oY1Eq810Czp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp06dOnTp034kcj/ZkENNVGBDg+TQET3b7JKywkdXeMllV9PqGNRKvNdAnm92eEAfrGgSIIwAY8gksFAJAJECAAP7/26YAAAAACl/hoC5q3/Bps3A3sQ7OPLXEW+6zW8f/8O/qudtcjsCpuaEuy2R+RTjGidtUXrL9FTwu5aJvs11UeFADT3aoesath3LOtTfD1uqcdx3F78j9T4ALevey/5kb2pUmH8KanoKNUv/s5P/WlpwWzqrJKzz1ArrCc9shfnRuAr/fFULdZfIIyU/JHYx2Ni3+DQG8sH6feRzl6lMG+y+sbH9yrd3XZ9x4GCyQwpAIVCJKlnsZf+3/+ohxfS/AmF8SjdZ7Yq/WmfwgqXxT7P/R0Lz+OiPtpPMro3TKbKVsO47V2wH40X21gvi15mp6UlQ8+N5cswzhIILKzjffbvns17IQ3EJZA9nnjWikRrQ45bI53yWlonGrt6ghljSCq6Tb9QvAYepKptfi3gZglssQZlygX3YBlF4hPNCfuEUZE6k/PoLvCMXoqJRCarvBFXZOJkU5jSbFOjcISJV12ZJNs/xX5YqkebH6d+pniu2KL1s/m5+NR1JOAoQi29esTenNgWqiT014I1mMmE1PhKNMT44V0o47bGnO1vtc0LG9d5+L7MVUJ28qhFkiMjOG4bteAwQQLkN9Rf4kGnEOAAAAAAAAAAAAAAAEaQ+BcoZsgR2QglZlnswtrfOFCs7A8oZDYx6it3uVW2z2wdjZZPHhsi86w5vrwznDYgAe2arRLS0WEXb7p+XBqsPXuHlCrT8SzGRa6DKTWr4+tW0uHQMCU77YeRMlFgRL9fsfeuSObECsgvg2cX1yC5DwxkWZgbbkputJTcp3AMplLyRZf+ATltATbLscVKx0YajBRnFzO72Dsd/eeTph7brGmwPdBfQ20XNubKunU9HR/etAx+4qgDhhzueAqChOYMp2mcliJAiZYHFubeZvEjv7oirDQzIo/P6RkP/YlCh9ESKqaUQ02OzBvVIu6lUILcaEBmCNb+Ovpuo8j356lvKEY7NYBjBb5ydaGuKLAHBYBCyet1hPfcPA08eAcolzBNmr5MptpJAkqzmcGx8e4iEVON+r1MfKDXfNte+AiCW7abbbdAAAAEFOTUZyBwAAKgAAMQAAigAALAAA7gcAAEFMUEgsAAAAAQ8w/xERQm3bNpK7K7X/lt7g/rvKqSP6PwH9v4XA1dhxPCQZPebrV8rxGAZWUDggJgcAANQfAJ0BKosALQA+kTyYSALRRteAASCWkAE+RfYPxx87fB1589ldx9vb/tn5W+mP+o8AeAF6u/w35U8JnWb/M+oR7T/Mf8V9xvoA/y/of9Pv9d6oP5x/uPKI8LWgB/J/6b/v/8B+UvxX/6v+Q/w37Ve4b81/uf/V/xXwGfzD+qf8X/AdqP0af2xGul2TpiZ1tPny1JRZHijDkf4JsUOqOAuFGUpNijKf3bXl1B265NzKVw1zCNHoWdlyofxRCAmnPXq8nApv5joP2iMCJroHMPCAqXV9tAbBCptAgv8ii2sEt/IZRh8aTz6Ave97yMT5I3v+ltL3H/oShACxdaMgEcJz39GDfPkAAP76S8A/xbOcqR7eOVHpHjN/y1oEaJAtZgA+1cRhm1XseIK/7VlQyML4BNNrvHo/FIvaHmzBYOjbISY0NS/tIHjIlGnJrNAs3mtWgkM5uvBqh7SR3cRQ47vVru3KtOAevioHLGio5jxW8bPp/4gacYtsVVoMdwCKy4R4Lljlr0T6I45yU/yCd/NHzZlczK2/Sibpv1zRCNyA+5L8XuZlokCSudZUsqcCxPBgohzmuq2eAVSXGQ4ZKKMmyg9wkZRPRBVLjXrnI+TfBAzW6NfxZeVUHlQFgTb1kpnEiZXdjhzAPok+8XUVPkYNw3yI/fw34GsSJOyMq/Djyx03pO4ce0SeckWBa3RKhD2Ew7UEyhNVCMQ7fh4htGW6JbSzF9mc7tBOm6OtN6G2tRb/LapGYLjh7QkHfqfDti3xtrUJiOMGxDGgPAHUbD3vG1ORR6wRL+sMGXQTGflKaFstEN3hHUdVxFnqf10fTC1QQsEZITXZK8l1PesFnMoHNv8XL6DQ2bOQImNc78HV2X1p3RsGj+4zJrbsoX65GJ+N49vPHaSfa1Vt2FiqVNdWZNlH/RhHaxH9Iy7PaFNrLz4KBHfLStNDke1E40c/fXOAZwPtMlDXohzIlARodOtioY3eRldVdAO0RopA9uy3NTBtdwfZqC4aA7ijMB32Sn2vxtajox3y1/S6Eq+LXFLqGH/yuiavKpMfIi8AoxN8Fu3V+Sz/GJFXyscyXvpKh/gQ9j43J3S8kc+DAX7m5z5Ja2Unmzsjeb3KL/wM8EhF16gyVY7DFz/aygsIMn87fxVAw89I1JwLZybQveT70Ohs5kxwGWD5k/jY8UAv/Olz5uoZbn+hCP6V4z+dR6zMKvH3KW9WmNL8l69clNav5q1mZP+Hx6sdtHMWbH1yox7gfsSbce5I9f2akaaxYb5pHeaUuT5kg4TfxRbt11X1H5fqWQRZQF0wFdU1nxNOsRBfJ0Bk2+7VfWFnEMZkiXsRpcLdqnUdRJ71RdSJS1mcgL1D2ILJctUaJTNntVot778F5dZtfH38nwn0ieFOH/LazgCTdqUga/n1R0iXlb5o3jBoj+plMEH1Sz/aFPJB2S7txbIxWdmuokLeOpZb3g9yWxygVzQ+aqe0b0Vf70gG/1GPgG2CraymVsPY5ng7mza9dZ935gcrV8W8bpcZNTlPpHcmHwk6CB0OmIuQGHp8y9kNmvIIs5AhVLfFhXbPcEMAFdQQUZNVozsiIP92YnmMw8YA8t7MnocGdoeupHmYYHzYXSBhrzARe3BRKecxlCncvNxaWnxTGAv2XAPVXQI3RUPuOlBDBJyfHa/9Iu+cJ+34S0QcPf44IMvQpdlYOoEBN8nCFd07LBwBaqC/egUiN/NIRK/fdQ5d6xZuZ5S3g/D9vtIvmKsu7d7LOcJ7D9SqmnkL/x7y/gYfEp6buq0KwGgtBe8mIXHHr2483vkla5yv2T1OEPVMkrHv/h15EJ7xdFSjeIGGvL5cIP7f2ydoVWW2CB+E3fkUXVf+5v80HZwPaz6SeWfq1ZEXj9zxpr8RMRnr4DZrx6RT/M6L0q8fCqayCSwsvmjSVFylrpdZMgvw0kdKSJARryRJov6Ycc3QkCvHyl/QCmkK16lsZa3cYY0YdQwwk6AHTSTVWkNK12UZUhN38I/6mjNXOJn9CzPNSnkPtfXF5T+hHX8gbR8zLJa/Eg6unwnvXMt7nye5/hX5gHg8MVDr4iIb8NO861X6ZO8rIS6Rc8CTnf9Ejw6SXeYnBZZ/S5aalUrdrpGi0xhuKGSZkpHuvkgQ5e/+bF9vhZBEYWqDEjDkc7WvHEMKmKxqV63weZ8qNQFT+N/evVpg68d3zHRbbRKmrvZpl4BnltyLe6Z1umKoRkJ9meOBXdfFOPywtZyc3AtOl+i1MUVzE4ehqhiSTwnc7ryS48YUD4g0NJpQHxFilw2NrgtNe45aE0/n7AITH438dTl17UeBvegJE7jXkF8cUhsTSL+tOeIOo22vTM97EDnPzc70D63IF4fXV4oOsjXMcMzBsT+iN6YBPK6pZRuEVZivBNo/Y9UezNOrMWCTRb5peK19oNNjbEwAAEFOTUYUBwAAKgAAPgAAjwAAKwAAsgcAAEFMUEgqAAAAAQ8w/xERQk3bSNIwCH+Ww2Dvr26njuj/BNSPcRo7bg9JRq/urq7EUUUIVlA4IMoGAADUHwCdASqQACwAPpE4lkiDCTfYgAEglpABPX/0n8hPOvxxedvaL9xdBH9evtP5n+wn+S8D/UP6gXp7+t+ED/K9zVMd6gXr78z/yv5kf1j0XtTjuP/ivU3/MP8Z6ld8l9O/0n0AfYB/Mf6x/uPuW+lX+D/4X+e/KL2y/RH/W/yHwEfzX+mf7/sL+iqTL/tWDpGaERt0ejAtgj8VDtsH3gShloOWhVdi/dfweV6vw33F/hPJhu+tV2V3vFm/nGIH+GQY3yo9pX8UZQL2paBk3JzTHIXcoGsC4A+qYYLDUJayEhWVO9uXCD4wefxH9V5wzO3PsodwuaR946hp5sSKcn+U2/piVBygAAD+92tVIVCQ+e3NXdfrjF4R5lhrxP+gK9YNtTD/8u1eBl3DhP4x9mHYgWzHAkApxeiZ4mc98xnfoOHnIBu3ctlT/kTDaqNLUO9aFOeJb0LHoUI9j4gcCNbtv5Zm5p0W8k9ZQUDzU41JNfwxEDU/6afGfipc8yOomW/jiYVaXQ4mvkuu0/Czs/qjUHw22R4dddzPZCi0OTrH/sLKv6UPNQ4Hps5CPAcAmSVdtbUVH1L1ZpBe/j8HnEh9OzDCxvwCHSmKFvQ0/DnpzeLABH1uxqTPQG18rfEBDbaxCGMfNo7rYLShJyLXvY+YHJ08GFZjaAeTDgxw86/EfiQ5q4zdZyEYWs/d5qER6qcPMQJVsUMugzKaAd6wHEYDqW/2GGuO4AygKiOmR/xbptwK+uC5E5+/cOW6ozM/wfG4+yO8dUFKSvp0lRE0Z1ETpfUbhdWvLJ1E9TbcEnKRePUdTIepnwIeDi2OwSyaVC226gtpC8Uqvc+nSC7OtOetJqwYRdvF/W8Wg2nfuO0sfIyOJE9FoBol9/pelWud3sQDAssf0SPMhH1gmf9oudyWbrbE69A2HJSqgHcchZaG/MM4+rAD4Shc6RpolAXX23LdA1s9DPVmLEUAIQJ9704yoInjXarqZ626d9vfFcDRbych9ez3nsSZRmY9tcOtnTOXsyvMUD2gqARItLnGTpti0yOXopDzp2/ZM0ohaxV2R8Y3KEeLYfCeeoA446nFcBEYRA/2a48wMuf9yqaMN32jFlf/GXax8FC3eRNY8DgukOfmDDyWqGQScml0NP+Me0rXW3X5lbYWkx1zmVCfz7f0AjQBSb9a3C6Cpml2Fmee+QZ9ou8jNRlm2yRylHe3LJshyVI9BU0Diz/IjLWotT7fS8Scc7fwY+D+9oaulP+aIZhSdtF8u90Qgm/m8G7x/mjypN+DdTqSIXKRyC38vmf9jTE/m9AjR99yajYVMif6u3LNpkEMTDRJeopN5lrq1hCm0Jk4k/moxvYrBbId4Eu3mPf3TJiCH7L35lkaiD2YmmLtF3kC1AP3HmS/i2s7VrGHrRrM5sggPsR4yi0/gT+ZRSm20JqfIBbkJmmKQRzq2WaaR4JUerfSkVb/lkmvRnqa09UbJOSmnYRgk8fQFNL+b7KaNYwsUnOScXdQyztPBhenF4fyL5qQcJ9Vck6xhbOXG8Nv0s5+w2YWDSQTIy3k9DNPW3TFO9pg6bsdcymhU3eZSxuMAWlnsrFxzDuB+e0DCmHDiZpU/1jNZ/gpEOjXuBRmgkHQUbl4UqBqi7PQHLRENBrjVOuXob77W4iiAutkaMTD5cAOQga4JD+0Hx/xVDHBaG0gbgIr9iddBeiGPth1XC+PGWDnpb+5zb3lxwDPmBUHpfR9btWQKQAaVKnEPns2E6dGx36bUTcllc1qIYU/RM33OLW08vFiBic2cEhtRf0yW0bMGkjAvmiVVGk03AyIErM3w31COGT6yEsU0lwIgJF2rMirqA3515ss5iTF/X0NlmZa/e22MYzsFCbgeSnCg0n4WiCXp9FuWfnto+qSzz3kK/16LLDw75S2sqqvnam/9LBVeFjnurKlDnkKIFJGBfVmmFjqeos7bJqV9KhURJVEhrvhSyXW8x5x5ZwWL90vjw8XVKU6oUqjACqPAwXgX6hlWX8X5xdeeMcp5zPWUFEPvCIA9KtazO6T7gr6KkcPdA18ks6HBxH9ARfhuS51YNzLhO89eBjY7z8qIyNJGlJP95f66CSJ9O/LrG4VEUH7d7TSVf7gWzZXW+wb/lNK7PwijvTC11UUGNUyFm+v/+OQTI6q6esEH//5bdSZfdOP6WVG1H1WdJSB7Ucg+zbKfrsk7UoXkiqux2hrY9PF2XcuV115o69LbqPNAh85fGzypNasEkYkjq5J0ysOthIZ3XWcuQloK00ZJOJyrx52th+FTEH/L0X5Z/sseH59vRi/AoxQAAAAQU5NRkIFAAAqAABKAACPAAATAAAwAgAAQUxQSB8AAAABDzD/ERGCLMAEYf6WGUyguiP6LyAo8sAMGyPrPr0QAFZQOCACBQAAFBgAnQEqkAAUAD6RPplKAo+ZpwABIJaQAW22zL+4+Efgl8OezO5T3uf2v8t/Vn+q+APpV9QL1d/g/yb4G2YD1AvWX5p/lO9T/h+6A9UH8k/3fGw0AP49/V/+B9xn0pfwn/J/y3nl+e/+P7hf8u/rf+97IPoiftuMxQ7yijbHQpTnwUlEsJ9MRHGRoq7YKcWtaylxv5UxtrGmsF3GifGgrRAwIPxfENy7QwnzXjVlvn4P5qSzgY1hqEwkdfomdF5/fOqKgJ4Yv3MAAP7/JIFe9IfQpNqg7bqGmM9swlv54eLiWdICi73bGg+g5W0zjAFjUh/NvUR0Vel1uTdhxmUYvMibMaYL4XMT2YSXuztCfUQIciGEKGzSfy+P/g1T5jufsz4I25oP03NLC8fEwkMpWgk0ILDGGQRfc4zq7TBMcjT7E5uD7fVi31M3n5ycYhp70AEk/ziqmukOyQUA404Fgz0tWAwinyhtYtN2Dd0MdZHAH5PMASCkhLpCMokbyxnSxUft9SX/YnHsD8F7JRIYFVFd2sevo+WVj2P0mmm0ypF3EPGlXfYDnvA0X/MBhCxd46ZEs6Nm+EiExEscSAcn4/o9b+7KtjblQX0V36vT9hUYy3Z6zwtOry1zoDDbOJvYJivBVMR/DtB5xPO7F92F8yC5t7csn+Z5UoHn6qVl+YWGwUE4FKt4HWq5/2J75S5yuZ1ndkOVuolADPgAe3muDiSe847NFPg0p/QOY4D7rXEaOee6vGx35NJJoFBQmjV681L+rEHf0G0H+C8NjaHZ+6AsTKLb/SheTWv9zUE8HZ+SEbLRpl9l8kzrJ/weCvNk1m4Q7nWL3ZqKqnmCHUAbnAhuQA3ByY7ieaOD869ebJR2v55VHZ90ZLUlkcX4wHOePiVS5GH2azXOra9QdR2YYMceX1qlaYSH6XeYBcgi9WOU1nZSAzmic5bJ5gJy5nlTohbTrXyfixr8YCBX//yu6boX/mmkQczMPqiziVZuee+Qaf8rGXuiDN/lh0TP6oY/+8sY9xuO5uWY+uE8qxH54VfNYHz9h48GscO3kPoz+gqR7IvyTd7xn/Bk+3lLWqMCDmGOiQwzEHeVCI8T/lXN8rsD0Xi2up8bUvrU/4sFuWvXqvOBpIHAHTmSDto5mJfANE3mqEXV/4wrvmUv5HwFP/wHBKvBLazz2oIP6OMI406bdRr9pecifzV2NUJ4G7AE+HRwM3w0PDz0DRpBs7MxdX9AmFhgwHgog4R2SCIe9SShv8WKheXZPrl1yXw8uDgdFFdZeWMdmG7zY+Y8VkjOaHJcq8bE269kkQYk4vfNVdqaLxxTSQPHnfcYoG1ylbuwJsxnWv3wWPahMfeqhYkG3zDBaVpASq/uVdw1O5tVro3jZsQ7SZyxvRJMP8UfZulMfLwDa9Trmc8MouP4o+Vppt3nFS0qNXZv1FJ6NQfyb7l63keDEweUCChUWTDoi78Uvj5UQXaJXz//Z6Ry04GCQA6xZOBcysBLPJ8d5ojV4H2zDTnpRu5D5SPar9PcZ+cwuS2x6kSZ1GUVVGCta/8E3DrJ5366pdNaAHCgXSqmLtv4IDtKhnC5yb6/clGrHLqrO+z9r4NJ3cjrqLn4QT4s6RlBFdEAi/WoRhzvQ2QknhsGU7jilinistPpP7LUV/R+jF9/At6PhOHi8ryplHu925PHp6M0FXSa5/s1lRL1ltAAAA==