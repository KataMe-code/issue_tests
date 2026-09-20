---
name: gh-issue-create
description: 要件をテンプレ単位の粒度に分割し、リポジトリの Issue Forms を項目ごとに埋めて GitHub Issue を作成し、Projects の Status / Priority / Size、ラベル、担当者、親子関係まで設定する。必ず下書きの承認後に作成する。「Issue を作って」「要件を Issue に分割して」で起動。
disable-model-invocation: true
---

# gh-issue-create

要件から GitHub Issue を作る。流れは「テンプレ取得 → 分割 → 充填 → 下書き提示 → 承認 → 作成 → Project 設定」。

## 既定値

- 対象リポジトリ: 引数で `owner/repo` が渡されたらそれを使う。無ければカレントの git リモートから `gh repo view --json nameWithOwner -q .nameWithOwner` で取得する
- 担当者: `@me`
- Project: 引数で番号が渡されたらそれを使う。無ければ `gh project list --owner <owner> --format json` で取得し、1つならそれを使い、複数ならユーザーに選んでもらう

## 絶対ルール

- 対象は public リポジトリ。Issue 作成・Project 追加・sub-issue 紐付けは外部から見える操作。**ユーザーが承認した Issue についてだけ**実行する。
- 承認前に `gh issue create`、`gh project item-add`、`gh project item-edit`、`gh api -X POST` を実行しない。
- 途中で失敗したら、作成済みのものを報告して止める。再実行する前に重複が出ないかユーザーに確認する。

## 事前確認

1. `gh auth status` が成功し、scopes に `project` と `repo` があること
2. `gh project --help` が使えること（無ければ gh が古い。公式 apt リポジトリ版への更新を依頼する）

どちらか失敗したら、そこで止めて原因を報告する。

## 手順

### 1. 入力を整理する

- 入力は要件と、あれば `gh-issue-research` の調査メモ。要件が渡されていなければ、最初に要件を聞く
- 既存コードや既存 Issue との関連が分割に影響しそうなのに調査メモが無い場合は、先に `gh-issue-research` の実行を提案する（調査は読み取り専用なので実行してよい）

### 2. テンプレを取得する

- ローカルに clone があれば `.github/ISSUE_TEMPLATE/*.yml` を読む
- 無ければ `gh api repos/<o>/<r>/contents/.github/ISSUE_TEMPLATE --jq '.[].name'` で一覧を取り、各ファイルを `gh api repos/<o>/<r>/contents/.github/ISSUE_TEMPLATE/<name> -H "Accept: application/vnd.github.raw"` で取得する
- テンプレが1つも無ければ、止めて報告する

### 3. 分割する

- 1 Issue = 1 PR で完結し、1つのテンプレで表せる粒度にする
- 完了条件が1〜3個で書ける大きさを目安にする
- 分割が不要なら1件にする
- 分割後に子が2件以上になる場合だけ、親 Issue（`feature_request`）を作り、子（`task` / `bug_report`）を sub-issue として紐付ける
- テンプレの選び方: 不具合は `bug_report`、新機能・改善は `feature_request`、それ以外の作業は `task`

### 4. テンプレを埋める

- 本文の組み立ては [references/issue-forms.md](references/issue-forms.md) に従う
- `required: true` の項目は必ず埋める
- dropdown は `options` の中から選ぶ
- 要件や調査メモに根拠が無い項目は推測で埋めず、ユーザーに聞く
- タイトルはテンプレの `title` 接頭辞 + 具体的な題にする
- 言語は日本語

### 5. Project フィールドの値を決める

- `gh project field-list <番号> --owner <owner> --format json` で Status / Priority / Size のフィールド ID と選択肢を取得する。ID は毎回取得し、書き写して固定しない
- Status は原則 Todo にする
- Priority と Size は提案値として下書きに載せる。ユーザーが修正できる
- フィールドが Project に無い場合は、作成を提案する（例: `gh project field-create <番号> --owner <owner> --name Priority --data-type SINGLE_SELECT --single-select-options "P0,P1,P2"`）。承認後にだけ実行する
- テンプレの `labels` にあるラベルがリポジトリに無い場合は、作成を提案する（例: `gh label create task`）。承認後にだけ実行する

### 6. 下書きを提示する

Issue ごとに次を一覧で出す。

- 通し番号、親子関係（親 / 子 / 単独）
- タイトル、テンプレ種別、ラベル、担当者
- Status / Priority / Size
- 本文（組み立て後の Markdown）

承認の受け取り方: 「全部」「#1 と #3 だけ」「#2 を修正」。修正が入ったら下書きを更新して再提示する。

### 7. 作成する（承認した Issue のみ。親 → 子の順）

1. 本文を一時ファイルに書く
2. Issue を作る
   - `gh issue create -R <repo> --title "<題>" --body-file <file> --label <label> --assignee @me`
   - 出力された URL から Issue 番号を取る
3. Project に追加する
   - `gh project item-add <番号> --owner <owner> --url <issue-url> --format json` で item ID を取る
   - `gh project view <番号> --owner <owner> --format json` で project ID を取る
4. フィールドを設定する（Status / Priority / Size のそれぞれ）
   - `gh project item-edit --id <item-id> --project-id <project-id> --field-id <field-id> --single-select-option-id <option-id>`
5. 親子関係を設定する（子のみ）
   - 子の numeric id: `gh api repos/<o>/<r>/issues/<子番号> --jq .id`
   - `gh api -X POST repos/<o>/<r>/issues/<親番号>/sub_issues -F sub_issue_id=<numeric id>`

### 8. 結果を報告する

- 作成した Issue の URL 一覧、設定した Status / Priority / Size、親子関係
- 失敗した操作があれば、どの Issue のどの操作かを明記する
