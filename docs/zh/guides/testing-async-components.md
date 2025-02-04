## 测试异步行为

在编写测试代码时你将会遇到两种异步行为：

1. 来自 Vue 的更新
2. 来自外部行为的更新

## 来自 Vue 的更新

Vue 会异步的将未生效的 DOM 批量更新，避免因数据反复变化而导致不必要的渲染。

_你可以阅读[Vue 文档](https://vuejs.org/v2/guide/reactivity.html#Async-Update-Queue)了解更多关于异步指更新的信息。_

在实践中，往往意味着你在更新会引发 DOM 变化的属性后必须使用 `Vue.nextTick（）` 来等待 Vue 完成 DOM 更新。

使用 `Vue.nextTick()` 最简单的方法是在你的测试代码中使用异步函数：

```js
// 在文件头部引用Vue库
import Vue from 'vue'

// 其它的代码片断...

// 在测试框架中，编写一个测试用例
it('button click should increment the count text', async () => {
  expect(wrapper.text()).toContain('0')
  const button = wrapper.find('button')
  button.trigger('click')
  await Vue.nextTick()
  expect(wrapper.text()).toContain('1')
})
```

## 来自外部行为的更新

在 Vue 之外最常见的一种异步行为就是在 Vuex 中进行 API 调用。以下示例将展示如何测试在 Vuex 中进行 API 调用的方法。本示例使用 Jest 运行测试并模拟 HTTP 库`axios`。可以在[这里](https://jestjs.io/docs/en/manual-mocks.html#content)找到有关 Jest Mock 的更多信息。

`axios` Mock 的实现如下所示：

```js
export default {
  get: () => Promise.resolve({ data: 'value' })
}
```

当按钮被点击时，组件将会产生一个 API 调用，并且将响应的返回内容赋值给 `value`。

```html
<template>
  <button @click="fetchResults" />
</template>

<script>
  import axios from 'axios'

  export default {
    data() {
      return {
        value: null
      }
    },

    methods: {
      async fetchResults() {
        const response = await axios.get('mock/service')
        this.value = response.data
      }
    }
  }
</script>
```

可以这样编写测试：

```js
import { shallowMount } from '@vue/test-utils'
import Foo from './Foo'
jest.mock('axios')

it('fetches async when a button is clicked', () => {
  const wrapper = shallowMount(Foo)
  wrapper.find('button').trigger('click')
  expect(wrapper.vm.value).toBe('value')
})
```

上面的代码代码会执行失败，这是因为我们在 `fetchResults` 方法执行完毕前就对结果进行断言。绝大多数单元测试框架都会提供一个回调来通知你测试将在何时完成。Jest 和 Mocha 都使用`done` 这个方法。我们可以将 `done` 与 `$nextTick` 或 `setTimeout` 结合使用，以确保在进行断言前已经处理完所有的 Promise 回调。

```js
it('fetches async when a button is clicked', done => {
  const wrapper = shallowMount(Foo)
  wrapper.find('button').trigger('click')
  wrapper.vm.$nextTick(() => {
    expect(wrapper.vm.value).toBe('value')
    done()
  })
})
```

setTimeout 也可以使测试通过的原因是，Promise 回调的微任务队列会排在 setTimeout 回调的微任务队列之前。这意味着当 setTimeout 回调执行时，微任务队列上的所有 Promise 回调已经被执行过了。另一方面，`$nextTick` 也存在调度微任务的情况，但是由于微任务队列是先进先出的，因此也保证了在进行断言时已经处理完所有的 Promise 回调。请参阅[此处]（https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/）了解更多详细说明。

另外一个使用 `async` 方法的解决方案是使用 npm 仓库中的 `flush-promises`。`flush-promises` 会刷新所有处于 pending 状态或 resolved 状态的 Promise。你可以用 `await` 语句来等待 `flushPromises` 刷新 Promise 的状态，这样可以提升你代码的可读性。

修改以后的测试代码：

```js
import { shallowMount } from '@vue/test-utils'
import flushPromises from 'flush-promises'
import Foo from './Foo'
jest.mock('axios')

it('fetches async when a button is clicked', async () => {
  const wrapper = shallowMount(Foo)
  wrapper.find('button').trigger('click')
  await flushPromises()
  expect(wrapper.vm.value).toBe('value')
})
```

相同的技术细节也可以应用在处理 Vue Actions 上，默认情况下，它也会返回一个 Promise。
