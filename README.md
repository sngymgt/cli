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

読めるのは**あなた自身が Slack で見えるものだけ**。参加していないプライベートチャンネルや他人の
DM は返らない。検索結果が 0 件でも「存在しない」とは限らない。

投稿・削除はできない。書き込みの権限を要求していない。

## インストール

```
git clone git@github.com:sngymgt/slk.git
cd slk
ln -s "$PWD/slk" ~/.local/bin/slk
git config core.hooksPath .githooks
```

置き場所はどこでもよい。`~/.local/bin` に PATH が通っていること。更新は `git pull` だけ。

## セットアップ（ワークスペースに 1 回だけ）

1. `slk manifest` の出力をコピーする
2. <https://api.slack.com/apps> → Create New App → **From a manifest** → ワークスペースを選んで貼る
3. **Distribution を有効にしない。** 有効化すると履歴の取得が 15 件・1 req/分に落ちて元に戻せない。
   同じワークスペースのメンバーに配る分には有効化は不要
4. OAuth & Permissions → Redirect URLs に着地ページを**末尾スラッシュまで完全一致**で登録
   （`https://sngymgt.github.io/slk/callback/`）
5. Basic Information の `client_id` を `app.json` に書いてコミットする
6. `client_secret` は**コミットしない**。1Password に入れておく（`slk login` の初回に 1 度だけ聞かれ、
   以降は Keychain から読む）

scope の組み合わせが Slack に拒否されたら `slk manifest --search legacy` か `--search rts` で作り直す。

別のアプリやワークスペースを使う人は、`~/.config/slk/app.json` を置けばリポジトリを書き換えずに
上書きできる。

## 認証

### 使う人ごとに 1 回

```
slk login
```

ブラウザで承認すると、その人自身のトークンが Keychain に入る。**トークンは人ごとに別**で、互いに
失効させない。読める範囲は各自の Slack の権限どおりになる。

自動でターミナルに戻れない場合は着地ページにコードが出るので、それを貼れば同じ結果になる
（`slk login --manual` で最初から貼り付け方式）。

`client_secret` を打ち間違えた、または Slack 側でローテーションしたときは
`slk login --reset-secret` で入れ直す。交換時に拒否された場合はその場で聞き直すので、
ブラウザの承認からやり直す必要はない。

### アプリ設定画面からトークンをコピーする方法

`app.json` も `client_secret` も要らない代わりに、アプリの collaborator である必要がある。

```
slk profile add <プロファイル名>      # xoxp- を貼る（画面には出ない）
```

### トークンの置き場所

macOS Keychain（service `slk`、account はプロファイル名）。平文ファイルに入っている古い
プロファイルは `slk profile keychain <name>` で移せる。`slk profile rm` は Keychain の項目も消す。

## 新しい PC で使い始める

持ち越す必要があるものは無い。以下だけで元に戻る。

```
git clone git@github.com:sngymgt/slk.git
cd slk
ln -s "$PWD/slk" ~/.local/bin/slk
slk login
```

- **Slack アプリはワークスペース側に残る**ので作り直さなくてよい（PC とは無関係）
- `client_secret` は 1Password から。`slk login` の初回に 1 度だけ聞かれる
- **持ち越せないのは Keychain のトークンとキャッシュだけ**。トークンは `slk login` で取り直せて、
  キャッシュは自動で作り直される。古い PC のトークンを消したい場合は Slack の管理画面から

`slk login` を使わない場合は、`slk profile add <name>` にトークンを貼るだけでも同じ状態になる。

## コマンド

```
slk manifest [--search both|legacy|rts]
slk login [<profile 名>] [--manual] [--reset-secret]
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

`slk ask` は Slack の Real-time Search API を使う。名前に反してキーワード一致なので、文章を
丸ごと渡すと 0 件になる。単語で渡すこと。

---

このリポジトリを変更するときの決まりごとは [AGENTS.md](AGENTS.md)。
