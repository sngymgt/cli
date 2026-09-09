# cli

個人用の小さな CLI 群。どれも**単一ファイル・依存パッケージなし**（Python 3 標準ライブラリか
POSIX シェルのみ）で、設定は `~/.config/<名前>/` に置く。

| | 何をするか | 認証 |
| --- | --- | --- |
| [`slk`](slk/) | Slack を利用者として読む（read-only、ワークスペース切替つき） | OAuth（ブラウザ） |
| [`zoho`](zoho/) | Zoho CRM API v8（`get` / `coql` / 書き込み） | OAuth Self Client |
| [`mf`](mf/) | マネーフォワード クラウド（請求書・会計） | OAuth + PKCE |
| [`tldv`](tldv/) | tl;dv（ミーティング・文字起こし・AI ノート） | API キー |
| [`gws`](gws/) | `googleworkspace-cli` に認証プロファイル切替を足すラッパー | 本体に委譲 |

## インストール

```
git clone git@github.com:sngymgt/cli.git
cd cli
git config core.hooksPath .githooks
for t in slk zoho mf tldv gws; do ln -s "$PWD/$t/$t" ~/.local/bin/$t; done
```

必要なものだけ symlink すればよい。`~/.local/bin` に PATH が通っていること。更新は `git pull` だけ。

## セットアップ

**認証はツールごとに違うので、各ディレクトリの README を読むこと。**
コマンドの一覧と引数は `<ツール名> --help` が権威。README には書かない。

## 共通の作法

- 出力は既定が人間向け。JSON が要るものは `--json` / `--all` を持つ
- 設定とトークンは `~/.config/<名前>/`。`slk` だけはトークンを macOS Keychain に置く
- どのツールも、そのサービスの API が返した内容をそのまま見せる。手元での再解釈をしない

---

変更するときの決まりごとは [AGENTS.md](AGENTS.md)。
