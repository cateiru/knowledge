# ホーム画面への追加（PWA のインストール）を勧めるバナーを出す

Next.js（App Router）の PWA で、ホーム画面から開いていないときだけ画面の下にバナーを出し、Android ではその場で追加の確認を出し、iOS では追加方法のヘルプへ案内するようにしたときのメモ（2026年11月）。

## 方針

- ホーム画面から開いているか（`display-mode: standalone`、iOS Safari は `navigator.standalone`）を見て、開いていればバナーを出さない。
- Chrome などは、インストールできると判断したときに `beforeinstallprompt` を発火する。このイベントの `prompt()` を呼ぶと、ブラウザの追加の確認が出る。
- iOS の Safari には `beforeinstallprompt` がなく、共有メニューから自分で「ホーム画面に追加」をする必要がある。そのため、iOS では手順を説明するページへのリンクにする。
- iOS 以外で `beforeinstallprompt` が来ないブラウザ（Firefox など）では、押しても何もできないのでバナーを出さない。

## beforeinstallprompt はモジュールの読み込み時から受け取る

React の `useEffect` で `addEventListener` すると、登録はハイドレーションの後になる。ページを読み込んですぐにイベントが来た場合は取りこぼすおそれがあるため、クライアントのモジュールが読み込まれた時点で登録し、受け取ったイベントをモジュールの変数に保持する。部品からは `useSyncExternalStore` で読む。

なお、実際に取りこぼしが起きるかは確かめていない。予防のための実装。

```ts
let deferredPrompt: BeforeInstallPromptEvent | null = null;
const listeners = new Set<() => void>();

if (typeof window !== "undefined") {
  window.addEventListener("beforeinstallprompt", (event) => {
    // ブラウザ既定の案内を出さず、バナーのボタンから出す
    event.preventDefault();
    deferredPrompt = event as BeforeInstallPromptEvent;
    for (const listener of listeners) listener();
  });
}

export function subscribe(listener: () => void) {
  listeners.add(listener);
  return () => listeners.delete(listener);
}
export const getSnapshot = () => deferredPrompt;
export const getServerSnapshot = () => null;
```

`BeforeInstallPromptEvent` は TypeScript の DOM の型にないため、`prompt(): Promise<void>` と `userChoice: Promise<{ outcome: "accepted" | "dismissed" }>` を持つ型を自分で定義する。

`prompt()` は 1 つのイベントで 1 回しか呼べないため、呼んだらそのイベントを捨てる。`appinstalled` を受け取ったときも捨てる。

## iPad は Mac と同じ UA になる

iPadOS 13 以降の Safari は、既定で Mac と同じ UA（`Macintosh`）を返す。`/iphone|ipad|ipod/i` だけでは iPad を見分けられないため、`Macintosh` で `navigator.maxTouchPoints > 1` のときも iOS として扱う。

## 画面の下に固定するときは、同じ高さの余白を空ける

`position: fixed` のバナーは、ページの最後の内容（フッターや保存ボタン）を隠してしまう。バナーの高さを `ResizeObserver` で測り、ページの最後に同じ高さの要素を置いて、最後までスクロールすれば内容が見えるようにした。

`position: sticky; bottom: ...` にすれば余白は要らないが、今回のページは `html` と `body` の両方に `overflow-x: hidden` があった。両方に `overflow` があると `body` の `overflow` はビューポートに引き継がれず、`body` 自体がスクロールコンテナになるため、`sticky` が期待どおりに効かないと判断して使わなかった（試してはいない）。

## Playwright でバナーを撮る

### ヘッドレスの Chromium では beforeinstallprompt が来ない

開発サーバーでは Service Worker も登録していないなど条件がそろわず、Playwright のヘッドレスの Chromium では `beforeinstallprompt` が発火しなかった。VRT やヘルプのスクリーンショットでは、`prompt` と `userChoice` を持たせた `Event` を自分で発火してバナーを出す。

ハイドレーション前に発火すると受け取られないことがあるため、バナーのボタンが出るまで `toPass` で繰り返す。

```ts
const installButton = page.getByRole("button", { name: "<ボタンの名前>" });
await expect(async () => {
  await page.evaluate(() => {
    window.dispatchEvent(
      Object.assign(new Event("beforeinstallprompt"), {
        prompt: async () => {},
        userChoice: new Promise(() => {}),
      }),
    );
  });
  await expect(installButton).toBeVisible({ timeout: 2000 });
}).toPass({ timeout: 30_000 });
```

iOS 向けの表示は、`test.use({ userAgent: "<iPhone の Safari の UA>", viewport, isMobile: true, hasTouch: true })` で撮れる。`devices["iPhone 13"]` をそのまま `test.use` に広げると `defaultBrowserType` が含まれ、describe の中では使えないので、必要な項目だけを書く。

### fullPage では固定表示の要素がページの途中に写る

`page.screenshot({ fullPage: true })` で撮ると、`position: fixed` のバナーは撮影時のビューポートの位置に描かれ、縦に長いページの途中に浮いて写った。画面の下に固定した見た目を撮りたいときは `fullPage: false` にして、見えている範囲だけを撮る。

### Vitest でのテスト

jsdom でも `window.dispatchEvent` で同じイベントを発火すればバナーの表示と `prompt()` の呼び出しを確かめられる。`navigator.userAgent` は `vi.spyOn(navigator, "userAgent", "get")`、`matchMedia` は `vi.stubGlobal` で差し替える。モジュールの変数に残ったイベントは、テストごとに捨てる関数を呼んで消す。
