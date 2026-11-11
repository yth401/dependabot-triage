# dependabot-triage が作る Issue

dependabot-triage が作る Issue は 2 種類ある。

| 種類 | いつ作るか | ラベル |
|---|---|---|
| 要対処 | 「要対処」のものを、担当者が Issue にすると選んだとき（`action_required.mode` が `issue` のときだけ） | `dependabot-triage` と `action_required.labels` |
| 保留 | 「要対処」のものを、担当者が保留にすると選んだとき（`action_required.mode` にかかわらず） | `dependabot-triage` と `dependabot-triage:deferred` |

## 目印

本文の 1 行目に、次の目印を必ず入れる。check はこの目印で、どのアラートの Issue かを見分ける。

```
<!-- dependabot-triage alert=42 kind=action-required -->
<!-- dependabot-triage alert=42 kind=deferred until=2026-12-31 -->
```

- `alert`：Dependabot アラートの番号
- `kind`：`action-required`（要対処）か `deferred`（保留）
- `until`：保留の期限（`YYYY-MM-DD`）。保留のときだけ

## 開いている Issue の読み方

```bash
gh issue list --state open --label dependabot-triage --limit 1000 --json number,title,body,url
```

本文の 1 行目の目印から、アラートの番号、種類、期限を取り出す。

## ラベルがないとき

`dependabot-triage` と `dependabot-triage:deferred` のラベルがリポジトリになければ、Issue を作る前に、作ってよいか確認してから作る。

```bash
gh label create dependabot-triage --description "dependabot-triage が作った Issue" --color 5319E7
gh label create dependabot-triage:deferred --description "dependabot-triage：期限付きで保留中" --color FBCA04
```

## タイトル

```
[dependabot-triage] 要対処：<パッケージ名> <GHSA の ID または CVE の ID>（アラート #<番号>）
[dependabot-triage] 保留：<パッケージ名> <GHSA の ID または CVE の ID>（アラート #<番号>、<期限> まで）
```

`guidance.md` にタイトルの決まりがあれば、それに従う（目印は本文に入れるので、タイトルを変えても見分けられる）。

## 本文（要対処）

`action_required.template` を指定しているなら、そのテンプレートの見出しに沿って、下の内容を振り分けて書く。指定がなければ、この形で書く。

```markdown
<!-- dependabot-triage alert=42 kind=action-required -->
## 概要

- アラート：<アラートの URL>
- パッケージ：<パッケージ名>（<マニフェストのパス>）
- 勧告：<GHSA の ID> / <CVE の ID>
- Dependabot の評価：<深刻度>、CVSS <値>、EPSS <値>%、KEV <あり／なし／照合できず>

## 判定

- このプロジェクトでの深刻度：<緊急／高／中／低>（<上げ下げの経緯>）
- 対処期限：<YYYY-MM-DD、または「期限なし」>

## やること

<何をどのバージョンに上げるか、回避策 など>

## 根拠

<観点ごとの答え。攻撃が成り立つ筋書き>

---
dependabot-triage による判定（<判定した日>）
```

## 本文（保留）

```markdown
<!-- dependabot-triage alert=42 kind=deferred until=2026-12-31 -->
## 概要

- アラート：<アラートの URL>
- パッケージ：<パッケージ名>（<マニフェストのパス>）
- 勧告：<GHSA の ID> / <CVE の ID>

## 保留の理由

<担当者が入力した理由>

## 期限

<YYYY-MM-DD>。期限が過ぎたら、次の `/dependabot-triage:check` でもう一度判定する。

## 保留にしたときの判定

- このプロジェクトでの深刻度：<緊急／高／中／低>
- 根拠：<観点ごとの答えの要約>

---
dependabot-triage による記録（<保留にした日>）。保留を取りやめるときは、この Issue を閉じる。
```

## 期限が過ぎた保留の Issue

check でもう一度判定したら、次の順で進める。このアラートは、check のふつうの「要対処」「不要」「不明」の処理には入れない（要対処の Issue を重ねて作ったり、保留の Issue を開いたままアラートだけ閉じたりしないため）。

1. 判定の結果と、その Issue に書くコメントの文面を見せる
2. 担当者に、下のどれかを選んでもらう
3. 選ばれた操作とコメントを、まとめて確認をとってから行う

| 選択 | すること |
|---|---|
| 保留を延ばす | 新しい期限と理由を聞く。本文 1 行目の `until` を書き換え（`gh issue edit --body-file`）、タイトルの期限も直す（`gh issue edit --title`）。判定の結果と延ばした理由をコメントする |
| 要対処に切り替える | `action_required.mode` が `issue` なら、本文を要対処の形で書き直し（1 行目の `kind` を `action-required` にし、`until` を外す）、タイトルを要対処の形に直し、`dependabot-triage:deferred` のラベルを外して `action_required.labels` と `action_required.assignees` を付ける。判定の結果をコメントする。`report` なら、判定の結果をコメントして Issue を閉じ、結果を見せるだけにする |
| 不要として閉じる | 判定が「不要」のときだけ選べる。アラートを閉じ（check の手順 8 の「不要」と同じやり方）、判定の結果をコメントして Issue を閉じる |
| 何もしない | 判定が「不明」のときなど。Issue もアラートもそのままにする。次の check でもう一度判定される |

コメントの本文はファイルに書いてから渡す。

```bash
gh issue comment <Issue の番号> --body-file <コメントのファイル>
```
