---
name: init
description: このリポジトリで dependabot-triage を使い始めるための初期設定をする。Dependabot alerts と権限を確かめ、コードベースを読み取ってプリセット・公開範囲の当たりと guidance.md の案を作り、それをもとに判定の基準、プロジェクトの考慮事項、閾値、「要対処」の扱いを質問して、dependabot-triage.yml と .dependabot-triage/guidance.md を作る。「dependabot-triage を導入したい」「dependabot-triage の初期設定をして」と頼まれたときに使う。
---

# dependabot-triage:init

使う側のリポジトリに、dependabot-triage の設定ファイルを用意する。

## 守ること

- コミットと push はしない。作ったファイルを見せて、担当者に任せる
- 書き込む前に、作る・変える内容を見せて確認をとる
- 質問は選択肢で示し、推奨のものに「（推奨）」を付ける。どの質問でも、自由に書いて答えられるようにする

## 手順

### 1. 事前確認

[事前確認](${CLAUDE_PLUGIN_ROOT}/reference/preflight.md) の手順をすべて行う。止まったら、それ以上は進めない。

### 2. 今の設定を読む

- `dependabot-triage.yml` があれば読む。あれば、以降の質問では今の値を初期値として示す（「今の値：〜」）
- `.dependabot-triage/guidance.md` があれば読む
- `.github/ISSUE_TEMPLATE/` があれば、どんなテンプレートがあるかを見ておく（手順 4 の「要対処」の扱いで使う）

### 3. コードベースを読み取る

[コードベースの読み取り](survey.md) に沿ってリポジトリを読み、次の 3 つを作る。

- プリセットの当たり（根拠つき）
- 公開範囲の当たり（根拠つき）
- `guidance.md` の案（項目ごとに、どこから読み取ったか。推測なら推測と書く）

`guidance.md` がすでにあれば、案は「今の内容に足す・直すもの」として作る。今の内容と食い違う読み取り結果があれば、それも挙げる。

読み取った結果を、質問に入る前にまとめて見せる。

### 4. 質問する

設定の各項目の意味は [設定の仕様](${CLAUDE_PLUGIN_ROOT}/reference/config.md) を参照する。次の順に聞く。手順 3 で当たりを付けた項目は、その値を推奨にして、根拠を添える。

**判定の基準**

1. プリセット（`preset`）：[プリセット](${CLAUDE_PLUGIN_ROOT}/reference/presets.md) の「想定するプロジェクト」を見せて選んでもらう

**プロジェクトの考慮事項**

2. 公開範囲（`exposure`）：インターネットに公開している／認証の内側や社内ネットワークからだけ使える
3. `guidance.md` の案：手順 3 で作った案を見せ、次を確かめる
   - 推測の項目が合っているか。合っていれば「（推測：…）」を外し、違っていれば直すか消す
   - 足りない事情がないか。例として、「管理画面は社内ネットワークからしか開けない」「ユーザーがアップロードしたファイルを処理する」「CI はフォークからの PR では動かない」を示す
   - 案に何も書けなかったときは、自由に書いてもらう（なくてもよい）

**閾値**

4. 判定の対象にする最低の深刻度（`thresholds.min_severity`）：low（すべて。推奨）／medium／high／critical
5. 深刻度にかかわらず対象にする EPSS（`thresholds.epss_min_percent`）：使わない／1% 以上（推奨）／10% 以上
6. KEV に載ったものを常に対象にするか（`thresholds.always_include_kev`）：する（推奨）／しない
7. 開発時だけの依存を対象にするか（`thresholds.include_development`）：する（推奨）／しない
8. 深刻度ごとの対処期限（`deadlines`）：次を推奨として示し、変えたいものがあるか聞く
   - 緊急：当日（0）
   - 高：7 日
   - 中：30 日
   - 低：期限なし（null）

閾値を下回ったアラートは判定に回らないことと、深刻度はプロジェクトの使い方を考慮しない一般的な値なので、取りこぼしを避けたいなら低めにするとよいことを、質問のときに伝える。

**「要対処」の扱い**

9. 「要対処」と判定したもの（`action_required.mode`）：Issue にする／見せるだけ
   - 見せるだけを選んだら、記録が残らず、次の check でもう一度判定されることを伝える
   - どちらを選んでも、保留にしたものは Issue になることを伝える
10. Issue にするなら、次も聞く
    - 足すラベル（`action_required.labels`）：なくてもよい
    - 担当者（`action_required.assignees`）：なくてもよい
    - テンプレート（`action_required.template`）：手順 2 で見つけたテンプレートがあれば選択肢に出す。使わないなら dependabot-triage の形式になる
    - そのほかのルール（タイトルの決まり など）は、`guidance.md` に書く。手順 3 で CONTRIBUTING などから読み取れた決まりがあれば、案に入れておく

### 5. ファイルを作る

回答から、次のファイルの内容を作り、見せて確認をとってから書き込む。すでにあるファイルは、変わる部分がわかるように見せる。

- `dependabot-triage.yml`：[設定の仕様](${CLAUDE_PLUGIN_ROOT}/reference/config.md) の例と同じ並びとコメントで書く
- `.dependabot-triage/guidance.md`：質問 3 で確かめた案と、質問 10 で決めたルール。何もなければ、見出しと書き方の例だけを書いておく。担当者が確かめなかった推測の項目は、「（推測：…）」を付けたまま残す
- `.claude/settings.json`：チームのメンバーにインストールを促すため、次を足す。ファイルがあれば、ほかの設定を消さずに足す

```json
{
  "extraKnownMarketplaces": {
    "dependabot-triage": {
      "source": {
        "source": "github",
        "repo": "yth401/dependabot-triage"
      }
    }
  },
  "enabledPlugins": {
    "dependabot-triage@dependabot-triage": true
  }
}
```

書き込んだら、作った設定を [設定の仕様](${CLAUDE_PLUGIN_ROOT}/reference/config.md) の「確かめること」で確かめる。

### 6. 終わりに

次を伝える。

- 作った・変えたファイルの一覧
- コミットと push は担当者が行うこと
- 判定するときは `/dependabot-triage:check` を実行すること
- 設定を変えるときは `/dependabot-triage:settings` を実行すること
