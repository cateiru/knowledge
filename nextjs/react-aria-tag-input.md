# react-aria-components の ComboBox でタグの入力欄（TagInput）を作る

2026年11月に、文字を入力して Enter でタグにし、× で外せる入力欄を、react-aria-components 1.22 の `ComboBox` で作った。既存のタグを候補に出し、候補にない文字は新しいタグにできるようにした。そのときに確かめたことを書く。

## 構成

タグの並びと `ComboBox` を、枠を描く 1 つの要素の中に並べる。タグは `ComboBox` の外に置く。

```tsx
<label id={labelId} htmlFor={inputId}>{label}</label>
<div ref={controlRef} onKeyDownCapture={handleKeyDown}>
  <ul>{/* タグと × のボタン */}</ul>
  <ComboBox
    aria-labelledby={labelId}
    items={options}
    inputValue={inputValue}
    onInputChange={setInputValue}
    selectedKey={null}
    onSelectionChange={(key) => { /* 選んだ候補をタグにする */ }}
    allowsCustomValue
    menuTrigger="focus"
    validationBehavior="aria"
  >
    <Input ref={inputRef} id={inputId} />
    <Popover triggerRef={controlRef} style={{ width: menuWidth }}>
      <ListBox>{(option) => <ListBoxItem id={option.id}>{option.label}</ListBoxItem>}</ListBox>
    </Popover>
  </ComboBox>
</div>
```

- 選んだ候補は自分の状態（タグの配列）に足し、`selectedKey` は常に `null` にする。`inputValue` と `selectedKey` の両方を渡すと、選択した後の入力欄の書き換えは利用側に任されるため、自分で空にする。
- `items` を渡すと `ComboBox` は候補を絞り込まない。付いているタグの除外、入力した文字での絞り込み、「「…」を追加」の候補の追加は自分で行う。
- `allowsCustomValue` を付けないと、フォーカスが外れたときに入力途中の文字が消える。

`ComboBox` の子は、候補を集めるためにもう一度見えないところで描かれる。`Tag` を中に置くと候補と混ざるおそれがあると考えて外に出した（中に置いて試してはいない）。

## Enter の扱い

`onKeyDownCapture` で、`ComboBox` とフォームより先に Enter を受け取る。

```ts
if (event.target !== inputRef.current) return;
if (event.nativeEvent.isComposing || event.keyCode === 229) {
  event.stopPropagation();
  return;
}
if (event.key === "Enter") {
  // 矢印キーで候補を選んでいるときは、ComboBox に確定させる
  if (inputRef.current.getAttribute("aria-activedescendant")) return;
  event.preventDefault();
  event.stopPropagation();
  addTag(inputValue.trim());
}
```

- 日本語の変換を確定する Enter では追加しない。Safari は、変換を確定する Enter を、変換が終わった後の `keyCode` が 229 のキー入力として送ることが知られているため、`isComposing` と両方を見る（Safari の実機では試していない）。
- 変換中のキーは `stopPropagation` で `ComboBox` にも渡さない。渡すと、`isComposing` が `false` で `keyCode` が 229 の Enter（Safari の場合）で候補の一覧が閉じた。`preventDefault` はしないので、変換の操作には影響しない。
- `preventDefault` しないと、フォームの中では Enter で送信されてしまう。
- 候補を選んでいるかは `aria-activedescendant` で分かる。見ないと、選んだ候補と入力途中の文字の 2 つが付く。

### 変換の操作を確かめる

Playwright の `fill`・`press` では変換を再現できない。Chromium では DevTools Protocol で再現できた。

```ts
const cdp = await page.context().newCDPSession(page);
// 変換中の文字を表示する
await cdp.send("Input.imeSetComposition", { text: "病院", selectionStart: 2, selectionEnd: 2 });
// 変換中のキー入力。keyCode は 229 になる
await cdp.send("Input.dispatchKeyEvent", { type: "rawKeyDown", key: "Enter", code: "Enter", windowsVirtualKeyCode: 229 });
// 変換を確定する
await cdp.send("Input.insertText", { text: "病院" });
```

`compositionstart`・`compositionend` と、`isComposing` が `true` の `keydown` が発火する。確定の後に `keyCode` 229 の Enter を送ると、Safari の順序も再現できる。

## 候補の幅

`Popover` の `--trigger-width` は、`triggerRef` に枠の要素を渡しても、文字の入力欄の幅になった。タグと同じ行に並ぶと候補の一覧が狭くなるため、`onOpenChange` で枠の `offsetWidth` を測って `style` で渡した。

## 候補を開いている間は、ほかの要素が読み上げから隠れる

`ComboBox` は、候補を開いている間、入力欄と候補の外に `aria-hidden` を付ける。タグを足した後も候補が開いたままだと、次のようになる。

- `getByRole` でタグの × のボタンが見つからない（Playwright で確認）。`includeHidden: true` を付けるか、Escape で候補を閉じてから探す。隠れていても、クリックはできる。
- `role="status"` で「追加しました」と伝えていても、その要素ごと隠れる。`data-live-announcer="true"` を付けると、隠す対象から外れる（`ariaHideOutside` の実装が、この属性の要素を残している）。

## フォームで送る

タグごとに hidden の input を置く。状態から描いているため、`<form action>` の後に React がフォームをリセットしても消えない（[チェックボックスが元に戻る問題](react-aria-checkbox-form-reset.md) が起きない）。

Enter を押さずに保存したときの付け忘れを防ぐため、入力途中の文字も hidden の input で一緒に送った。フォーカスが外れたときにタグにする方法もあるが、タグが増えて行が折り返すと、押そうとしていたボタンの位置がずれてクリックが外れるおそれがあるため採らなかった。

タグの実体がサーバーにある（名前を変えられる）ときは、文字だけを送ってはいけない。フォームを開いている間に別の画面で名前が変わると、保存したときに古い名前のタグが作り直され、元のタグが外れてしまう。次のようにした。

- 既存のタグは `{ id, name }`、新しく入力したタグは `{ id: null, name }` を JSON にして、hidden の input の値にする。
- サーバーでは、ID のタグがあれば ID で付ける。名前が変わっていても元のタグのままになる。
- ID のタグが削除されていたら、名前で作って付ける。エラーにすると、同じ文字を入力し直しても古い ID が送られ、フォームを開き直すまで保存できない。

2 つのページを開き、片方で編集の画面を開いたまま、もう片方で名前を変えてから保存する E2E で確かめた。

## エラーを出した後に送信できなくなる

`isInvalid` を渡すと、既定ではブラウザの検証に反映され、直しても送信できなくなる。`ComboBox` でも起きたため、`validationBehavior="aria"` を付けた（[サーバーの入力エラーの後に送信できない問題](react-aria-server-validation-resubmit.md)）。送信のボタンを押して `onSubmit` が呼ばれるかを確かめる単体テスト（jsdom）で再現できた。
