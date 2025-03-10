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

### クライアントサイドレンダリング
従来のVue.jsアプリケーションはブラウザでレンダリングされます。 そして、Vue.jsはブラウザが現在のインターフェイスを作成するための命令を含むすべてのJavaScriptコードをダウンロードして解析した後、HTML要素を生成します。

**クライアントサイドレンダリングの利点**
開発スピード： 完全にクライアントサイドで作業する場合、例えばwindowオブジェクトのようなブラウザ専用のAPIを使うことで、コードのサーバー互換性を心配する必要がありません。

安い： JavaScriptをサポートするプラットフォーム上で実行する必要があるため、サーバーを実行するとインフラのコストがかかります。 私たちは、HTML、CSS、JavaScriptファイルを持つ静的なサーバー上で、クライアントのみのアプリケーションをホストすることができます。

オフライン： コードはすべてブラウザで実行されるため、インターネットが利用できない状態でもうまく動作し続けることができます。

**クライアントサイドレンダリングの欠点**
パフォーマンス： ユーザーは、ブラウザがJavaScriptファイルをダウンロード、解析、実行するのを待つ必要があります。 ダウンロード部分のネットワークと、解析と実行のためのユーザーのデバイスによっては、時間がかかり、UXに影響を与える可能性があります。

検索エンジンの最適化： クライアントサイドレンダリングで配信されたコンテンツのインデックス化と更新には、サーバーサイドレンダリングのHTMLドキュメントより時間がかかります。 検索エンジンのクローラーは、ページのインデックスを最初に試みるときに、インターフェイスが完全にレンダリングされるのを待たないからです。 純粋なクライアントサイドレンダリングでは、検索結果ページにコンテンツが表示され、更新されるまでに時間がかかります。

クライアントサイドレンダリングは、インデックス作成が不要な、またはユーザーが頻繁にアクセスするような、インタラクティブ性の高いウェブアプリケーションに適しています。 ブラウザのキャッシュを活用して、SaaS、バックオフィスアプリケーション、オンラインゲームなど、次回以降の訪問時にダウンロードフェーズをスキップすることができます。

Nuxtでは、`nuxt.config.ts`でクライアントサイドレンダリングを有効にすることができます。

`nuxt.config.ts`
```ts
export default defineNuxtConfig({
  ssr: false
})
```

### ハイブリッドレンダリング
ハイブリッドレンダリングは、ルートルールを使用してルートごとに異なるキャッシュルールを可能にし、指定されたURLの新しいリクエストに対してサーバーがどのように応答すべきかを決定します。

以前は、Nuxtアプリケーションとサーバーのすべてのルート/ページで、ユニバーサルまたはクライアントサイドの同じレンダリングモードを使用する必要がありました。 さまざまなケースで、ビルド時に生成されるページもあれば、クライアントサイドでレンダリングされるページもあります。 例えば、管理セクションのあるコンテンツウェブサイトを考えてみましょう。 すべてのコンテンツページは主に静的で、一度生成されたものであるべきですが、管理セクションは登録を必要とし、動的なアプリケーションのように振る舞います。

Nuxtにはルートルールとハイブリッドレンダリングのサポートが含まれています。 ルートルールを使用すると、nuxtルートのグループに対してルールを定義したり、レンダリングモードを変更したり、ルートに基づいてキャッシュ戦略を割り当てることができます！

Nuxtサーバーは、対応するミドルウェアを自動的に登録し、Nitroキャッシュレイヤーを使用してキャッシュハンドラでルートをラップします。

`nuxt.config.ts`
```ts
export default defineNuxtConfig({
  routeRules: {
    // Homepage pre-rendered at build time
    '/': { prerender: true },
    // Products page generated on demand, revalidates in background, cached until API response changes
    '/products': { swr: true },
    // Product pages generated on demand, revalidates in background, cached for 1 hour (3600 seconds)
    '/products/**': { swr: 3600 },
    // Blog posts page generated on demand, revalidates in background, cached on CDN for 1 hour (3600 seconds)
    '/blog': { isr: 3600 },
    // Blog post page generated on demand once until next deployment, cached on CDN
    '/blog/**': { isr: true },
    // Admin dashboard renders only on client-side
    '/admin/**': { ssr: false },
    // Add cors headers on API routes
    '/api/**': { cors: true },
    // Redirects legacy urls
    '/old-page': { redirect: '/new-page' }
  }
})
```

### ルートルール
使用できるプロパティは以下の通りです。

`redirect: string` - サーバーサイドリダイレクトを定義する

`ssr: boolean` - アプリのセクションのHTMLのサーバーサイドレンダリングを無効にし、ssr: falseでブラウザーのみでレンダリングする

`cors: boolean` - 自動的にcorsヘッダーを追加する cors: true - headersでオーバーライドすることで出力をカスタマイズできる

`headers: object` - サイトのセクションに特定のヘッダを追加します - 例えば、アセット

`swr: number | boolean` - サーバーレスポンスにキャッシュヘッダを追加し、設定可能な TTL (time to live) の間、サーバーまたはリバースプロキシにキャッシュします。 Nitroのnode-serverプリセットは、フルレスポンスをキャッシュすることができます。 TTLが切れると、キャッシュされたレスポンスが送信され、バックグラウンドでページが再生成されます。 true を使うと、MaxAge なしで stale-while-revalidate ヘッダが追加されます。

`isr: number | boolean` - 動作はswrと同じですが、CDNキャッシュをサポートしているプラットフォーム（現在のところNetlifyまたはVercel）では、レスポンスをCDNキャッシュに追加することができます。

`prerender: boolean` - ビルド時にルートをプリレンダリングし、静的アセットとしてビルドに含めます。

`experimentalNoScripts: boolean` - サイトのセクションに対して、NuxtスクリプトとJSリソースヒントのレンダリングを無効にします。

`appMiddleware: string | string[] | Record<string, boolean>` - アプリケーションのVueアプリ部分（つまり、Nitroルートではない）内のページパスに対して実行する、または実行しないミドルウェアを定義できます。

可能な限り、ルートルールはデプロイプラットフォームのネイティブルールに自動的に適用され、最適なパフォーマンスを実現します（現在、NetlifyとVercelがサポートされています）。

### エッジサイドレンダリング
エッジサイドレンダリング（ESR）はNuxtに導入された強力な機能で、コンテンツデリバリーネットワーク（CDN）のエッジサーバを経由して、Nuxtアプリケーションのレンダリングをユーザに近づけることができます。 ESRを活用することで、パフォーマンスの向上と待ち時間の短縮を実現し、ユーザーエクスペリエンスの向上を図ることができます。

ESRでは、レンダリングプロセスはネットワークの「エッジ」、つまりCDNのエッジサーバーにプッシュされます。 ESRは、実際のレンダリングモードというよりも、デプロイメントターゲットであることに注意してください。

ページへのリクエストが行われると、元のサーバーに送られるのではなく、最も近いエッジサーバーによってインターセプトされます。 このサーバーがページのHTMLを生成し、ユーザーに送り返します。 このプロセスにより、データの物理的な移動距離が最小化され、待ち時間が短縮され、ページの読み込みが速くなります。

Nuxt3を支えるサーバーエンジンNitroのおかげで、エッジサイドレンダリングが可能になりました。 Node.js、Deno、Cloudflare Workersなどをクロスプラットフォームでサポートしています。

現在ESRを活用できるプラットフォームは以下の通りです。
* gitインテグレーションとnuxtビルドコマンドを使用した設定ゼロの**Cloudflare Pages**

* nuxtビルドコマンドとNITRO_PRESET=vercel-edge環境変数を使用した**Vercel Edge Functions**

* nuxtビルドコマンドとNITRO_PRESET=netlify-edge環境変数を使用した**Netlify Edge Functions**

## データフェッチ
Nuxtはアプリケーション内のデータ取得を処理するためのコンパイラを提供します。

Nuxtにはブラウザまたはサーバ環境でデータ・フェッチを実行するための2つのComposableと組み込みライブラリが用意されています。`useFetch`、`useAsyncData`、`$fetch`です。

一言で言うと
* `$fetch`はネットワーク・リクエストを行う最も簡単な方法です。
* `useFetch`は`$fetch`のラッパーで、ユニバーサルレンダリングで一度だけデータをフェッチします。
* `useAsyncData`は`useFetch`に似ていますが、よりきめ細かい制御が可能です。

`useFetch`も`useAsyncData`も、共通のオプションとパターンを持っています。

### useFetchとuseAsyncDataの必要性
Nuxtはサーバーとクライアントの両方の環境で同型のコードを実行できるフレームワークです。Vueコンポーネントのセットアップ関数で`$fetch`関数を使用してデータフェッチを実行すると、データが2回フェッチされる可能性があります。1回はサーバで（HTMLをレンダリングするために）、もう1回はクライアントで（HTMLがハイドレートされるときに）です。これによりハイドレーションの問題が発生したり、インタラクティブになるまでの時間が長くなったり、予測できない動作が発生したりする可能性があります。

`useFetch`および`useAsyncData`コンポーザブルは、サーバー上でAPIコールが行われた場合、データがペイロードでクライアントに転送されるようにすることでこの問題を解決します。

ペイロードは`useNuxtApp().payload`を通じてアクセス可能なJavaScriptオブジェクトです。これはハイドレーション中にブラウザでコードが実行されたときに、同じデータの再フェッチを回避するためにクライアントで使用されます。
`app.vue`
```ts
<script setup lang="ts">
const { data } = await useFetch('/api/data')

async function handleFormSubmit() {
  const res = await $fetch('/api/submit', {
    method: 'POST',
    body: {
      // My form data
    }
  })
}
</script>

<template>
  <div v-if="data == null">
    No data
  </div>
  <div v-else>
    <form @submit="handleFormSubmit">
      <!-- form input tags -->
    </form>
  </div>
</template>
```

上記の例では`useFetch`はリクエストがサーバーで発生し、適切にブラウザに転送されることを確認します。`$fetch`にはそのような仕組みはなく、リクエストがブラウザからのみ行われる場合に使用するのがよいでしょう。

#### Suspense
NuxtはVueの`<Suspense>`コンポーネントを使用して、すべての非同期データがビューで利用可能になる前にナビゲーションが行われないようにします。データ取得のコンポーザブルはこの機能を活用し、呼び出しごとに最適なものを使用するのに役立ちます。

### $fetch
Nuxtにはofetchライブラリが含まれており、アプリケーション全体で`$fetch`エイリアスとして自動インポートされます。
`pages/todos.vue`
```ts
<script setup lang="ts">
async function addTodo() {
  const todo = await $fetch('/api/todos', {
    method: 'POST',
    body: {
      // My todo data
    }
  })
}
</script>
```

#### クライアント・ヘッダーをAPIに渡す
サーバで`useFetch`を呼び出す場合、Nuxtは`useRequestFetch`を使用してクライアントのヘッダとCookieをプロキシします（host のような転送されないヘッダを除く）。

```ts
<script setup lang="ts">
const { data } = await useFetch('/api/echo');
</script>
```

```ts
// /api/echo.ts
export default defineEventHandler(event => parseCookies(event))
```

あるいは以下の例では、`useRequestHeaders`を使用して、(クライアントを起点とする) サーバ側リクエストからAPIにアクセスしてクッキーを送信する方法を示しています。同型の`$fetch`呼び出しを使用することで、APIエンドポイントがユーザのブラウザによって元々送信されたのと同じCookieヘッダにアクセスできるようにしています。これは`useFetch`を使用していない場合にのみ必要です。

```ts
<script setup lang="ts">
const headers = useRequestHeaders(['cookie'])

async function getCurrentUser() {
  return await $fetch('/api/me', { headers })
}
</script>
```

### useFetch
`useFetch`コンポーザブルは`$fetch`を内部で使い、setup関数内でSSRセーフなネットワークコールを行います。
`app.vue`
```ts
<script setup lang="ts">
const { data: count } = await useFetch('/api/count')
</script>

<template>
  <p>Page visits: {{ count }}</p>
</template>
```

このコンポーザブルは`useAsyncData`コンポーザブルと`$fetch`ユーティリティのラッパーです。

### useAsyncData
`useAsyncData`コンポーザブルは非同期ロジックをラップし、解決したら結果を返します。

たとえばCMSやサードパーティが独自のクエリ層を提供している場合などです。このような場合は`useAsyncData`を使用して呼び出しをラップすることができます。
`pages/users.vue`
```ts
<script setup lang="ts">
const { data, error } = await useAsyncData('users', () => myGetFunction('users'))

// This is also possible:
const { data, error } = await useAsyncData(() => myGetFunction('users'))
</script>
```

`pages/users/[id].vue`
```ts
<script setup lang="ts">
const { id } = useRoute().params

const { data, error } = await useAsyncData(`user:${id}`, () => {
  return myGetFunction('users', { id })
})
</script>
```

`useAsyncData`コンポーザブルは複数の`$fetch`リクエストが完了するのをラップして待ち、結果を処理する素晴らしい方法です。

```ts
<script setup lang="ts">
const { data: discounts, status } = await useAsyncData('cart-discount', async () => {
  const [coupons, offers] = await Promise.all([
    $fetch('/cart/coupons'),
    $fetch('/cart/offers')
  ])

  return { coupons, offers }
})
// discounts.value.coupons
// discounts.value.offers
</script>
```

### 戻り値
`useFetch`と`useAsyncData`の戻り値は同じです。

* `data`: 渡された非同期関数の結果。
* `refresh`/`execute`: ハンドラー関数から返されたデータをリフレッシュするために使用できる関数。
* `clear`: データを`undefined`に設定し、エラーを`null`に設定し、ステータスを`idle`に設定し、現在保留中のリクエストをキャンセル済みとしてマークするために使用できる関数。
* `error`: データフェッチに失敗した場合のエラーオブジェクト
* `status`: データリクエストのステータスを示す文字列（"idle"、"pending"、"success"、"error"）。

`data`、`error`、`status`は、`<script setup>`の`.value`でアクセス可能なVueの`ref`です。
デフォルトではNuxtは`refresh`が終了するまで待ってから再実行します。

### オプション
`useAsyncData`と`useFetch`は同じオブジェクト型を返し、共通のオプションを最後の引数として受け取ります。これらはナビゲーションのブロック、キャッシュ、実行などコンポーザブルの動作を制御するのに役立ちます。

#### Lazy
デフォルトでデータ取得のコンポーザブルはVueの`Susppense`を使用して、新しいページに移動する前に非同期関数の解決を待ちます。この機能は`lazy`オプションを使用すると、クライアント側のナビゲーションで無視できます。その場合は`status`を使用して手動でロード状態を処理する必要があります。
`app.vue`
```ts
<script setup lang="ts">
const { status, data: posts } = useFetch('/api/posts', {
  lazy: true
})
</script>

<template>
  <!-- you will need to handle a loading state -->
  <div v-if="status === 'pending'">
    Loading ...
  </div>
  <div v-else>
    <div v-for="post in posts">
      <!-- do something -->
    </div>
  </div>
</template>
```

代わりに`useLazyFetch`や`useLazyAsyncData`を便利なメソッドとして使うこともできる。

```ts
<script setup lang="ts">
const { status, data: posts } = useLazyFetch('/api/posts')
</script>
```

#### クライアントのみのフェッチ
デフォルトではデータフェッチコンポーザブルは、クライアントとサーバーの両方の環境で非同期機能を実行します。クライアント側でのみ呼び出しを実行するには、サーバーオプションを`false`に設定します。最初のロードではハイドレーションが完了する前にデータはフェッチされないので、保留状態を処理する必要があります。

`lazy`オプションと組み合わせることで、最初のレンダリングで必要とされないデータ（例えばSEO上重要でないデータなど）に有効である。

```ts
/* This call is performed before hydration */
const articles = await useFetch('/api/article')

/* This call will only be performed on the client */
const { status, data: comments } = useFetch('/api/comments', {
  lazy: true,
  server: false
})
```

`useFetch`コンポーザブルはsetup関数内で呼び出されるか、ライフサイクルフックの関数のトップレベルで直接呼び出されることを意図しています。

#### ペイロードサイズの最小化
`pick`オプションを使用するとコンポーザブルから返したいフィールドのみを選択することで、HTMLドキュメントに保存されるペイロードサイズを最小限に抑えることができます。

```ts
<script setup lang="ts">
/* only pick the fields used in your template */
const { data: mountain } = await useFetch('/api/mountains/everest', {
  pick: ['title', 'description']
})
</script>

<template>
  <h1>{{ mountain.title }}</h1>
  <p>{{ mountain.description }}</p>
</template>
```

もっとコントロールが必要な場合や複数のオブジェクトをマッピングする必要がある場合は、`transform`関数を使ってクエリの結果を変更することができます。

#### キャッシュと再フェッチ
##### Keys
`useFetch`と`useAsyncData`は同じデータの再フェッチを防ぐためにキーを使用します。
* `useFetch`は指定されたURLをキーとして使用する。また最後の引数として渡されるオプションでキー値を指定することもできます。
* `useAsyncData`は最初の引数が文字列の場合、その引数をキーとして使用します。最初の引数がクエリを実行するハンドラ関数の場合、`useAsyncData`のインスタンスのファイル名と行番号に固有のキーが生成されます。

#### リフレッシュと実行
データを手動でフェッチまたはリフレッシュしたい場合は、コンポーザブルが提供する`execute`または`refresh`関数を使用します。

```ts
<script setup lang="ts">
const { data, error, execute, refresh } = await useFetch('/api/users')
</script>

<template>
  <div>
    <p>{{ data }}</p>
    <button @click="() => refresh()">Refresh data</button>
  </div>
</template>
```

`execute`関数は`refresh`の別名でまったく同じように動作しますが、フェッチが即時でない場合にはよりセマンティックです。

#### Clear
何らかの理由で`clearNuxtData`に渡す特定のキーを知らなくても、提供されたデータをクリアしたい場合はcomposablesが提供する`clear`関数を使うことができます。

```ts
<script setup lang="ts">
const { data, clear } = await useFetch('/api/users')

const route = useRoute()
watch(() => route.path, (path) => {
  if (path === '/') clear()
})
</script>
```

#### Watch
アプリケーション内の他のリアクティブな値が変更されるたびにフェッチ関数を再実行するには、`watch`オプションを使用します。このオプションは1つまたは複数の監視可能な要素に使用できます。
```ts
<script setup lang="ts">
const id = ref(1)

const { data, error, refresh } = await useFetch('/api/users', {
  /* Changing the id will trigger a refetch */
  watch: [id]
})
</script>
```

リアクティブな値を見てもフェッチされるURLは変わらないことに注意してください。例えばこの関数が呼び出された瞬間にURLが構築されるため、これはユーザーの同じ初期IDをフェッチし続けます。
```ts
<script setup lang="ts">
const id = ref(1)

const { data, error, refresh } = await useFetch(`/api/users/${id.value}`, {
  watch: [id]
})
</script>
```

反応値に基づいてURLを変更する必要がある場合は、代わりに`Computed URL`を使用するとよいでしょう。

#### Computed URL
時にはリアクティブな値からURLを計算し、それらが変更されるたびにデータを更新する必要があるかもしれません。このような場合は各パラメータをリアクティブ値としてアタッチします。Nuxtは自動的にリアクティブ値を使用し、値が変更されるたびに再取得します。
```ts
<script setup lang="ts">
const id = ref(null)

const { data, status } = useLazyFetch('/api/user', {
  query: {
    user_id: id
  }
})
</script>
```

より複雑なURL構築の場合、URL文字列を返す計算ゲッターとしてコールバックを使うことができる。

依存関係が変更されるたびに、新しく構築されたURLを使用してデータがフェッチされます。これを`not-immediate`と組み合わせると、フェッチする前にリアクティブ要素が変更されるまで待つことができます。
```ts
<script setup lang="ts">
const id = ref(null)

const { data, status } = useLazyFetch(() => `/api/users/${id.value}`, {
  immediate: false
})

const pending = computed(() => status.value === 'pending');
</script>

<template>
  <div>
    <!-- disable the input while fetching -->
    <input v-model="id" type="number" :disabled="pending"/>

    <div v-if="status === 'idle'">
      Type an user ID
    </div>

    <div v-else-if="pending">
      Loading ...
    </div>

    <div v-else>
      {{ data }}
    </div>
  </div>
</template>
```

他の反応値が変化したときに強制的に更新する必要がある場合は、他の値を監視することもできます。

#### Not immediate
`useFetch`コンポーザブルは呼び出された瞬間にデータのフェッチを開始します。これを防ぐにはたとえば`immediate: false`を設定して、ユーザーとの対話を待つようにします。

これによりフェッチのライフサイクルを処理するステータスと、データフェッチを開始する`execute`の両方が必要になります。
```ts
<script setup lang="ts">
const { data, error, execute, status } = await useLazyFetch('/api/comments', {
  immediate: false
})
</script>

<template>
  <div v-if="status === 'idle'">
    <button @click="execute">Get data</button>
  </div>

  <div v-else-if="status === 'pending'">
    Loading comments...
  </div>

  <div v-else>
    {{ data }}
  </div>
</template>
```

より詳細な制御のために、`status`変数は次のようにすることができる：

* `idle`（フェッチが開始されていないとき）
* `pending`（フェッチが開始されたがまだ完了していないとき）
* `error`（フェッチが失敗したとき）
* `success`（フェッチが正常に完了したとき）。

### ヘッダーとクッキーを渡す
ブラウザで`$fetch`を呼び出すと、クッキーのようなユーザーヘッダが直接APIに送られます。

通常サーバサイドレンダリングの間、セキュリティへの配慮から`$fetch`はユーザのブラウザ・クッキーを含めず、フェッチ・レスポンスからクッキーを渡すこともありません。

しかし、サーバ上の相対URLで`useFetch`を呼び出す場合、Nuxtは`useRequestFetch`を使用してヘッダとクッキーをプロキシします（hostのように転送されることを意図していないヘッダは例外です）。

#### SSRレスポンスでサーバー側APIコールからクッキーを渡す
内部リクエストからクライアントに戻る、別の方向にクッキーを渡す/プロキシしたい場合、これを自分で処理する必要があります。

`composables/fetch.ts`
```ts
import { appendResponseHeader } from 'h3'
import type { H3Event } from 'h3'

export const fetchWithCookie = async (event: H3Event, url: string) => {
  /* Get the response from the server endpoint */
  const res = await $fetch.raw(url)
  /* Get the cookies from the response */
  const cookies = res.headers.getSetCookie()
  /* Attach each cookie to our incoming Request */
  for (const cookie of cookies) {
    appendResponseHeader(event, 'set-cookie', cookie)
  }
  /* Return the data of the response */
  return res._data
}
```

```ts
<script setup lang="ts">
// This composable will automatically pass cookies to the client
const event = useRequestEvent()

const { data: result } = await useAsyncData(() => fetchWithCookie(event!, '/api/with-cookie'))

onMounted(() => console.log(document.cookie))
</script>
```

#### Options APIサポート
NuxtはOptions API内で`asyncData`フェッチを実行する方法を提供します。これを動作させるには`defineNuxtComponent`でコンポーネント定義をラップする必要があります。
```ts
<script>
export default defineNuxtComponent({
  /* Use the fetchKey option to provide a unique key */
  fetchKey: 'hello',
  async asyncData () {
    return {
      hello: await $fetch('/api/hello')
    }
  }
})
</script>
```

※ `<script setup>`または`<script setup lang="ts">`を使用するのが、Nuxt3でVueコンポーネントを宣言する推奨方法です。

#### サーバーからクライアントへのデータのシリアライズ
`useAsyncData`と`useLazyAsyncData`を使用してサーバーで取得したデータをクライアントに転送する場合（Nuxtペイロードを使用する他のすべてのものと同様）、ペイロードはdevalueでシリアライズされます。これにより基本的なJSONだけでなく、正規表現、日付、MapとSet、ref、reactive、shallowRef、shallowReactive、NuxtErrorなどより高度な種類のデータをシリアライズおよび復活/デシリアライズして転送できます。

また、Nuxtがサポートしていない型に対して独自のシリアライザ/デシリアライザを定義することも可能です。詳しくは`useNuxtApp`のドキュメントを参照してください。

#### APIルートからのデータのシリアライズ
サーバディレクトリからデータをフェッチする場合、レスポンスはJSON.stringifyを使用してシリアライズされます。しかし、シリアライズはJavaScriptのプリミティブ型のみに制限されているため、Nuxtは`$fetch`と`useFetch`の戻り値の型を実際の値と一致するように変換するよう最善を尽くします。

##### 例
`server/api/foo.ts`
```ts
export default defineEventHandler(() => {
  return new Date()
})
```

`app.vue`
```ts
<script setup lang="ts">
// Type of `data` is inferred as string even though we returned a Date object
const { data } = await useFetch('/api/foo')
</script>
```

##### カスタムシリアライザー関数
シリアライズの動作をカスタマイズするには、返されたオブジェクトにtoJSON関数を定義します。toJSONメソッドを定義するとNuxtは関数の戻り値の型を尊重し、型の変換を試みません。

`server/api/bar.ts`
```ts
export default defineEventHandler(() => {
  const data = {
    createdAt: new Date(),

    toJSON() {
      return {
        createdAt: {
          year: this.createdAt.getFullYear(),
          month: this.createdAt.getMonth(),
          day: this.createdAt.getDate(),
        },
      }
    },
  }
  return data
})
```

`app.vue`
```ts
<script setup lang="ts">
// Type of `data` is inferred as
// {
//   createdAt: {
//     year: number
//     month: number
//     day: number
//   }
// }
const { data } = await useFetch('/api/bar')
</script>
```






// TODO: 後で調べる
* useRequestHeaders
* lazy（useLazyFetch）いつ使う？
* pick説明がわからない
* SSRレスポンスでサーバー側APIコールからクッキーを渡す
* サーバーからクライアントへのデータのシリアライズ
