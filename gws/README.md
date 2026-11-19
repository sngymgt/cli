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

### Windows

Git for Windows（Git Bash）の上で動く。Claude Code は Windows でも Git Bash でコマンドを叩くので、
AI から使う分にはこれで足りる。clone 先は以下 `<clone 先>` と書く。

1. 本体を置く。PowerShell で:

   ```
   $d = "$env:LOCALAPPDATA\Programs\gws"
   New-Item -ItemType Directory -Force $d | Out-Null
   Invoke-WebRequest https://github.com/googleworkspace/cli/releases/latest/download/google-workspace-cli-x86_64-pc-windows-msvc.zip -OutFile "$env:TEMP\gws.zip"
   Expand-Archive "$env:TEMP\gws.zip" $d -Force
   ```

   ラッパーは PATH に無くても `%LOCALAPPDATA%\Programs\gws\gws.exe` を見つける
2. Git Bash で入口を置く（Git Bash は `#!` を読まず `ln -s` も実体のコピーになるため。
   `~/bin` に PATH が通っている前提）:

   ```
   mkdir -p ~/bin
   printf '#!/bin/sh\nexec bash "%s" "$@"\n' "$(cd <clone 先> && pwd)/gws/gws" > ~/bin/gws
   chmod +x ~/bin/gws
   ```

3. OAuth クライアント（`client_secret.json`。管理者から受け取る）を取り込んでログイン:
   `gws profile add <name> --client <client_secret.json のパス>`

- 鍵は Windows 資格情報マネージャーに入る。プロファイルの置き場は Mac と同じ `~/.config/gws-profiles/`
- Windows では `cache` をプロファイル間で共有しない（各プロファイルに数 MB ずつできる）
- PowerShell から手で叩くのは勧めない。Windows PowerShell 5.1 は外部コマンドに渡す引数から
  `"` を落とすので、`--params '{"pageSize":1}'` のような JSON が壊れる

## 注意

- discovery ドキュメントは数 MB あるがアカウントに依存しないため、`add` で作った
  プロファイルは元の `cache` を symlink で共有する。`rename` / `rm` はその張り替えまで面倒を見る
- プロファイルは**利便性の境界であってセキュリティ境界ではない**。`credentials.enc` は
  プロファイルごとに分かれるが、復号鍵は Keychain の 1 項目で共有される
  （鍵は service と account だけで、config dir は入らない）。バイナリに一度アクセスを
  許可すれば、どのプロファイルの認証情報も復号できる。
  `gws auth logout` は当該 config dir のファイルを消すだけで鍵には触らないので、
  他のプロファイルは巻き添えにならない
- 本体の `auth login` の選択画面は `cloud-platform` / `pubsub` に既定でチェックが入っている。
  Workspace のアカウントでこれを許可すると Google Cloud のセッション制御の対象になり、
  十数時間ごとに `invalid_rapt` で切れる。そのためラッパーの `auth login` は選択画面を出さず、
  Cloud 系を除いた既定のスコープ（`DEFAULT_SCOPES`）で認可する。プロファイルごとに変えるときは
  `<プロファイル>/scopes` に 1 行 1 スコープを書く（短い名前は URL に展開する。本体の `--scopes` は展開しない）。
  `--scopes` / `--services` / `--full` / `--readonly` を明示したときはそのまま本体に渡す
- `GOOGLE_WORKSPACE_CLI_{TOKEN,CREDENTIALS_FILE,CLIENT_ID,CLIENT_SECRET}` が設定されていると
  プロファイルより優先されるため、ラッパー側で解除して警告する
