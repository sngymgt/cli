# mf

マネーフォワード クラウドの API CLI。請求書（invoice）と会計（accounting）を 1 本で扱う。
両者は認可サーバを共有し、API ホスト・スコープ・一覧の封筒だけが違う。

コマンドの一覧と引数は `mf --help` が権威。

## セットアップ

1. `~/.config/mf/env` に `MF_CLIENT_ID` と `MF_CLIENT_SECRET` を書く
   （マネーフォワードの開発者向け管理画面で発行したもの）
2. `mf login` でブラウザ認可（OAuth 2.0 authorization code + PKCE）
   - サービスを絞るなら `mf login invoice` / `mf login accounting`
   - 既定は全サービスの参照系スコープ
3. `mf token` でアクセストークンの状態を確認できる

## 注意

- 請求書 API は API キーに対応しておらず、OAuth のみ
- リダイレクトは追わない。urllib は転送先が別ホストでも Authorization を持ち回るため
- `mf-invoice` という旧コマンド名を使っていた場合は、`mf invoice …` に読み替える
