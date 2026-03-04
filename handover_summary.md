# 規定チェックシステム 設計検討まとめ
## 別アカウントへの引き継ぎ用サマリー

---

## プロジェクト概要

**目的：** 総務人事部の規定文章が、法令や他の社内規定に照らして適合しているかをAIで自動チェックするシステムを構築する。

**制約・方針：**
- フロントエンド：Go製GUIツール（既存）
- バックエンド：Microsoft Azure
- RAG素材：法令取得API（e-Gov等）＋ SharePointに保管されている社内規定
- 方針：低コスト・シンプル・高精度

---

## 採用アーキテクチャ：Plan 03

**Azure AI Foundry + Prompt Flow 構成**を採用。

3案を検討した結果、以下の理由でPlan 03を選定：
- AI Foundryハブ自体が無料
- Prompt FlowのGUI管理でプロンプト改善サイクルが速い
- Foundryの評価機能でRAG精度を定量測定できる
- 単純クエリはGPT-4o-miniに落としてコスト削減できる

---

## 使用サービス一覧

| サービス | 役割 | PoC月額 |
|---|---|---|
| Azure AI Foundry | AI開発統合プラットフォーム（ハブ） | 無料 |
| Azure AI Search | Hybrid検索インデックス（Vector + BM25） | 無料（Freeプラン） |
| Azure Blob Storage | ファイル中間保管庫 | 〜¥100 |
| Azure OpenAI（GPT-4o / mini） | 判定生成AIエンジン | 〜¥500〜1,000 |
| Azure Logic Apps | データ取込自動化ワークフロー | 〜¥200〜500 |
| Prompt Flow | AI処理パイプライン（4ノード） | 無料（Foundry内） |

**PoC月額合計：¥1,000〜2,000**  
**本番月額合計：¥6,000〜10,000**（AI Search Basic移行・Managed Endpoint追加後）

---

## データフロー

### データ取込（自動化済み）

```
法令取得API
    → Logic Apps（週次スケジュール実行）
    → Blob Storage（/hourei/フォルダ）

SharePoint Online
    → Logic Apps（ファイル更新トリガー）
    → Blob Storage（/kisoku/フォルダ）

Blob Storage
    → AI Search Blob Indexer（5分おき差分検知）
    → Skillset（テキスト抽出 → チャンキング → Embedding）
    → AI Search Index（Hybrid: Vector + BM25）
```

### クエリ処理

```
Go GUI
    → Managed Endpoint（HTTPS POST）
    → Prompt Flow
        NODE01: クエリ変換（GPT-4o mini）
        NODE02: AI Search Hybrid検索（top_k=5〜10）
        NODE03: 判定生成（GPT-4o）
        NODE04: レスポンス整形
    → Go GUI（判定結果 + 根拠条文 + 修正提案）
```

---

## 重要な設計決定事項

### SharePoint連携方式
- **SharePoint直結インデクサーは使わない**
- 理由：2025年3月時点でもプレビュー版のまま。本番非推奨とMicrosoft公式が明記
- 代替：Logic Apps の SharePointコネクタ（GA済み）でファイルを取得し、Blob経由でインデックス化

### SharePoint → Blobの仕組み
- Logic AppsにSharePointコネクタが標準搭載（Graph APIの直接実装不要）
- Azure ADアプリ登録が事前に1件必要（IT管理者依頼、30〜60分）
- SharePointのファイル更新をトリガーにして自動でBlobにコピー

### PDFのOCR対応
- 対象ファイルはテキストPDF（デジタル作成）のため**OCR不要**
- AI SearchのBlobインデクサーが標準でテキスト抽出
- Word / Excel / PowerPoint / JSON / XMLも混在対応（自動判別）
- パスワードロック付きPDFのみ要注意（インデクサーがスキップする）

### Azure Functionsについて
- **今回の構成では不要**
- Managed Endpointがリクエスト受付を担当
- Logic AppsがスケジューラとSharePoint連携を担当
- AI SearchのBlobインデクサーが自動インデックス化を担当
- 唯一必要になるケース：Logic Appsで処理できない複雑なデータ変換が生じた場合のみ

### インデックス設計
- チャンキングは**条文・条番号単位**で行う（段落単位では精度が落ちる）
- メタデータフィールド必須：`source`（hourei/kisoku）、`law_name`、`article_no`、`effective_date`、`doc_title`
- **Hybrid Search（RRF）をON**：BM25とベクトル検索を統合することで精度が大幅向上

### コスト最適化
- NODE01（クエリ変換）：GPT-4o mini使用（低コスト）
- NODE03（判定生成）：GPT-4o使用（精度重視）
- AI Search Free tier：50MBまで無料、PoC規模で十分

---

## PoCと本番の差分

| 項目 | PoC | 本番 |
|---|---|---|
| データ取込 | Logic Apps自動化（済み） | 同左 |
| Go GUI接続 | Foundryテスト画面で代替 | Managed Endpoint接続 |
| AI Searchプラン | Free（50MB） | Basic（〜¥1,400/月） |
| チェック履歴 | なし | Blob or CosmosDBに永続化 |
| 認証・RBAC | 最小限 | 本番セキュリティ設計 |
| 構築期間 | 1〜2週間 | 追加2〜4週間 |

---

## PoCで検証すること

1. 関連条文が正しく検索にヒットするか（Recall）
2. 判定根拠が正確に引用されているか
3. ハルシネーション（でたらめな条文引用）が許容範囲内か
4. チャンク粒度が条文単位で正しく分割されているか

**本番化判断の目安：** 正解率80%以上、根拠条文引用ミス5%以下、1クエリ応答10秒以内

---

## PoC開始前に必要な作業

### IT管理者への依頼（先決事項）
1. Azureサブスクリプションの確認・作成
2. 開発者アカウントへのContributorロール付与（リソースグループ単位）
3. Azure ADアプリ登録（Logic Apps用SharePoint読み取り権限）
4. **Azure OpenAI利用申請（審査1〜3営業日、最初に着手すること）**

### 開発者が準備するもの
- 検証用社内規定ファイル 20〜50件
- 対応する法令データのリスト
- テストケース（入力と期待判定）10〜20件

---

## Azure契約について

- **サービス単位の個別契約は不要**。サブスクリプション1つで全サービスが使える
- 既存のMicrosoft 365テナント（会社ドメイン）にAzureサブスクリプションを紐付けて利用
- 開発者は会社のEntra IDアカウント（名前@会社.com）でAzureポータルにサインイン
- IT管理者がRBACで操作範囲を制御（PoC向けには「共同作成者」ロールを推奨）
- 課金：従量課金（Pay-As-You-Go）でPoC開始、本番化時に年間契約へ移行検討

---

## AI Foundry vs Copilot Studioの整理

| | Copilot Studio | Azure AI Foundry |
|---|---|---|
| 対象 | ビジネス担当者 | 開発者 |
| カスタマイズ性 | 低い | 高い（今回採用） |
| SharePoint連携 | 標準搭載 | Logic Apps経由で実装 |
| RAG精度制御 | ほぼ不可 | 完全制御可能 |
| 向いている用途 | 社内FAQチャットボット | 業務特化の判定システム |

今回は「条文単位チャンキング」「モデル使い分け」「判定根拠の正確な引用」が必要なためFoundryを選択。

---

## 出力済みの成果物

1. **アーキテクチャ提案書（3案）** - Plan01〜03の比較
2. **全体フロー図** - 4フェーズの詳細フロー
3. **PoC構成資料** - 本番との差分・2週間スケジュール
4. **PoC解説資料** - 非技術者向けサービス説明・コスト詳細・Q&A
5. **設計図3点セット** - フローチャート・シーケンス図・アーキテクチャ図（タブ切替HTML）
