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

## レイアウト
Nuxtは、一般的なUIパターンを再利用可能なレイアウトに抽出するレイアウト・フレームワークを提供します。

### レイアウトを有効にする
レイアウトは、`app.vue`に`<NuxtLayout>`を追加することで有効になります。

```ts
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

レイアウトを使用するには
* `definePageMeta`でページにレイアウト・プロパティを設定します
* `<NuxtLayout>`の`name`プロパティを設定します

レイアウトが指定されていない場合は、`layouts/default.vue`が使用されます。
アプリケーションにレイアウトが1つしかない場合は、代わりに`app.vue`を使うことをお勧めします。

### デフォルトレイアウト
`~layouts/default.vue`を追加します。

`layouts/default.vue`
```ts
<template>
  <div>
    <p>Some default layout content shared across all pages</p>
    <slot />
  </div>
</template>
```
レイアウトファイルでは、ページの内容は`<slot />`コンポーネントに表示されます。

### 名前付きレイアウト
```ts
-| layouts/
---| default.vue
---| custom.vue
```
以下のようにページでカスタムレイアウトを使うことができます。
`pages/about.vue`
```ts
<script setup lang="ts">
definePageMeta({
  layout: 'custom'
})
</script>
```
`<NuxtLayout>`の`name`プロパティを使用すると、すべてのページのデフォルト・レイアウトを直接オーバーライドできます。

`app.vue`
```ts
<script setup lang="ts">
// You might choose this based on an API call or logged-in status
const layout = "custom";
</script>

<template>
  <NuxtLayout :name="layout">
    <NuxtPage />
  </NuxtLayout>
</template>
```
ネストされたディレクトリにレイアウトがある場合、レイアウトの名前はそれ自身のパスディレクトリとファイル名に基づき、重複するセグメントは削除されます。

| File | Layout Name |
| ---- | ---- |
| ~/layouts/desktop/default.vue | desktop-defaultD |
| ~/layouts/desktop-base/base.vue | desktop-base |
| ~/layouts/desktop/index.vue | desktop |

わかりやすくするため、レイアウトのファイル名はレイアウト名と同じにすることをお勧めします。
| File | Layout Name |
| ---- | ---- |
| ~/layouts/desktop/DesktopDefault.vue | desktop-defaultD |
| ~/layouts/desktop-base/DesktopBase.vue | desktop-base |
| ~/layouts/desktop/Desktop.vue | desktop |

### レイアウトを動的に変更する
`setPageLayout` ヘルパーを使うと、動的にレイアウトを変更することもできます。
```ts
<script setup lang="ts">
function enableCustomLayout () {
  setPageLayout('custom')
}
definePageMeta({
  layout: false,
});
</script>

<template>
  <div>
    <button @click="enableCustomLayout">Update layout</button>
  </div>
</template>
```

### ページ単位でレイアウトを上書きする
ページを使用している場合は、`layout: false`を設定し、ページ内で`<NuxtLayout>`コンポーネントを使用することで、完全に制御することができます。

`pages/index.vue`
```ts
<script setup lang="ts">
definePageMeta({
  layout: false,
})
</script>

<template>
  <div>
    <NuxtLayout name="custom">
      <template #header> Some header template content. </template>

      The rest of the page
    </NuxtLayout>
  </div>
</template>
```

## レンダリングモード
Nuxtで利用可能なさまざまなレンダリングモードについてご紹介します。

Nuxtはさまざまなレンダリングモード、ユニバーサルレンダリング、クライアントサイドレンダリングをサポートしていますが、ハイブリッドレンダリングやCDNエッジサーバでのアプリケーションのレンダリングも可能です。 ブラウザとサーバの両方がJavaScriptコードを解釈して、Vue.jsコンポーネントをHTML要素に変換することができます。 このステップをレンダリングと呼びます。 Nuxtはユニバーサルレンダリングとクライアントサイドレンダリングの両方をサポートしています。 この2つのアプローチには利点と欠点があり、これから説明します。

デフォルトでは、Nuxtはより良いユーザーエクスペリエンスとパフォーマンスを提供し、検索エンジンのインデックスを最適化するためにユニバーサルレンダリングを使用しますが、1行の設定でレンダリングモードを切り替えることができます。

### ユニバーサルレンダリング
このステップは、PHPやRubyアプリケーションで実行される従来のサーバーサイドレンダリングに似ています。 ブラウザがユニバーサルレンダリングを有効にしたURLを要求すると、NuxtはJavaScript (Vue.js) コードをサーバ環境で実行し、完全にレンダリングされたHTMLページをブラウザに返します。 ページが事前に生成されている場合は、Nuxtはキャッシュから完全にレンダリングされたHTMLページを返すこともあります。 ユーザーは、クライアントサイドレンダリングとは逆に、アプリケーションの初期コンテンツ全体を即座に取得できます。

HTMLドキュメントがダウンロードされると、ブラウザがこれを解釈し、Vue.jsがドキュメントを制御します。 かつてサーバー上で実行されたのと同じJavaScriptコードが、クライアント（ブラウザ）上でバックグラウンドで再び実行され、リスナーをHTMLにバインドすることで、インタラクティブ性（したがってユニバーサルレンダリング）を実現します。 これをハイドレーションと呼びます。 ハイドレーションが完了すると、ページはダイナミックなインターフェイスやページ遷移などのメリットを享受できるようになる。

ユニバーサルレンダリングにより、Nuxtアプリケーションはクライアントサイドレンダリングの利点を維持しながら、ページのロード時間を短縮できます。 さらに、コンテンツはすでにHTMLドキュメントに存在しているため、クローラはオーバーヘッドなしにインデックスを作成できます。

**何がサーバーサイドレンダリングで、何がクライアントレンダリングなのか？**

ユニバーサルレンダリングモードでは、Vueファイルのどの部分がサーバやクライアントで実行されるかを確かめるのが普通です。
```ts
<script setup lang="ts">
const counter = ref(0); // executes in server and client environments

const handleClick = () => {
  counter.value++; // executes only in a client environment
};
</script>

<template>
  <div>
    <p>Count: {{ counter }}</p>
    <button @click="handleClick">Increment</button>
  </div>
</template>
```

最初のリクエストでは、カウンターrefは`<p>`タグ内にレンダリングされるため、サーバ内で初期化されます。 `handleClick`の内容がここで実行されることはありません。 ブラウザでのハイドレーションの間、カウンターrefは再度初期化されます。 したがって、`handleClick`の本体は常にブラウザ環境で実行されると推論するのが妥当です。

ミドルウェア（`Middlewares`）とページ（`pages `）はハイドレーションの間、サーバーとクライアントで実行されます。 プラグイン（`Plugins`）はサーバーまたはクライアント、あるいはその両方でレンダリングできます。 コンポーネント（`Components`）は、強制的にクライアントのみで実行させることもできます。 コンポーザブル（`Composables`）とユーティリティ（`utilities`）は、その使用状況に応じてレンダリングされます。

**サーバーサイドレンダリングの利点**
パフォーマンス： ブラウザは静的コンテンツをJavaScriptで生成したコンテンツよりもはるかに高速に表示できるため、ユーザーはページのコンテンツにすぐにアクセスできます。 同時に、Nuxtはハイドレーション処理中もWebアプリケーションのインタラクティブ性を維持します。

検索エンジン最適化： ユニバーサルレンダリングは、ページのHTMLコンテンツ全体を古典的なサーバーアプリケーションとしてブラウザに配信します。 Webクローラはページのコンテンツを直接インデックスできるため、ユニバーサルレンダリングは、すばやくインデックスしたいコンテンツに最適です。

**サーバーサイドレンダリングの欠点**
開発上の制約： サーバ環境とブラウザ環境は同じAPIを提供しておらず、両側でシームレスに実行できるコードを書くのは厄介です。 幸いなことに、Nuxtはコードの一部がどこで実行されるかを判断するためのガイドラインと特定の変数を提供しています。

コストがかかる： オンザフライでページをレンダリングするためには、サーバーが稼働している必要があります。 これには従来のサーバーと同様に毎月のコストがかかります。 しかし、クライアント側のナビゲーションをブラウザが引き継ぐユニバーサルレンダリングのおかげで、サーバー呼び出しは大幅に削減されます。 エッジサイドレンダリングを活用すれば、コスト削減が可能になります。

ユニバーサルレンダリングは非常に汎用性が高く、ほとんどのユースケースに適合します。特に、ブログ、マーケティングウェブサイト、ポートフォリオ、eコマースサイト、マーケットプレイスなど、コンテンツ指向のウェブサイトに適しています。
