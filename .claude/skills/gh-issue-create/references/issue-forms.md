# Issue Forms の本文レンダリング規則

Issue Forms（`.github/ISSUE_TEMPLATE/*.yml`）は UI 専用で、CLI や API からはフォームとして送信できない。UI が生成するのと同じ形式の Markdown を自前で組み立て、`gh issue create --body-file` に渡す。

## テンプレの frontmatter

| キー        | 扱い                                                           |
| ----------- | -------------------------------------------------------------- |
| `title`     | 接頭辞としてタイトルの先頭に付ける                             |
| `labels`    | `--label` に渡す。リポジトリに無いラベルは事前に作成を提案する |
| `assignees` | 指定があれば `--assignee` に渡す。無ければ既定の `@me`         |
| `projects`  | 使わない。Project への追加は別手順で行う                       |

## body[] の項目

`body[]` の順に、項目間を空行1つで並べる。

| type                        | 出力                                                                                                       |
| --------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `markdown`                  | 出力しない（UI 上の説明文）                                                                                |
| `input` / `textarea`        | `### {label}` + 空行 + 値。値が空なら `_No response_`                                                      |
| `textarea`（`render` あり） | 値を ` ```{render} ` のコードフェンスで囲む                                                                |
| `dropdown`                  | `### {label}` + 空行 + 選んだ option の文字列。`multiple: true` なら `, ` 区切り。未選択は `_No response_` |
| `checkboxes`                | `### {label}` + 空行 + 各 option を `- [x] {label}` または `- [ ] {label}` で1行ずつ                       |

- `required: true` の checkboxes option はチェック済み（`- [x]`）にする。ただし、その内容が事実である場合に限る。例: 「同じ Issue が無いことを確認した」は、実際に検索して確認してから付ける
- `label` は `attributes.label` の文字列をそのまま使う

## 例

```markdown
### 概要

ログイン後に画面が真っ白になる。

### 再現手順

1. ログインする
2. トップに遷移する

### 環境

_No response_

### 確認事項

- [x] 同じ内容の Issue が無いことを確認した
```

## 検証状況

上の規則は GitHub の Issue Forms が生成する本文の形式に基づく。UI から同じテンプレで作った Issue の本文と、この規則で作った本文を実際に比べて確認する（未実施）。差異が見つかったらこのファイルを更新する。
