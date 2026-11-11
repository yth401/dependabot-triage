# 事前確認

init と check の最初に、すべての手順を行う。settings は、手順 1 と 2 だけを行う。どれかで止まったら、理由と対処を示して、それ以上は進めない。

## 1. gh の認証

```bash
gh auth status
```

ログインしていなければ、`! gh auth login` を実行するよう案内する（`!` を付けるとこのセッションの中で実行できる）。

## 2. 対象のリポジトリ

```bash
gh repo view --json nameWithOwner --jq .nameWithOwner
```

以降の `OWNER/REPO` はこの値に置き換える。取れなければ、GitHub のリポジトリの中で実行しているかを確かめるよう案内する。

## 3. 権限と Issue 機能

```bash
gh api repos/OWNER/REPO --jq '{permissions, has_issues}'
```

**権限**：`push`、`maintain`、`admin` のどれかが `true` であること。

- どれも `false` なら、リポジトリでの役割が Read か Triage で、ふつうはアラートを閉じられない。ただし、組織でセキュリティの管理を任された役割（security manager など）なら、閉じられる場合がある
- そのため、すぐには止めない。「アラートを閉じられない可能性が高い。判定と結果を見ることはできる」と伝え、続けるか聞く。続けるなら、閉じる操作が失敗したときに、そのエラーを示す

**Issue 機能**：`has_issues` が `true` であること。

- `false` なら、dependabot-triage は Issue を作れない。保留も Issue で記録するので、保留も使えない
- 止めずに、その旨を伝える。init では「要対処」の扱いを「見せるだけ」にするよう勧める。check では、保留と「Issue にする」を選択肢から外す

## 4. Dependabot alerts

```bash
gh api "repos/OWNER/REPO/dependabot/alerts?state=open&per_page=1" --jq length
```

成功すれば、Dependabot alerts が有効で、トークンにも読む権限がある。

失敗したら、エラーの内容を読んで、人にわかる言葉で案内する。よくある原因：

- Dependabot alerts が有効になっていない：リポジトリの Settings → Code security で有効にする（管理者の権限が要る）
- トークンのスコープが足りない：`! gh auth refresh -s security_events` を実行する
