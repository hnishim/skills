---
name: git-add-commit-push
description: Use when the user asks to stage changes, create a Git commit, and push the current branch in one action. Safely perform git add, git commit, and git push with explicit scope checks, secret detection, protected-branch safeguards, and remote-state validation.
---

# Git Add, Commit, Push

現在のGitリポジトリで、対象をステージングし、コミットし、現在のブランチをリモートへプッシュする一連の処理を1回のスキル実行で行う。各段階が成功した場合だけ次へ進み、失敗した段階以降は実行しない。

## 実行エージェント

このスキルは、カスタムエージェント `git-actions` で実行する。`git-actions` は `gpt-5.6-luna`、推論レベル `low` を使用する。

- 別のエージェントからこのスキルを呼び出した場合、Gitの状態変更を始める前に `git-actions` へ委譲する。
- 委譲できない実行環境では、`git add`、`git commit`、`git push` を実行せず、必要なエージェント名を報告して停止する。
- `agents/openai.yaml` は表示用メタデータであり、モデル選択の代替にはしない。

## 入力とステージング範囲

- ユーザーが明示したファイル・ディレクトリだけを対象にする。パスを受け取ったら `git add -- <paths>` を使う。
- ユーザーが「全変更」「すべて」などと明示した場合だけ、リポジトリ全体を対象に `git add -A -- :/` を使う。
- 対象範囲が指定されていない場合、変更候補を一覧表示して確認を求める。`git add .`、無指定の `git add -A`、暗黙の全量ステージングは行わない。
- 実行前から存在するステージ済み変更が対象範囲に含まれると明示されていない場合、その変更をコミットせず停止する。既存のステージ済み変更を勝手に解除・上書きしない。
- 対象外の変更、未追跡ファイル、削除ファイルを保持し、ステージング・コミット・削除の対象にしない。
- ユーザーが指定したコミットメッセージはそのまま使う。指定がなければ、ステージ済み差分と直近のコミット形式から、変更内容と目的を説明するメッセージを生成する。

## 実行手順

### 1. リポジトリと状態を確認する

作業ディレクトリを対象リポジトリのルートに移し、すべてのGitコマンドをそのディレクトリで直接実行する。`git -C <path>` は使わない。

読み取り専用の確認を可能な範囲で並列に行う。

```bash
git rev-parse --show-toplevel
git status --short --branch
git branch --show-current
git diff --name-status
git diff --cached --name-status
git log --oneline -5
git branch -vv
git remote -v
```

次の場合はステージング前に停止する。

- Gitリポジトリではない。
- detached HEAD である。
- merge、rebase、cherry-pick の途中である。
- 対象ファイルが存在しない、または対象範囲を特定できない。
- リポジトリルート外のパスを対象にしようとしている。

### 2. 保護対象と機密ファイルを確認する

次のファイルは自動的にコミットしない。

```text
.env
.env.*
.env.local
.env.*.local
credentials.json
secrets.*
*.pem
*.key
*.p12
config/local.*
```

対象差分に該当ファイルがあれば、ファイル名を示して停止する。ユーザーが明示的に指定しても、機密情報を含む可能性を説明し、通常の一括処理には含めない。

現在のブランチが `main`、`master`、`develop` のいずれかである場合、コミットとプッシュの直前に、対象・コミットメッセージ・プッシュ先を示して明示確認を求める。確認なしに保護ブランチへ直接反映しない。

### 3. 対象をステージする

初期状態のステージ済みファイルを記録してから、指定された範囲だけをステージする。

```bash
git add -- <explicit-paths>
```

全量が明示された場合だけ、次を使う。

```bash
git add -A -- :/
```

ステージ後に必ず確認する。

```bash
git status --short
git diff --cached --name-status
git diff --cached --stat
git diff --cached --check
```

ステージ済み差分に対象外ファイル、機密ファイル、競合マーカー、意図しない大きな変更があれば、コミット前に停止する。ステージ済みの内容を確認できない場合も停止する。

### 4. コミットする

ステージ済み差分が空なら停止する。差分と既存のコミット形式に基づき、メッセージには少なくとも以下を含める。

- 変更の要約
- 背景または目的
- 主な実装内容
- 必要に応じた設計判断・参照情報

ユーザー指定のメッセージがある場合は、内容・構成を勝手に変更しない。`--no-verify`、`--no-gpg-sign`、`--amend` は使わない。

```bash
git commit -m "<title>" -m "<body>"
```

コミット後に次を確認する。

```bash
git status --short --branch
git log -1 --oneline
```

コミットフックがファイルを変更した場合は、その変更を自動的にamendせず、状態を報告して停止する。コミットが成功した後にプッシュが失敗しても、`reset` や自動修正を行わず、コミットハッシュを報告する。

### 5. プッシュ先とリモート状態を確認する

アップストリームとリモートの先行状態を確認する。

```bash
git branch -vv
git rev-parse --abbrev-ref --symbolic-full-name '@{u}'
git log '@{u}..HEAD' --oneline
git log 'HEAD..@{u}' --oneline
```

- アップストリームが設定済みなら、そのブランチへ `git push` する。
- アップストリームがなく、`origin` が1つだけ存在するなら、現在のブランチ名を確認して `git push -u origin <branch>` を使う。
- `origin` がない、または複数の候補から選ぶ必要がある場合は停止して確認を求める。
- リモートが先行またはローカルと分岐している場合は停止し、pull/rebaseなどの判断をユーザーに委ねる。自動でpull、rebase、mergeを行わない。
- force push が必要な場合は、このスキルでは実行しない。特に `main`、`master`、`develop` へのforce pushは禁止する。

### 6. プッシュして結果を検証する

通常のfast-forwardプッシュだけを実行する。

```bash
git push
```

アップストリーム未設定時だけ、確認済みのリモートとブランチに対して次を使う。

```bash
git push -u origin <branch>
```

成功後に次を確認し、ローカルの未コミット変更と未プッシュコミットを報告する。

```bash
git status --short --branch
git log '@{u}..HEAD' --oneline
```

## GitHub CLIを使う場合

このスキルの通常処理は `git` だけで完結させる。認証、リモート情報、チェック結果などのために `gh` を使う必要が生じた場合は、sandbox・restricted execution内で実行しない。通常のmacOSユーザーコンテキストで実行し、キーチェーンを含む実際の認証状態を確認する。

```bash
gh auth status -h github.com
```

restricted executionで `The token in default is invalid` などと表示されても、それだけでトークン期限切れや再認証必要とは判断しない。先に通常コンテキストで `gh auth status -h github.com` を確認し、認証と必要スコープが有効なら再認証せずに処理を続ける。最終的なリモートプッシュも、必要に応じて同じ通常コンテキストで検証する。

## 禁止事項

- `git -C`、`git reset --hard`、`git clean`、無確認のcheckoutによる変更破棄を行わない。
- `git config`を変更しない。
- hookをスキップしない。
- force pushを行わない。
- 対象外のユーザー変更をステージ、コミット、破棄しない。
- push失敗時に自動で履歴を書き換えない。

## 完了報告

成功時は次を簡潔に報告する。

- ステージしたファイル
- コミットハッシュとメッセージ
- プッシュ先のリモート・ブランチ
- 検証した状態
- 残った未コミット変更または未確認事項

途中停止時は、実行済みの段階、停止理由、コミット済みかどうか、ユーザー変更を保持したかを明示する。

## 使用例

```text
Use $git-add-commit-push. src/app.ts と tests/app.test.ts をコミットしてpushし、メッセージは「ログイン処理を更新」にしてください。
```

```text
Use $git-add-commit-push. 全変更を確認してコミットし、現在のブランチへpushしてください。
```

参考実装: https://github.com/biwakonbu/cc-plugins/tree/main/plugins/git-actions
