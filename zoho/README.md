# zoho

Zoho CRM API v8 を直接叩く CLI。MCP では届かない操作（項目の削除、カスタムビュー、COQL）や、
全ページ取得が必要な処理に使う。

コマンドの一覧と引数は `zoho --help` が権威。

## 注意

- `--all` と `coql` は取り切れなかったら exit 1 になる（出力は途中まで）。
  COQL は 1 回 200 件が上限。`coql --all` は LIMIT / OFFSET を自動で回す
  （`order by id` を付けないとページ境界で行が重複・欠落しうるので警告が出る）
- `get -H 'If-Modified-Since: …'` で更新なしなら 304。stderr に出して exit 0、stdout は `{}`
- `get -o <file>` は JSON 以外（添付・Bulk の zip）をそのまま保存する（600、既存なら `-y` で上書き）。
  `upload <path> <file>` は multipart で送る。`content.zohoapis.*` にも送れる
- `zoho revoke <refresh_token | ->` は refresh_token を 1 本失効させる。`tokens.json` の分は
  Secret Manager のコネクタも止まるので `-y` が要る。Zoho にトークンの一覧 API は無い
- 一括書き込みは HTTP 200 の中にレコード単位の失敗が入る。全件・一部が失敗したら exit 1 になる
- `zoho delete` は確認を訊く。非対話で流すなら `-y`
- `zoho fn push` は Deluge を**その場で公開する**（下書きが無い）。直し始めた版を `--base` に渡すと、
  画面で誰かが直していたときは送らずに止まる。送ったあとは読み直して一致しなければ exit 1

## セットアップ

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/sngymgt/cli/main/zoho/zoho -o ~/.local/bin/zoho
chmod +x ~/.local/bin/zoho
zoho init
```

1. Client ID / Client Secret を手に入れる
   1. ブラウザで共有アカウントとして Zoho にログインした状態で <https://api-console.zoho.jp/> を開く
   2. クライアントの一覧から **CLI**（Non-browser Applications）を開く
   3. 「Client Secret」タブに出る Client ID と Client Secret を、次の `zoho init` に貼る
      （人には送らない。ログインできる人は各自でここから写す）
2. `zoho init` が 2 つを訊くので貼る。`~/.config/zoho/env`（600）に入る。既にあれば何もしない
   （端末でないときは空の雛形を作る。データセンターは `ZOHO_DC=jp`）
3. `zoho login` で表示される URL をブラウザで開いて承認する。既定は読むだけで、書き込みも要るなら
   `zoho login --write`

**サーバーのジョブと同じクライアントを人に配らない。** Zoho の refresh_token は同じクライアント・
同じユーザーで 20 本までで、超えると使用中でも一番古いものが消える。人が `login` を繰り返して
ジョブのトークンを押し出さないよう、ジョブは Self Client、人は Non-browser Applications と分ける。
Self Client の grant token は `zoho auth <grant_token>` で交換する。

トークンは `~/.config/zoho/tokens.json`（600）に入り、失効時は refresh token で自動更新される。
`zoho token` で状態を確認できる。1 台で 2 つのクライアントを使い分けるときは `ZOHO_CONFIG_DIR` で
設定ディレクトリを切り替える。

## 注意

- 同じ API を秒間に叩きすぎると CONCURRENCY で落ちるため、CLI 側で間隔を空けている
- 引数に絶対 URL を渡せるのは、設定した Zoho のオリジン（`api_domain` と `ZOHO_API_BASE`、
  それと対の `content.zohoapis.*`）に一致する場合だけ。`-H` で Authorization は上書きできない。それ以外はアクセストークンを付けないよう拒否する。リダイレクトも追わない
- サブフォームは一覧の GET では返らない。**エラーにならず「明細が空」と区別できない**ので、
  サブフォーム自体をモジュールとして読むこと
