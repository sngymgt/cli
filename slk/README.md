# slk

Slack を利用者として読むための CLI。ワークスペースはプロファイルで切り替える。
書き込みは、自分で許可したチャンネルへの投稿だけ（既定はどこにも書けない）。
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

## 書き込み（任意）

既定では書き込みの権限を要求しない。投稿したいときだけ次の 3 つを行う。

1. アプリの OAuth & Permissions → **User Token Scopes に `chat:write` を足す**
   （新しく作るなら `slk manifest --write`）
2. `slk login --write` でトークンを取り直す
3. 書いてよい宛先を登録する（プロファイルごと。**既定は空で、どこにも書けない**）

```
slk writable add #times-taro @hanako     # 宛先を許可する（名前は完全一致）
slk writable                             # 一覧
slk writable rm #times-taro

slk post #times-taro "デプロイしました"
slk post https://example.slack.com/archives/C0123ABCD/p1757049780123456 "確認しました"   # スレッドに返信
echo "長い本文" | slk post #times-taro - --yes
```

- `chat:write` 自体はチャンネル単位に絞れない（参加している所ならどこへでも書ける権限）。
  宛先を絞っているのは slk の許可リストだけで、`slk api` からは書き込みメソッドを呼べない
- 投稿の前に宛先と本文を表示して確認する。端末でない（パイプ・スクリプト）ときは `--yes` が要る
- **`slk writable add` は必ず人の確認を挟む。** 端末なら y/N、端末でない（Claude Code などの AI が
  叩いた）ときは確認ダイアログが出る（macOS は標準のダイアログ、Windows は MessageBox、Linux は
  zenity。どれも出せない環境では追加できず、利用者が自分の端末で実行する）。`--yes` は無い。AI が勝手に宛先を増やせないための門で、
  許可リストに無い宛先へ投稿しようとしたときのエラーは、AI に「利用者に追加するか聞け」と指示する文面に
  なっている
- `& < >` はエスケープして送るので、`<!channel>` や `<@U…>` は通知にならない。
  メンションを効かせたいときだけ `--no-escape`
- 許可リストはチャンネル ID で持つ。改名されても同じチャンネルを指し続ける
- 一度 `--write` で認可すると、`--write` なしで取り直しても権限は残る（Slack が足し合わせる）。
  外すにはアプリの認可を取り消してから `slk login` し直す

## インストール

リポジトリ全体の手順は [../README.md](../README.md)。slk だけなら:

```
ln -s "$PWD/slk/slk" ~/.local/bin/slk
```

### Windows

Python 3（python.org などから。インストール時に PATH に追加する）と Git for Windows が要る。
Windows は先頭の `#!` を読まず `ln -s` も実体のコピーになるので、`python` 経由で起動する
小さな入口を PATH の通った場所に置く。clone 先は以下 `<clone 先>` と書く。

Claude Code などの AI から使う場合（Git Bash。`~/bin` に PATH が通っている前提）:

```
mkdir -p ~/bin
printf '#!/bin/sh\nexec python "%s" "$@"\n' "$(cd <clone 先> && pwd -W)/slk/slk" > ~/bin/slk
chmod +x ~/bin/slk
slk --help
```

PowerShell から使う場合は `$PROFILE` に足す:

```
function slk { python "<clone 先>\slk\slk" @args }
```

**最初の `slk login` は PowerShell かコマンドプロンプトで行う。** Git Bash（mintty）は端末として
見えないことがあり、そのときは `client_secret` を入力できずに止まる。

- トークンと `client_secret` は **Windows 資格情報マネージャー**に入る（「資格情報マネージャー」→
  「Windows 資格情報」→ 汎用資格情報の `slk:<プロファイル名>`）
- 書き込み先の追加（`slk writable add`）の確認は、端末なら y/N、AI から叩いたときは Windows の
  ダイアログになる
- パイプやファイルへの出力は UTF-8。PowerShell で受けて文字化けするときは
  `[Console]::OutputEncoding = [Text.Encoding]::UTF8` を先に実行する

## セットアップ（ワークスペースに 1 回だけ）

1. `slk manifest` の出力をコピーする
2. <https://api.slack.com/apps> → Create New App → **From a manifest** → ワークスペースを選んで貼る
3. **Distribution を有効にしない。** 有効化すると履歴の取得が 15 件・1 req/分に落ちて元に戻せない。
   同じワークスペースのメンバーに配る分には有効化は不要
4. OAuth & Permissions → Redirect URLs に着地ページを**末尾スラッシュまで完全一致**で登録
   （`https://sngymgt.github.io/cli/slk/callback/`）
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

**同じワークスペースの同じアカウント**のプロファイルが既にあれば、それを更新する（名前は増えない）。
同じワークスペースでも別アカウントで入った場合は、既存を上書きせず別のプロファイルを作る。
名前を明示すれば（`slk login <名前>`）そのプロファイルに入れられる。
名前を変えたくなったら `slk profile rename <old> <new>`。

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

OS の資格情報ストア。macOS は Keychain（service `slk`、account はプロファイル名）、Windows は
資格情報マネージャー（`slk:<プロファイル名>`）。どちらも無い OS では `profiles/<名前>/env` に
平文で置く（そのときは警告が出る）。平文ファイルに入っている古いプロファイルは
`slk profile keychain <name>` で移せる。`slk profile rm` はストアの項目も消す。

## 新しい PC で使い始める

持ち越す必要があるものは無い。以下だけで元に戻る。

```
git clone git@github.com:sngymgt/cli.git
cd cli
ln -s "$PWD/slk/slk" ~/.local/bin/slk
slk login
```

- **Slack アプリはワークスペース側に残る**ので作り直さなくてよい（PC とは無関係）
- `client_secret` は 1Password から。`slk login` の初回に 1 度だけ聞かれる
- **持ち越せないのは資格情報ストアのトークンとキャッシュだけ**。トークンは `slk login` で取り直せて、
  キャッシュは自動で作り直される。古い PC のトークンを消したい場合は Slack の管理画面から

`slk login` を使わない場合は、`slk profile add <name>` にトークンを貼るだけでも同じ状態になる。

## コマンド

```
slk manifest [--search both|legacy|rts] [--write]
slk login [<profile 名>] [--manual] [--reset-secret] [--write]
slk profile [list|current|use|add|keychain [--force]|rename|label|path|rm]
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
slk file <F…|permalink|URL> [-o <先>|-] [--force] [--info]
slk cache [refresh|clear]
slk api <method> [k=v ...] [--all]

slk writable [list|add|rm] [<#channel|@user|C…> ...]
slk post <#channel|@user|C…|permalink> [本文|-] [--thread <ts|permalink>] [--yes] [--no-escape]
```

`slk file` は添付を落とすだけで、**中身は解釈しない**（PDF や docx のテキスト抽出には依存
パッケージが要り、単一ファイルで動く前提が壊れる）。落としたものを別のツールに渡すこと。
`-o -` で標準出力に流せる。

`slk ask` は Slack の Real-time Search API を使う。名前に反してキーワード一致なので、文章を
丸ごと渡すと 0 件になる。単語で渡すこと。

---

このリポジトリを変更するときの決まりごとは [../AGENTS.md](../AGENTS.md)。
