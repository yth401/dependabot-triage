# dependabot-triage.yml の仕様

使う側のリポジトリのルートに置く設定ファイル。init が作り、settings で変える。

## 例

```yaml
# dependabot-triage の設定
preset: static-site          # 判定の前提にするプリセット
exposure: internal           # internet / internal

thresholds:                  # 判定の対象にするアラート
  min_severity: low          # low / medium / high / critical
  epss_min_percent: 1        # この値（%）以上なら深刻度にかかわらず対象。null なら使わない
  always_include_kev: true   # KEV に載ったものは深刻度にかかわらず対象
  include_development: true  # 開発時だけの依存（Dependabot の scope が development）も対象

deadlines:                   # このプロジェクトでの深刻度ごとの対処期限（日数）
  urgent: 0                  # 緊急。0 は当日
  high: 7                    # 高
  medium: 30                 # 中
  low: null                  # 低。null は期限を決めない

action_required:             # 「要対処」と判定したものの扱い
  mode: issue                # report（見せるだけ） / issue（Issue にする）
  labels: [security]         # Issue に足すラベル（mode が issue のとき）
  assignees: []              # Issue の担当者（mode が issue のとき）
  template: null             # Issue のテンプレートのパス。null なら dependabot-triage の形式
```

## 項目

| 項目 | 必須 | 値 | 意味 |
|---|---|---|---|
| `preset` | ○ | `web-service` / `internal-service` / `static-site` / `library` / `cli-tool` | 判定の前提。中身は [presets.md](presets.md) |
| `exposure` | ○ | `internet` / `internal` | 公開範囲。`internal` は認証の内側や社内ネットワークからだけ使える場合 |
| `thresholds.min_severity` | ○ | `low` / `medium` / `high` / `critical` | Dependabot の深刻度がこれ以上のアラートを判定の対象にする |
| `thresholds.epss_min_percent` | ○ | 0〜100 の数値、または `null` | EPSS がこの値以上なら、深刻度にかかわらず対象にする。深刻度を上げる条件の「EPSS が高い」の基準にも使う |
| `thresholds.always_include_kev` | ○ | `true` / `false` | KEV に載ったものを、深刻度にかかわらず対象にする |
| `thresholds.include_development` | ○ | `true` / `false` | `false` なら、Dependabot の `scope` が `development` のアラートを対象から外す |
| `deadlines.urgent` 〜 `deadlines.low` | ○ | 0 以上の整数（日数）、または `null` | このプロジェクトでの深刻度ごとの対処期限。判定した日から数える |
| `action_required.mode` | ○ | `report` / `issue` | 「要対処」を見せるだけにするか、Issue にするか |
| `action_required.labels` | | 文字列の配列 | Issue に足すラベル。dependabot-triage のラベル（[issues.md](issues.md)）とは別に付ける |
| `action_required.assignees` | | GitHub のユーザー名の配列 | Issue の担当者 |
| `action_required.template` | | リポジトリのルートからのパス、または `null` | Issue のテンプレート。指定があれば、その見出しに沿って本文を書く |

## 確かめること

設定を読んだら、次を確かめる。問題があれば、どこがどう違うかを示し、`/dependabot-triage:settings` で直すよう案内する。

- YAML として読める
- 必須の項目がそろっている
- 値が上の表の範囲に入っている
- 表にない項目がない（項目名の書き間違いが、黙って無視されるのを防ぐため）
- `action_required.template` を指定しているなら、そのファイルがある
- `preset` と `exposure` の組み合わせが食い違っていない（[プリセット](presets.md) の「プリセットと公開範囲の組み合わせ」）。食い違っていれば警告する

## guidance.md

`.dependabot-triage/guidance.md` は自由に書くファイル。判定のときに AI に考慮してほしい事情や、プリセットに足りない観点を書く。形式は決めない。

例：

```markdown
# dependabot-triage への補足

- 管理画面（/admin）は社内ネットワークからしか開けない
- ユーザーがアップロードした画像をサーバーで変換している
- CI はフォークからの PR では動かない
- Issue のタイトルの先頭には [security] を付ける
```
