---
id: concurrent-mode-patterns
title: Concurrent UI Patterns (Experimental)
permalink: docs/concurrent-mode-patterns.html
prev: concurrent-mode-suspense.html
next: concurrent-mode-adoption.html
---

>警告：
>
> このページでは**安定リリースで[まだ利用できない](/docs/concurrent-mode-adoption.html)実験的機能**を説明しています。本番のアプリケーションで React の実験的ビルドを利用しないでください。これらの機能は React の一部となる前に警告なく大幅に変更される可能性があります。
>
> このドキュメントは興味のある読者やアーリーアダプター向けのものです。React が初めての方はこれらの機能を気にしないで構いません -- 今すぐに学ぶ必要はありません。

通常、state を更新した場合、画面に即座に変化が現れることを期待します。ユーザ入力に対してアプリケーションをレスポンシブに保ちたいので、これは理に適っています。しかし、**画面に更新が現れるのを遅延**させたい場合があります。

例えば、ある画面から別の画面に切り替えたいが、次の画面に必要なコードやデータが何もロードされていないという場合、切り替え直後にロード中のインジケータのある空のページを見るのはいらだたしいものです。前の画面にもうしばらく残りたいと思うでしょう。歴史的に React ではこのパターンの実装は困難でした。並列モードはこれを行うための新たなツール群を提供します。

- [Transitions](#transitions)
  - [Wrapping setState in a Transition](#wrapping-setstate-in-a-transition)
  - [Adding a Pending Indicator](#adding-a-pending-indicator)
  - [Reviewing the Changes](#reviewing-the-changes)
  - [Where Does the Update Happen?](#where-does-the-update-happen)
  - [Transitions Are Everywhere](#transitions-are-everywhere)
  - [Baking Transitions Into the Design System](#baking-transitions-into-the-design-system)
- [The Three Steps](#the-three-steps)
  - [Default: Receded → Skeleton → Complete](#default-receded-skeleton-complete)
  - [Preferred: Pending → Skeleton → Complete](#preferred-pending-skeleton-complete)
  - [Wrap Lazy Features in `<Suspense>`](#wrap-lazy-features-in-suspense)
  - [Suspense Reveal “Train”](#suspense-reveal-train)
  - [Delaying a Pending Indicator](#delaying-a-pending-indicator)
  - [Recap](#recap)
- [Other Patterns](#other-patterns)
  - [Splitting High and Low Priority State](#splitting-high-and-low-priority-state)
  - [Deferring a Value](#deferring-a-value)
  - [SuspenseList](#suspenselist)
- [Next Steps](#next-steps)

## トランジション {#transitions}

前のページ [Suspense for Data Fetching](/docs/concurrent-mode-suspense.html) にある[こちらのデモ](https://codesandbox.io/s/infallible-feather-xjtbu)について改めて考えましょう。

"Next" ボタンをクリックしてアクティブなプロフィールを切り替えた際、既存のページデータは即座に消えて、新しい画面のためのローディングインジケータを見ることになります。これは「望ましくない」ローディング中状態と呼べるでしょう。**新しい画面に遷移する前に新しいコンテンツを待ち、これをスキップすることができれば良さそうです。**

React はこれを補助するたに `useTransition()` という新しい組み込みフックを提供します。

これは以下の 3 ステップで利用できます。

まず、実際に並列モードを利用していることを確かめます。後で[並列モードの利用開始](/docs/concurrent-mode-adoption.html)方法については述べますが、今の所は `ReactDOM.render()` の代わりに `ReactDOM.createRoot()` を使うことでこの機能が使えるということを知っていれば十分です。

```js
const rootElement = document.getElementById("root");
// Opt into Concurrent Mode
ReactDOM.createRoot(rootElement).render(<App />);
```

次に、React から `useTransition` フックをインポートする文を追加します：

```js
import React, { useState, useTransition, Suspense } from "react";
```

最後に、`App` コンポーネント内でそれを利用します：

```js{3-5}
function App() {
  const [resource, setResource] = useState(initialResource);
  const [startTransition, isPending] = useTransition({
    timeoutMs: 3000
  });
  // ...
```

**このコードはそれ自体ではまだ何もしません。**このフックの戻り値を使って state の遷移をセットアップします。`useTransition` からの戻り値は 2 つです：

* `startTransition` は関数です。これを使って、*どの* state の更新を遅延させたいのかを React に伝えます。
* `isPending` は真偽値です。React はこれを使って現在トランジションが進行中かどうかを伝えます。

このすぐ後で使ってみます。

`useTransition` に設定オブジェクトを渡したことに気をつけてください。この `timeoutMs` プロパティで**遷移が終了するまでどれだけ待てるか**を指定します。`{timeoutMs: 3000}` を渡すことで、「次のプロフィール画面がロードされるのに 3 秒以上かかったら、大きなスピナーを表示せよ、ただしそれまでは前の画面を表示しつづけていて構わない」ということを伝えています。

### トランジション内で setState をラップする {#wrapping-setstate-in-a-transition}

"Next" ボタンのクリックハンドラーは、現在のプロフィールを切り替えるための state を設定しています：

```js{4}
<button
  onClick={() => {
    const nextUserId = getNextId(resource.userId);
    setResource(fetchProfileData(nextUserId));
  }}
>
```

このステートの更新を `startTransition` でラップします。これが、もしこのステートの更新が望ましくないローディング中状態を引き起こすのであれば、**React によって遅らされても構わない**、と伝える方法です：

```js{3,6}
<button
  onClick={() => {
    startTransition(() => {
      const nextUserId = getNextId(resource.userId);
      setResource(fetchProfileData(nextUserId));
    });
  }}
>
```

**[CodeSandbox で試す](https://codesandbox.io/s/musing-driscoll-6nkie)**

何度か "Next" を押してみましょう。既に大きく違っていることが分かるでしょう。**クリックの直後に空の画面を見させられる代わりに、しばらくは前のページが表示され続けます。**データがロードされたら、React が次の画面に遷移します。

もし API のレスポンスが 5 秒かかるように変えると、React は 3 秒後に「諦めて」ともかく画面の遷移を行うことが[確認できます](https://codesandbox.io/s/relaxed-greider-suewh)。これは `useTransition()` に `{timeoutMs: 3000}` を渡したからです。もし `{timeoutMs: 60000}` を代わりに渡したら、丸々 1 分間待つことになるでしょう。

### 保留中インジケータの追加 {#adding-a-pending-indicator}

There's still something that feels broken about [our last example](https://codesandbox.io/s/musing-driscoll-6nkie). Sure, it's nice not to see a "bad" loading state. **But having no indication of progress at all feels even worse!** When we click "Next", nothing happens and it feels like the app is broken.

Our `useTransition()` call returns two values: `startTransition` and `isPending`.

```js
  const [startTransition, isPending] = useTransition({ timeoutMs: 3000 });
```

`useTransition` はもう使っています。ここで `isPending` も使うようにします。React がこの真偽値を渡してくれることで、**このトランジションの終了を待っているところかどうか**が分かります。何かが起こっているということを示すためにこれを使いましょう：

```js{4,14}
return (
  <>
    <button
      disabled={isPending}
      onClick={() => {
        startTransition(() => {
          const nextUserId = getNextId(resource.userId);
          setResource(fetchProfileData(nextUserId));
        });
      }}
    >
      Next
    </button>
    {isPending ? " Loading..." : null}
    <ProfilePage resource={resource} />
  </>
);
```

**[CodeSandbox で試す](https://codesandbox.io/s/jovial-lalande-26yep)**

ずっと良くなりました！ Next をクリックした際に、何度も押しても意味がないのでボタンが無効化されます。そして新たに表示される "Loading..." がユーザにアプリケーションがフリーズしていないということを伝えています。

### 変更のおさらい {#reviewing-the-changes}

[元の例](https://codesandbox.io/s/infallible-feather-xjtbu)以降に行った変更をもう一度すべて見てみましょう：

```js{3-5,9,11,14,19}
function App() {
  const [resource, setResource] = useState(initialResource);
  const [startTransition, isPending] = useTransition({
    timeoutMs: 3000
  });
  return (
    <>
      <button
        disabled={isPending}
        onClick={() => {
          startTransition(() => {
            const nextUserId = getNextId(resource.userId);
            setResource(fetchProfileData(nextUserId));
          });
        }}
      >
        Next
      </button>
      {isPending ? " Loading..." : null}
      <ProfilePage resource={resource} />
    </>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/jovial-lalande-26yep)**

このトランジションを加えるのに必要だったコードはわずか 7 行でした。

* `useTransition` フックをインポートして、state を更新するコンポーネント内で使いました。
* `{timeoutMs: 3000}` を渡すことで、最高 3 秒間は前の画面に留まるようにしました。
* state の更新を `startTransition` でラップし、この更新は遅延可能であると伝えました。
* `isPending` を使って、ユーザにトランジションの進行状況を伝えるとともに、ボタンを無効化しました。

結果として、"Next" をクリックしても「望ましくない」ローディング中状態に直接遷移するのではなく、前の画面に留まってユーザに進行状況を伝えるようになりました。

### 更新はどこで起こるのか？ {#where-does-the-update-happen}

これを実装するのはあまり難しくありませんでした。しかしこれがどうやって動作しているのかを考え始めると、ちょっと混乱しそうになります。state を設定したのに、なぜその結果がすぐ現れなかったのでしょうか。次の `<ProfilePage>` は*どこで*レンダーされているのでしょうか？

明らかに、`<ProfilePage>` の両方の「バージョン」が同時に存在しているのです。古いバージョンが存在していることは、画面に表示されており進行中のインジケータまで表示しているということから分かります。また新しいバージョンが*どこかに*存在しているということも分かります。まさにそれが完了するのを待っているのですから！

**ですが、同じコンポーネントの 2 つのバージョンが何故同時に存在しているのでしょうか？**

これが並列モードの根底にあたる部分です。これは React が state の更新を「ブランチ」で行っているようなものだと[以前述べました](/docs/concurrent-mode-intro.html#intentional-loading-sequences)。これを概念化する別の方法として、`startTransition` で state の更新をラップすることで、SF 映画のごとくレンダーが*「別の宇宙で」*始まるのだと考えることができます。その宇宙を直接「観測する」ことはできません -- しかしその宇宙からは何かが起きているという信号 (`isPending`) を得ることはできます。更新の準備が完了したところで、「2 つの宇宙」がマージされ、画面に結果が表示されます！

[デモ](https://codesandbox.io/s/jovial-lalande-26yep)で遊んでみて、そのようなことが起きているところを想像してみてください。

もちろん、ツリーの 2 つのバージョンが*同時に*レンダーされているというのは幻覚であり、それはあなたのコンピュータ上のプログラムが全部同時に実行されていると考えることが幻覚であるのと同じです。オペレーティング・システムは複数のアプリケーションを非常に素早く切り替えているのです。同様に React は、画面上に見えているツリーのバージョンと、次に表示されるために「準備中」のバージョンとを切り替えています。

`useTransition` のような API を使うことで、望ましいユーザ体験に集中でき、それがどのような仕組みで実現されているのかについて気にしないでよくなります。それでも、`startTransition` でラップした更新が「ブランチ」や「別世界」で起こっていると想像するのは例え話としては便利です。

### トランジションは至る所にある {#transitions-are-everywhere}

[サスペンスの解説](/docs/concurrent-mode-suspense.html)で学んだ通り、コンポーネントはそれが必要とするデータがまだ準備できていない場合にいつでも「サスペンド」することができます。計画的にツリー内の様々な場所に `<Suspense>` バウンダリを配置することでこれを制御できますが、それでは十分でないことがあります。

プロフィールが 1 つだけだった最初の[サスペンスのデモ](https://codesandbox.io/s/frosty-hermann-bztrp)に戻りましょう。今のところはデータを 1 回だけ取得しています。サーバ側に更新があるかを確認する "Refresh" ボタンを追加しましょう。

まずはこのようにしてみました：

```js{6-8,13-15}
const initialResource = fetchUserAndPosts();

function ProfilePage() {
  const [resource, setResource] = useState(initialResource);

  function handleRefreshClick() {
    setResource(fetchUserAndPosts());
  }

  return (
    <Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <button onClick={handleRefreshClick}>
        Refresh
      </button>
      <Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline resource={resource} />
      </Suspense>
    </Suspense>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/boring-shadow-100tf)**

この例では、ロード時**および** "Refresh" を押下する度に、データ取得が開始されます。`fetchUserAndPosts()` を呼び出した結果を state 内に入れることで、配下のコンポーネントがたった今開始したリクエストから新しいデータを読み出せるようにします。

[こちら](https://codesandbox.io/s/boring-shadow-100tf)で試せるとおり、"Refresh" ボタンの押下は動作はしています。`<ProfileDetails>` と `<ProfileTimeline>` コンポーネントは新しいデータを表す `resource` を props として受け取り、レスポンスがまだ存在しないため「サスペンド」し、フォールバックが表示されます。レスポンスがロードされると、更新された投稿を見ることができます（このフェイク API は 3 秒ごとに投稿を追加するようになっています）。

しかしながらユーザ体験はとても煩わしいものとなっています。ページをブラウズしたのに、操作した瞬間にローディング中状態で置き換わってしまうのです。これはユーザを混乱させます。**これまでと同様に、望ましくないローディング中状態を回避するために、state の更新をトランジションでラップできます：**

```js{2-5,9-11,21}
function ProfilePage() {
  const [startTransition, isPending] = useTransition({
    // Wait 10 seconds before fallback
    timeoutMs: 10000
  });
  const [resource, setResource] = useState(initialResource);

  function handleRefreshClick() {
    startTransition(() => {
      setResource(fetchProfileData());
    });
  }

  return (
    <Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <button
        onClick={handleRefreshClick}
        disabled={isPending}
      >
        {isPending ? "Refreshing..." : "Refresh"}
      </button>
      <Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline resource={resource} />
      </Suspense>
    </Suspense>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/sleepy-field-mohzb)**

ずっと良く感じられます！ "Refresh" をクリックすることで今ブラウズしていたページから引き離されることがなくなりました。何かがロード中であると「インライン」で見ることができ、データの準備が完了したらそれが表示されます。

### トランジションをデザインシステムに組み込む {#baking-transitions-into-the-design-system}

これで `useTransition` の要求は*非常に*よくあるものであることが分かったでしょう。コンポーネントのサスペンドを引き起こすような、ほぼあらゆるボタンクリックやユーザ操作は、`useTransition` でラップして、ユーザが触っていたものをうっかり隠さないようにする必要があります。

これはコンポーネント間で多くのコードの繰り返しを引き起こす可能性があります。このため、**`useTransition` を*デザインシステム*コンポーネントに組み込む**ことをお勧めします。例えば、トランジションのロジックを独自の `<Button>` コンポーネントに抽出することができます。

```js{7-9,20,24}
function Button({ children, onClick }) {
  const [startTransition, isPending] = useTransition({
    timeoutMs: 10000
  });

  function handleClick() {
    startTransition(() => {
      onClick();
    });
  }

  const spinner = (
    // ...
  );

  return (
    <>
      <button
        onClick={handleClick}
        disabled={isPending}
      >
        {children}
      </button>
      {isPending ? spinner : null}
    </>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/modest-ritchie-iufrh)**

このボタンは*どの* state を更新しようとしているのか関知しないということに注意してください。これは `onClick` ハンドラー内部で起こる*あらゆる* state の更新をトランジションにラップしています。`<Button>` がトランジションの作成を行ってくれるようになったので、`<ProfilePage>` コンポーネントは自分で作成しなくて良くなります：

```js{4-6,11-13}
function ProfilePage() {
  const [resource, setResource] = useState(initialResource);

  function handleRefreshClick() {
    setResource(fetchProfileData());
  }

  return (
    <Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <Button onClick={handleRefreshClick}>
        Refresh
      </Button>
      <Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline resource={resource} />
      </Suspense>
    </Suspense>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/modest-ritchie-iufrh)**

ボタンがクリックされると、トランジションが開始され、内部の `props.onClick()` が呼びされます。それが `<ProfilePage>` コンポーネント内の `handleRefreshClick` をトリガします。新しいデータの取得が開始されますが、トランジションの内部におり、かつ `useTransition` 呼び出しで指定されている 10 秒のタイムアウトがまだ経過していないため、フォールバックは呼び出されません。トランジションが動作している間、ボタンはインラインでローディングインジケータを表示します。

これで、コンポーネントの独立性やモジュール性を犠牲にせずに良いユーザ体験を実現するために、並列モードがどのように役立つのがが分かったと思います。React がトランジションの調整を行うのです。

## The Three Steps {#the-three-steps}

ここまでで、更新したときに経由する可能性のあるすべての視覚的な状態について説明が終わりました。このセクションでは、それらに名前を付け、それらの間での移行について説明します。

<br>

<img src="../images/docs/cm-steps-simple.png" alt="Three steps" />

最後に存在しているのは **Complete** 状態です。ここが最終的に到達したい状態です。次の画面が完全に描画されており、それ以上データを読み込んでいない瞬間を表します。

しかし画面が Complete になる前に、何らかのデータやコードを読み込む必要があります。次の画面を表示しようとしているが、一部がまだロード中である場合、それを **Skeleton** 状態と呼びます。

最後に、Skeleton 状態に至るまでの経路が主に 2 つあります。具体的な例を使ってそれらの違いを述べたいと思います。

### デフォルト：Receded → Skeleton → Complete {#default-receded-skeleton-complete}

[この例](https://codesandbox.io/s/prod-grass-g1lh5)を開いて "Open Profile" をクリックしてください。複数の視覚的な状態を 1 つずつ見ることができます。

* **Receded**: 1 秒間の間、`<h1>Loading the app...</h1>` フォールバックが表示されます。
* **Skeleton:** `<ProfilePage>` コンポーネントが表示され、中で `<h2>Loading posts...</h2>` が表示されます。
* **Complete:** `<ProfilePage>` コンポーネントが表示され、内部のフォールバックも表示されません。すべての取得が完了しています。

Receded（退避）状態と Skeleton 状態はどのように区別するのでしょうか？ これらの違いは、**Receded** 状態はユーザからは「一歩下がっている」ように見え、**Skeleton** 状態はよりコンテンツを見せるため「一歩進んでいる」ように見えるということです。

In this example, we started our journey on the `<HomePage>`:

```js
<Suspense fallback={...}>
  {/* previous screen */}
  <HomePage />
</Suspense>
```

After the click, React started rendering the next screen:

```js
<Suspense fallback={...}>
  {/* next screen */}
  <ProfilePage>
    <ProfileDetails />
    <Suspense fallback={...}>
      <ProfileTimeline />
    </Suspense>
  </ProfilePage>
</Suspense>
```

Both `<ProfileDetails>` and `<ProfileTimeline>` need data to render, so they suspend:

```js{4,6}
<Suspense fallback={...}>
  {/* next screen */}
  <ProfilePage>
    <ProfileDetails /> {/* suspends! */}
    <Suspense fallback={<h2>Loading posts...</h2>}>
      <ProfileTimeline /> {/* suspends! */}
    </Suspense>
  </ProfilePage>
</Suspense>
```

When a component suspends, React needs to show the closest fallback. But the closest fallback to `<ProfileDetails>` is at the top level:

```js{2,3,7}
<Suspense fallback={
  // We see this fallback now because of <ProfileDetails>
  <h1>Loading the app...</h1>
}>
  {/* next screen */}
  <ProfilePage>
    <ProfileDetails /> {/* suspends! */}
    <Suspense fallback={...}>
      <ProfileTimeline />
    </Suspense>
  </ProfilePage>
</Suspense>
```

This is why when we click the button, it feels like we've "taken a step back". The `<Suspense>` boundary which was previously showing useful content (`<HomePage />`) had to "recede" to showing the fallback (`<h1>Loading the app...</h1>`). We call that a **Receded** state.

As we load more data, React will retry rendering, and `<ProfileDetails>` can render successfully. Finally, we're in the **Skeleton** state. We see the new page with missing parts:

```js{6,7,9}
<Suspense fallback={...}>
  {/* next screen */}
  <ProfilePage>
    <ProfileDetails />
    <Suspense fallback={
      // We see this fallback now because of <ProfileTimeline>
      <h2>Loading posts...</h2>
    }>
      <ProfileTimeline /> {/* suspends! */}
    </Suspense>
  </ProfilePage>
</Suspense>
```

Eventually, they load too, and we get to the **Complete** state.

This scenario (Receded → Skeleton → Complete) is the default one. However, the Receded state is not very pleasant because it "hides" existing information. This is why React lets us opt into a different sequence (**Pending** → Skeleton → Complete) with `useTransition`.

### Preferred: Pending → Skeleton → Complete {#preferred-pending-skeleton-complete}

When we `useTransition`, React will let us "stay" on the previous screen -- and show a progress indicator there. We call that a **Pending** state. It feels much better than the Receded state because none of our existing content disappears, and the page stays interactive.

You can compare these two examples to feel the difference:

* Default: [Receded → Skeleton → Complete](https://codesandbox.io/s/prod-grass-g1lh5)
* **Preferred: [Pending → Skeleton → Complete](https://codesandbox.io/s/focused-snow-xbkvl)**

The only difference between these two examples is that the first uses regular `<button>`s, but the second one uses our custom `<Button>` component with `useTransition`.

### Wrap Lazy Features in `<Suspense>` {#wrap-lazy-features-in-suspense}

Open [this example](https://codesandbox.io/s/nameless-butterfly-fkw5q). When you press a button, you'll see the Pending state for a second before moving on. This transition feels nice and fluid.

We will now add a brand new feature to the profile page -- a list of fun facts about a person:

```js{8,13-25}
function ProfilePage({ resource }) {
  return (
    <>
      <ProfileDetails resource={resource} />
      <Suspense fallback={<h2>Loading posts...</h2>}>
        <ProfileTimeline resource={resource} />
      </Suspense>
      <ProfileTrivia resource={resource} />
    </>
  );
}

function ProfileTrivia({ resource }) {
  const trivia = resource.trivia.read();
  return (
    <>
      <h2>Fun Facts</h2>
      <ul>
        {trivia.map(fact => (
          <li key={fact.id}>{fact.text}</li>
        ))}
      </ul>
    </>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/focused-mountain-uhkzg)**

If you press "Open Profile" now, you can tell something is wrong. It takes whole seven seconds to make the transition now! This is because our trivia API is too slow. Let's say we can't make the API faster. How can we improve the user experience with this constraint?

If we don't want to stay in the Pending state for too long, our first instinct might be to set `timeoutMs` in `useTransition` to something smaller, like `3000`. You can try this [here](https://codesandbox.io/s/practical-kowalevski-kpjg4). This lets us escape the prolonged Pending state, but we still don't have anything useful to show!

There is a simpler way to solve this. **Instead of making the transition shorter, we can "disconnect" the slow component from the transition** by wrapping it into `<Suspense>`:

```js{8,10}
function ProfilePage({ resource }) {
  return (
    <>
      <ProfileDetails resource={resource} />
      <Suspense fallback={<h2>Loading posts...</h2>}>
        <ProfileTimeline resource={resource} />
      </Suspense>
      <Suspense fallback={<h2>Loading fun facts...</h2>}>
        <ProfileTrivia resource={resource} />
      </Suspense>
    </>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/condescending-shape-s6694)**

This reveals an important insight. React always prefers to go to the Skeleton state as soon as possible. Even if we use transitions with long timeouts everywhere, React will not stay in the Pending state for longer than necessary to avoid the Receded state.

**If some feature isn't a vital part of the next screen, wrap it in `<Suspense>` and let it load lazily.** This ensures we can show the rest of the content as soon as possible. Conversely, if a screen is *not worth showing* without some component, such as `<ProfileDetails>` in our example, do *not* wrap it in `<Suspense>`. Then the transitions will "wait" for it to be ready.

### Suspense Reveal "Train" {#suspense-reveal-train}

When we're already on the next screen, sometimes the data needed to "unlock" different `<Suspense>` boundaries arrives in quick succession. For example, two different responses might arrive after 1000ms and 1050ms, respectively. If you've already waited for a second, waiting another 50ms is not going to be perceptible. This is why React reveals `<Suspense>` boundaries on a schedule, like a "train" that arrives periodically. This trades a small delay for reducing the layout thrashing and the number of visual changes presented to the user.

You can see a demo of this [here](https://codesandbox.io/s/admiring-mendeleev-y54mk). The "posts" and "fun facts" responses come within 100ms of each other. But React coalesces them and "reveals" their Suspense boundaries together. 

### Delaying a Pending Indicator {#delaying-a-pending-indicator}

Our `Button` component will immediately show the Pending state indicator on click:

```js{2,13}
function Button({ children, onClick }) {
  const [startTransition, isPending] = useTransition({
    timeoutMs: 10000
  });

  // ...

  return (
    <>
      <button onClick={handleClick} disabled={isPending}>
        {children}
      </button>
      {isPending ? spinner : null}
    </>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/floral-thunder-iy826)**

This signals to the user that some work is happening. However, if the transition is relatively short (less than 500ms), it might be too distracting and make the transition itself feel *slower*.

One possible solution to this is to *delay the spinner itself* from displaying:

```css
.DelayedSpinner {
  animation: 0s linear 0.5s forwards makeVisible;
  visibility: hidden;
}

@keyframes makeVisible {
  to {
    visibility: visible;
  }
}
```

```js{2-4,10}
const spinner = (
  <span className="DelayedSpinner">
    {/* ... */}
  </span>
);

return (
  <>
    <button onClick={handleClick}>{children}</button>
    {isPending ? spinner : null}
  </>
);
```

**[CodeSandbox で試す](https://codesandbox.io/s/gallant-spence-l6wbk)**

With this change, even though we're in the Pending state, we don't display any indication to the user until 500ms has passed. This may not seem like much of an improvement when the API responses are slow. But compare how it feels [before](https://codesandbox.io/s/thirsty-liskov-1ygph) and [after](https://codesandbox.io/s/hardcore-http-s18xr) when the API call is fast. Even though the rest of the code hasn't changed, suppressing a "too fast" loading state improves the perceived performance by not calling attention to the delay.

### Recap {#recap}

The most important things we learned so far are:

* By default, our loading sequence is Receded → Skeleton → Complete.
* The Receded state doesn't feel very nice because it hides existing content.
* With `useTransition`, we can opt into showing a Pending state first instead. This will keep us on the previous screen while the next screen is being prepared.
* If we don't want some component to delay the transition, we can wrap it in its own `<Suspense>` boundary.
* Instead of doing `useTransition` in every other component, we can build it into our design system.

## Other Patterns {#other-patterns}

Transitions are probably the most common Concurrent Mode pattern you'll encounter, but there are a few more patterns you might find useful.

### Splitting High and Low Priority State {#splitting-high-and-low-priority-state}

When you design React components, it is usually best to find the "minimal representation" of state. For example, instead of keeping `firstName`, `lastName`, and `fullName` in state, it's usually better keep only `firstName` and `lastName`, and then calculate `fullName` during rendering. This lets us avoid mistakes where we update one state but forget the other state.

However, in Concurrent Mode there are cases where you might *want* to "duplicate" some data in different state variables. Consider this tiny translation app:

```js
const initialQuery = "Hello, world";
const initialResource = fetchTranslation(initialQuery);

function App() {
  const [query, setQuery] = useState(initialQuery);
  const [resource, setResource] = useState(initialResource);

  function handleChange(e) {
    const value = e.target.value;
    setQuery(value);
    setResource(fetchTranslation(value));
  }

  return (
    <>
      <input
        value={query}
        onChange={handleChange}
      />
      <Suspense fallback={<p>Loading...</p>}>
        <Translation resource={resource} />
      </Suspense>
    </>
  );
}

function Translation({ resource }) {
  return (
    <p>
      <b>{resource.read()}</b>
    </p>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/brave-villani-ypxvf)**

Notice how when you type into the input, the `<Translation>` component suspends, and we see the `<p>Loading...</p>` fallback until we get fresh results. This is not ideal. It would be better if we could see the *previous* translation for a bit while we're fetching the next one.

In fact, if we open the console, we'll see a warning:

```
Warning: App triggered a user-blocking update that suspended.

The fix is to split the update into multiple parts: a user-blocking update to provide immediate feedback, and another update that triggers the bulk of the changes.

Refer to the documentation for useTransition to learn how to implement this pattern.
```

As we mentioned earlier, if some state update causes a component to suspend, that state update should be wrapped in a transition. Let's add `useTransition` to our component:

```js{4-6,10,13}
function App() {
  const [query, setQuery] = useState(initialQuery);
  const [resource, setResource] = useState(initialResource);
  const [startTransition, isPending] = useTransition({
    timeoutMs: 5000
  });

  function handleChange(e) {
    const value = e.target.value;
    startTransition(() => {
      setQuery(value);
      setResource(fetchTranslation(value));
    });
  }

  // ...

}
```

**[CodeSandbox で試す](https://codesandbox.io/s/zen-keldysh-rifos)**

Try typing into the input now. Something's wrong! The input is updating very slowly.

We've fixed the first problem (suspending outside of a transition). But now because of the transition, our state doesn't update immediately, and it can't "drive" a controlled input!

The answer to this problem **is to split the state in two parts:** a "high priority" part that updates immediately, and a "low priority" part that may wait for a transition.

In our example, we already have two state variables. The input text is in `query`, and we read the translation from `resource`. We want changes to the `query` state to happen immediately, but changes to the `resource` (i.e. fetching a new translation) should trigger a transition.

So the correct fix is to put `setQuery` (which doesn't suspend) *outside* the transition, but `setResource` (which will suspend) *inside* of it.

```js{4,5}
function handleChange(e) {
  const value = e.target.value;
  
  // Outside the transition (urgent)
  setQuery(value);

  startTransition(() => {
    // Inside the transition (may be delayed)
    setResource(fetchTranslation(value));
  });
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/lively-smoke-fdf93)**

With this change, it works as expected. We can type into the input immediately, and the translation later "catches up" to what we have typed.

### Deferring a Value {#deferring-a-value}

By default, React always renders a consistent UI. Consider code like this:

```js
<>
  <ProfileDetails user={user} />
  <ProfileTimeline user={user} />
</>
```

React guarantees that whenever we look at these components on the screen, they will reflect data from the same `user`. If a different `user` is passed down because of a state update, you would see them changing together. You can't ever record a screen and find a frame where they would show values from different `user`s. (If you ever run into a case like this, file a bug!)

This makes sense in the vast majority of situations. Inconsistent UI is confusing and can mislead users. (For example, it would be terrible if a messenger's Send button and the conversation picker pane "disagreed" about which thread is currently selected.)

However, sometimes it might be helpful to intentionally introduce an inconsistency. We could do it manually by "splitting" the state like above, but React also offers a built-in Hook for this:

```js
import { useDeferredValue } from 'react';

const deferredValue = useDeferredValue(value, {
  timeoutMs: 5000
});
```

To demonstrate this feature, we'll use [the profile switcher example](https://codesandbox.io/s/musing-ramanujan-bgw2o). Click the "Next" button and notice how it takes 1 second to do a transition.

Let's say that fetching the user details is very fast and only takes 300 milliseconds. Currently, we're waiting a whole second because we need both user details and posts to display a consistent profile page. But what if we want to show the details faster?

If we're willing to sacrifice consistency, we could **pass potentially stale data to the components that delay our transition**. That's what `useDeferredValue()` lets us do:

```js{2-4,10,11,21}
function ProfilePage({ resource }) {
  const deferredResource = useDeferredValue(resource, {
    timeoutMs: 1000
  });
  return (
    <Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline
          resource={deferredResource}
          isStale={deferredResource !== resource}
        />
      </Suspense>
    </Suspense>
  );
}

function ProfileTimeline({ isStale, resource }) {
  const posts = resource.posts.read();
  return (
    <ul style={{ opacity: isStale ? 0.7 : 1 }}>
      {posts.map(post => (
        <li key={post.id}>{post.text}</li>
      ))}
    </ul>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/vigorous-keller-3ed2b)**

The tradeoff we're making here is that `<ProfileTimeline>` will be inconsistent with other components and potentially show an older item. Click "Next" a few times, and you'll notice it. But thanks to that, we were able to cut down the transition time from 1000ms to 300ms.

Whether or not it's an appropriate tradeoff depends on the situation. But it's a handy tool, especially when the content doesn't change very visible between items, and the user might not even realize they were looking at a stale version for a second.

It's worth noting that `useDeferredValue` is not *only* useful for data fetching. It also helps when an expensive component tree causes an interaction (e.g. typing in an input) to be sluggish. Just like we can "defer" a value that takes too long to fetch (and show its old value despite others components updating), we can do this with trees that take too long to render.

For example, consider a filterable list like this:

```js
function App() {
  const [text, setText] = useState("hello");

  function handleChange(e) {
    setText(e.target.value);
  }

  return (
    <div className="App">
      <label>
        Type into the input:{" "}
        <input value={text} onChange={handleChange} />
      </label>
      ...
      <MySlowList text={text} />
    </div>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/pensive-shirley-wkp46)**

In this example, **every item in `<MySlowList>` has an artificial slowdown -- each of them blocks the thread for a few milliseconds**. We'd never do this in a real app, but this helps us simulate what can happen in a deep component tree with no single obvious place to optimize.

We can see how typing in the input causes stutter. Now let's add `useDeferredValue`:

```js{3-5,18}
function App() {
  const [text, setText] = useState("hello");
  const deferredText = useDeferredValue(text, {
    timeoutMs: 5000
  });

  function handleChange(e) {
    setText(e.target.value);
  }

  return (
    <div className="App">
      <label>
        Type into the input:{" "}
        <input value={text} onChange={handleChange} />
      </label>
      ...
      <MySlowList text={deferredText} />
    </div>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/infallible-dewdney-9fkv9)**

Now typing has a lot less stutter -- although we pay for this by showing the results with a lag.

How is this different from debouncing? Our example has a fixed artificial delay (3ms for every one of 80 items), so there is always a delay, no matter how fast our computer is. However, the `useDeferredValue` value only "lags behind" if the rendering takes a while. There is no minimal lag imposed by React. With a more realistic workload, you can expect the lag to adjust to the user’s device. On fast machines, the lag would be smaller or non-existent, and on slow machines, it would be more noticeable. In both cases, the app would remain responsive. That’s the advantage of this mechanism over debouncing or throttling, which always impose a minimal delay and can't avoid blocking the thread while rendering.

Even though there is an improvement in responsiveness, this example isn't as compelling yet because Concurrent Mode is missing some crucial optimizations for this use case. Still, it is interesting to see that features like `useDeferredValue` (or `useTransition`) are useful regardless of whether we're waiting for network or for computational work to finish.

### SuspenseList {#suspenselist}

`<SuspenseList>` is the last pattern that's related to orchestrating loading states.

Consider this example:

```js{5-10}
function ProfilePage({ resource }) {
  return (
    <>
      <ProfileDetails resource={resource} />
      <Suspense fallback={<h2>Loading posts...</h2>}>
        <ProfileTimeline resource={resource} />
      </Suspense>
      <Suspense fallback={<h2>Loading fun facts...</h2>}>
        <ProfileTrivia resource={resource} />
      </Suspense>
    </>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/proud-tree-exg5t)**

The API call duration in this example is randomized. If you keep refreshing it, you will notice that sometimes the posts arrive first, and sometimes the "fun facts" arrive first.

This presents a problem. If the response for fun facts arrives first, we'll see the fun facts below the `<h2>Loading posts...</h2>` fallback for posts. We might start reading them, but then the *posts* response will come back, and shift all the facts down. This is jarring.

One way we could fix it is by putting them both in a single boundary:

```js
<Suspense fallback={<h2>Loading posts and fun facts...</h2>}>
  <ProfileTimeline resource={resource} />
  <ProfileTrivia resource={resource} />
</Suspense>
```

**[CodeSandbox で試す](https://codesandbox.io/s/currying-violet-5jsiy)**

The problem with this is that now we *always* wait for both of them to be fetched. However, if it's the *posts* that came back first, there's no reason to delay showing them. When fun facts load later, they won't shift the layout because they're already below the posts.

Other approaches to this, such as composing Promises in a special way, are increasingly difficult to pull off when the loading states are located in different components down the tree.

To solve this, we will import `SuspenseList`:

```js
import { SuspenseList } from 'react';
```

`<SuspenseList>` coordinates the "reveal order" of the closest `<Suspense>` nodes below it:

```js{3,11}
function ProfilePage({ resource }) {
  return (
    <SuspenseList revealOrder="forwards">
      <ProfileDetails resource={resource} />
      <Suspense fallback={<h2>Loading posts...</h2>}>
        <ProfileTimeline resource={resource} />
      </Suspense>
      <Suspense fallback={<h2>Loading fun facts...</h2>}>
        <ProfileTrivia resource={resource} />
      </Suspense>
    </SuspenseList>
  );
}
```

**[CodeSandbox で試す](https://codesandbox.io/s/black-wind-byilt)**

The `revealOrder="forwards"` option means that the closest `<Suspense>` nodes inside this list **will only "reveal" their content in the order they appear in the tree -- even if the data for them arrives in a different order**. `<SuspenseList>` has other interesting modes: try changing `"forwards"` to `"backwards"` or `"together"` and see what happens.

You can control how many loading states are visible at once with the `tail` prop. If we specify `tail="collapsed"`, we'll see *at most one* fallback at the time. You can play with it [here](https://codesandbox.io/s/adoring-almeida-1zzjh).

Keep in mind that `<SuspenseList>` is composable, like anything in React. For example, you can create a grid by putting several `<SuspenseList>` rows inside a `<SuspenseList>` table.

## Next Steps {#next-steps}

Concurrent Mode offers a powerful UI programming model and a set of new composable primitives to help you orchestrate delightful user experiences.

It's a result of several years of research and development, but it's not finished. In the section on [adopting Concurrent Mode](/docs/concurrent-mode-adoption.html), we'll describe how you can try it and what you can expect.
