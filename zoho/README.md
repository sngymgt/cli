# zoho

Zoho CRM API v8 を直接叩く CLI。MCP では届かない操作（項目の削除、カスタムビュー、COQL）や、
全ページ取得が必要な処理に使う。

コマンドの一覧と引数は `zoho --help` が権威。

## セットアップ

1. `zoho init` で `~/.config/zoho/env` の雛形を作る
2. <https://api-console.zoho.jp/> の **Self Client** で client_id / client_secret を発行して記入する
   （データセンターが jp なら `ZOHO_DC=jp`。accounts は `accounts.zoho.jp`、API は `zohoapis.jp`）
3. 同じ Self Client 画面で scope を指定して grant token を作り、`zoho auth <grant_token>` で交換する

トークンは `~/.config/zoho/tokens.json`（600）に入り、失効時は refresh token で自動更新される。
`zoho token` で状態を確認できる。

## 注意

- 同じ API を秒間に叩きすぎると CONCURRENCY で落ちるため、CLI 側で間隔を空けている
- 引数に絶対 URL を渡せるのは、設定した Zoho のオリジン（`api_domain` と `ZOHO_API_BASE`）に
  一致する場合だけ。それ以外はアクセストークンを付けないよう拒否する。リダイレクトも追わない
- サブフォームは一覧の GET では返らない。**エラーにならず「明細が空」と区別できない**ので、
  サブフォーム自体をモジュールとして読むこと
