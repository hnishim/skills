---
name: build-terminology-glossary
description: 会議ページや資料から固有名詞・専門用語・略称・表記揺れ・文字起こし誤り候補を抽出し、根拠付きのcanonical terminologyを作る場合に使う。
---

# Build Terminology Glossary

対象資料内の情報から、固有名詞・専門用語・略称・表記揺れ・文字起こし誤り候補を抽出し、要約や文書作成で使用する canonical terminology を集約する。文字起こしを正本として扱わず、対象資料内で確認できる根拠に基づいて正式表記を決める。

## Procedure

1. 対象範囲を確認する。
  - 対象資料のタイトル、メタデータ、本文、見出し、表、リンクを確認する。
  - AI Meeting Notes の summary、notes、transcript を確認する。
  - 添付資料、画像、スクリーンショットは、必要に応じて [$extract-file-contents](../extract-file-contents/SKILL.md) または [$ocr-screenshots](../ocr-screenshots/SKILL.md) で確認する。
  - 対象資料から直接参照されていない別資料や過去会議へ、自動的に検索範囲を広げない。ユーザーが範囲を指定した場合だけ含める。
2. 用語候補を抽出する。
  - 企業名、組織名、プロジェクト名、製品名、社内資産名。
  - 人名、部署名、役職名。
  - 遺伝子、タンパク質、標的、抗原、受容体、化合物、天然リガンド。
  - 配列名、clone名、変異、construct、format。
  - assay、実験手法、モダリティ、モデル、ツール、データセット。
  - 略称、記号、番号、同音・類似音による文字起こし誤り候補。
  - 表記揺れが技術的解釈に影響する単位・記号。
3. 同一対象を照合する。
  - 発音の類似だけで統合しない。
  - 会議の対象、周辺文脈、資料上の表記、関連説明が一致するか確認する。
  - 確定できない場合は統合せず、`要確認` とする。
4. canonical 表記を決定する。
  - 確認済みの正式表記を、原表記・原ケースのまま canonical term とする。
  - 英語表記が標準の専門用語・固有名詞は、翻訳・展開・正規化しない。
  - 数値・単位は、資料と文字起こしが一致する場合、または発言内で明確に確認できる場合のみ補正する。
5. 対象文書へ統合する。
  - 既存の `Canonical terminology` セクションがあれば更新・整理する。
  - 既存セクションがなければ、対象文書の文脈に合わせて `## Canonical terminology` を作る。
  - 単に文書末尾へ追記せず、資料・会議概要の後、AI Meeting Notes または議事本文の前に置く。
  - ユーザーが手動で確認済みとした表記は保持する。

## Evidence priority

1. 対象資料内に明示された正式データソースの値
2. 対象資料の添付・提示資料に記載された正式表記
3. 対象資料のタイトル、メタデータ、本文にタイプされた表記
4. AI Meeting Notes 内のユーザー作成 notes
5. 会議内で明瞭かつ反復された発言
6. 自動文字起こし

この優先順位は名称・表記の確認にのみ使う。資料の内容を、会議で発言・合意された内容として扱う根拠にはしない。

## Output format

<table fit-page-width="true" header-row="true">
<tr>
<td>Canonical term</td>
<td>Category</td>
<td>Variants / ASR candidates</td>
<td>Evidence</td>
<td>Status / Notes</td>
</tr>
<tr>
<td>確認済みの正式表記</td>
<td>対象種別</td>
<td>略称・表記揺れ・文字起こし候補</td>
<td>確認元の資料・本文・プロパティ</td>
<td>Confirmed / 要確認 / Transcript only、および必要な補足</td>
</tr>
</table>

## Hard constraints

- 一般語を網羅的に列挙しない。表記統一または文字起こし補正に有用な項目に絞る。
- `Variants / ASR candidates` には、実際にページ内で確認した表記だけを書く。
- 想像した誤認識候補を追加しない。
- 発音の類似だけで用語を統合しない。
- canonical term の原表記・原ケースを維持する。
- 数値・単位を裏付けなく補正しない。
- 既存の手動確認済み表記を上書きしない。
- 重複する `Canonical terminology` セクションを作らない。
- glossary は表記集約のためのものとし、会議要約そのものを書き換えない。
