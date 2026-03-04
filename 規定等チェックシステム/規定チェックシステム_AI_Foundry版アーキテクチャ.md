# 規定チェックシステム アーキテクチャ設計書（Azure AI Foundry 版）

> 本書は [Functions版アーキテクチャ](./規定チェックシステム_アーキテクチャ設計書.md) の **代替パターン** として、  
> **Azure AI Foundry** を中核に据えた構成を提案するものです。

---

## 1. Azure AI Foundry とは

Azure AI Foundry（旧 Azure AI Studio）は、AI アプリケーションの **開発・評価・運用を一元管理** するプラットフォーム。

```
┌─────────────────────────────────────────────────────────┐
│                  Azure AI Foundry                        │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ モデルカタログ │  │ Prompt Flow  │  │  評価・監視    │  │
│  │ GPT-4o      │  │ (AIパイプ    │  │ (品質メトリ   │  │
│  │ Embedding   │  │  ライン)     │  │  クス)        │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Agent Service│  │ 接続管理     │  │ コンテンツ    │  │
│  │ (AIエージェ │  │ (AI Search,  │  │ フィルター    │  │
│  │  ント構築)  │  │  Blob等)     │  │ (安全性)     │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Functions 版との根本的な違い

| 観点 | Functions 版 | AI Foundry 版 |
|------|-------------|--------------|
| AI パイプラインの構築 | コードで全ロジックを実装 | **Prompt Flow** でビジュアル / YAML 定義 |
| プロンプト管理 | コード内にハードコード or 設定ファイル | **Foundry 上で一元管理・バージョン管理** |
| RAG 構築 | 自前でインデックス作成コードを実装 | **接続設定のみ**で AI Search と統合 |
| 評価・品質管理 | 自前テストスクリプト | **組込み評価ツール**（正確性・根拠性・関連性） |
| モデル切替 | コード変更 + 再デプロイ | **Foundry UI 上で即時切替** |
| 運用監視 | Application Insights を個別設定 | **Foundry ダッシュボード**で AI 固有メトリクス |

---

## 2. PoC（パイロット版）アーキテクチャ — AI Foundry 版

> **設計方針**: AI Foundry の Prompt Flow でチェックパイプラインを構築。コード量を最小化し、プロンプトの反復改善に集中する。

### 2.1 構成図

```mermaid
flowchart TB
    subgraph ユーザー端末
        GUI["Go GUI デスクトップアプリ<br/>（Fyne / Wails）"]
    end

    subgraph Azure["Azure クラウド"]
        subgraph AI_Foundry["Azure AI Foundry プロジェクト"]
            PF_LEGAL["Prompt Flow<br/>リーガルチェック"]
            PF_CONSIST["Prompt Flow<br/>整合性チェック"]
            AOAI["Azure OpenAI Service<br/>（GPT-4o / Embedding）"]
            EVAL["評価パイプライン<br/>（品質スコアリング）"]
        end

        APIM["Azure API Management<br/>（エンドポイント統合）"]
        SEARCH["Azure AI Search<br/>（社内規定インデックス）"]
        BLOB["Azure Blob Storage<br/>（文書保管）"]
    end

    subgraph 外部API
        EGOV["e-Gov 法令API v2<br/>（法令取得）"]
    end

    GUI -- "HTTPS / ファイルアップロード" --> APIM
    APIM --> PF_LEGAL
    APIM --> PF_CONSIST

    PF_LEGAL -- "プロンプト実行" --> AOAI
    PF_LEGAL -- "法令検索" --> EGOV
    PF_LEGAL -- "ファイル保存" --> BLOB

    PF_CONSIST -- "プロンプト実行" --> AOAI
    PF_CONSIST -- "類似規定検索（RAG）" --> SEARCH

    SEARCH -. "インデックス元" .-> BLOB

    PF_LEGAL -- "結果" --> APIM
    PF_CONSIST -- "結果" --> APIM
    APIM -- "統合レポート" --> GUI
```

### 2.2 Prompt Flow パイプライン設計

リーガルチェックと整合性チェックそれぞれを **Prompt Flow** として定義します。

#### リーガルチェック フロー

```mermaid
flowchart LR
    subgraph "Prompt Flow: リーガルチェック"
        A["入力ノード<br/>（文書テキスト）"]
        B["Python ノード<br/>キーワード抽出"]
        C["Python ノード<br/>e-Gov API 呼出"]
        D["LLM ノード<br/>法令適合性チェック<br/>（GPT-4o）"]
        E["Python ノード<br/>リスクスコア算出"]
        F["出力ノード<br/>（チェック結果JSON）"]
    end

    A --> B --> C --> D --> E --> F
```

```yaml
# legal_check_flow.yaml（概念）
inputs:
  document_text:
    type: string
    description: チェック対象の規定文書テキスト
  law_categories:
    type: list
    description: 対象法令カテゴリ

nodes:
  - name: extract_keywords
    type: python
    source: extract_keywords.py
    inputs:
      text: ${inputs.document_text}

  - name: fetch_laws
    type: python
    source: fetch_laws_egov.py
    inputs:
      keywords: ${extract_keywords.output}
      categories: ${inputs.law_categories}

  - name: legal_check
    type: llm
    source: legal_check_prompt.jinja2
    inputs:
      model: gpt-4o
      document: ${inputs.document_text}
      laws: ${fetch_laws.output}

  - name: calculate_risk
    type: python
    source: calculate_risk_score.py
    inputs:
      check_result: ${legal_check.output}

outputs:
  result:
    type: object
    value: ${calculate_risk.output}
```

#### 整合性チェック フロー

```mermaid
flowchart LR
    subgraph "Prompt Flow: 整合性チェック"
        A["入力ノード<br/>（文書テキスト）"]
        B["Embedding ノード<br/>ベクトル化"]
        C["Index Lookup ノード<br/>AI Search 検索"]
        D["LLM ノード<br/>整合性分析<br/>（GPT-4o）"]
        E["Python ノード<br/>結果整形"]
        F["出力ノード<br/>（チェック結果JSON）"]
    end

    A --> B --> C --> D --> E --> F
```

### 2.3 処理フロー

```mermaid
sequenceDiagram
    actor User as 人事総務部ユーザー
    participant GUI as Go GUI アプリ
    participant API as API Management
    participant PF1 as Prompt Flow<br/>(リーガルチェック)
    participant PF2 as Prompt Flow<br/>(整合性チェック)
    participant Law as e-Gov 法令API v2
    participant Search as Azure AI Search
    participant AI as Azure OpenAI

    User->>GUI: 規定文書ファイルを選択
    GUI->>API: ファイル送信

    par リーガルチェック
        API->>PF1: 文書テキスト + 法令カテゴリ
        PF1->>PF1: キーワード抽出（Python ノード）
        PF1->>Law: 関連法令の検索・取得
        Law-->>PF1: 法令条文データ
        PF1->>AI: 文書 + 法令 → 適合性チェック（LLM ノード）
        AI-->>PF1: チェック結果
        PF1->>PF1: リスクスコア算出（Python ノード）
        PF1-->>API: リーガルチェック結果
    and 整合性チェック
        API->>PF2: 文書テキスト
        PF2->>PF2: 文書ベクトル化（Embedding ノード）
        PF2->>Search: 類似規定検索（Index Lookup ノード）
        Search-->>PF2: 関連規定文書
        PF2->>AI: 入力文書 + 既存規定 → 整合性分析（LLM ノード）
        AI-->>PF2: チェック結果
        PF2-->>API: 整合性チェック結果
    end

    API-->>GUI: 統合レポート
    GUI-->>User: 結果表示（リスク箇所ハイライト付き）
```

### 2.4 PoC 技術スタック

| レイヤー | 技術 | 選定理由 |
|---------|------|---------|
| GUI | **Go + Wails** | モダンなWebベースUI |
| API Gateway | **Azure API Management**（Consumption） | Prompt Flow エンドポイントの統合 |
| AI オーケストレーション | **Azure AI Foundry — Prompt Flow** | ノーコード/ローコードでAIパイプライン構築 |
| LLM | **Azure OpenAI Service**（GPT-4o） | Foundry から直接接続 |
| ベクトル検索 | **Azure AI Search** | Foundry の Index Lookup ノードで直接統合 |
| ストレージ | **Azure Blob Storage** | 文書保管・規定原本 |
| 法令取得 | **e-Gov 法令API v2** | Prompt Flow の Python ノードから呼出 |
| AI評価 | **Foundry 組込み評価** | プロンプト品質の自動スコアリング |

### 2.5 AI Foundry 版 PoC の利点

```
✅ AI Foundry 版ならではの利点
  ├── Prompt Flow でチェックロジックをビジュアルに構築・変更
  ├── プロンプトのバージョン管理が Foundry 上で完結
  ├── 組込み評価ツールでチェック精度を定量的に測定
  ├── RAG 構成が Index Lookup ノードで簡単に実現
  ├── モデル変更（GPT-4o → GPT-4.1 等）がUI操作のみ
  └── 非エンジニア（法務担当）もプロンプト改善に参加可能

⚠ 制約・注意点
  ├── Prompt Flow の Python ノード実行環境に制約あり
  ├── e-Gov API 呼出など外部連携はカスタムコードが必要
  ├── 複雑な分岐ロジックは Prompt Flow だけでは限界あり
  └── ランタイム（Managed Online Endpoint）の最低コストが Functions より高い
```

---

## 3. 本番アーキテクチャ — AI Foundry 版

> **設計方針**: AI Foundry を AI オーケストレーション層として中心に配置。外部連携やビジネスロジックは Container Apps が担い、両者のハイブリッドで堅牢なシステムを構築。

### 3.1 構成図

```mermaid
flowchart TB
    subgraph ユーザー端末
        GUI["Go GUI デスクトップアプリ<br/>（Wails + WebView）"]
    end

    subgraph Azure["Azure クラウド"]
        direction TB

        subgraph フロント層
            FD["Azure Front Door<br/>（WAF + CDN）"]
            APIM["Azure API Management<br/>（Standard v2）"]
        end

        subgraph 認証・セキュリティ
            AAD["Microsoft Entra ID<br/>（旧 Azure AD）"]
            KV["Azure Key Vault<br/>（シークレット管理）"]
        end

        subgraph アプリケーション層["アプリケーション層（Go API）"]
            ACA["Azure Container Apps<br/>（メインAPI / Go）"]
            SB["Azure Service Bus<br/>（非同期ジョブキュー）"]
        end

        subgraph AI_Foundry["Azure AI Foundry プロジェクト"]
            PF_LEGAL["Prompt Flow<br/>リーガルチェック<br/>（Managed Endpoint）"]
            PF_CONSIST["Prompt Flow<br/>整合性チェック<br/>（Managed Endpoint）"]
            AGENT["AI Foundry Agent Service<br/>（マルチステップ推論）"]
            AOAI["Azure OpenAI Service<br/>（GPT-4o / Embedding）"]
            DI["Azure AI Document Intelligence<br/>（文書構造解析）"]
            EVAL["評価 & モニタリング<br/>（品質ダッシュボード）"]
        end

        subgraph 検索・データ層
            SEARCH["Azure AI Search<br/>（ハイブリッド検索）"]
            BLOB["Azure Blob Storage<br/>（文書保管）"]
            COSMOS["Azure Cosmos DB<br/>（チェック結果・メタデータ）"]
            SQL["Azure SQL Database<br/>（規定マスタ・ユーザー管理）"]
        end

        subgraph 運用・監視
            MON["Azure Monitor"]
            AI_INSIGHTS["Application Insights"]
        end
    end

    subgraph 外部API
        EGOV["e-Gov 法令API v2"]
    end

    GUI -- "HTTPS + OAuth 2.0" --> FD
    FD --> APIM
    APIM -- "トークン検証" --> AAD
    APIM --> ACA

    ACA -- "文書アップロード" --> BLOB
    ACA -- "非同期ジョブ投入" --> SB
    ACA -- "結果永続化" --> COSMOS
    ACA -- "マスタ参照" --> SQL
    ACA -- "シークレット取得" --> KV

    SB -- "ジョブ取得" --> ACA

    ACA -- "リーガルチェック実行" --> PF_LEGAL
    ACA -- "整合性チェック実行" --> PF_CONSIST
    ACA -- "複合チェック（Agent）" --> AGENT

    PF_LEGAL --> AOAI
    PF_LEGAL --> EGOV
    PF_LEGAL --> DI

    PF_CONSIST --> AOAI
    PF_CONSIST --> SEARCH

    AGENT --> AOAI
    AGENT --> SEARCH
    AGENT --> EGOV

    SEARCH -. "インデックス元" .-> BLOB

    ACA --> AI_INSIGHTS
    AI_Foundry -. "AIメトリクス" .-> EVAL
    AI_INSIGHTS --> MON
```

### 3.2 レイヤー別の役割分担

```mermaid
flowchart LR
    subgraph "Container Apps（Go API）の責務"
        CA1["認証・認可"]
        CA2["ファイル受付・保存"]
        CA3["ジョブ管理（Service Bus）"]
        CA4["結果の永続化（Cosmos DB）"]
        CA5["Prompt Flow エンドポイント呼出"]
        CA6["レスポンス統合・整形"]
    end

    subgraph "AI Foundry の責務"
        AF1["文書構造解析<br/>（Document Intelligence）"]
        AF2["法令検索 + 適合性チェック<br/>（Prompt Flow）"]
        AF3["規定検索 + 整合性チェック<br/>（Prompt Flow）"]
        AF4["マルチステップ推論<br/>（Agent Service）"]
        AF5["チェック品質の評価<br/>（評価パイプライン）"]
        AF6["プロンプト管理<br/>（バージョン管理）"]
    end
```

### 3.3 処理フロー（本番）

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant GUI as Go GUI アプリ
    participant Auth as Entra ID
    participant API as Container Apps (Go API)
    participant Queue as Service Bus
    participant PF1 as Prompt Flow<br/>(リーガルチェック)
    participant PF2 as Prompt Flow<br/>(整合性チェック)
    participant DI as Document Intelligence
    participant Law as e-Gov 法令API v2
    participant Search as AI Search
    participant AI as Azure OpenAI
    participant DB as Cosmos DB
    participant Eval as Foundry 評価

    User->>GUI: ログイン
    GUI->>Auth: OAuth 2.0 認証
    Auth-->>GUI: アクセストークン

    User->>GUI: 規定文書アップロード
    GUI->>API: ファイル送信 + トークン
    API->>API: Blob Storage に文書保存
    API->>Queue: チェックジョブ投入
    API-->>GUI: ジョブID（進捗確認用）

    Queue->>API: ジョブ取得（Worker ロール）

    rect rgb(240, 240, 255)
        Note over API,DI: 文書前処理
        API->>PF1: 文書データ送信
        PF1->>DI: 文書構造解析（PDF→構造化テキスト）
        DI-->>PF1: 構造化テキスト + レイアウト情報
    end

    par リーガルチェック（Prompt Flow）
        rect rgb(230, 245, 255)
            PF1->>PF1: キーワード抽出（Python ノード）
            PF1->>Law: 関連法令の取得
            Law-->>PF1: 法令条文
            PF1->>AI: 文書 + 法令 → 適合性チェック（LLM ノード）
            AI-->>PF1: リーガルチェック結果
            PF1->>PF1: リスクスコア算出
            PF1-->>API: リーガルチェック結果
        end
    and 整合性チェック（Prompt Flow）
        rect rgb(255, 245, 230)
            API->>PF2: 文書テキスト
            PF2->>PF2: Embedding 生成
            PF2->>Search: ハイブリッド検索
            Search-->>PF2: 類似規定文書
            PF2->>AI: 文書 + 規定 → 整合性分析（LLM ノード）
            AI-->>PF2: 整合性チェック結果
            PF2-->>API: 整合性チェック結果
        end
    end

    API->>DB: 統合結果を保存
    API->>Eval: 結果をログ（品質評価用）

    GUI->>API: 進捗ポーリング
    API->>DB: 結果取得
    API-->>GUI: 統合レポート
    GUI-->>User: 結果表示（詳細レポート + リスクスコア）
```

### 3.4 Agent Service の活用（本番拡張）

本番フェーズでは、AI Foundry **Agent Service** を活用し、単純なチェックでは判断が難しい **複合的な問題** をマルチステップで推論します。

```mermaid
flowchart TB
    subgraph "Agent Service: 複合チェックエージェント"
        PLAN["ステップ1<br/>チェック計画の策定"]
        LEGAL["ステップ2<br/>法令チェック実行<br/>（ツール呼出）"]
        CONSIST["ステップ3<br/>整合性チェック実行<br/>（ツール呼出）"]
        CROSS["ステップ4<br/>クロスチェック<br/>法令×規定の矛盾検出"]
        REPORT["ステップ5<br/>総合レポート生成"]
    end

    PLAN --> LEGAL --> CONSIST --> CROSS --> REPORT

    LEGAL -.- T1["ツール: e-Gov API"]
    CONSIST -.- T2["ツール: AI Search"]
    CROSS -.- T3["ツール: LLM 再推論"]
```

**Agent Service の利用シーン**:

| シーン | 説明 |
|-------|------|
| 法令改正の影響分析 | 改正法令に対し、影響を受ける社内規定を自律的に洗い出し |
| 複数規定の一括チェック | 関連する複数規定をまとめてチェックし、相互矛盾を検出 |
| 改善案の自動生成 | リスク箇所に対して修正案を複数パターン提示 |

---

## 4. 本番 技術スタック

| レイヤー | 技術 | 選定理由 |
|---------|------|---------|
| GUI | **Go + Wails v2** | WebViewベースでリッチUI |
| API Gateway | **Azure Front Door + API Management** | WAF・レート制限 |
| 認証 | **Microsoft Entra ID** | SSO・RBAC |
| メインAPI | **Azure Container Apps**（Go） | ジョブ管理・結果統合・外部連携 |
| 非同期キュー | **Azure Service Bus** | 大容量文書の非同期処理 |
| **AI オーケストレーション** | **Azure AI Foundry — Prompt Flow** | **チェックパイプラインをビジュアル管理** |
| **マルチステップ推論** | **Azure AI Foundry — Agent Service** | **複合チェック・改善案生成** |
| 文書解析 | **Azure AI Document Intelligence** | Foundry 接続経由で利用 |
| LLM | **Azure OpenAI Service**（GPT-4o） | Foundry プロジェクト内でモデル管理 |
| ベクトル検索 | **Azure AI Search**（ハイブリッド） | Foundry の Index Lookup で統合 |
| ファイル保管 | **Azure Blob Storage**（GRS） | 文書保管 |
| メタデータDB | **Azure Cosmos DB** | チェック結果・ジョブ管理 |
| マスタDB | **Azure SQL Database** | 規定マスタ・ユーザー管理 |
| シークレット | **Azure Key Vault** | API キー・接続文字列管理 |
| **AI 品質管理** | **Foundry 評価パイプライン** | **チェック精度の定量評価・回帰テスト** |
| 監視 | **Application Insights + Azure Monitor** | E2E トレース |

---

## 5. AI Foundry 固有の運用機能

### 5.1 評価パイプライン

AI Foundry には組込みの評価機能があり、チェック結果の品質を **自動的かつ定量的に** 測定できます。

```mermaid
flowchart LR
    subgraph "評価パイプライン"
        TD["テストデータセット<br/>（正解付き規定文書）"]
        RUN["Prompt Flow<br/>バッチ実行"]
        METRICS["評価メトリクス算出"]
        DASH["評価ダッシュボード"]
    end

    TD --> RUN --> METRICS --> DASH

    subgraph "評価メトリクス"
        M1["Groundedness<br/>（根拠性）"]
        M2["Relevance<br/>（関連性）"]
        M3["Coherence<br/>（一貫性）"]
        M4["F1 Score<br/>（検出精度）"]
    end

    METRICS --> M1
    METRICS --> M2
    METRICS --> M3
    METRICS --> M4
```

| メトリクス | 説明 | 規定チェックでの意味 |
|-----------|------|-------------------|
| Groundedness | 回答が提供されたコンテキストに基づいているか | 法令条文の引用が正確か |
| Relevance | 回答が質問に対して関連しているか | 指摘が対象規定の内容に関連しているか |
| Coherence | 回答が論理的に一貫しているか | リスク指摘の理由が論理的か |
| F1 Score | 検出漏れ・過検出のバランス | 法令違反を見逃さず、かつ過剰指摘しないか |

### 5.2 プロンプトのバージョン管理

```
AI Foundry プロジェクト
  └── Prompt Flow: リーガルチェック
       ├── v1.0  ← 初版（PoC開始時）
       ├── v1.1  ← 労働基準法のチェック精度改善
       ├── v1.2  ← 育児介護休業法対応追加
       ├── v2.0  ← GPT-4.1 対応 + プロンプト最適化
       └── v2.1  ← 法令改正（2026年4月施行）対応 ← current
```

Functions 版ではコードデプロイが必要だったプロンプト変更が、**Foundry UI 上での操作のみ** で完了します。法務担当者がプロンプト改善に直接参加できる点が大きなメリットです。

### 5.3 コンテンツフィルター（安全性）

AI Foundry のコンテンツフィルター設定により、不適切な出力を防止します。

| フィルター | 設定 | 理由 |
|-----------|------|------|
| ヘイトスピーチ | 高 | 規定文書チェックで不要 |
| 暴力的表現 | 高 | 同上 |
| 自傷行為 | 高 | 同上 |
| 性的表現 | 高 | 同上 |
| プロンプトインジェクション | 有効 | 入力文書経由の攻撃防止 |

---

## 6. Functions 版 vs AI Foundry 版 比較

### 6.1 総合比較

| 観点 | Functions 版 | AI Foundry 版 |
|------|-------------|--------------|
| **開発スピード** | 中（全てコード実装） | **速い**（Prompt Flow でビジュアル構築） |
| **プロンプト改善** | デプロイが必要 | **UI操作のみ。非エンジニアも参加可能** |
| **評価・品質管理** | 自前実装が必要 | **組込み評価ツールで自動化** |
| **カスタマイズ性** | **高い**（コードで自由に制御） | 中（Prompt Flow の制約あり） |
| **外部API連携** | **容易**（コードで直接呼出） | Python ノード経由（やや制約あり） |
| **最低コスト** | **安い**（Consumption プラン） | やや高い（Managed Endpoint 最低料金） |
| **モデル管理** | コード変更が必要 | **UI上で即時切替** |
| **運用監視（AI固有）** | 個別設定が必要 | **ダッシュボード標準装備** |
| **スケーラビリティ** | **高い**（Functions 自動スケール） | 高い（Endpoint スケール設定） |
| **学習コスト** | Go/Python の開発経験が必要 | **Prompt Flow の学習で開始可能** |

### 6.2 コスト比較

#### PoC 月額概算

| サービス | Functions 版 | AI Foundry 版 |
|---------|-------------|--------------|
| コンピュート | Functions Consumption: ~¥1,000 | Managed Endpoint (1 instance): ~¥15,000 |
| Azure OpenAI（GPT-4o） | ~¥15,000 | ~¥15,000 |
| Azure AI Search（Basic） | ~¥12,000 | ~¥12,000 |
| Azure Blob Storage | ~¥500 | ~¥500 |
| Azure API Management | ~¥5,000 | ~¥5,000 |
| AI Foundry プロジェクト | — | ~¥0（プロジェクト自体は無料） |
| **合計** | **~¥33,500/月** | **~¥47,500/月** |

#### 本番 月額概算

| サービス | Functions 版 | AI Foundry 版 |
|---------|-------------|--------------|
| Container Apps | ~¥15,000 | ~¥15,000 |
| Prompt Flow Endpoint（2 instance） | — | ~¥30,000 |
| Functions / Agent Service | ~¥20,000 | ~¥10,000 |
| Azure OpenAI（GPT-4o） | ~¥50,000 | ~¥50,000 |
| Azure AI Search（Standard S1） | ~¥35,000 | ~¥35,000 |
| Document Intelligence | ~¥15,000 | ~¥15,000 |
| Cosmos DB | ~¥5,000 | ~¥5,000 |
| SQL Database | ~¥1,000 | ~¥1,000 |
| Blob Storage | ~¥2,000 | ~¥2,000 |
| Front Door + APIM | ~¥25,000 | ~¥25,000 |
| Key Vault | ~¥500 | ~¥500 |
| Monitor + App Insights | ~¥3,000 | ~¥3,000 |
| **合計** | **~¥171,500/月** | **~¥191,500/月** |

> AI Foundry 版は Managed Endpoint のコストが追加される分 **月額 ~¥20,000 高くなる** が、  
> プロンプト改善の工数削減・評価自動化による **人件費削減効果** を考慮すると十分にペイする。

### 6.3 推奨パターン

```mermaid
flowchart TD
    Q1{"プロンプトの<br/>反復改善が多い？"}
    Q2{"非エンジニアも<br/>プロンプト改善に参加？"}
    Q3{"外部API連携が<br/>複雑？"}
    Q4{"コストを<br/>最小化したい？"}

    Q1 -- "はい" --> AI_FOUNDRY["✅ AI Foundry 版を推奨"]
    Q1 -- "いいえ" --> Q3
    Q2 -- "はい" --> AI_FOUNDRY
    Q2 -- "いいえ" --> Q3
    Q3 -- "はい" --> FUNCTIONS["✅ Functions 版を推奨"]
    Q3 -- "いいえ" --> Q4
    Q4 -- "はい" --> FUNCTIONS
    Q4 -- "いいえ" --> HYBRID["✅ ハイブリッド版を推奨<br/>（Container Apps + AI Foundry）"]
```

**本プロジェクトの推奨**:

> 人事総務部の法務担当者がプロンプト改善に参加し、チェック精度を継続的に高めていく運用を想定する場合、**AI Foundry 版（ハイブリッド構成）を推奨** します。  
> 法令改正への迅速な対応（プロンプト更新のみでデプロイ不要）と、組込み評価ツールによる品質管理が大きなアドバンテージです。

---

## 7. PoC → 本番 移行ロードマップ（AI Foundry 版）

```mermaid
gantt
    title PoC → 本番 移行スケジュール（AI Foundry 版）
    dateFormat YYYY-MM
    axisFormat %Y/%m

    section Phase 1: PoC構築
    要件定義・設計                :a1, 2026-04, 1M
    AI Foundry プロジェクト構築   :a2, after a1, 1M
    Prompt Flow 開発（2フロー）   :a3, after a2, 2M
    Go GUI 開発                  :a4, after a1, 2M
    社内規定登録・RAG構築         :a5, after a3, 1M
    評価データセット作成・精度検証  :a6, after a5, 1M

    section Phase 2: パイロット運用
    人事総務部パイロット運用       :b1, after a6, 3M
    プロンプト改善（法務担当と共同）:b2, after a6, 3M
    評価パイプラインで精度モニタリング :b3, after a6, 3M

    section Phase 3: 本番構築
    Container Apps + Foundry 統合  :c1, after b3, 2M
    Entra ID認証・セキュリティ     :c2, after b3, 2M
    Agent Service 構築            :c3, after c1, 1M
    監視・運用基盤整備             :c4, after c3, 1M

    section Phase 4: 全社展開
    段階的部門展開                :d1, after c4, 3M
    法令カバレッジ拡大            :d2, after c4, 3M
```

---

## 8. まとめ

| 観点 | PoC版（AI Foundry） | 本番版（AI Foundry） |
|------|-------------------|-------------------|
| AI オーケストレーション | Prompt Flow（2フロー） | Prompt Flow + Agent Service |
| 構成 | Prompt Flow + APIM の最小構成 | Container Apps + Foundry ハイブリッド |
| 対象 | 人事総務部（~20名） | 全社（~数百名） |
| 処理 | 同期（Endpoint 直接呼出） | 非同期（Service Bus + Worker） |
| 認証 | APIキー | Entra ID + RBAC |
| 文書解析 | テキスト直接処理 | Document Intelligence 構造解析 |
| 品質管理 | Foundry 評価で手動実行 | 評価パイプライン自動実行 |
| プロンプト管理 | Foundry 上でバージョン管理 | 同左 + CI/CD 連携 |
| 月額 | ~¥47,500 | ~¥191,500 |
| 構築期間 | ~4ヶ月 | ~6ヶ月（PoC後） |

### どちらを選ぶべきか

| こんな場合は… | 推奨パターン |
|-------------|------------|
| コスト最優先、エンジニア主導で開発 | **Functions 版** |
| プロンプト改善を頻繁に行いたい | **AI Foundry 版** |
| 法務・総務担当者もAI改善に参加させたい | **AI Foundry 版** |
| 外部API連携が複雑、カスタムロジックが多い | **Functions 版** |
| 両方のメリットを活かしたい | **ハイブリッド版**（本書の本番構成） |
