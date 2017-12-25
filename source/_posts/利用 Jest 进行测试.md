---
title: 利用 Jest 进行测试
date: 2017-08-25
---

## [Jest](https://facebook.github.io/jest/)

常用的单元测试框架是 jasmine ，Mocha + Chai，不同于这些测试框架，jest 的集成度更高，提供的功能也更丰富，利用好 jest 所提供的功能，能大大提升测试用例的执行效率。

<!-- more -->

Jest 特点：

1. 测试用例并行执行，更高效
2. 强大的 Mock 功能
3. 内置的代码覆盖率检查，不需要在引入额外的工具
4. 集成 JSDOM，可以直接进行 DOM 相关的测试
4. 更易用简单，几乎不需要额外配置
5. 可以直接对 ES Module Import 的代码测试
6. 有快照测试功能，可对 React 等框架进行 UI 测试

## 断言库基本用法

jest 使用的断言风格与 jasmine 是类似的。

```javascript
// be and equal
expect(4 * 2).toBe(8);                      // ===
expect({bar: 'bar'}).toEqual({bar: 'baz'}); // deep equal
expect(1).not.toBe(2);

// boolean
expect(1 === 2).toBeFalsy();
expect(false).not.toBeTruthy();

// comapre
expect(8).toBeGreaterThan(7);
expect(7).toBeGreaterThanOrEqual(7);
expect(6).toBeLessThan(7);
expect(6).toBeLessThanOrEqual(6);

// Promise
expect(Promise.resolve('problem')).resolves.toBe('problem');
expect(Promise.reject('assign')).rejects.toBe('assign');

// contain
expect(['apple', 'banana']).toContain('banana');
expect([{name: 'Homer'}]).toContainEqual({name: 'Homer'});

// match
expect('NBA').toMatch(/^NB/);
expect({name: 'Homer', age: 45}).toMatchObject({name: 'Homer'});
```

相比于 Mocha 的书写风格(`expect(8).to.be.equal(8)`)，jest 的断言风格更加简洁，同时保持着优秀的可读性。

同时可以根据需要扩展自己的断言库：

```javascript
expect.extend({
  toBeEven(received) {
    const even = (received % 2 === 0);
    if (event) {
      return {
        message: () => (`expected ${received} not to be even Number`),
        pass: true,
      };
    } else {
      return {
        message: () => (`expected ${received} to be even number`),
        pass: false,
      };
    }
  },
});

expect(10).toBeEven();
expect(9).not.toBeEven();
```

## Mock Function

以上的断言基本是用在测试同步函数的返回值，如果所测试的函数存在异步逻辑。那么在测试时就应该利用 jest 的 mock function 来进行测试。通过 mock function 可以轻松地得到回调函数的调用次数、参数等调用信息，而不需要编写额外的代码去获取相关数据，让测试用例变得更可读。

```javascript
function getDouble(val, callback) {
  if(val < 0) {
    return;
  }
  setTimeout(() => {
    callback(val * val);
  }, 100);
};

const mockFn = jest.fn();
getDouble(10, mockFn);

expect(mockFn).not.toHaveBeenCalled()
setTimeout(() => {
  expect(mockFn).toHaveBeenCalledTimes(1);
  expect(mockFn).toHaveBeenCalledWith(20);
}, 110);
```

除了可以创建一个 mock function 作为回调函数以外，jest 还可以利用 mock function 跟踪对象上已有的方法。

```javascript
const api = {
  getRandom(range) {
    return Math.floor(Math.Random() * range);
  },
};

const spy = hest.spyOn(api, 'getRandomID');

api.getRandom(1000);
expect(spy).toHaveBeenCalled();
expect(spy).toHaveBeenCalledWith(1000);

spy.mockReset();
spy.mockRestore();      // 恢复原有的方法
```
## Mock 模块

某些情况下，我们要测试模块可能会依赖于另一个模块。

```javascript
// getRandom.js
export default function getRandom(range) {
  return Math.floor(Math.Random() * range);
};

// createModule.js
import getRandom from './getRandom';
export default function createModule(name) {
  return {
    name,
    id: `${name}-${getRandom(10000)}`,
  };
};
```

getRandom 返回的值是随机的，这样每次调用 createModule 时得到的 id 是无法确定的，意味着无法对 id 的值进行全等的断言测试。

为此，jest 提供了 mock 功能，让我们可以对这些依赖模块进行 mock。

在 `getRandom.js` 的同一级路径下，创建一个子目录`__mocks__`，并把新写的 mock 模块放到里面。

```javascript
//__mocks__/getRandom.js
let num = 0;
function getRandom() {
  return num;
}
getRandom.__set = function(_num) {
  num = _num;
};
export default getRandom;
```

那么在测试脚本当中，调用 `jest.mock` 方法就可实现对该模块的 mock。

```javascript
import getRandom from './getRandom';
import createModule from './createModule';
jest.mock('./getRandom');

describe('test mock', function() {
  it('test', function() {
    getRandom.__set(100);
    const module = createModule('module');
    expect(module.id).toBe('module-100');
  });
});
```

对模块进行 mock 的最大好处不仅仅在于方便地控制所依赖模块的返回值，还可以提高测试执行的效率。假如有一个模块需要调用 fs 模块进行文件读写，在进行测试时，就可以对这个模块进行 mock。那么在测试中，就不需要真正地去进行硬盘读写，提升了测试的效率。

```javascript
// __mocks__/fs.js
const path = require('path');
const fs = jest.genMockFromModule('fs');

let mockFiles = Object.create(null);
function __setMockFiles(newMockFiles) {
  mockFiles = Object.create(null);
  for (const file in newMockFiles) {
    const dir = path.dirname(file);

    if (!mockFiles[dir]) {
      mockFiles[dir] = [];
    }
    mockFiles[dir].push(path.basename(file));
  }
}

function readdirSync(directoryPath) {
  return mockFiles[directoryPath] || [];
}

fs.__setMockFiles = __setMockFiles;
fs.readdirSync = readdirSync;

export default fs;
```

除了 fs 模块，在封装与异步请求相关的接口时，也可以通过这个功能对异步请求返回的数据进行 mock，而不必要建一个 mock server 去执行真正的异步请求。

## Mock Timers

jest 除了为我们提供 mock 整个模块的功能外，还继承了对 timers mock，也就是 jest 可以劫持 `setTimout`、`setInterval`、`clearTimeout`、`clearInterval` 等方法，模拟 timer 的功能。

例如本文中的第一个例子，被测试函数中需要用到定时器，在这里就可以利用 jest.useFaceTimers 来进行 mock。这样，就不需要真正地去等待 timers 执行完才去进程断言。

```javascript
function getDouble(val, callback) {
  if(val < 0) {
    return;
  }
  setTimeout(() => {
    callback(val * val);
  }, 100);
};

jest.useFakeTimers();
const mockFn = jest.fn();
getDouble(10, mockFn);

expect(mockFn).not.toHaveBeenCalled()
jest.runAllTimers();
expect(mockFn).toHaveBeenCalledTimes(1);
expect(mockFn).toHaveBeenCalledWith(20);
jest.useRealTimers();
```

再看一个简单的 debounce 的例子。

```javascript
function debounce(fn, wait) {
  let timestamp = null, timer = null, context;
  return function(...args) {
    timestamp = +new Date();
    context = this;
    function later() {
      const last = (+new Date()) - timestamp;
      if (last < wait && last > 0) {
        clearTimer(timer);
        timer = setTimeout(later, wait - last);
      } else {
        fn.call(context, ...args);
        clearTimer(timer);
      }
    }
    if (!timer) {
      timer = setTimeout(later, wait);
    }
    
  };
}
```

测试脚本如下

```javascript
describe('debounce', function() {
  it('should be called after 100 ms', function() {
    const mockFn = jest.fn();
    const run = debounce(mockFn, 100);
    jest.useFakeTimers();
    
    run();
    
    jest.runTimersToTime(50);   // 第 50 ms
    run();
    expect(mockFn).not.toHaveBeenCalled();
    
    jest.runTimersToTime(50);   // 第 100 ms
    expect(mockFn).not.toHaveBeenCalled();
    
    jest.runTimersToTime(50);   // 第 150 ms
    expect(mockFn).toHaveBeenCalledTime(1);
    
    jest.useRealTimers();
  });
});
```

通过 mock timer，我们期待的是当定时器运行到第 150 ms 时，mockFn 才会执行一次。但这个用例会执行失败。因为当定时器运行到第 100 ms 时，mockFn 就被执行了。这是由于 jest 的 mock timer 的实现机制导致的。

jest 会将 setTimeout 替换为自带的 setTimeout 方法，该方法调用时会将对应的回调函数登记到对应的_timers列表当中。当中会通过 expiry 记录定时器的到期时间。当执行 jest.runTimersToTime(time) 时，就会进行判断 _now + time >= _timer.expiry ，如果达到过期时间，_timer.callback 就会被立即执行。

```javascript
// _timers 列表中的数据结构
{
  callback: () => callback.apply(null, args),
  expiry: _now + delay,
  interval: null,
  type: 'timeout',
}
```

因此在 jest 的 mock timer 环境下， 定时器回调函数的执行实际上已经变成了同步的了，它会在调用 jest.runTimers 这类方法时进行判断并执行符合条件的回调方法。

在定时器回调执行时，实现的时间并没有流逝，而 debounce 方法中需要通过 Date 类记录调用时间。所以无论是在第 50 ms 还是第 100 ms， run 执行时的 timestamp 跟第一次执行时是一样的，所以在第 100 ms时， last === 0，因此 mockFn 被执行。

mock timer 能模拟定时器的行为，但并不是真正地加速时间运行，所以通过 Date 获取的时间不会跟着一起增加。当被测模块需中还需要调用 Date 时，就还需要对 Date 进行模拟。为此我们可以利用 mockdate 这个 npm 包。

mockdate 通过修改全局的 Date 类，达到控制 Date 的时间的目的。

因此测试用例需要作调整，在调用 jest.runTimersToTime 之前先修改 Date 的当前时间。

```javascript
import MockDate from 'mockdate';
let now = +new Date();
function fastforward(time) {
  now += time;
  MockDate.set(now);
  jest.runTimersTo(time);
}

describe('debounce', function() {
  it('should be called after 100 ms', function() {
    const mockFn = jest.fn();
    const run = debounce(mockFn, 100);
    jest.useFakeTimers();
    
    run();
    
    fastforward(50);   // 第 50 ms
    run();
    expect(mockFn).not.toHaveBeenCalled();
    
    fastforward(50);   // 第 100 ms
    expect(mockFn).not.toHaveBeenCalled();
    
    fastforward(50);   // 第 150 ms
    expect(mockFn).toHaveBeenCalledTime(1);
    
    jest.useRealTimers();
    MockDate.reset();
  });
});
```

在 mock timer 的帮助下，我们可以测试在实际使用时可能会用到很长时间间隔定时器的模块，例如一个跨天的倒计时模块。它可以方便地让测试覆盖到代码的每一个分支。

## 总结

代码测试能够保障代码的质量和功能，在开发过程进行测试能够提前发现 bug，在进行代码维护、移植或重构时，测试能够保障代码功能的完整性。对于一些复用度高、需要长期维护的公用代码来说，利用测试来进行质量保障是非常有必要的。


by [ELCARIM](https://github.com/elcarim5efil)
