# zp

Zoom Phone API の CLI。通話履歴・録音・録音の文字起こしを取る。

コマンドの一覧と引数は `zp --help` が権威。ドキュメントに無い API の癖は
`zp` の冒頭 docstring にまとめてある。

## セットアップ

1. `zp init` で `~/.config/zp/env` の雛形を作る
2. <https://marketplace.zoom.us/> の **Develop → Build App → Server-to-Server OAuth** で
   アプリを作り、Account ID / Client ID / Client Secret を記入する
3. 同じアプリの **Scopes** に phone の read を足す。粒度スコープなら以下、
   旧表記しか出ないアカウントなら `phone:read:admin` と `phone_recording:read:admin`

   ```
   phone:read:list_users:admin
   phone:read:list_recordings:admin
   phone:read:recording:admin
   phone:read:recording_transcript:admin
   phone:read:list_call_history:admin
   phone:read:list_account_settings:admin
   ```

4. アプリを **Activate** する
5. `zp health` で、実際に付与された scope と疎通を確認する

アクセストークンはディスクに置かない（1 時間で切れるうえ、毎回取り直しても 1 リクエストしか
増えないため）。`~/.config/zp` は 700、`env` は 600 に、`zp init` と実行のたびに絞り直す
（緩ければ直したと表示する）。

## 使いはじめ

録音系は**アカウントによって通るエンドポイントが違う**（冒頭 docstring 参照）ので、
何かを書く前に当たりを取る。

```
zp probe                       # 直近 7 日
zp probe 2026-01-01 2026-01-07
```

`probe` は付与された scope → 各エンドポイントの可否 → 録音レコードの生の形 →
長い通話 3 件の文字起こし、の順に出す。

## 注意

- **文字起こしは録音の処理時に作られるので、有効化より前の通話には遡及しない。**
  「1 件も無い」を設定漏れと区別できないので、有効化したらテスト架電で確かめる
- **文字起こしの言語はユーザー設定の「デフォルトの文字起こし言語」で決まり、既定は英語。**
  日本語の通話を英語として起こしてもエラーにならず、読めない結果だけが返る
- 文字起こしが無い録音は 404 ではなく `code 12000`。未対応・処理中・設定漏れを区別できない
- 申請した scope と実際に付いた scope はズレる（Activate し直していない等）。
  実際の値は `zp health` にしか出ない
- 保存した録音は 600 で作る（音声は `env` と同じ機微なので、`--force` の有無で権限が変わらない）
- 録音と文字起こしの download は署名付きの配信先へ転送される。CLI は 1 ホップずつ自分で追い、
  **転送先には資格情報を付け直さない**。`recordings` が返す `download_url` を curl で直接
  叩くとトークンが配信先に渡るので、保存は `zp download` を使うこと
- `from` / `to` は `YYYY-MM-DD`。遡れる範囲と 1 回の幅に上限があるので、長期間は日付を割る。
  片方だけ書いた場合、もう片方は補われない（書いていない側で上限を超えるため）
- `--all` が最後まで辿れなかったときは exit 1 になり、`next_page_token` を残したまま返す。
  途中までの JSON が完全な取得と見分けられなくなるため。`zp health` も NG があれば exit 1
