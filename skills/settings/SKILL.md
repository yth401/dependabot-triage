---
name: settings
description: dependabot-triage の設定（dependabot-triage.yml と .dependabot-triage/guidance.md）を表示し、間違いがないか確かめ、項目を変える。「dependabot-triage の設定を見せて」「閾値を変えたい」「プリセットを変えたい」と頼まれたときに使う。
argument-hint: "[変えたい項目]"
---

# dependabot-triage:settings

使う側のリポジトリの dependabot-triage の設定を、表示・確認・変更する。

引数に変えたい項目が渡されたら（例：`/dependabot-triage:settings 閾値`）、表示と確認のあと、その項目だけを変える。

引数：$ARGUMENTS

## 守ること

- コミットと push はしない
- 書き込む前に、変わる部分を見せて確認をとる

## 手順

### 1. 設定を読む

- [事前確認](${CLAUDE_PLUGIN_ROOT}/reference/preflight.md) の手順 1 と 2（gh の認証、対象のリポジトリ）を行う。手順 3 で保留の Issue を読むのに使う
- `dependabot-triage.yml` を読む。なければ、`/dependabot-triage:init` を実行するよう案内して止める
- `.dependabot-triage/guidance.md` があれば読む

### 2. 表示する

設定を、人が読みやすい形で示す。項目名だけでなく意味もわかるようにする（[設定の仕様](${CLAUDE_PLUGIN_ROOT}/reference/config.md) と [プリセット](${CLAUDE_PLUGIN_ROOT}/reference/presets.md) を参照）。

例：

```
判定の基準    static-site（ビルドして静的なファイルを配信するサイト）
公開範囲      社内のみ（internal）
閾値          深刻度 low 以上 ／ EPSS 1% 以上は深刻度にかかわらず対象 ／ KEV は常に対象 ／ 開発時だけの依存も対象
対処期限      緊急 当日 ／ 高 7 日 ／ 中 30 日 ／ 低 期限なし
要対処の扱い  Issue にする（ラベル：security、担当者：なし、テンプレート：なし）
guidance.md   3 項目（管理画面は社内ネットワークからしか開けない、ほか）
```

### 3. 確かめる

[設定の仕様](${CLAUDE_PLUGIN_ROOT}/reference/config.md) の「確かめること」を確かめ、問題があれば示す。

あわせて、次も確かめて示す。

- 保留の Issue のうち、期限が過ぎているもの（[dependabot-triage が作る Issue](${CLAUDE_PLUGIN_ROOT}/reference/issues.md) の「開いている Issue の読み方」で読む）。あれば、`/dependabot-triage:check` で判定し直すよう案内する
- `guidance.md` の内容が、プリセットや公開範囲と食い違っていないか（例：プリセットが `static-site` なのに「サーバーでファイルを処理する」と書いてある）

### 4. 変える

変えたい項目を聞く（引数で渡されていれば、それを変える）。問題が見つかっていれば、それを直すことも選択肢に入れる。

- 質問の仕方と選択肢は、[init](../init/SKILL.md) の手順 4 と同じにする。今の値を初期値として示す
- `guidance.md` を変えるときは、足す・直す・消す内容を聞く

変わる部分を見せて確認をとってから書き込み、もう一度「確かめること」で確かめる。

### 5. 終わりに

変えたファイルを示し、コミットと push は担当者が行うことを伝える。何も変えなかったときは、その旨を伝える。
