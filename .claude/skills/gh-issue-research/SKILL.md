---
name: gh-issue-research
description: GitHub リポジトリの既存 Issue・PR とソースコードを読み取り専用で調査し、調査メモを作る。Issue 作成前の関連・重複確認や影響範囲の把握に使う。「〇〇を調査して」「関連Issueを探して」で起動。
---

# gh-issue-research

Issue を作る前に、対象リポジトリの Issue・PR・ソースを読んで調査メモを作る。**読み取り専用**。Issue の作成・編集・コメント、push は行わない。

## 既定値

- 対象リポジトリ: 引数で `owner/repo` が渡されたらそれを使う。無ければカレントの git リモートから `gh repo view --json nameWithOwner -q .nameWithOwner` で取得する
- 前提: `gh auth status` が成功すること。失敗したらそこで止めて、ユーザーに認証を依頼する

## 手順

1. 調査テーマと対象リポジトリを確認する。テーマが曖昧なら質問は1つだけにする。
2. Issue / PR を調べる。キーワードは最低2パターン（名詞、別名、エラー文言など）で検索する。
   - `gh issue list -R <repo> --state all --search "<キーワード>" --limit 30 --json number,title,state,labels,url`
   - `gh pr list -R <repo> --state all --search "<キーワード>" --limit 30 --json number,title,state,url`
   - 関連しそうな Issue は `gh issue view <n> -R <repo> --comments` で本文とコメントを読む
3. ソースを調べる。
   - カレントディレクトリが対象リポジトリならそこで Grep / Read する
   - 無ければ一時ディレクトリに `gh repo clone <repo> <dir> -- --depth 1` して読む
   - 見る対象: テーマに関連するファイル・関数、README・docs、テスト、`.github/ISSUE_TEMPLATE`
   - コミットが無い空リポジトリなら「ソースなし」と報告し、Issue 調査だけで終える
4. 調査メモを下の形式で出力する。

## 出力形式

```
## 調査メモ: <テーマ>
対象: owner/repo（調査日: YYYY-MM-DD）

### 関連 Issue / PR
- #n タイトル（open|closed）: 関連理由。重複の疑い: あり|なし

### 関連コード
- path:line: 内容

### 分かったこと
- 確認できた事実のみ

### 未確認・不明点
- 推測で埋めず、ここに列挙する

### Issue 化の示唆
- 分割の観点（実装 / 調査 / ドキュメントなど）と、既存 Issue で足りるか
```

## ルール

- 事実と推測を分ける。ソースやIssueで確認できないことは「未確認・不明点」に書く。
- ログやファイルの全文を貼らない。`path:line` と要点だけにする。
- 出力は `gh-issue-create` にそのまま渡せる形にする。
- `gh` が `authentication failed` を返し、環境変数 `GITHUB_TOKEN` が原因の場合は、ユーザーに無効なトークンの解除を依頼する。
