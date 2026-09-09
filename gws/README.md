# gws

[googleworkspace-cli](https://github.com/googleworkspace/cli) のラッパー。
本体に無い**認証プロファイルの切り替え**だけを足している。

プロファイルの実体は gws の設定ディレクトリで、`GOOGLE_WORKSPACE_CLI_CONFIG_DIR` で選ぶ。
`profile` 以外の引数はすべて本体にそのまま渡す。

```
gws profile                     一覧（* が有効）
gws profile use <name>          切り替え
gws profile add <name>          作ってログイン
GWS_PROFILE=<name> gws ...      その 1 コマンドだけ別プロファイルで
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
