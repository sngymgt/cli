# gws

[googleworkspace-cli](https://github.com/googleworkspace/cli) のラッパー。
本体に無い**認証プロファイルの切り替え**だけを足している。

プロファイルの実体は gws の設定ディレクトリで、`GOOGLE_WORKSPACE_CLI_CONFIG_DIR` で選ぶ。
`profile` と `--profile` 以外の引数はすべて本体にそのまま渡す。

```
gws profile                     一覧（* が有効）
gws profile use <name>          切り替え
gws profile add <name>          作ってログイン
gws --profile <name> ...        その 1 コマンドだけ別プロファイルで（位置は問わない）
GWS_PROFILE=<name> gws ...      同上
```

詳しくは `gws profile help`。

## セットアップ

1. 本体を入れる: `brew install googleworkspace-cli`
2. このラッパーを PATH の先（`~/.local/bin`）に置く。本体は PATH 上の別の場所か
   `/opt/homebrew/bin` から自動で探す
3. `gws profile add <name> --client <client_secret.json>` で OAuth クライアントを取り込んでログイン

プロファイルは `~/.config/gws-profiles/<name>/`、有効なものは同ディレクトリの `active`。

## 注意

- discovery ドキュメントは数 MB あるがアカウントに依存しないため、`add` で作った
  プロファイルは元の `cache` を symlink で共有する。`rename` / `rm` はその張り替えまで面倒を見る
- プロファイルは**利便性の境界であってセキュリティ境界ではない**。`credentials.enc` は
  プロファイルごとに分かれるが、復号鍵は Keychain の 1 項目で共有される
  （鍵は service と account だけで、config dir は入らない）。バイナリに一度アクセスを
  許可すれば、どのプロファイルの認証情報も復号できる。
  `gws auth logout` は当該 config dir のファイルを消すだけで鍵には触らないので、
  他のプロファイルは巻き添えにならない
- `GOOGLE_WORKSPACE_CLI_{TOKEN,CREDENTIALS_FILE,CLIENT_ID,CLIENT_SECRET}` が設定されていると
  プロファイルより優先されるため、ラッパー側で解除して警告する
