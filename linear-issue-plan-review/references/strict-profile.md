# Strict Profile

厳格プロファイルは、ユーザーの確認・承認後にだけ適用します。軽量プロファイルの範囲を超えるリスクがあっても、自動で切り替えず、承認なしではAgent起動、レビューComment、Status変更、Plan保存を行いません。

## 開始ゲート

1. 親Agentは、厳格プロファイルの適用が必要になった根拠を示し、ユーザーの確認・承認を得ます。承認がない場合は停止し、`PLAN_BLOCKED` または `BLOCKED` として報告します。
2. 承認後、Issue、全Comments、Repository、ローカル指示、既存Planを確認し、`description_baseline`、`status_baseline`、`comments_baseline` を固定します。
3. Statusが `Backlog` または `Todo` 以外、Issue・Repository・Plan領域を一意に特定できない、または入力を取得できない場合は停止します。
4. 通常modeではIssueにない仕様を確定しません。Spike modeで置く暫定判断は、根拠、影響範囲、検証方法、変更可能性をPlanに明記します。

## Agent契約

- Plannerは `gpt-5.6-luna`/`xhigh`、Reviewerは `gpt-5.6-sol`/`high` を原則とします。
- 名前付きAgentは契約と実効設定が完全一致する場合だけ使います。不一致・不明時は `default` に各モデル・推論設定を明示します。実効設定を確認できない場合は `BLOCKED` とします。
- PlannerとReviewerは必ず別Agentとし、直列に起動します。ReviewerはPlannerの出力を独立に確認します。
- 両Agentには同じRepository root、ローカル指示、確認対象codebase、Issue、全Comments、既存Planを渡します。SubagentにLinear、ファイル、外部システムを書き込ませません。
- Agentが利用できない場合の `default` 置換はCycle 1–2だけ、実効Luna/xhighまたはSol/highを確認できる場合に限ります。それ以外は `BLOCKED` とします。

## Plan作成と保存

1. Plannerには、Issue、全Comments、確定Repository、ローカル指示、対象codebase、既存Planを渡します。Plannerは目的、範囲、要求との対応、Repositoryの根拠、実施項目、受入条件、検証、未確認事項を返します。情報・方針不足は推測せず `PLAN_BLOCKED` とします。
2. 親Agentは、最新Descriptionを基礎に同じcanonical Plan marker内だけをマージし、Descriptionを保存します。マーカー外の内容、タイトル、担当者、ラベル、関連付けは保持します。
3. 保存前後にIssueを再取得し、意図したPlan、マーカー外保持、保存前の期待Statusを確認します。成功時だけ再取得値を次のbaselineにします。初回未達かつ再取得値が保存前baselineと完全一致する場合だけ1回再試行し、それ以外は `BLOCKED` とします。
4. 初回Plan保存と再取得確認後だけ、Statusを `In Plan Review` へ更新します。Status更新前後にIssueを再取得して期待値を確認します。

## 独立Review

Reviewerは要求、Repository、Plan、範囲、依存関係、受入条件、失敗経路、検証、未確認事項、Spike modeの暫定判断を独立に確認します。好みや任意の改善だけで判定を下げず、次の1つだけを返します。

- `APPROVE`: Planを承認できる
- `REVISE`: 同じPlan領域の局所修正が必要
- `REPLAN`: baselineを安全にrefineできない重大な矛盾がある

情報・方針不足はReviewer判定ではなく、親Agentの `PLAN_BLOCKED` とします。

## Review CommentとCycle

判定Commentは保存直前に全ページを取得してID baselineを固定し、次の完全本文で保存します。

```text
Issue ID: <issue-identifier>
Cycle n/3
Decision: APPROVE|REVISE|REPLAN
Findings: reviewerの指摘全文
```

Comment保存後は全ページを再取得し、正常応答では返却ID・本文を、応答不明ではbaseline外の完全一致Commentが1件だけ存在することを確認します。それ以外は再投稿・Status更新をせず `BLOCKED` とします。

Comment確認後、Issueが `In Plan Review` であることを再取得確認します。`REVISE`/`REPLAN` はStatusを `Todo` へ戻し、Cycle 1–2だけ同じPlannerで同じcanonical Planを更新して再開します。修正版Planは保存・再取得確認後、別のReviewerで再確認します。`APPROVE` はStatusを `Test Implementation` へ進めます。

Cycle 3の非承認は `REVIEW_LIMIT_REACHED` として停止し、Planner・Reviewerを起動せず、Planを勝手に承認しません。

## 終了報告

`APPROVED`、`PLAN_BLOCKED`、`REVIEW_LIMIT_REACHED`、`BLOCKED` のいずれかを報告します。Issue ID/title、実効Agent・モデル・推論設定、各Cycle判定、Description/Comment/Statusの再取得結果、最終Plan、未確認事項を含めます。実Linearやクラウド側を実行していない場合は `UNVERIFIED` と明記し、成功扱いしません。
