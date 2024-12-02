# Nuxt チュートリアル勉強会の草案

## 目的

## 手順

## スケジュール

## 備考

## メモ（随時更新）
* 思ったよりサクサク進むので,1単位/1時間は内容薄めになる
* 単位によるが,2単位+ドキュメント内リンク読解/1時間くらいがいいかも

# 各章のTips

## Nuxt のコンセプト
* 自動インポート
  * 自動インポートをオフにもできる.インポート元を明示したい場合に用いたい.
  * サードパーティライブラリなども自動インポートに対応させることが可能.
    * https://nuxt.com/docs/guide/concepts/auto-imports#auto-import-from-third-party-packages
* コード分割
  * https://nuxt.com/docs/guide/concepts/server-engine
  * Nitroはunjsのh3を利用して作られたサーバーフレームワーク
    * https://nitro.unjs.io/guide

## アプリケーションのエントリ
* `nuxt.config.ts`でアプリケーション全体の設定を行う
  * https://nuxt.com/docs/getting-started/configuration

## ルーティング
* `NuxtPage`はVue Routerの`RouterView`に該当する
* `NuxtLink`はVue Routerの`RouterLink`に該当する
* `NuxtLink`の`to="/sample"`の部分は`name`を使用できる
  * `:to={name: dir-file, params: {id: id}}`

## ミドルウェア
Nuxtは特定のルートにナビゲートする前にコードを実行するミドルウェアを提供する。

Nuxtはアプリケーション全体で使用できるカスタマイズ可能なルートミドルウェアフレームワークを提供し、特定のルートにナビゲートする前に実行したいコードを抽出するのに理想的です。

ルートミドルウェアには3種類ある：
1. 匿名(またはインライン)ルートミドルウェアはページ内で直接定義されます。
2. 名前付きルートミドルウェアは`middleware/`に配置され、ページで使用されるときに非同期インポートによって自動的にロードされる。
3. グローバルルートミドルウェアは`.global`接尾辞を持つ`middleware/`に置かれ、ルート変更のたびに実行される。

最初の2種類のルートミドルウェアは`definePageMeta`で定義できます。

#### 使用方法
ルートミドルウェアは現在のルートと次のルートを引数として受け取るナビゲーションガードである。

`middleware/my-middleware.ts`
```ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.params.id === '1') {
    return abortNavigation()
  }
  // In a real app you would probably not redirect every route to `/`
  // however it is important to check `to.path` before redirecting or you
  // might get an infinite redirect loop
  if (to.path !== '/') {
    return navigateTo('/')
  }
})
```

Nuxtはミドルウェアから直接返すことができるグローバルに利用可能なヘルパーを2つ提供している。
1. `navigateTo` - 指定されたルートにリダイレクトする
2. `abortNavigation` - オプションのエラーメッセージとともにナビゲーションを中止する。

可能な戻り値は以下の通り：
* nothing（単純なリターン、またはまったくリターンしない） - ナビゲーションをブロックせず、次のミドルウェア関数があればそれに移行するか、ルートナビゲーションを完了する。
* `return navigateTo('/')` - 与えられたパスにリダイレクトし、サーバー側でリダイレクトされた場合はリダイレクトコードを**302 Found**に設定する。
* `return navigateTo('/', { redirectCode: 301 })` - リダイレクトがサーバー側で行われる場合は、リダイレクトコードを**301 Moved Permanently**に設定します。
* `return abortNavigation()` - 現在のナビゲーションを停止する。
* `return abortNavigation(error)` - 現在のナビゲーションをエラーで拒否する。

> https://nuxt.com/docs/api/utils/navigate-to

> https://nuxt.com/docs/api/utils/abort-navigation

#### ミドルウェアの順序
ミドルウェアは以下の順序で実行される：
1. グローバルミドルウェア
2. ページ定義のミドルウェア（配列構文で宣言されたミドルウェアが複数ある場合）

例えば次のようなミドルウェアとコンポーネントがあるとする：
```ts
-| middleware/
---| analytics.global.ts
---| setup.global.ts
---| auth.ts
```

`pages/profile.vue`
```ts
<script setup lang="ts">
definePageMeta({
  middleware: [
    function (to, from) {
      // Custom inline middleware
    },
    'auth',
  ],
});
</script>
```

ミドルウェアは次のような順序で実行されると考えてよい：
1. `analytics.global.ts`
2. `setup.global.ts`
3. `Custom inline middleware`
4. `auth.ts`

グローバルミドルウェアの順序

デフォルトでグローバルミドルウェアはファイル名に基づいてアルファベット順に実行される。

しかし、特定の順序を定義したい場合もあるでしょう。例えば`setup.global.ts`を`analytics.global.ts`の前に実行する必要があるかもしれません。その場合はグローバルミドルウェアの前に「アルファベット順」の番号を付けることをお勧めします。

```ts
-| middleware/
---| 01.setup.global.ts
---| 02.analytics.global.ts
---| auth.ts
```

「アルファベット順」のナンバリングに慣れていない人のために言っておくと、ファイル名は数値ではなく文字列としてソートされることを覚えておいてほしい。例えば`10.new.global.ts`は`2.new.global.ts`の前に来る。これがこの例で一桁の数字の前に0をつけている理由である。

#### ミドルウェアが動作するとき
サイトがサーバーレンダリングされる場合、初期ページのミドルウェアはページがレンダリングされる時とクライアントで再度実行される時の両方で実行されます。ミドルウェアがブラウザの環境を必要とする場合、たとえば生成されたサイトがある場合や積極的にレスポンスをキャッシュする場合、ローカルストレージから値を読み込みたい場合などに、この処理が必要になることがあります。

しかし、この行動を避けたいのであれば、そうすることができる：

`middleware/example.ts`
```ts
export default defineNuxtRouteMiddleware(to => {
  // skip middleware on server
  if (import.meta.server) return
  // skip middleware on client side entirely
  if (import.meta.client) return
  // or only skip middleware on initial client load
  const nuxtApp = useNuxtApp()
  if (import.meta.client && nuxtApp.isHydrating && nuxtApp.payload.serverRendered) return
})
```

#### ミドルウェアを動的に追加する
`addRouteMiddleware()`ヘルパー関数を使うことで、手動でグローバルもしくは名前付きルートミドルウェアを追加することができます。

```ts
export default defineNuxtPlugin(() => {
  addRouteMiddleware('global-test', () => {
    console.log('this global middleware was added in a plugin and will be run on every route change')
  }, { global: true })

  addRouteMiddleware('named-test', () => {
    console.log('this named middleware was added in a plugin and would override any existing middleware of the same name')
  })
})
```

> https://nuxt.com/docs/api/utils/add-route-middleware

#### ビルド時のミドルウェア設定
各ページで`definePageMeta`を使う代わりに、`pages:extend`フックの中に名前付きルートミドルウェアを追加することができます。
