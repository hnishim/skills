---
name: linear-issue-plan-review
description: "指定された Linear Issue の initial-plan をRepository上の事実で検証・具体化し、既存Planがなければ新規Planを作成して保存する。"
---

# Linear Issue Plan Review

指定IssueをSource of TruthとしてRepositoryを確認し、initial-plan由来の既存Planがあれば同じPlanをRepository-awareにrefineします。既存Planがなければ新規Planを作成します。承認済みPlanの実装は扱いません。

## プロファイル選択

このスキルを呼び出しただけでは、厳格プロファイルを選びません。IssueとRepositoryから、単一ユーザーのローカル用途、短いスクリプト、共有サービス・本番データ・機密情報・不可逆な外部副作用がないと確認でき、ユーザーが「厳格」「レビュー」「独立レビュー」「テスト先行」「本番」「共有」「公開」「セキュリティ」などを明示していない場合は軽量プロファイルを使います。

それ以外は厳格プロファイルを適用する可能性があるため、実行前にユーザーへ厳格プロファイルを適用するか確認してください。

- 厳格プロファイルはユーザーの確認・承認なしには実行しません
- 厳格プロファイルを適用することが確認・承認されたら、実行前に [references/strict-profile.md](references/strict-profile.md) を読みます

### 軽量プロファイル（標準）

- 親AgentでIssue、既存Plan、Repositoryを確認し、既存Planがあればbaselineとしてrefineし、なければ新規Planを作る
- PlanはIssueの目的に直接対応する範囲に絞り、不要な要件、抽象化、依存関係、リスク項目を追加しない
- DescriptionのPlan領域だけを更新し、外側の内容を保持する
- 親Agentは別Agentの軽量Reviewerを1つ起動し、Planの要求適合、範囲、Repositoryの根拠、最小限の検証を読み取り専用で確認させる。ReviewerにLinear、ファイル、外部システムを書き込ませない
- Reviewerの確認は1回とし、具体的な問題があれば親AgentがPlan領域だけを修正して同じReviewerに1回だけ再確認させる。好みや任意の改善はブロッカーにしない
- レビューComment、Cycle管理、Status変更は行わない
- Issueから共有・本番・外部副作用・機密情報・権限・データ損失のリスクが判明したら、厳格プロファイルへ自動で切り替えず、ユーザーの確認・承認を求めて停止するか `PLAN_BLOCKED` とする

### 厳格プロファイル

ユーザーの確認・承認後に限り、[references/strict-profile.md](references/strict-profile.md) のPlanner、Reviewer、レビューComment、Status遷移、最大3 Cycleの契約を適用します。

## 共通契約

- 開始対象はStatusが `Backlog` または `Todo` のIssue。`Backlog` はcanonical Planなしの新規作成、`Todo` の既存PlanはRepository未検証のbaselineとして扱う
- Issue IDがない、複数ある、対象を特定できない、またはDescriptionのPlan領域を安全に特定できない場合はLinearを変更せず停止する
- Descriptionのマーカー外、タイトル、担当者、ラベル、関連付けは保持する
- 許可する外部書き込みは対象IssueのDescription、レビューComment、workflow Statusだけとし、実行するのは親Agentだけとする
- Plan本文にStatus、Review cycle、終端状態を保存しない。Workflow Statusを唯一の状態管理とする
- Linear、Agent、Repository、保存・再取得が不明・競合・失敗した場合は推測せず `BLOCKED` とする

## Issueのモード

- タイトルが `Spike:` で始まる、またはラベルに `Spike` が含まれる場合はSpike modeとする。タイトルとラベルが矛盾する場合は、タイトルとDescriptionの目的を優先し、判定理由をPlanに記載する
- 通常modeではIssueにない仕様を確定しない。Spike modeでは、検証を進めるための局所的で可逆的な事項だけ暫定判断を置いてよい
- 暫定判断は事実や確定要件として扱わず、`暫定判断`、根拠、影響範囲、検証方法、変更可能性をPlanに記載する
- Spike modeで `PLAN_BLOCKED` にするのは、外部APIへの実データ送信、本番データ、認証情報、課金、権限、セキュリティ、不可逆な変更など、Issueから安全に選べず暫定判断でも検証できない事項に限る。ホットキー、検証用モデル、ファイル配置、最小限のUI、検証用エラー表示など、検証用の可逆的な細部は合理的な暫定値を置く
- Spike modeの受入条件は完成品の品質ではなく、各検証ポイントについて成功・失敗・未検証を判定でき、採用した暫定判断、技術的制約、未実装範囲、次の判断事項を記録できることとする

## Plan handoffとmarker

Initial-planとCodex Planningは、Planの由来を別々に記録せず、同じcanonical Plan領域を段階的に更新します。Planの由来を見出し名や内容から推測しません。

LinearがHTMLコメントをMarkdownとして正規化するため、Descriptionの区切りにはHTMLコメントを使いません。新しいcanonical markerは、Linearが変換しないASCIIの単独行とします。

```text
CODEX_LINEAR_ISSUE_PLAN_START
## Implementation Plan

（initial-planまたはCodex PlanningのPlan本文）
CODEX_LINEAR_ISSUE_PLAN_END
```

- 完全なmarkerが1組ある場合だけ、そこをbaselineまたは更新対象とする
- Markerが複数ある、片側が欠落する、順序が逆、または範囲を一意に決められない場合は `BLOCKED`
- Markerがない場合はcanonical Planなしとして扱い、Descriptionの見出しからinitial-plan由来と推測しない
- 旧HTML markerは、取得結果で開始・終了境界を完全一致として確認できる場合だけ1回限り新markerへ移行する。Linear正規化後の `→` などを推測で復元しない
- 旧markerの断片や変換された終端記号が残っているのに完全なmarkerを特定できない場合は、新markerを追加せず `BLOCKED` とする
- Plan Reviewの `REVISE`/`REPLAN` でも新しいPlan領域を追加せず、同じcanonical Planを更新する

Initial-planのクラウド側Skillを修正する必要がある場合は、既存のクラウド版を最初に読み取り取得し、内容を変更せずRepositoryへ取り込んでから修正します。Repository版を検証した後、利用可能なクラウド側の書き込み経路で反映し、再取得して一致を確認します。クラウド版の取得・書き込み経路が利用できない場合は、Skillを捏造・代替作成せず `PLAN_BLOCKED` とします。

## ツールとAgent

- Issue: `linear_get_issue`、保存： `linear_save_issue`、Comment: `linear_list_comments`/`linear_save_comment`
- 軽量プロファイルは親Agentと別AgentのReviewerを使い、厳格プロファイルは別AgentのPlannerとReviewerを使う。各Agentには同じRepository root、ローカル指示、確認対象codebaseを渡す
- SubagentにLinear、ファイル、外部システムを書き込ませない。Agentのモデル・推論設定、フォールバック、終了条件は厳格プロファイルでは [references/strict-profile.md](references/strict-profile.md) に従う

## Repositoryと入力固定

1. Repositoryは、明示パス、現在workspace、workspaceから一意に決まるGit rootの順で決定する。不明・複数候補・検証不能なら書き込み前に `BLOCKED`
2. `linear_get_issue` でIssue、Description、Status、identifier、labels、project/teamを取得する。StatusがBacklog/Todo以外なら停止する。Descriptionが空の場合、通常modeでは推測で要件を補わず情報不足として停止し、Spike modeでは検証目的が取得情報から明確な場合だけ最小計画を許可する
3. `linear_list_comments` をCursorで最後まで取得する。取得不能、重複識別不能、またはbaselineを固定できない場合は停止する
4. 開始時に `description_baseline`、`status_baseline`、`comments_baseline` を固定する。Canonical marker内のPlanは未検証baselineとしてPlannerへ渡し、marker外は保持対象とする

## 実行

### Plan作成・refinement

親Agentは、Issue、全Comments、確定Repository、ローカル指示、対象codebase、既存Planを確認します。

既存Planがある場合は次を守ります。

- 既存Planを読み、Repositoryの事実で各項目を検証する
- 正しい部分は維持し、誤り・曖昧さ・不足がある箇所だけを変更する
- Repository固有のファイル、関数、変更箇所、実装順序、検証方法を既存Planの該当箇所へ追加する
- 目的・要件・制約・受入条件は、Repositoryとの矛盾または明確な不足がない限り再記述しない
- `Initial Plan` と `Repository-aware Plan` の別セクションを作らない

既存Planがない場合は、Issueの要求とRepositoryの事実から新規Planを作成します。Planには目的、範囲、要求との対応、確認済みファイル・依存関係・テスト根拠、実施項目、受入条件、検証、未確認事項を含めます。Issueから導けない仕様、互換性、エラー処理、セキュリティ方針、期限を発明しません。

### 軽量プロファイルのReview

親Agentは作成・refineしたPlan、Issue、Repositoryの根拠、検証方法、既存変更を別AgentのReviewerへ渡します。Reviewerは読み取り専用で1回確認し、要求適合、範囲、Repositoryの根拠、検証可能性、未確認事項だけを判定します。具体的な問題がある場合、親AgentはPlan領域だけを修正し、同じReviewerに1回だけ再確認させます。

Reviewerが好みや任意の改善だけを指摘した場合はブロッカーにしません。Reviewerが情報不足・方針不足・競合を示した場合は、親Agentは推測で補わず `PLAN_BLOCKED` として停止します。

### Description保存

Plan markerがない場合は、最新Descriptionの末尾にcanonical markerを1組だけ追加します。既存の場合は、最初の完全なcanonical markerの内側だけを置換します。Marker外のDescriptionは最新値を基礎に保持します。

すべてのPlan保存で次を行います。

1. 保存直前にIssueを再取得し、固定baselineのDescriptionとStatusに一致することを確認する。不一致・取得不能は `BLOCKED`
2. 最新Descriptionを基礎にmarker内だけをマージして保存する
3. 保存直後に再取得し、意図したPlan、marker外保持、保存前の期待Statusを確認する。成功時だけ再取得値を次のbaselineにする
4. 初回保存が未達で、再取得値が保存前baselineと完全一致する場合だけ1回再試行する。再試行後の未達、差分、取得不能、検証不能は `BLOCKED`

## 厳格プロファイルの実行

ユーザーの確認・承認後に限り、[references/strict-profile.md](references/strict-profile.md) を読み、そこに定めるPlanner、Reviewer、レビューComment、Status遷移、最大3 Cycleの契約を適用します。

## 終了報告

軽量プロファイルではReviewer判定、Plan保存の再取得結果、未確認事項を報告します。厳格プロファイルでは [references/strict-profile.md](references/strict-profile.md) の終了報告に従います。実Linearやクラウド側を実行していない場合は `UNVERIFIED` と明記し、成功扱いしません。
