# P02: データアーキテクトプロンプト — 全データ構造の標準化

## このプロンプトの使い方

1. 新しい Claude (claude.ai) との会話を開く
2. このプロンプト全文をコピペ
3. 末尾の「入力ファイル」に示されているファイルを添付
4. Claude の出力を受け取りoutputs/P02_output/ に保存

---

## あなたの役割

あなたはデータ駆動アプリケーションのシニアアーキテクトです。専門は：

- スケーラブルな JSON Schema 設計
- 命名規則とファイル構成の体系化
- Google Drive・Sheets を使った構造化データ管理
- 数百〜数千のコンテンツファイルを扱う運用設計

今日は、英語ダンジョンというサービスの全データ仕様の標準化を行います。
114コンテンツ(100ステージ+観測1+ミッション13)を長期運用する基盤を設計します。

---

## 背景・制約

### 扱うコンテンツ総数

| 種別 | 数 | 1個あたりの音声数 | 音声総数 |
|---|---:|---:|---:|
| コアステージ | 100 | 14 | 1,400 |
| 定点観測 | 1 | 18 | 18 |
| ミッション | 13 | 約21 | 273 |
| 合計 | 114 | | 約1,691 |

### すでに決まっている制約

1. 音声は事前生成 → Drive保存 (ElevenLabs Default voices廃止対策)
2. sho先生の声 (voice_id 9tnqrcdSuRMggTdDrs9C) は日本語のみ
3. 英語は Rachel / Adam / Bella の3声
4. GAS は既存 dungeon_audio_gen.gs を拡張
5. マニフェストは Google Sheets で管理
6. ホスティングは sho-blog.com (Xserver)

### 設計の目的

この出力は以下全てが参照する共通規格になります:
- P05 ステージDATA生成 (100回実行)
- P06 観測パッセージ生成
- P07 ミッション設計 (13回実行)
- P08 フロントエンド実装
- P09 音声バッチ拡張
- P10 録音判定AI
- P11 QA レビュー

---

## 設計すべき7項目

### 設計1: Stage DATA JSON Schema

1ステージのDATAブロックの完全な型定義。

必須フィールド全て列挙:
- voices (4種: sho/rachel/adam/bella)
- stage (id, dungeon, tier, title, titleEn, subtitle, prevUrl, nextUrl, mapUrl, bookshelfUrl)
- soundTheme (type, shape, label, tagline, tipText, shoNpcText, shoNpcAudioId)
- rooms[] (9要素)
  - roomNum, enemy{name, type, shape}, voice, en, ja, audioId, linkings[], svoc[]
- boss (name, nameJa, shape, description, voice, en, ja, audioId)
- vocab[] (4要素)
  - id, word, pos, meaning, sentence, highlight, voice, audioId, tags[]
- rewards (xpPerRoom, xpPerfect, xpBossClear, xpVictory, maxXp)
- audio (全音声のURLマッピング)
- schemaVersion, createdAt, updatedAt, reviewStatus

出力形式:
- JSON Schema (Draft 2020-12) 準拠
- TypeScript型定義も併記
- 各フィールドに解説コメント

### 設計2: Mission DATA JSON Schema

独自フィールド:
- missionType: 'sound_rule' / 'goal_oriented' / 'seasonal'
- days[] (7要素)
- prerequisite
- certificateDesign
- pricing

### 設計3: Proving Ground DATA JSON Schema

独自フィールド:
- attempt[] (ユーザーの挑戦履歴)
- ruleBreakdown[]

### 設計4: 命名規則

- ステージID: D1-S1, D10-S4
- ルームID: D1-S1-R1
- ボスID: D1-S1-BOSS
- 語彙ID: D1-S1-V1
- ミッションID: M1, M10
- 音声ファイル名: {stageId}_{key}_{voice}.mp3
- HTMLファイル名: /all/dungeon/d1/s01.html
- Driveフォルダ: sho-eigo-dungeon-audio/{stageId}/

各命名に正規表現パターン・許容文字・大文字小文字ルールを明記。

### 設計5: Drive ディレクトリ構成

    sho-eigo-dungeon/
    audio/stages/D1-S1/
    audio/proving_ground/
    audio/missions/M1/day01/
    recordings/{userId}/stages/
    recordings/{userId}/proving_ground/
    backups/
    exports/

各フォルダの作成タイミング・権限・容量上限・削除ポリシーを明記。

### 設計6: マスター管理スプレッドシート

シート構成:
1. Stages (100+行)
2. Missions (13行)
3. Vocab (400+行)
4. Audio_Manifest
5. Proving_Attempts
6. Mission_Enrollments

各シートのカラム名・型・PK・FK・バリデーションを明記。

### 設計7: API仕様 (GAS v3)

エンドポイント:
- POST /stages/generate
- POST /recordings/upload
- POST /recordings/analyze
- GET /stages/{id}/manifest
- GET /progress/{userId}
- POST /progress/{userId}
- POST /bookshelf/{userId}/add

各エンドポイントのrequest/response JSON Schema・エラー・認証・CORS。

---

## 設計判断で考慮すべき質問

- Q1: schemaVersion の更新ポリシーとマイグレーション戦略
- Q2: 多言語化の布石（将来の韓国語・中国語対応）
- Q3: オフライン対応（IndexedDB vs localStorage）
- Q4: バージョン管理（ステージ修正時の既存学習者データ）
- Q5: プライバシー（録音データのGDPR対応）
- Q6: 価格変更への対応
- Q7: コラボ学習の将来設計

YAGNI原則を適用し、無駄な複雑化を避けること。

---

## 出力ファイル構成

    outputs/P02_output/
    data_schema.json
    schemas/stage.schema.json
    schemas/mission.schema.json
    schemas/proving_ground.schema.json
    schemas/vocab.schema.json
    schemas/audio_manifest.schema.json
    typescript_types.ts
    naming_conventions.md
    drive_structure.md
    master_sheet_design.md
    api_spec.md
    design_decisions.md
    migration_guide.md

---

## 品質基準

- 100ステージ+観測+13ミッションの全データ型を網羅
- 既存 stage_v2.html の DATA がロスなく移行できる
- JSON Schema が valid
- 命名規則に例外がゼロ
- Drive容量見積もりが含まれる
- GDPR対応済み
- P05がこのスキーマだけでステージを完成させられる

---

## 入力ファイル（この会話に添付してください）

1. dungeon_master_plan.md — サービス全体設計
2. stage_v2.html — 既存のDATA構造（最も重要）
3. dungeon_audio_gen.gs — 既存のGAS実装
4. dungeon_500_design.md — 元の設計
5. P01出力 (uiux_spec.md) — UIが何を必要とするか
