# tldv

tl;dv の API（v1alpha1）CLI。ミーティング一覧・文字起こし・AI ノートを取る。

コマンドの一覧と引数は `tldv --help` が権威。ドキュメントに無い API の癖は
`tldv` の冒頭 docstring にまとめてある。

## セットアップ

1. `tldv init` で `~/.config/tldv/env` の雛形を作る
2. tl;dv の管理画面で発行した API キーを `TLDV_API_KEY` に記入する
3. `tldv health` で疎通を確認する

## 注意

- API は alpha 版。**未知のパラメータはエラーにならず黙って捨てられる**ので、効いていないことに
  気づけない
- transcript / notes が 403 になるかは、**そのミーティングの主催者の席のプラン**で決まる。
  キーの持ち主のプランではない
