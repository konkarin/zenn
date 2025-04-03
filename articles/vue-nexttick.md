---
title: "nextTickを正しく理解する〜Vueとマイクロタスク〜"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "javascript"]
published: false
publication_name: comm_vue_nuxt
---

::: message
本記事は以下のバージョンで検証しています。

- Vue: 3.5.13
:::

突然ですがこのVueコンポーネントでボタンをクリックした時、どの順番でログが出力されると思いますか？分かる人はこの記事を読む必要はありません！解答は最後の方に書いてあります。

```vue:App.vue
<script setup>
import { ref, nextTick } from 'vue';

const count = ref(0);

function increment() {
  count.value++;

  nextTick(() => {
    console.log('nextTick');
  });

  setTimeout(() => {
    console.log('setTimeout');
  }, 0);

  Promise.resolve().then(() => {
    console.log('then');
  });
}
</script>

<template>
  <button @click="increment">count is {{ count }}</button>
</template>
```

https://play.vuejs.org/#eNp9kk9v2zAMxb8KoUscNHAGbKfOCfYHPWyHrWhz1CVVmVStTBkSlQYw/N1LWXWaQ5GjyPekHx/Vq59dVx8SqmvVRBNsxxCRU7fWZNvOB4YeAu4WQHjkjTUvMMAu+BZmYpp916TJeIoMxidiWGVx9WU+NnaJDFtPYMkEbJG4mkOvCYq4PmxdwqurUQunByoRrdZFl5UUvcPa+X01mySzfD/AUJ6BTLyxLfrEF8wfosm+gHdQgFsZyUasA4rhgNW85iekC7fl9jnGoKlZlgAlOjkwtp3bMsoJoHlIzBLED+MEf6XVKRGt1iU5G6Hv31MchmZZHOJulqer1EJxFIyd3dfP0ZMsbWTTyvi2sw7D/y4HHrW6nqi12jrnX/+ONQ4JF1PdPKF5+aT+HI+5ptWtpIHhgFqderwNexTo3L65/yf7OGu2/jE5UV9o3uV8U2Yssl+JHgX7TDfS/hm/nqX9Jt4cGSlOQ2XQsrsyt3zC3xdG/8D9Wn8bfbIpNbwB3T/7mA==

## 俺たちは雰囲気で`nextTick`を使っている

`nextTick`、たまに使いますが使うときは雰囲気で使いがちですよね。少なくとも私はそうです。
今日は雰囲気で`nextTick`を使うのを卒業しましょう！

ひとまず、公式ドキュメントを見てみます。

> 次の DOM 更新処理を待つためのユーティリティーです。
> 
> Vue でリアクティブな状態を変更したとき、その結果の DOM 更新は同期的に適用されません。その代わり、Vue は「次の tick」まで更新をバッファリングし、どれだけ状態を変更しても各コンポーネントの更新が一度だけであることを保証します。
> 
> 状態を変更した直後に `nextTick()` を使用すると、DOM 更新が完了するのを待つことができます。引数としてコールバックを渡すか、戻り値の Promise を使用できます。

https://ja.vuejs.org/api/general.html#nexttick

`nextTick`は次のDOM更新処理を待つためのユーティリティです。では、`nextTick`はどうやってDOM更新の完了を待っているのでしょうか？

### `nextTick`の実装を見てみる

https://github.com/vuejs/core/blob/v3.5.13/packages/runtime-core/src/scheduler.ts#L56-L62 

これだけ見てもわからないですね😇

## `nextTick`を正しく理解するための前提知識

正しく理解するうえで、前提となる知識がいくつかあります。

1. Promiseと`then`
2. イベントループとタスク・マイクロタスク
3. Vueとマイクロタスクの関係

それぞれ見ていきましょう！

### Promiseと`then`

PromiseはJavaScriptにおける非同期な振る舞いの基本的な概念です。すべて説明しようとするととてつもない量になるので、ここでは`then`の一部の振る舞いだけピックアップします。

`then`はPromiseオブジェクトのプロトタイプメソッドで、新しいPromiseを返します。このPromiseは`then` の**引数の関数の実行完了とともに解決**されます。
例えば、以下のコードの1つ目の`then`の引数の関数は実行完了に約1秒かかるので、1つ目の`then`が返すPromiseが解決されるのは1秒後です。その直後に2つ目の`then`の引数の関数が実行されます。

```jsx
// p1は解決済みのPromise
const p1 = Promise.resolve()

// p2は1秒後に解決されるPromise
const p2 = p1.then(() => {
  const startTime = Date.now();
  
  while (Date.now() - startTime < 1000) {} // 1秒間ループ
  
  return 'done!'
})

// p2が解決されたらすぐに引数の関数が実行される
p2.then((value) => {
  console.log(value) // -> done!
})
```

`then`の振る舞いは、いくつかパターンがあるので詳しくはMDNをご参照ください。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/then#%E8%BF%94%E5%80%A4

### イベントループとタスク・マイクロタスク

次にイベントループとタスク・マイクロタスクです。あまり聞き慣れない人もいるかもしれませんが、これもJavaScriptにおける重要な仕組みです。

- イベントループ：無限ループでJavaScriptエンジンはタスクを待機し、それらを実行し、また次のタスクを待機します。
- タスク：`setTimeout()`、scriptタグの読み込み、イベントリスナーなど
- マイクロタスク：`Promise.prototype.then()`, `queueMicrotask()` など

https://developer.mozilla.org/ja/docs/Web/API/HTML_DOM_API/Microtask_guide#%E3%82%BF%E3%82%B9%E3%82%AF%E3%81%A8%E3%83%9E%E3%82%A4%E3%82%AF%E3%83%AD%E3%82%BF%E3%82%B9%E3%82%AF


::: message

私は理解するのにとても時間がかかり、何度も挫折しながら繰り返し学び、ようやく雰囲気をつかめるようになりました。1回で理解できなくとも、めげずに何度も学んでいきましょう！

:::

タスクとマイクロタスクを簡単なサンプルコードで理解していきましょう。

```js
console.log('a')

setTimeout(() => {
  console.log('b')
}, 0)

Promise.resolve().then(() => {
  console.log('c')
})

console.log('d')
```

この出力順は **a → d → c → b** です。

JavaScriptのイベントループは **タスク → マイクロタスク → タスク → マイクロタスク → ...** の無限ループで実行されています。
`setTimeout`は第二引数のミリ秒後に引数の関数のタスクを追加します。
`then`はPromiseが解決された時にマイクロタスクを追加します。

先程のサンプルコードをタスクとマイクロタスクに分けると以下のようになります。

```jsx
// <-- Task 1
console.log('a')

setTimeout(() => {
  // <-- Task 2
  console.log('b')
  // Task 2 -->
}, 0)

Promise.resolve().then(() => {
  // <-- Microtask 1
  console.log('c')
  // Microtask 1 -->
})

console.log('d')
// Task 1 -->
```

つまり、**Task 1 → Microtask 1 → Task 2** の順番で実行されます。

#### マイクロタスクで新たに追加されたマイクロタスクはすぐに実行される

では次のコードはどうでしょうか？

```jsx
console.log('a')

setTimeout(() => {
  console.log('b')
}, 0)

Promise.resolve().then(() => {
  console.log('c')
}).then(() => {
  console.log('d')
})

console.log('e')
```

この出力順は **a → e → c → d → b** です。

マイクロタスクは、実行されているマイクロタスクが新たにマイクロタスクを追加した場合、次のタスクの実行前に追加されたマイクロタスクを実行します。これが`setTimeout`の引数の関数のよりも2つ目の`then`の引数の関数が先に実行される理由です。

分解すると以下のようになります。

```jsx
// <-- Task 1
console.log('a')

setTimeout(() => {
  // <-- Task 2
  console.log('b')
  // Task 2 -->
}, 0)

Promise.resolve().then(() => {
  // <-- Microtask 1
  console.log('c')
  // Microtask 1 -->
}).then(() => {
  // <-- Microtask 2
  console.log('d')
  // Microtask 2 -->
})

console.log('e')
// Task 1 -->
```

つまり、**Task 1 → Microtask 1 → Microtask 2 → Task 2** の順番で実行されます。
もし視覚的に理解したい場合は、[JS Visualizer 9000](https://www.jsv9000.app/?code=Ly8gPC0tIFRhc2sgMQpjb25zb2xlLmxvZygnYScpCgpzZXRUaW1lb3V0KCgpID0%2BIHsKICAvLyA8LS0gVGFzayAyCiAgY29uc29sZS5sb2coJ2InKQogIC8vIFRhc2sgMiAtLT4KfSwgMCkKClByb21pc2UucmVzb2x2ZSgpLnRoZW4oKCkgPT4gewogIC8vIDwtLSBNaWNyb3Rhc2sgMQogIGNvbnNvbGUubG9nKCdjJykKICAvLyBNaWNyb3Rhc2sgMSAtLT4KfSkudGhlbigoKSA9PiB7CiAgLy8gPC0tIE1pY3JvdGFzayAyCiAgY29uc29sZS5sb2coJ2QnKQogIC8vIE1pY3JvdGFzayAyIC0tPgp9KQoKY29uc29sZS5sb2coJ2UnKQovLyBUYXNrIDEgLS0%2B) がおすすめです。

これでイベントループやタスク・マイクロタスクの仕組みをざっくりと理解できました。


### Vueとマイクロタスク

次はVueとマイクロタスクの関係を見ていきましょう。公式ドキュメントにこんな記述があります。

> リアクティブな状態を変化させると、DOM は自動的に更新されます。しかし、DOM の更新は同期的に適用されないことに注意する必要があります。

https://ja.vuejs.org/guide/essentials/reactivity-fundamentals.html#dom-update-timing


「同期的に適用されない＝非同期」ということです。この非同期とは、「マイクロタスクで実行される」と置き換えられるので、VueはマイクロタスクでDOMを更新しています。
つまり、マイクロタスクを理解すれば、どのタイミングでDOMが更新されるかわかりますね。

例えば、`ref`が更新されるとマイクロタスクが一つ追加され、そのマイクロタスク内でDOMを更新します。

```vue
<script setup>
import { ref } from 'vue';
const count = ref(0);
</script>

<template>
  <!-- count.value++ でVueはマイクロタスクを1つ追加する -->
  <button @click="count.value++">{{ count }}</button>
</template>
```

`count.value++`をすると、`ref`の内部で`value`を更新する際に、同時にDOM更新のマイクロタスクが1つ追加されます。

`ref`のイメージは以下のコードのような感じです。

```jsx
// refの擬似コード https://ja.vuejs.org/guide/extras/reactivity-in-depth.html#how-reactivity-works-in-vue
function ref(value) {
  const refObject = {
    get value() {
      // getterでも実は色々してる
      track(refObject, 'value')
      return value
    },
    set value(newValue) {
      value = newValue
      // ↓triggerの実行から紆余曲折があって、最終的にDOMを更新するマイクロタスクが1つ追加される
      trigger(refObject, 'value')
    }
  }
  return refObject
}
```

この例は`ref`ですが、`reactive`などの他のリアクティビティAPIでも同様です。

これでVueとマイクロタスクの関係を掴めました。

## `nextTick`を理解する

前提となる知識を覚えた状態で、改めて`nextTick`の実装とその周辺コードを見てみます。

```jsx:core/packages/runtime-core/src/scheduler.ts
const resolvedPromise = Promise.resolve()
let currentFlushPromise = null

// ...

export function nextTick(fn) {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}

// ...

function queueFlush() {
  if (!currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```
[https://github.com/vuejs/core/blob/v3.5.13/packages/runtime-core/src/scheduler.ts](https://github.com/vuejs/core/blob/v3.5.13/packages/runtime-core/src/scheduler.ts)

`currentFlushPromise`には、`ref`の更新時になんやかんやで追加されたDOM更新処理に関わるPromiseが入っています。このPromiseはDOMの更新処理が完了したら解決されます。

```jsx:core/packages/runtime-core/src/scheduler.ts
// refの更新時に実行される関数
function queueFlush() {
  if (!currentFlushPromise) {
    // flushJobs ≒ DOM更新の関数
    // thenはPromiseを返す。このPromiseはflushJobsが完了したら解決される。
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```
[https://github.com/vuejs/core/blob/v3.5.13/packages/runtime-core/src/scheduler.ts#L114-L118](https://github.com/vuejs/core/blob/v3.5.13/packages/runtime-core/src/scheduler.ts#L114-L118)

そして`nextTick`は`currentFlushPromise` の解決後（＝DOM更新後）に、引数の関数を実行するマイクロタスクを追加します。これが`nextTick`がDOM更新を待つ仕組みになっています。

```jsx:core/packages/runtime-core/src/scheduler.ts
export function nextTick(fn) {
  // currentFlushPromiseはDOM更新処理のPromiseが入っている
  const p = currentFlushPromise || resolvedPromise
  
  // p.thenでflushJobsの完了後に、引数の関数を実行するマイクロタスクを追加する
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

また、`nextTick`は引数の関数があってもなくてもPromiseを返すため、`await`や`then`も使えますね（むしろこっちの使い方のほうが多いかも）。

```jsx
await nextTick()
console.log('after DOM update')

// あまり見ないがこうも書ける
nextTick().then(() => {
  console.log('after DOM update')
})
```

これで`nextTick`を理解できました！🥳

最後に冒頭のコードを見てみましょう。

```vue:App.vue
<script setup>
import { ref, nextTick } from 'vue';

const count = ref(0);

function increment() {
  count.value++;

  nextTick(() => {
    console.log('nextTick');
  });

  setTimeout(() => {
    console.log('setTimeout');
  }, 0);

  Promise.resolve().then(() => {
    console.log('then');
  });
}
</script>

<template>
  <button @click="increment">count is {{ count }}</button>
</template>
```

これはタスクとマイクロタスクに分解すると以下のようになります。

```jsx
function increment() {
	// <-- Task 1
  count.value++; // この裏でVueがMicrotask 1を追加する

  nextTick(() => {
    // <-- Microtask 3 (Microtask 1の完了後に追加される)
    console.log('nextTick');
    // Microtask 3 -->
  });

  setTimeout(() => {
    // <-- Task 2
    console.log('setTimeout');
    // Task 2 -->
  }, 0);
  
  Promise.resolve().then(() => {
    // <-- Microtask 2
    console.log('then');
    // Microtask 2 -->
  });
  // Task1 -->
}
```

ということは、**Task 1 → Microtask 1 → Microtask 2 → Microtask 3 → Task 2** の順に実行されますね。
つまり、最初の質問の解答の出力順は **then → nextTick → setTimeout** です。

## おわりに

今回は雰囲気で使いがちな`nextTick`を細かく深掘ってみました。Vueの奥深さが感じられて楽しいですね。
この記事を書きながら、複数のリアクティブな状態を変化させても更新が一度だけなのか（＝バッチ処理されるのか）、が気になったので、また調べて記事にしようと思っています。