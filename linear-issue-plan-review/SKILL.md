---
name: linear-issue-plan-review
description: "指定された Linear Issue の計画をRepository-awareに作成して保存する。個人用・低リスクのスクリプトは短い軽量計画を標準とし、独立レビューは明示時または高リスク時に使う。"
---

# Linear Issue Plan Review

指定IssueをSource of TruthとしてRepositoryを確認したImplementation Planを作成し、プロファイルに応じた確認後にLinearへ保存します。承認後の実装は扱いません。

## プロファイル選択

このスキルを呼び出しただけでは、厳格プロファイルを選ばない。IssueとRepositoryから、単一ユーザーのローカル用途、短いスクリプト、共有サービス・本番データ・機密情報・不可逆な外部副作用がないと確認でき、ユーザーが「厳格」「レビュー」「独立レビュー」「テスト先行」「本番」「共有」「公開」「セキュリティ」などを明示していない場合は、軽量プロファイルを使う。条件に当てはまらない場合は厳格プロファイルを使う。

### 軽量プロファイル（標準）

- 親AgentだけでIssueと対象Repositoryを確認し、Issueの目的に直接対応する短いPlanを作る。Planner・Reviewer Agent、レビューComment、Status変更、Cycle管理は行わない
- Planは `目的`、`変更対象`、`実施内容`、`最小限の検証`、`範囲外` に絞る。Issueにない仕様、互換性、失敗処理、セキュリティ方針を補わない
- 変更対象と検証が明確なら、受入条件やリスクマトリクスを増やさない。コードや行数を増やさないこと、依存関係を増やさないこと、後から読んで修正しやすいことを優先する
- DescriptionのPlanマーカーだけを更新し、1回の再取得で保存結果とマーカー外の保持を確認する。競合・取得不能・対象不明なら `BLOCKED`
- Issueから共有・本番・外部副作用・機密情報・権限・データ損失のリスクが判明したら、軽量プロファイルを続けず厳格プロファイルへ切り替えるか、必要な判断を `PLAN_BLOCKED` として停止する

### 厳格プロファイル

ユーザーが明示的に求めた場合、または軽量プロファイルの範囲を超えるリスクがある場合だけ、以下のPlanner、Reviewer、Comment、Status、Cycleの契約を適用する。

以下の「ツールとAgent」以降の独立レビュー、Status更新、レビューComment、Cycle制限は、厳格プロファイルに限る。

## 契約

- 開始対象はStatusが `Backlog` または `Todo` のIssue。`Backlog` はinitial-planなし、`Todo` のinitial-planは未検証入力として扱う
- Issue IDがない、複数ある、対象を特定できない場合はLinearを変更せず停止する
- Descriptionのマーカー外、タイトル、担当者、ラベル、関連付けは保持する。許可する外部書き込みは対象IssueのDescription、レビューComment、workflow Statusだけで、実行するのは親Agentだけ
- Workflow Statusは `In Plan Review`、`Todo`、`Test Implementation` だけを使う。Description内にStatus、Review cycle、終端状態を保存しない
- Linear、Agent、Repository、保存・再取得の不明・競合・失敗は推測せず `BLOCKED` で停止する

### Issueのモード

- Issueのタイトルが `Spike:` で始まる、または取得結果のラベルに `Spike` が含まれる場合は `Spike mode` として扱う。タイトルとラベルが矛盾する場合は、タイトルとdescriptionの目的を優先し、判定理由を計画に記載する
- 通常の実装計画では、Issueにない仕様を勝手に確定しない。Spike modeでは、検証を進めるために、局所的で可逆的な事項について暫定判断を置いてよい
- 暫定判断は事実や確定要件として書かず、`暫定判断`、根拠、影響範囲、検証方法、変更可能性を計画に明記する
- Spike modeで `PLAN_BLOCKED` にするのは、外部APIへの実データ送信、本番データ、認証情報、課金、権限、セキュリティ、不可逆な変更など、Issueから安全に選べず暫定判断でも検証できない事項に限る。ホットキー、検証用モデル、ファイル配置、最小限のUI、検証用エラー表示など、検証用の可逆的な細部は合理的な暫定値を置く
- Spike modeの受け入れ条件は完成品の品質ではなく、各検証ポイントについて成功・失敗・未検証を判定でき、採用した暫定判断、技術的制約、未実装範囲、次の判断事項を記録できることとする

## ツールとAgent

- Issue: `linear_get_issue`、保存： `linear_save_issue`、Comment: `linear_list_comments`/`linear_save_comment`
- Planner: `gpt-5.6-luna`/`xhigh`; Reviewer: `gpt-5.6-sol`/`high`
- 起動前に名前付きAgentの契約・実効設定を確認し、完全一致時だけ使う。不一致・不明時は `default` へ各モデル・推論設定を明示する。実効設定を確認できない、または契約外モデルしか使えない場合は `BLOCKED`
- PlannerとReviewerは別Agent。両者に同じRepository root、ローカル指示、確認対象codebaseを渡し、Reviewerは独立確認する。SubagentにLinear・ファイル・外部システムを書き込ませない

## Repositoryと入力固定

1. Repositoryは、明示パス → 現在workspace → workspaceから一意に決まるGit rootの順で決定する。不明・複数候補・検証不能なら書き込み前に `BLOCKED`
2. `linear_get_issue` でIssue、Description、Status、identifier、labels、project/teamを取得する。StatusがBacklog/Todo以外なら停止する。Descriptionが空の場合、通常modeでは推測で要件を補わず情報不足として停止し、Spike modeでは検証目的が取得情報から明確な場合だけ最小計画を許可する。Commentsは `linear_list_comments` をCursorで最後まで取得する。取得不能・重複識別不能なら停止する
3. 開始時に `description_baseline`、`status_baseline`、`comments_baseline` を固定する。既存Planマーカーは候補PlanとしてPlannerへ渡し、マーカー外は保持する

## 実行

### Planner

PlannerにはIssue、全Comments、確定Repository、ローカル指示、対象codebaseを渡す。Planには目的、範囲、要求との対応、確認済みファイル・依存関係・テスト根拠、実施項目、受入条件、検証、未確認事項を含める。通常modeで判断が不足する場合は `PLAN_BLOCKED`、Spike modeでは可逆的な暫定判断を明記する。

### Description保存

Planマーカー内は次の本文だけにする。マーカーがなければ末尾に追加し、既存なら最初の完全なペア内だけを置換する。複数・終了欠落・範囲不明の場合はDescriptionを上書きせず `BLOCKED`。

```markdown
<!-- codex:linear-issue-plan:start -->
## Implementation Plan

<!-- planner output starts -->
（PlannerのPlan）
<!-- planner output ends -->
<!-- codex:linear-issue-plan:end -->
```

初回PlanおよびCycle 1–2のREVISE/REPLAN後の修正版Planを含む、すべてのDescription保存に次を共通適用する。

1. 保存直前にIssueを再取得し、固定baselineのDescriptionとStatusに一致することを確認。不一致・取得不能は `BLOCKED`
2. 最新Descriptionを基礎にマーカー外を保持し、Plan本文だけをマージして保存
3. 保存試行（初回・再試行を問わず）直後に再取得する。意図した完全Description、Plan本文、マーカー外保持、保存前の期待Statusをすべて確認できた場合だけ成功とし、完全な再取得値を次baselineへ進める
4. 初回試行が未達で、再取得値が保存前baselineと完全一致する場合だけ1回再試行する。再試行前はbaselineを進めない。再試行後の未達、差分、取得不能、検証不能はbaseline未更新の `BLOCKED` とし、再々試行しない

初回保存の確認後だけStatusを `In Plan Review` へ更新する。すべてのStatus更新は前後にIssueを再取得して期待値を確認し、Status不一致・保存確認不能は `BLOCKED`。

### Review

Reviewerは次の1つだけを返す。

- `APPROVE`: 親Agentが最終状態 `APPROVED` へ変換し、`Test Implementation` へ進む
- `REVISE`: Planを修正する
- `REPLAN`: Planを再作成する

レビューは要求、codebase、範囲、依存関係、受入条件、失敗経路、未確認事項、Spike modeの暫定判断を確認する。好みや任意の改善だけで `REVISE`/`REPLAN` にしない。情報・方針不足はReviewer判定ではなく親Agentの `PLAN_BLOCKED` とする。

判定Commentは保存直前に全ページを取得してID baselineを固定し、次の完全本文を保存する。

```text
Issue ID: <issue-identifier>
Cycle n/3
Decision: APPROVE|REVISE|REPLAN
Findings: reviewerの指摘全文
```

正常応答でも返却ID・本文を全ページ再取得する。不明応答はbaseline外かつ完全本文一致の新規Commentが1件だけの場合に限り成功。それ以外は再投稿・Status更新せず `BLOCKED`。

Comment確認後、Issueが `In Plan Review` であることを再取得確認する。`REVISE`/`REPLAN` はStatusを `Todo` へ、`APPROVE` は `Test Implementation` へ、いずれも更新前後に期待値を再取得確認する。Cycle 1–2だけ同じPlannerを再利用してDescription保存共通プロトコルから再開する。Cycle 3は `REVIEW_LIMIT_REACHED` として停止し、Planner・Reviewerを起動しない。Agentが利用不能な場合のdefault置換はCycle 1–2だけ、実効Luna/xhighまたはSol/highを確認できる場合に限る。それ以外は `BLOCKED`。

## 終了報告

`APPROVED`、`PLAN_BLOCKED`、`REVIEW_LIMIT_REACHED`、`BLOCKED` のいずれかを報告する。Issue ID/title、実効Agentとモデル・推論設定、各Cycle判定、Description/Comment/Statusの再取得結果、最終Plan、未確認事項を含める。実Linear等を実行していない場合は `UNVERIFIED` と明記し、成功扱いしない。
