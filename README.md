
在上一篇测试指南中，我们介绍了**Jest 的背景、如何初始化项目、常用的匹配器语法以及钩子函数**的使用。这一篇篇将继续深入探讨 Jest 的高级特性，包括 **Mock 函数、异步请求的处理、Mock 请求的模拟、类的模拟以及定时器的模拟、snapshot 的使用**。通过这些技术，我们将能够更高效地编写和维护测试用例，尤其是在处理复杂异步逻辑和外部依赖时。


### Mock 函数


假设存在一个 `runCallBack` 函数，其作用是判断入参是否为函数，如果是，则执行传入的函数。



```
export const runCallBack = (callback) => {
  typeof callback == "function" && callback();
};

```

#### 编写测试用例


我们先尝试编写它的测试用例：



```
import { runCallBack } from './func';
test("测试 runCallBack", () => {
  const fn = () => {
    return "hello";
  };
  expect(runCallBack(fn)).toBe("hello");
});

```

此时，命令行会报错提示 `runCallBack(fn)` 执行的返回值为 `undefined`，而不是 `"hello"`。如果期望得到正确的返回值，就需要修改原始的 `runCallBack` 函数，但这种做法不符合我们的测试预期——我们不希望为了测试而改变原有的业务功能。


这时，`mock` 函数就可以很好地解决这个问题。mock 可以用来模拟一个函数，并可以自定义函数的返回值。我们可以通过 mock 函数来分析其调用次数、入参和出参等信息。


#### 使用 mock 解决问题


上述测试用例可以改为如下形式：



```
test("测试 runCallBack", () => {
  const fn = jest.fn();
  runCallBack(fn);
  expect(fn).toBeCalled();
  expect(fn.mock.calls.length).toBe(1);
});

```

这里，`toBeCalled()` 用于检查函数是否被调用过，`fn.mock.calls.length` 用于检查函数被调用的次数。


mock 属性中还有一些有用的参数：


* calls: 数组，保存着每次调用时的入参。
* instances: 数组，保存着每次调用时的实例对象。
* invocationCallOrder: 数组，保存着每次调用的顺序。
* results: 数组，保存着每次调用的执行结果。


#### 自定义返回值


`mock` 还可以自定义返回值。可以在  `jest.fn` 中定义回调函数，或者通过 `mockReturnValue`、`mockReturnValueOnce` 方法定义返回值。



```
test("测试 runCallBack 返回值", () => {
  const fn = jest.fn(() => {
    return "hello";
  });
  createObject(fn);
  expect(fn.mock.results[0].value).toBe("hello");
 
  fn.mockReturnValue('alice') // 定义返回值
  createObject(fn);
  expect(fn.mock.results[1].value).toBe("alice");
  fn.mockReturnValueOnce('x') // 定义只返回一次的返回值
  createObject(fn);
  expect(fn.mock.results[2].value).toBe("x");
  createObject(fn);
  expect(fn.mock.results[3].value).toBe("alice");
});

```

#### 构造函数的模拟


构造函数作为一种特殊的函数，也可以通过 `mock` 实现模拟。



```
// func.js
export const createObject = (constructFn) => {
  typeof constructFn == "function" && new constructFn();
};

// func.test.js
import { createObject } from './func';
test("测试 createObject", () => {
    const fn = jest.fn();
    createObject(fn);
    expect(fn).toBeCalled();
    expect(fn.mock.calls.length).toBe(1);
});

```

通过使用 `mock` 函数，我们可以更好地模拟函数的行为，并分析其调用情况。这样不仅可以避免修改原有业务逻辑，还能确保测试的准确性和可靠性。


### 异步代码


在处理异步请求时，我们期望 Jest 能够等待异步请求结束后再对结果进行校验。测试请求接口地址使用 `http://httpbin.org/get`，可以将参数通过 query 的形式拼接在 URL 上，如 `http://httpbin.org/get?name=alice`。这样接口返回的数据中将携带 `{ name: 'alice' }`，可以依此来对代码进行校验。


![](https://img2024.cnblogs.com/blog/1408181/202409/1408181-20240916222041177-811931182.png)


以下分别通过异步请求回调函数、Promise 链式调用、await 的方式获取响应结果来进行分析。


#### 回调函数类型


回调函数的形式通过 `done()` 函数告诉 Jest 异步测试已经完成。


在 `func.js` 文件中通过 `Axios` 发送 `GET` 请求：



```
const axios = require("axios");

export const getDataCallback = (url, callbackFn) => {
  axios.get(url).then(
    (res) => {
      callbackFn && callbackFn(res.data);
    },
    (error) => {
      callbackFn && callbackFn(error);
    }
  );
};

```

在 `func.test.js` 文件中引入发送请求的方法：



```
import { getDataCallback } from "./func";
test("回调函数类型-成功", (done) => {
  getDataCallback("http://httpbin.org/get?name=alice", (data) => {
    expect(data.args).toEqual({ name: "alice" });
    done();
  });
});

test("回调函数类型-失败", (done) => {
  getDataCallback("http://httpbin.org/xxxx", (data) => {
    expect(data.message).toContain("404");
    done();
  });
});

```

#### promise类型


在 `Promise` 类型的用例中，需要使用 `return` 关键字来告诉 `Jest` 测试用例的结束时间。



```
// func.js
export const getDataPromise = (url) => {
  return axios.get(url);
};

```

`Promise` 类型的函数可以通过 then 函数来处理：



```
// func.test.js
test("Promise 类型-成功", () => {
  return getDataPromise("http://httpbin.org/get?name=alice").then((res) => {
    expect(res.data.args).toEqual({ name: "alice" });
  });
});

test("Promise 类型-失败", () => {
  return getDataPromise("http://httpbin.org/xxxx").catch((res) => {
    expect(res.response.status).toBe(404);
  });
});

```

也可以直接通过 `resolves` 和 `rejects` 获取响应的所有参数并进行匹配：



```
test("Promise 类型-成功匹配对象t", () => {
  return expect(
    getDataPromise("http://httpbin.org/get?name=alice")
  ).resolves.toMatchObject({
    status: 200,
  });
});

test("Promise 类型-失败抛出异常", () => {
  return expect(getDataPromise("http://httpbin.org/xxxx")).rejects.toThrow();
});

```

#### await 类型


上述 `getDataPromise` 也可以通过 await 的形式来编写测试用例：



```
test("await 类型-成功", async () => {
  const res = await getDataPromise("http://httpbin.org/get?name=alice");
  expect(res.data.args).toEqual({ name: "alice" });
});

test("await 类型-失败", async () => {
  try {
    await getDataPromise("http://httpbin.org/xxxx")
  } catch(e){
    expect(e.status).toBe(404)
  }
});

```

通过上述几种方式，可以有效地编写异步函数的测试用例。`回调函数`、`Promise 链式调用`以及 `await` 的方式各有优劣，可以根据具体情况选择合适的方法。


### Mock 请求/类/Timers


在前面处理异步代码时，是根据真实的接口内容来进行校验的。然而，这种方式并不总是最佳选择。一方面，每个校验都需要发送网络请求获取真实数据，这会导致测试用例执行时间较长；另一方面，接口格式是否满足要求是后端开发者需要着重测试的内容，前端测试用例并不需要涵盖这部分内容。


在之前的函数测试中，我们使用了 `Mock` 来模拟函数。实际上，`Mock` 不仅可以用来模拟函数，还可以模拟网络请求和文件。


#### Mock 网络请求


Mock 网络请求有两种方式：一种是直接模拟发送请求的工具（如 `Axios`），另一种是模拟引入的文件。


##### 直接模拟 Axios


首先，在 request.js 中定义发送网络请求的逻辑：



```
import axios from "axios";

export const fetchData = () => {
  return axios.get("/").then((res) => res.data);
};

```

然后，使用 `jest` 模拟 axios 即  `jest.mock("axios")`，并通过 `axios.get.mockResolvedValue` 来定义响应成功的返回值：



```
const axios = require("axios");
import { fetchData } from "./request";

jest.mock("axios");
test("测试 fetchData", () => {
  axios.get.mockResolvedValue({
    data: "hello",
  });
  return fetchData().then((data) => {
    expect(data).toEqual("hello");
  });
});

```

##### 模拟引入的文件


如果希望模拟 `request.js` 文件，可以在当前目录下创建 `__mocks__` 文件夹，并在其中创建同名的 `request.js` 文件来定义模拟请求的内容：



```
// __mocks__/request.js
export const fetchData = () => {
  return new Promise((resolve, reject) => {
    resolve("world");
  });
};

```

使用 `jest.mock('./request')` 语法，`Jest` 在执行测试用例时会自动将真实的请求文件内容替换成 `__mocks__/request.js` 的文件内容：



```
// request.test.js
import { fetchData } from "./request";
jest.mock("./request");

test("测试 fetchData", () => {
  return fetchData().then((data) => {
    expect(data).toEqual("world");
  });
});

```

如果部分内容需要从真实的文件中获取，可以通过 `jest.requireActual()` 函数来实现。取消模拟则可以使用  `jest.unmock()`。


#### Mock 类


假设在业务场景中定义了一个工具类，类中有多个方法，我们需要对类中的方法进行测试。



```
// util.js
export default class Util {
  add(a, b) {
    return a + b;
  }
  create() {}
}

// util.test.js
import Util from "./util";
test("测试add方法", () => {
  const util = new Util();
  expect(util.add(2, 5)).toEqual(7);
});

```

此时，另一个文件如 `useUtil.js` 也用到了 `Util` 类：



```
// useUtil.js
import Util from "./util";

export function useUtil() {
  const util = new Util();
  util.add(2, 6);
  util.create();
}

```

在编写 `useUtil` 的测试用例时，我们只希望测试当前文件，并不希望重新测试 `Util` 类的功能。这时也可以通过 `Mock` 来实现。


##### 在  `__mock__` 文件夹下创建模拟文件


可以在 `__mock__` 文件夹下创建 `util.js` 文件，文件中定义模拟函数：



```
// __mock__/util.js
const Util = jest.fn()
Util.prototype.add = jest.fn()
Util.prototype.create = jest.fn();
export default Util;

// useUtil.test.js
jest.mock("./util");
import Util from "./util";
import { useUtilFunc } from "./useUtil";

test("useUtil", () => {
  useUtilFunc();
  expect(Util).toHaveBeenCalled();
  expect(Util.mock.instances[0].add).toHaveBeenCalled();
  expect(Util.mock.instances[0].create).toHaveBeenCalled();
});

```

##### 在当前 `.test.js` 文件定义模拟函数


也可以在当前 `.test.js` 文件中定义模拟函数：



```
// useUtil.test.js
import { useUtilFunc } from "./useUtil";
import Util from "./util";
jest.mock("./util", () => {
  const Util = jest.fn();
  Util.prototype.add = jest.fn();
  Util.prototype.create = jest.fn();
  return Util
});
test("useUtil", () => {
  useUtilFunc();
  expect(Util).toHaveBeenCalled();
  expect(Util.mock.instances[0].add).toHaveBeenCalled();
  expect(Util.mock.instances[0].create).toHaveBeenCalled();
});

```

这两种方式都可以模拟类。


#### Timers


在定义一些功能函数时，比如防抖和节流，经常会使用 `setTimeout` 来推迟函数的执行。这类功能也可以通过 `Mock` 来模拟测试。



```
// timer.js
export const timer = (callback) => {
  setTimeout(() => {
    callback();
  }, 3000);
};

```

##### 使用 `done` 异步执行


一种方式是使用 `done` 来异步执行：



```
import { timer } from './timer'

test("timer", (done) => {
  timer(() => {
    done();
    expect(1).toBe(1);
  });
});

```

##### 使用 Jest 的 timers 方法


另一种方式是使用 `Jest` 提供的 `timers` 方法，通过 `useFakeTimers` 启用假定时器模式，`runAllTimers` 来手动运行所有的定时器，并使用 `toHaveBeenCalledTimes` 来检查调用次数：



```
beforeEach(()=>{
    jest.useFakeTimers()
})

test('timer测试', ()=>{
    const fn = jest.fn();
    timer(fn);
    jest.runAllTimers();
    expect(fn).toHaveBeenCalledTimes(1);
})

```

此外，还有 `runOnlyPendingTimers` 方法用来执行当前位于队列中的 timers，以及 `advanceTimersByTime` 方法用来快进 X 毫秒。


例如，在存在嵌套的定时器时，可以通过 `advanceTimersByTime` 快进来模拟：



```
// timer.js
export const timerTwice = (callback) => {
  setTimeout(() => {
    callback();
    setTimeout(() => {
      callback();
    }, 3000);
  }, 3000);
};

// timer.test.js
import { timerTwice } from "./timer";
test("timerTwice 测试", () => {
  const fn = jest.fn();
  timerTwice(fn);
  jest.advanceTimersByTime(3000);
  expect(fn).toHaveBeenCalledTimes(1);
  jest.advanceTimersByTime(3000);
  expect(fn).toHaveBeenCalledTimes(2);
});

```

无论是模拟网络请求、类还是定时器，`Mock` 都是一个强大的工具，可以帮助我们构建可靠且高效的测试用例。


#### snapshot


假设当前存在一个配置，配置的内容可能会经常变更，如下所示：



```
export const generateConfig = () => {
  return {
    server: "http://localhost",
    port: 8001,
    domain: "localhost",
  };
};

```

##### toEqual 匹配


如果对它进行测试用例编写，最简单的方式就是使用 `toEqual` 匹配，如下所示：



```
import { generateConfig } from "./snapshot";

test("测试 generateConfig", () => {
  expect(generateConfig()).toEqual({
    server: "http://localhost",
    port: 8001,
    domain: "localhost",
  });
});

```

但是这种方式存在一些问题：每当配置文件发生变更时，都需要修改测试用例。为了避免测试用例频繁修改，可以通过 snapshot 快照来解决这个问题。


##### toMatchSnapshot


通过 `toMatchSnapshot` 函数生成快照：



```
test("测试 generateConfig", () => {
  expect(generateConfig()).toMatchSnapshot();
});

```

第一次执行 `toMatchSnapshot` 时，会生成一个 `__snapshots__` 文件夹，里面存放着 xxx.test.js.snap 这样的文件，内容是当前配置的执行结果。


第二次执行时，会生成一个新的快照并与已有的快照进行比较。如果相同则测试通过；如果不相同，测试用例不通过，并且在命令行会提示你是否需要更新快照，如 “1 snapshot failed from 1 test suite. Inspect your code changes or press u to update them”。


按下 u 键之后，测试用例会通过，并且覆盖原有的快照。


##### 快照的值不同


如果该函数每次的值不同，生成的快照也不相同，例如每次调用函数返回时间戳：



```
export const generateConfig = () => {
  return {
    server: "http://localhost",
    port: 8002,
    domain: "localhost",
    date: new Date()
  };
};

```

在这种情况下，toMatchSnapshot 可以接受一个对象作为参数，该对象用于描述快照中的某些字段应该如何匹配：



```
test("测试 generateConfig", () => {
  expect(generateConfig()).toMatchSnapshot({
    date: expect.any(Date)
  });
});

```

##### 行内快照


上述的快照是在 `__snapshots__` 文件夹下生成的，还有一种方式是通过 `toMatchInlineSnapshot` 在当前的 .test.js 文件中生成。需要注意的是，这种方式通常需要配合 `prettier` 工具来使用。



```
test("测试 generateConfig", () => {
  expect(generateConfig()).toMatchInlineSnapshot({
    date: expect.any(Date),
  });
});

```

测试用例通过后，该用例的格式如下：



```
test("测试 generateConfig", () => {
  expect(generateConfig()).toMatchInlineSnapshot({
  date: expect.any(Date)
}, `
{
  "date": Any,
  "domain": "localhost",
  "port": 8002,
  "server": "http://localhost",
}
`);
});

```

使用 `snapshot` 测试可以有效地减少频繁修改测试用例的工作量。无论配置如何变化，只需要更新一次快照即可保持测试的一致性。


本篇及上一篇文章的内容合在一起涵盖了 Jest 的基本使用和高级配置。更多有关前端工程化的内容，请参考我的其他博文，持续更新中\~


 本博客参考[milou加速器](https://xinminxuehui.org)。转载请注明出处！
