---
title: "Chromeの翻訳でフォームの送信値が英語になる ── optionのvalue省略が原因"
emoji: "🌐"
type: "tech"
topics: ["javascript", "html", "chrome", "フロントエンド", "多言語対応"]
published: true
---

## はじめに

日本語のフォームを、外国の方がブラウザの翻訳機能で英語にして使うと、こうなることがあります。

- **送信される値が、翻訳後の英語になる**（`"会場で参加"` のはずが `"Attend at the venue"` で届く）
- **JavaScript が `RangeError: Maximum call stack size exceeded` で止まり**、条件付きで出るはずの設問が出てこない

2026年9月に、PC 版 Chrome と iPhone の Chrome（どちらもバージョン 153）で、実際に起きることを確認しました。

原因は、`<option>` の `value` 属性を省略していたことです。これはブラウザのバグではありません。**`value` を省略したときの動きは HTML の仕様で決まっており**、翻訳機能はその仕様どおりの値を変えてしまうだけです。だからこそ、フォームを書く側が気をつけるしかありません。

多言語版を作る予算はないけれど、インバウンドのお客様にも使ってほしい。そういう事業者のフォームほど、翻訳機能で使われる前提になります。この記事では、実際に壊れるデモを作り、再現と対策を確かめた結果をまとめます。

デモは次の URL で公開しています。お手元のブラウザで翻訳機能をオンにして試せます（入力内容はどこにも送信されません）。

https://www.brainy-software.co.jp/demo/translate-crash/

:::message
この記事では、書いている内容を次の3種類に分けています。

- **仕様**：HTML・DOM の標準仕様で決まっていること（リンクを付けています）
- **今回確認**：2026年9月に Chrome 153（PC／iOS）で確認した挙動。ブラウザの更新で変わる可能性があります
- **実測・推測**：今回の環境で測った数値と、そこからの推測
:::

## 何が起きるか

デモは、イベントの参加申込を想定した小さなフォームです。

- 「参加方法」で「会場で参加」を選ぶと、「懇親会に参加しますか」が出る
- 懇親会に「参加する」を選ぶと、「食事の制限」が出る
- 「オンラインで参加」を選ぶと、「視聴に使う端末」が出る

設問を隠したときは、その設問の入力をクリアし、後ろの設問にも伝わるよう `change` イベントを発火し直します。条件付き表示のフォームでよく見る作りです。

これをページごと英語に翻訳してから操作した結果が次のとおりです（**今回確認**）。

| | 翻訳なし | 翻訳あり |
|---|---|---|
| JavaScript | 正常 | **RangeError で止まる** |
| 条件付きの設問 | 出る | **出ない** |
| 送信される値 | `"会場で参加"` | **`"Attend at the venue"`** |

PC 版 Chrome で翻訳した状態です。「参加方法」を選んだ直後にエラーになり、送信される値も英語になっています。

![PC版Chromeで翻訳した対策前のフォーム。JavaScriptエラーの表示と、英語になった送信値](/images/translate-crash/pc-chrome-bad-translated.jpg)

iPhone の Chrome でも同じことが起きました。「会場で参加」を選んでも懇親会の設問が出ず、送信値は英語です。

![iPhoneのChromeで翻訳した対策前のフォーム](/images/translate-crash/iphone-chrome-bad-translated-3.png)

なお、ブラウザが翻訳を提案してこなくても、ユーザーはメニューから手動で翻訳できます。「うちのサイトでは翻訳の案内が出ないから関係ない」とは言えません。

## 原因

### value を省略すると、表示テキストが値になる（仕様）

`<option>` の値は、HTML の仕様で次のように決まっています。

> `value` 属性があればその値。なければ、要素のテキスト。
> （[HTML Standard: the option element の value](https://html.spec.whatwg.org/multipage/form-elements.html#dom-option-value) の要約）

つまり、次のように書いた場合、`select.value` は表示されている文字列そのものです。

```html
<!-- value を省略すると、表示テキストが値になる -->
<select name="q1">
  <option>選択してください</option>
  <option>会場で参加</option>
  <option>オンラインで参加</option>
</select>
```

短く書けるので、手書きの HTML でもテンプレートでもよく省略されます。翻訳を考えなければ、何も困りません。

### 翻訳で表示テキストが変わり、値も変わった（今回確認）

Chrome 153 で翻訳したところ、`<option>` の表示テキストが英語になり、それに合わせて `select.value` も英語になりました。

| `<option>` の書き方 | 翻訳後の表示 | 翻訳後の `select.value` |
|---|---|---|
| `<option>会場で参加</option>` | Attend at the venue | **`"Attend at the venue"`** |
| `<option value="会場で参加">会場で参加</option>` | Attend at the venue | `"会場で参加"` |

`value` 属性に書いた値は、翻訳後も変わりませんでした。ラジオボタンの `value` も同じです。**今回の Chrome では、翻訳機能は表示テキストを書き換え、`value` 属性は書き換えませんでした。**

`value` を省略していると、表示テキストの書き換えがそのまま値の書き換えになり、**サーバには英語の値が届きます**。エラーは出ないので、DB を見るまで気づけません。

### 条件判定も同じ理由で崩れる

条件付き表示は「選ばれた値が `"会場で参加"` なら表示する」と判定しています。翻訳後は値が `"Attend at the venue"` なので、何を選んでも条件を満たしません。**設問が出てこない**のはこのためです。

### RangeError になる仕組み

止まってしまう原因は、「未選択かどうか」の判定です。デモでは、よくある書き方として、表示テキストと比べていました。

```js
const PLACEHOLDER = '選択してください';

function hasValue(input) {
  if (input.tagName === 'SELECT') return input.value !== '' && input.value !== PLACEHOLDER;
  // ...
}

function clearValue(input) {
  if (input.tagName === 'SELECT') input.selectedIndex = 0; // 先頭の「選択してください」に戻す
  // ...
}
```

設問を隠すときの処理は、次の流れです。

1. 隠した設問に入力があれば（`hasValue` が真なら）、クリアする
2. クリアした入力に `change` イベントを発火し直す（`dispatchEvent`）
3. `change` を受けて、全設問の表示・非表示を判定し直す

翻訳前なら、クリアした `<select>` の値は `"選択してください"` なので、`hasValue` は偽になり、ここで止まります。

翻訳後は、クリアしても値が `"please select"` です。`"選択してください"` と一致しないので、**クリアしたのに「入力がある」と判定されます**。すると、またクリアして、また `change` を発火して、また判定し直して……と、終わらなくなります。

`dispatchEvent` は、登録された処理をその場で順に実行してから戻ります（**仕様**：[DOM Standard の dispatch](https://dom.spec.whatwg.org/#concept-event-dispatch)）。そのため、この繰り返しはすべて呼び出しスタックに積み上がり、上限に達したところで `RangeError: Maximum call stack size exceeded` になります。

デモで入れ子の深さを数えたところ、上限に達したのは **PC 版 Chrome で 43 段、iPhone の Chrome で 184 段**でした（**実測**）。段数はブラウザや端末によって変わると考えてください。

## 対策1：フォームだけ翻訳させない（`translate="no"`）

いちばん手軽なのは、フォームを翻訳の対象から外すことです。`translate` 属性は HTML の仕様にある属性です（[HTML Standard: the translate attribute](https://html.spec.whatwg.org/multipage/dom.html#attr-translate)）。

```html
<form translate="no">
  ...
</form>
```

ページの説明文は翻訳され、フォームの中だけ日本語のまま残ります。PC 版でも iPhone の Chrome でも、エラーは出ず、送信値も日本語のままでした（**今回確認**）。

![translate="no" を付けたフォーム。説明文は英語、フォームは日本語のまま](/images/translate-crash/pc-chrome-notranslate.jpg)

ただし、**外国の方にはフォームが読めません**。翻訳機能を使う人を締め出す対策なので、急ぎの応急処置として位置づけるのがよいと思います。

## 対策2：value 属性を明示する

翻訳させたまま直すには、`value` 属性を書きます。値は、今まで送られていた日本語のままで構いません。

```html
<select name="q1">
  <option value="">選択してください</option>
  <option value="会場で参加">会場で参加</option>
  <option value="オンラインで参加">オンラインで参加</option>
</select>
```

あわせて、未選択の判定を表示テキストとの比較から `value=""` での判定に変えます。

```js
function hasValue(input) {
  // 「選択してください」との比較をやめ、空文字かどうかだけで判定する
  return input.value !== '';
}
```

`value` 属性がある場合、値は属性の値で決まり、表示テキストには左右されません（**仕様**）。実際に翻訳しても、画面は英語、送信値は日本語のままで、条件付きの設問も正しく出ました（**今回確認**）。

![value を明示したフォーム。表示は英語、送信値は日本語](/images/translate-crash/iphone-chrome-fixed-value.jpeg)

**サーバ側も DB の値も変えずに済む**ので、既に動いているフォームを直すならこれが最小の修正です。

## 対策3：value をコード値にする

これから作るフォームなら、`value` を言語に依存しないコード値にしておくのがおすすめです。

```html
<select name="q1">
  <option value="">選択してください</option>
  <option value="venue">会場で参加</option>
  <option value="online">オンラインで参加</option>
</select>
```

表示テキストは翻訳してよく、値はコードのまま変わりません（**今回確認**）。画面の文言を変えても DB の値に影響しないので、将来きちんと多言語化するときにもそのまま使えます。

既存のフォームをコード値に切り替えるときは、サーバ側の処理や DB に入っている値の移行も必要になる点に注意してください。

![value をコード値にしたフォーム。表示は英語、送信値はコード値](/images/translate-crash/iphone-chrome-fixed-code.jpeg)

## 補足1：PC 版 Chrome では、このエラーを JavaScript で拾えなかった

デモを作っていて一番困ったのは、PC 版 Chrome では、この RangeError を JavaScript から捕まえられなかったことです（**今回確認**）。

- `window.addEventListener('error', ...)` に届かない
- `change` を受ける処理の中身を `try...catch` で囲んでも届かない
- コンソールには `Uncaught RangeError` と出る

一方、iPhone の Chrome では `error` イベントに届きました（**今回確認**）。

仕様で決まっているのは、「イベントを受け取る処理の中で例外が起きたら、ブラウザがそれを報告し、`dispatchEvent` を呼んだ側には伝わらない」というところまでです（[DOM Standard の inner invoke](https://dom.spec.whatwg.org/#concept-event-listener-inner-invoke)）。PC 版で `error` イベントにも届かなかったのは、スタックが上限に達した地点では、受け取る処理そのものを呼ぶ余裕がないためだと考えています（**推測**）。

実務への影響として、`window.onerror` でエラーを集める監視サービスを入れていても、**PC 版 Chrome で起きたこのエラーは届かない可能性があります**。「エラー通知には iPhone の分しか来ていない」という場合でも、PC で起きていないとは限りません。

デモでは代わりに、`change` の処理が何段入れ子になったかを数え、20 段を超えたら画面に出すようにしています。

## 補足2：デモを作るときの落とし穴

「送信される値」を画面に表示して確かめる場合、**その表示欄も翻訳されます**（**今回確認**）。表示した JSON の文字列が翻訳され、実際には日本語で送られているのに英語に見えてしまいました。

確認用の表示欄にも `translate="no"` を付けておく必要があります。

```html
<pre id="submitted" translate="no"></pre>
```

## 点検のしかた

手元のフォームが当てはまるかは、次の2点で確認できます。

1. **`value` 属性のない `<option>` がないか。** テンプレートも含めて検索します。
   ```bash
   grep -rn "<option>" resources/ templates/
   ```
2. **表示テキストと比べて判定していないか。** `'選択してください'` のような画面の文言が、JavaScript やサーバ側の条件式に出てこないかを見ます。

1 に当てはまれば、翻訳されたときに誤登録が起きます（**仕様**と、**今回確認**した翻訳の挙動から）。1 と 2 の両方に当てはまり、さらに値のクリアと `change` の再発火を組み合わせていれば、今回のように止まる可能性があります。

## 検証環境

2026年9月17日に確認しました。

- PC 版 Chrome 153（macOS）
- Chrome for iOS 153（iOS 27）

Safari や Edge の翻訳機能では確認していません。「**今回確認**」とした挙動は、ブラウザの更新で変わる可能性があります。お使いの環境では、デモで確かめてみてください。

https://www.brainy-software.co.jp/demo/translate-crash/
