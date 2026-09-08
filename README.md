# slk

Slack を利用者として読むための read-only CLI。ワークスペースはプロファイルで切り替える。
Python 3 標準ライブラリのみの単一ファイルで、依存パッケージは無い。

公式の `slack` CLI（アプリ開発用）とは別物なので名前を分けている。

```
slk search "デプロイ" --in general --from @taro --after 2026-09-01
slk read https://example.slack.com/archives/C0123ABCD/p1757049780123456 --context 3
slk history #general --limit 100 --threads
slk channels 営業 --member
slk profile use personal
```

`--json` で加工済み JSON、`--raw` で API 応答そのまま。人間向けの出力では stdout が結果だけに
なるよう、ワークスペース名や件数は stderr に出しているので `slk search … --json | jq` がそのまま通る。

## このリポジトリに入れてはいけないもの

public repo なので、git 履歴は取り消せない（fork とキャッシュに残る）。

- **取得したメッセージ・ログ**。作業ディレクトリをこのリポジトリにしないこと
- **実在のワークスペース URL・チャンネル名・実名**。README の例はすべてダミーにする
  （チャンネル名は組織構造と顧客名が出る）
- **トークン**と **client_secret**

上 2 つのうち機械で防げるものは `.githooks/pre-commit` が止める。clone 後に有効化すること:

```
git config core.hooksPath .githooks
```

## インストール

```
git clone git@github.com:sngymgt/slk.git ~/src/slk
ln -s ~/src/slk/slk ~/.local/bin/slk
git -C ~/src/slk config core.hooksPath .githooks
```

`~/.local/bin` に PATH が通っていること。更新は `git pull` だけ。

## セットアップ（ワークスペースに 1 回だけ）

1. `slk manifest` の出力をコピーする
2. <https://api.slack.com/apps> → Create New App → **From a manifest** → ワークスペースを選んで貼る
3. **Distribution は有効にしない**（後述）
4. OAuth & Permissions → Redirect URLs に GitHub Pages の着地ページを**末尾スラッシュまで完全一致**で登録
   （`https://sngymgt.github.io/slk/callback/`）
5. Basic Information の `client_id` と、4 の URL を `app.json` に書いてコミットする
6. `client_secret` は**コミットしない**。1Password に入れておく（`slk login` の初回に 1 度だけ聞かれ、
   以降は Keychain から読む）

`app.json` は `~/.config/slk/app.json` があればそちらが優先される。別のアプリやワークスペースを
使う人は、リポジトリを書き換えずに個人側で上書きできる。

scope の組み合わせが Slack に拒否されたら `slk manifest --search legacy` か `--search rts` で作り直す。

## 認証

### 使う人ごとに 1 回

```
slk login
```

ブラウザで承認すると、その人自身の user token が Keychain に入る。**トークンは人ごとに別**で、
互いに失効させない。読める範囲は各自の Slack の権限どおりになる。

自動でターミナルに戻れない場合は着地ページにコードが出るので、それを貼れば同じ結果になる
（`slk login --manual` で最初から貼り付け方式）。

`client_secret` は `client_id` ごとに別枠で Keychain に入るので、アプリを差し替えても前の値を
掴まない。打ち間違えた・Slack 側でローテーションした場合は `slk login --reset-secret` で入れ直す。
交換時に secret が拒否されたときは、その場で入力し直して同じコードのまま再試行する。

### アプリ設定画面からトークンをコピーする方法

`app.json` も client_secret も要らない代わりに、アプリの collaborator である必要がある。

```
slk profile add <プロファイル名>      # xoxp- を貼る（画面には出ない）
```

## 新しい PC で使い始める

持ち越す必要があるものは無い。以下だけで元に戻る。

```
git clone git@github.com:sngymgt/slk.git ~/src/slk
ln -s ~/src/slk/slk ~/.local/bin/slk
slk login
```

- **Slack アプリはワークスペース側に残る**ので作り直さなくてよい（PC とは無関係）
- `client_secret` は 1Password から。`slk login` の初回に 1 度だけ聞かれる
- **持ち越せないのは Keychain のトークンとキャッシュだけ**。トークンは `slk login` で取り直せて、
  キャッシュは自動で作り直される。古い PC のトークンを消したい場合は Slack の管理画面から

`slk login` を使わない場合は、`slk profile add <name>` にトークンを貼るだけでも同じ状態になる。

## 設計上の注意

- **アプリの Distribution を有効にしない。** 有効化すると `conversations.history` / `replies` が
  15 件・1 req/分に落ちる（2025-05 の配布アプリ向け制限）。内製アプリのままなら 1000 件・50+/分。
  同じワークスペースのメンバーに配る分には配布は不要
- **read-only を守っているのは 2 点だけ**。manifest が書き込み scope を要求しないことと、
  `READ_METHODS` の allowlist。HTTP メソッドは境界にならない（`chat.postMessage` も GET で通る）
- **トークンは macOS Keychain**（service `slk` / account はプロファイル名）。平文ファイルに
  入っている古いプロファイルは `slk profile keychain <name>` で移せる
- Keychain の項目は `(service, account)` だけが鍵で `SLK_CONFIG_DIR` は鍵に入らない。試験用の
  設定ディレクトリを作るときは `SLK_KEYCHAIN_SERVICE` も変えること

その他の「知らないと必ず一度踏む」Slack API の癖は `slk` の冒頭 docstring にまとめてある。

## コマンド

```
slk manifest [--search both|legacy|rts]
slk login [<profile 名>] [--manual]
slk profile [list|current|use|add|keychain [--force]|label|path|rm]
slk whoami

slk search "<query>" [--in #ch] [--from @user] [--after ...] [--before ...]
                     [--limit N|--all] [--sort score|timestamp] [--asc] [--files]
slk ask "<キーワード>" [--in #ch] [--context] [--types messages,channels,files] [--limit N]
slk channels [pattern] [--private|--public|--dm] [--member] [--archived] [--refresh]
slk users [pattern] [--email a@b] [--bots] [--deleted] [--refresh]
slk members <#channel>
slk history <#channel|@user> [--limit N] [--after ...] [--before ...] [--threads]
slk thread <permalink | #channel ts>
slk read <URL> [--context N]
slk cache [refresh|clear]
slk api <method> [k=v ...] [--all]
```
