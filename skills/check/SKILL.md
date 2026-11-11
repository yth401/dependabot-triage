---
name: check
description: 開いている Dependabot アラートを取ってきて、このプロジェクトで実際に対処が必要か（要対処／不要／不明）と、このプロジェクトでの深刻度を判定する。不要なものは確認のうえ理由を付けてアラートを閉じる。要対処のものは、設定に応じて Issue にするか結果を見せるだけにし、担当者が選べば期限付きの保留の Issue にする。Dependabot の通知が来たとき、気になる脆弱性が報じられたとき、「脆弱性をチェックして」「Dependabot のアラートを判定して」と頼まれたときに使う。
argument-hint: "[アラート番号 ...]"
---

# dependabot-triage:check

開いている Dependabot アラートについて、このプロジェクトで実際に対処が必要かを判定する。

引数にアラートの番号が渡されたら（例：`/dependabot-triage:check 42 57`）、そのアラートだけを判定の対象にする。閾値とは関係なく判定する。引数がなければ、開いているアラートすべてを対象にする。

引数：$ARGUMENTS

## 守ること

- アラートを閉じる、Issue を作る・書き換える・閉じる・コメントする、ラベルを作る、の前には、必ず担当者の確認をとる。まとめて確認してよいが、何をするかを 1 件ずつ見せる
- 保留にするかどうかは AI が決めない。担当者が選ぶ
- 判定に確信が持てないときは「不明」にする。迷ったら「不要」にしない
- 判定の途中でコードを書き換えない
- Bash ツールは呼び出しをまたいでシェルの変数を覚えていない。一時ファイルのパスなどは、変数に頼らず、1 回の呼び出しの中で使い切るか、パスをそのまま書く
- コメントや本文のように、利用者に見せる文章をコマンドに渡すときは、いったんファイルに書いてから渡す（コマンドラインに直接書くと、バッククォートや `$` が展開されて壊れるため）

## 手順

### 1. 事前確認

[事前確認](${CLAUDE_PLUGIN_ROOT}/reference/preflight.md) の手順をすべて行う。止まったら、それ以上は進めない。

### 2. 設定を読む

- `dependabot-triage.yml` を読み、[設定の仕様](${CLAUDE_PLUGIN_ROOT}/reference/config.md) の「確かめること」を確かめる。ファイルがなければ、`/dependabot-triage:init` を実行するよう案内して止める。問題があれば、どこがどう違うかを示して止める
- `.dependabot-triage/guidance.md` があれば読む
- `preset` に対応する前提を [プリセット](${CLAUDE_PLUGIN_ROOT}/reference/presets.md) で確かめる

### 3. アラートを取ってくる

```bash
gh api --paginate "repos/OWNER/REPO/dependabot/alerts?state=open&per_page=100" --jq '.[] | {
  number,
  url: .html_url,
  package: .dependency.package.name,
  ecosystem: .dependency.package.ecosystem,
  manifest: .dependency.manifest_path,
  scope: .dependency.scope,
  relationship: .dependency.relationship,
  ghsa: .security_advisory.ghsa_id,
  cve: .security_advisory.cve_id,
  summary: .security_advisory.summary,
  classification: .security_advisory.classification,
  severity: .security_advisory.severity,
  cvss: ([.security_advisory.cvss_severities.cvss_v4.score, .security_advisory.cvss_severities.cvss_v3.score, .security_advisory.cvss.score] | map(select(. != null and . > 0)) | first),
  epss: (.security_advisory.epss | if type == "array" then .[0] else . end | if . == null then null else .percentage end),
  vulnerable_range: .security_vulnerability.vulnerable_version_range,
  patched: .security_vulnerability.first_patched_version.identifier
}'
```

- `cvss` は、値がない（0 か null の）ものを飛ばして、v4、v3、旧来の値の順に最初のものを取る。取れなければ「CVSS なし」と示す
- `epss` は 0〜1 の割合として扱い、100 倍して % にしてから、設定の `epss_min_percent` と比べたり人に見せたりする
- 開いているアラートが 0 件なら、その旨を伝えて終わる
- 引数で渡された番号のアラートが、開いているアラートの中になければ、その番号を示して「開いていない（閉じた・直った・番号の誤り）」ことを伝える。ほかの番号の判定は続ける

### 4. KEV を照合する

CISA の KEV（実際に悪用が確認された脆弱性の一覧）を取ってきて、アラートの CVE ID を照合する。約 2 MB あるので、まるごと読まない。KEV のデータは項目ごとに改行されているので、改行を取り除いてから、1 件分（`{` から `}` の手前まで）を取り出す。取得から照合、一時ファイルの削除までを、**1 回の Bash の呼び出しの中で**行う。`for` の後ろには、手順 3 で取ったアラートの CVE ID（null でないもの）を、重複を除いて並べる。

```bash
KEV_FILE="$(mktemp)"
if curl -sSfL https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json | tr -d '\r\n' > "$KEV_FILE" \
   && [ "$(grep -o '"cveID"' "$KEV_FILE" | wc -l)" -ge 100 ]; then
  for id in CVE-2026-00001 CVE-2026-00002; do
    hit="$(grep -o "\"cveID\": *\"$id\"[^}]*" "$KEV_FILE" | head -n 1)"
    if [ -n "$hit" ]; then echo "KEV_HIT $id $hit"; else echo "KEV_MISS $id"; fi
  done
else
  echo "KEV_UNAVAILABLE"
fi
rm -f "$KEV_FILE"
```

- `KEV_HIT`：「KEV あり」。`dateAdded`（載った日）と `knownRansomwareCampaignUse`（ランサムウェアで使われているか）を判定の材料にする
- `KEV_MISS`：「KEV なし」
- アラートに CVE ID がなければ：「KEV：照合できず」
- `KEV_UNAVAILABLE`（取ってこられない、または中身が一覧として読めない）：すべて「KEV：照合できず」とし、まとめにその旨を書く。判定は続ける。KEV で判定の対象に入るはずのものを取りこぼさないよう、このときは `always_include_kev` を当てはめられないことも伝える

### 5. 判定の対象を決める

開いている Issue を [dependabot-triage が作る Issue](${CLAUDE_PLUGIN_ROOT}/reference/issues.md) の「開いている Issue の読み方」で読み、アラートを次のように分ける。

| 分類 | 条件 | 扱い |
|---|---|---|
| 起票済 | `kind=action-required` の Issue、または `until` が今日より後の `kind=deferred` の Issue がある | 判定しない |
| 保留の期限切れ | `kind=deferred` の Issue があり、`until` が今日以前 | 判定する。確認と反映は、手順 8 の「保留の期限切れ」だけで行う |
| 閾値で対象外 | 上のどれでもなく、閾値に当てはまらない | 判定しない |
| 判定の対象 | 上のどれでもなく、閾値に当てはまる | 判定する |

閾値（`thresholds`）の当てはめ方：

- `include_development` が `false` で、`scope` が `development` なら対象外
- 次のどれかに当てはまれば対象
  - Dependabot の深刻度が `min_severity` 以上
  - `epss_min_percent` が `null` でなく、EPSS がその値以上
  - `always_include_kev` が `true` で、KEV に載っている

引数でアラートの番号が渡されたときは、閾値の当てはめは行わない。ただし、起票済の分類はそのまま当てはめ、そのアラートは判定しない理由を示す。

### 6. 判定の進め方を決める

- 同じ勧告（GHSA）が複数のマニフェストに出ているものは、まとめて 1 件として判定する。結果は、それぞれのアラートに当てはめる
- 判定するものが 20 件を超えるときは、件数を伝え、判定する範囲を担当者に聞く（例：Dependabot の深刻度が高い順に 20 件ずつ、特定のマニフェストだけ）。残ったものは、次の check で判定できる

### 7. 判定する

決めた範囲を、[判定の基準](criteria.md) に沿って 1 件ずつ判定する。

- 勧告の詳しい内容が要るときは `gh api repos/OWNER/REPO/dependabot/alerts/<番号>` で取る。勧告の本文（`security_advisory.description`）と参考リンク（`security_advisory.references`）から、脆弱な関数、モジュール、設定、攻撃の条件を読み取る
- 依存の実際のバージョンは、`manifest` のファイル（lock ファイルなど）で確かめる
- 脆弱な機能を使っているかは、[判定の基準](criteria.md) の「観点 1 の確かめ方」に沿って確かめる

判定し終えたら、[出力の形](output.md) で結果を示す。

### 8. 確認して反映する

結果を示したあと、次の順で担当者に確認する。保留の期限切れのアラートは、「要対処」「不要」「不明」の処理には入れず、最後の「保留の期限切れ」だけで扱う。

**Issue を作る前に**

Issue を作る見込みがあるとき（要対処を Issue にする設定のとき、または保留を選んだとき）は、使うラベル（`dependabot-triage`、`dependabot-triage:deferred`、`action_required.labels`）がリポジトリにあるかを確かめる。

```bash
gh label list --limit 1000 --json name --jq '.[].name'
```

ないラベルは、作ってよいか確認してから作る（[dependabot-triage が作る Issue](${CLAUDE_PLUGIN_ROOT}/reference/issues.md) の「ラベルがないとき」）。作らないと言われたら、そのラベルを付けずに Issue を作る。ただし `dependabot-triage` と `dependabot-triage:deferred` がないと、次の check が Issue を見分けられないので、その旨を伝える。

**要対処**

1 件ずつ、次のどれかを選んでもらう。

- Issue にする（`action_required.mode` が `issue` のときだけ示す）
- 保留にする：理由と期限（`YYYY-MM-DD`）を聞く
- 何もしない（結果を見るだけ）

選ばれたとおりに、[dependabot-triage が作る Issue](${CLAUDE_PLUGIN_ROOT}/reference/issues.md) の形で本文をファイルに書き、Issue を作る。

```bash
gh issue create --title "<タイトル>" --body-file <本文のファイル> --label dependabot-triage [--label ...] [--assignee ...]
```

担当者（`--assignee`）が付けられずに失敗したときは、担当者なしで作り直してよいか確認し、その旨を伝える。

**不要**

閉じるアラートの一覧（番号、パッケージ、閉じる理由、コメント）を見せて確認をとり、閉じる。KEV に載っているものは、ほかとまとめずに 1 件ずつ確認をとる。

コメントはファイルに書いてから渡す。

```bash
gh api --method PATCH repos/OWNER/REPO/dependabot/alerts/<番号> \
  -f state=dismissed \
  -f dismissed_reason=<理由の区分> \
  -F dismissed_comment=@<コメントのファイル>
```

- 理由の区分は `fix_started`、`inaccurate`、`no_bandwidth`、`not_used`、`tolerable_risk` のどれか。選び方は [出力の形](output.md) に従う
- コメントは 280 文字以内

**不明**

何もしない。担当者が調べて判断するよう伝える。

**保留の期限切れ**

[dependabot-triage が作る Issue](${CLAUDE_PLUGIN_ROOT}/reference/issues.md) の「期限が過ぎた保留の Issue」に沿って進める。Issue へのコメントも、担当者の確認をとってから行う。

### 9. 終わりに

行ったこと（閉じたアラート、作った Issue、保留にしたもの、コメントした Issue）を短くまとめて示す。手順 6 で判定しなかったものが残っていれば、その件数を伝える。失敗した操作があれば、そのエラーと、手で行う方法を示す。
