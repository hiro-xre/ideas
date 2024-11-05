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
