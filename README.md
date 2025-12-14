# 概要

このリポジトリは[この記事](https://rsasage.com)の解説用のものになります。


## セットアップ
`Node.js`、`npm` がインストールされている環境で、下記コマンドを実行してください。

```
npm install
```

## 実行

下記コマンドで実行できます。

```sh
npm start
```

## 記事

下記が記事の全文です。

---

## はじめに

ローディングアニメーションにこだわるフロントエンジニアは少ないと思いますが、ここを改善するとUIはぐっと引き締まります。

![ちらつき例のGIFアニメーション](https://raw.githubusercontent.com/Arahabica/good-timing-loading/refs/heads/main/gifs/normal150ms_30fps.gif)

例えば、上記のローディング表示ですが、150msだけ表示されており、ユーザーにとってはちらつきとなり、洗練されていない印象になります。

## 2つのちらつきの抑え方

ローディングによるちらつきを抑える方法は以下の2つになります。

* ローディングを表示しない
  * 表示するまでに **遅延(Delay)** をいれる。遅延時間内に処理が終われば、ローディングは表示されない
* 一度出したローディングはすぐに消さない
  * ローディング表示が開始したら **最低表示時間 (Min Duration)** 経つまではローディングを消さない

下のアニメーションはこの2点を踏まえたローディング表示になります。ボタンの上の数字はローディング処理にかかる時間です。

![各時間のローディング表示](https://raw.githubusercontent.com/Arahabica/good-timing-loading/refs/heads/main/gifs/good_multi_30fps.gif)


1列目の300ms以下では、ローディングアニメーションは表示していません。300ms以下ぐらいの一瞬であれば、空白があっても不自然ではないでしょう。

2列目の400ms〜950msは同時にアニメーションが終了しています。これは処理が終わっても最低表示時間になるまであえて、待機しているからです。


3列目は普通に処理が終わったあとにすぐにアニメーションを終了しています。



先ほどのローディングで適用している遅延・最低表示時間は、

* 遅延: 350ms
* 最低表示時間: 600ms

です。

こちらが私のおすすめの設定値ですが、一応根拠のある数字になっています。詳しくは後の[「設定値の根拠」の章](#設定値の根拠)をご覧ください。

さらに挙動をまとめると下記のようになります。

| 処理時間 | ローディングの最終的な挙動 |
| ---- | ---- |
| ~ 350ms | ローディングを表示しない（Delayが終了する前に処理完了） |
| 350ms 〜 950ms | 350ms経ってからローディングが表示され、処理が950msになるまで表示が強制維持される (Min Durationが適用) |
| 950ms 〜 | 350ms経ってからローディングが表示され、処理が終わるまで表示される |

設定値を調整してローディング表示を試せるシミュレーターも作りました。

https://loading.rsasage.com/?x=2

ぜひ、触って体感してみてください。

## 実装

この挙動をWebで実現するためのコードを書きました。

遅延はCSSを使って実装しています。

```html:loading.html
<svg class="loading hidden" viewBox="0 0 40 40" xmlns="http://www.w3.org/2000/svg">
  <circle fill="none" stroke-width="4" cx="20" cy="20" r="18"></circle>
</svg>
```

```css:loading.css
:root {
  --loading-delay: 350ms;
  --loading-color: #2860d8;
  --primary-color: #2860d8;
}
.loading {
  width: 40px;
  height: 40px;
  animation: rotator 1.6s linear var(--loading-delay) infinite;
}

@keyframes rotator {
  0% {
    transform: rotate(-90deg);
  }

  100% {
    transform: rotate(180deg);
  }
}

.loading circle {
  stroke-dasharray: 114;
  stroke-dashoffset: 0;
  transform-origin: center;
  stroke: var(--loading-color);
  animation:
    spin-hidden var(--loading-delay),
    spin-fade-in 0.15s ease-out var(--loading-delay) forwards,
    spin-dash 1.6s ease-in-out var(--loading-delay) infinite;
}
/* 遅延を実現するための何も表示しないアニメーション */
@keyframes spin-hidden {
  0% {
    opacity: 0;
  }

  100% {
    opacity: 0;
  }
}
@keyframes spin-fade-in {
  0% {
    opacity: 0;
  }

  100% {
    opacity: 1;
  }
}
/* スピナーの円弧の長さを伸び縮みさせるアニメーション */
@keyframes spin-dash {
  0% {
    stroke-dashoffset: 111;
  }

  50% {
    stroke-dashoffset: 28.5;
    transform: rotate(135deg);
  }

  100% {
    stroke-dashoffset: 111;
    transform: rotate(450deg);
  }
}
```

下記は最低表示時間分、処理を遅延させるためのユーティリティメソッドです。

```js:waitLoadingAnimation.js
const waitLoadingAnimation = async (callback, opts) => {
  const delay = opts?.delay ?? 350; // ドハティ閾値(400ms) - バッファ(50ms)
  const minDuration = opts?.minDuration ?? 600; // 最低表示時間
  // Delayを超えた場合に強制される合計の最低待ち時間
  const requiredWaitTime = delay + minDuration; // 950ms

  const startTime = Date.now();

  // 1. コールバック関数を即座に実行
  const callbackPromise = callback();

  try {
    // 2. 処理の完了を待機
    const result = await callbackPromise;
    const finishedTime = Date.now();
    const actualTime = finishedTime - startTime;

    // 3. Min Durationの適用判定
    if (actualTime < delay) {
      // delay以内に完了した場合は即座に結果を返す (ノイズレス)
      return result;
    }
    // 処理時間がDelayを超えた場合、ローディングが表示されたとみなし、
    // Min Durationのルールを適用する。
    const remainingTime = Math.max(requiredWaitTime - actualTime, 0);

    // 4. 不足している時間分だけ人工的に待機
    if (remainingTime > 0) {
      // console.log("Waiting for min duration:", remainingTime);
      // Promiseが解決したにも関わらず、ローディング表示を維持する
      await new Promise((resolve) => {
        setTimeout(resolve, remainingTime);
      });
    }
    return result;
  } catch (error) {
    // エラー時も同様にMin Durationのルールを適用し、チラつきを防ぐ
    const errorTime = Date.now();
    const actualErrorTime = errorTime - startTime;

    if (actualErrorTime > delay) {
      const remainingTime = Math.max(requiredWaitTime - actualErrorTime, 0);
      if (remainingTime > 0) {
        // console.log("Waiting for min duration:", remainingTime);
        await new Promise((resolve) => {
          setTimeout(resolve, remainingTime);
        });
      }
    }
    // 待機完了後にエラーをスロー
    throw error;
  }
};
```

TypeScript版も作りました。
:::details TypeScript版
```ts:waitLoadingAnimation.ts
const waitLoadingAnimation = async <T>(
  callback: () => Promise<T>,
  {
    delay = 350, // delay: ドハティ閾値(400ms) - バッファ(50ms)
    minDuration = 600 // minDuration: 最低表示時間
  } = {} 
) => {
  // Delayを超えた場合に強制される合計の最低待ち時間
  const requiredWaitTime = delay + minDuration; // 950ms

  const startTime = Date.now();

  // 1. コールバック関数を即座に実行
  const callbackPromise = callback();

  try {
    // 2. 処理の完了を待機
    const result = await callbackPromise;
    const finishedTime = Date.now();
    const actualTime = finishedTime - startTime;

    // 3. Min Durationの適用判定
    if (actualTime < delay) {
      // delay以内に完了した場合は即座に結果を返す (ノイズレス)
      return result;
    }
    // 処理時間がDelayを超えた場合、ローディングが表示されたとみなし、
    // Min Durationのルールを適用する。
    const remainingTime = Math.max(requiredWaitTime - actualTime, 0);

    // 4. 不足している時間分だけ人工的に待機
    if (remainingTime > 0) {
      // console.log("Waiting for min duration:", remainingTime);
      // Promiseが解決したにも関わらず、ローディング表示を維持する
      await new Promise((resolve) => {
        setTimeout(resolve, remainingTime);
      });
    }
    return result;
  } catch (error) {
    // エラー時も同様にMin Durationのルールを適用し、チラつきを防ぐ
    const errorTime = Date.now();
    const actualErrorTime = errorTime - startTime;

    if (actualErrorTime > delay) {
      const remainingTime = Math.max(requiredWaitTime - actualErrorTime, 0);
      if (remainingTime > 0) {
        // console.log("Waiting for min duration:", remainingTime);
        await new Promise((resolve) => {
          setTimeout(resolve, remainingTime);
        });
      }
    }
    // 待機完了後にエラーをスロー
    throw error;
  }
};
```
:::

下記のような形でローディング処理を実行するメソッドを引数に入れるような形で使えます。

```js:sample.js
async function doSomething() {
   // 何らかのローディング処理を実行
}

nextButton.addEventListener("click", async () => {
   setState("loading");
   try {
     await waitLoadingAnimation(() => doSomething());
     setState("loaded");
   } catch (error) {
     setState("error");
   }
});
```

GitHubにもコードを上げているので、気になる方はご覧ください。

https://github.com/Arahabica/good-timing-loading/blob/main/random.html

## 設定値の根拠

今回、遅延時間を350ms、最低表示時間を600msとしました。これは個人的な体感だけではなく、ある程度根拠がある数字になっています。

### 遅延: 350ms

そもそもローディング表示が必要なのはシステムがちゃんと処理をしているのをユーザーに知らせるためです。
システムが一定時間以上反応がないとユーザーはストレスを感じ始めます。
この一定時間というのはちゃんと研究されていて、400msが閾値とされています（[ドハティの法則](https://lawsofux.com/doherty-threshold/)）。

逆にいうと400ms未満であれば無反応でも許容されるので、400ms以内に終わる処理ではローディングは表示しないのが良いUIと言えるでしょう。

350msというのはこの400msから少しバッファをもうけた値になります。

### 最低表示時間: 600ms

350ms経ってからローディングを始めても、450msで処理が終わったら、ローディングを表示する時間は100msになってしまい、結局ちらつきになってしまいます。

このような場合にはちらつきを防ぐために500ms〜600msほどの最低表示時間を設けるようにしましょう。

ユーザビリティ研究のヤコブ・ニールセンによると[ユーザーは1000msを超えると思考が途切れ始める](https://www.nngroup.com/articles/response-times-3-important-limits/)とのことです。表示を遅らせても遅延と最低表示時間の合計が1秒の閾値を超えていなければ、集中力をとぎらせることはないので安心して良いかと思います。

600msというのは350msと足して1秒にならないように調整した値になっています。

ユーザーに表示する準備が整っているのに、あえて表示を遅らせるのは、パフォーマンス命のエンジニアにとっては受け入れ難いかもしれません。
ただ、UIの観点に立つと、ちらつきが発生するよりは少し待たせる方がユーザー体験は向上します。どうしてもスピードを求めたい人は350ms以内を目指すべきでしょう。

## おわりに

ローディングの細かい挙動までデザイナーからの指示があることは少ないかもしれません。

だからこそ、見落とされがちであり、許容範囲内という扱いになって顧みられていないことが多いんだと思います。

ローディングにも気を遣って、あの人のUIはなんか洗練されてるよね、と思ってもらえるようにしましょう。
