# GCP Data Warehouse & ML Pipeline Architecture

## アーキテクチャ概要
複数のオンプレミスデータソースを統合し、BigQueryを基盤としたクラウドDWHを設計・構築。
ETL／データクオリティパイプライン、機械学習パイプライン、アクセス制御、監視・自動化を統合した高度なデータプラットフォームアーキテクチャ。

## アーキテクチャー
```mermaid

flowchart TB
    %% ==== ON PREMISE ==== 
    subgraph ON_PREMISE["On-Premise Environment"]
        style ON_PREMISE fill:#f0f0f0,stroke:#333,stroke-width:1px
        OPDB["Source Databases"]
        Debezium["Debezium CDC"]
        AirflowETL["ETL Pipeline"]
        AirflowDQ["Data Quality Pipeline"]

        OPDB --> Debezium
        OPDB --> AirflowETL
        OPDB --> AirflowDQ
    end

    %% ==== GCP DWH (vertical layers) ==== 
    subgraph GCP_DWH["GCP Data Warehouse Platform"]
        style GCP_DWH fill:#e8f4ff,stroke:#333,stroke-width:1px

        PubSubGCP["Pub/Sub Ingestion Channel"]

        %% Internal Data
        subgraph INTERNAL_DATA["Internal Data"]
            style INTERNAL_DATA fill:#fff3e6,stroke:#f3903d,stroke-width:1px
            InternalData["DQ Results & Internal Metrics"]
        end

        %% DWH Layers Vertical
        subgraph DWH_LAYERS["BigQuery Layers"]
            direction TB
            style DWH_LAYERS fill:#ffffff,stroke:#333,stroke-width:1px

            RawBQ["RAW Layer\nStore raw source data"]
            CleanBQ["CLEAN Layer\nCleaned, merged, basic calculations"]
            AnalyticBQ["ANALYTICS Layer\nML models & marketing analysis"]
            Share["SHARING Layer\nLooker/Tableau Reports & Exports"]
        end

        PubSubGCP --> RawBQ
        CleanBQ --> AnalyticBQ
        AnalyticBQ --> Share
        CleanBQ --> Share

        %% ==== Composer (GCP service outside BigQuery) ====
        Composer["Cloud Composer / Regular data processing & transformations"]
        RawBQ --> Composer --> CleanBQ

        %% ==== ML pipeline (GCP service outside BigQuery) ====
        mlp["ML Pipeline(Vertax AI)"]
        CleanBQ --> mlp --> AnalyticBQ

        %% ==== MONITORING ==== 
        subgraph GCP_MONITOR["Monitoring & Automation"]
            style GCP_MONITOR fill:#e6fff3,stroke:#33aa66,stroke-width:1px
            SDK["Cloud SDK Collector"]
            PubSubMon["Pub/Sub Alerts"]
            CF["Cloud Function Alert Logic"]
            SNSNotify["SNS Notify Developers"]
    
            SDK --> PubSubMon --> CF --> SNSNotify
        end
    end

    %% ==== IAM Roles (vertical) ==== 
    subgraph IAM["Access Control"]
        direction TB
        style IAM fill:#f0e8ff,stroke:#7b5fff,stroke-width:1px
        PA["Platform Admin / Dev\nFull Access All Layers"]
        BD["Business Data User\nRead DWH & Sharing"]
        BA["Business Analyst\nRead DWH & Analytics\nCreate Dashboards"]
    end

    %% IAM connections
    PA --- RawBQ
    PA --- Composer
    PA --- CleanBQ
    PA --- AnalyticBQ
    PA --- Share
 
    BD --- CleanBQ
    BD --- Share
 
    BA --- CleanBQ
    BA --- AnalyticBQ
    BA --- Share

    %% data flows
    Debezium --> PubSubGCP
    AirflowETL --> RawBQ
    AirflowDQ --> InternalData
    SDK --> InternalData

    %% monitoring reading
    RawBQ --- SDK
    Composer --- SDK
    CleanBQ --- SDK
    AnalyticBQ --- SDK
    AirflowETL --- SDK
    Debezium --- SDK

    %% === styling classes ===
    classDef userStyle fill:#ffd9d9,stroke:#ff4d4d,stroke-width:1px
    class PA,BD,BA userStyle

```

## 技術スタック

* **クラウド**: GCP (BigQuery, Pub/Sub, Cloud Composer, Vertex AI, Cloud Functions, IAM)
* **ETL / DQ**: Airflow, Python
* **ML**: Vertex AI, scikit-learn, Pandas, NumPy, Statsmodels, Prophet
* **可視化**: Looker, Tableau
* **DB**: MySQL, Oracle, Hive, BigQuery, Redshift

## 技術的アピールポイント

1. **リアルタイム & バッチ統合**

   * Debezium + Pub/Sub + Composerによるハイブリッドデータ収集
2. **スケーラブルなクラウドDWH**

   * BigQueryの垂直レイヤー設計でデータ整形・分析・共有を分離
3. **自動化データパイプライン**

   * Cloud ComposerによるETL／DQ／定期更新の完全自動化
4. **機械学習統合**

   * Vertex AIによる分析レイヤー強化
5. **アクセス制御 & セキュリティ**

   * IAMロールによる詳細権限管理、データ暗号化
6. **統合監視 & アラート**

   * Cloud SDK + Pub/Sub + Cloud Functionsでジョブ失敗・不正アクセス・負荷を監視

## データ連携フロー
### バッチETLフロー
```mermaid
flowchart TB
    %% ==== SOURCE DATABASES ====
    subgraph SOURCE_DB["Source Databases"]
        style SOURCE_DB fill:#f0f0f0,stroke:#333,stroke-width:1px
        SOURCE["MySQL"]
        ORACLE["Oracle"]
        HIVE["Hive"]
        REDSHIFT["Redshift"]
    end

    %% ==== AIRFLOW ETL PIPELINE ====
    subgraph AIRFLOW_ETL["Airflow ETL Pipeline"]
        style AIRFLOW_ETL fill:#fff3e6,stroke:#f3903d,stroke-width:1px
        OperatorMySQL["MySQL Operator"]
        OperatorOracle["Oracle Operator"]
        OperatorHive["Hive Operator"]
        OperatorRedshift["Redshift Operator"]
        RetryLogic["Retry Logic"]
        ErrorCallback["Error Callback"]
        ErrorHandlingDB["Error Handling DB"]
        ScriptCLIAPI["Run Script / CLI / API"]
        NotifyError["Notify Error (Email / Teams)"]
        AirflowGCS["Upload to GCS"]
    end

    %% ==== GCS & RAW LAYER ====
    subgraph STORAGE_RAW["GCS & RAW Layer"]
        style STORAGE_RAW fill:#e8f4ff,stroke:#333,stroke-width:1px
        GCSBucket["GCS Storage"]
        RawBQ["BigQuery RAW Layer"]
    end

    %% ==== ERROR NOTIFICATION FLOW ====
    subgraph NOTIFICATION["Error Notification & Monitoring"]
        style NOTIFICATION fill:#f0e8ff,stroke:#7b5fff,stroke-width:1px
        MailServer["Mail Server"]
        Email["Email"]
        Teams["Teams"]
        MonitorApp["Monitoring App"]
        AirflowDB["Airflow Metadata DB"]
        MobileApp["Mobile App Alert"]
    end

    %% ==== DATA FLOW ====
    SOURCE --> OperatorMySQL --> RetryLogic
    ORACLE --> OperatorOracle --> RetryLogic
    HIVE --> OperatorHive --> RetryLogic
    REDSHIFT --> OperatorRedshift --> RetryLogic

    RetryLogic --> AirflowGCS --> GCSBucket --> RawBQ
    RetryLogic -->|On Error| ErrorCallback
    ErrorCallback --> ErrorHandlingDB
    ErrorHandlingDB -->|If script exists| ScriptCLIAPI --> RetryLogic
    ErrorHandlingDB -->|If not exists| NotifyError
    NotifyError --> MailServer --> Email
    NotifyError --> MailServer --> Teams

    %% Monitoring reads Airflow DB
    AirflowDB --> MonitorApp --> MobileApp

    %% Styling
    classDef dbStyle fill:#d0f0c0,stroke:#2e7d32,stroke-width:1px
    class SOURCE,ORACLE,HIVE,REDSHIFT dbStyle
    classDef operatorStyle fill:#ffd9d9,stroke:#ff4d4d,stroke-width:1px
    class OperatorMySQL,OperatorOracle,OperatorHive,OperatorRedshift operatorStyle

```

### リアルタイムデータ連携フロー
```mermaid
flowchart TB
    %% Source DB to BigQuery RAW via CDC
    subgraph SOURCE_DB["Source Databases"]
        SOURCE["binlog"]
    end

    subgraph CDC_PIPELINE["CDC Pipeline"]
        Debezium["Debezium Connector"]
        Kafka["Kafka (optional buffer)"]
        PubSub["Google Cloud Pub/Sub"]
        RawBQ["BigQuery RAW Layer"]
    end

    SOURCE --> Debezium
    Debezium --> Kafka
    Kafka --> PubSub
    PubSub --> RawBQ

    %% Monitoring & Alerts
    subgraph MONITOR["Monitoring & Alert"]
        AirflowMonitor["Airflow / Monitoring App"]
        Mail["Email / Teams"]
        AutoRecover["Auto Recovery Script"]
    end

    Debezium --- AirflowMonitor
    Kafka --- AirflowMonitor
    PubSub --- AirflowMonitor

    AirflowMonitor --> Mail
    AirflowMonitor --> AutoRecover

```

## 監視とエラー通知システム
### アーキテクチャー
```mermaid
flowchart TB
    %% === Sources ===
    subgraph CLOUD_SERVICES["GCP Services"]
        IAM["IAM / Roles & Permissions"]
        BQ["BigQuery"]
        GCS["GCS"]
        CE["Compute Engine"]
        Composer["Cloud Composer"]
        VertexAI["Vertex AI"]
    end

    %% === Cloud Function Collector ===
    subgraph COLLECTOR["Cloud Function Collector"]
        style COLLECTOR fill:#e8f4ff,stroke:#333,stroke-width:1px
        METRICS_ANALYSIS["Metrics & Logs Analysis Script"]
        HISTORY_ANALYSIS["Historical & Performance Script"]
    end

    %% === Handler & DBs ===
    subgraph DB_LAYER["Databases"]
        ERROR_DB["Error DB\nStore irregular access/errors"]
        SCHEMA_DB["Schema DB\nStore performance history & roles"]
    end

    %% === Notifications & Actions ===
    subgraph NOTIFICATION["Notification & Action"]
        TEAMS["Teams Notification"]
        PLATFORM_MAINTAINER["Platform Maintainers"]
    end

    %% Data Flow
    IAM --> METRICS_ANALYSIS
    BQ --> METRICS_ANALYSIS
    GCS --> METRICS_ANALYSIS
    CE --> METRICS_ANALYSIS
    Composer --> METRICS_ANALYSIS
    VertexAI --> METRICS_ANALYSIS

    METRICS_ANALYSIS -->|Irregular/Error Detected| ERROR_DB
    METRICS_ANALYSIS -->|Notify / Execute Script| TEAMS

    HISTORY_ANALYSIS --> BQ
    HISTORY_ANALYSIS --> GCS
    HISTORY_ANALYSIS --> CE
    HISTORY_ANALYSIS --> Composer
    HISTORY_ANALYSIS --> VertexAI
    HISTORY_ANALYSIS --> IAM
    HISTORY_ANALYSIS --> SCHEMA_DB

    SCHEMA_DB -->|Visualize| PLATFORM_MAINTAINER
    ERROR_DB -->|Check & Investigate| PLATFORM_MAINTAINER

    %% Style
    classDef collectorStyle fill:#d1f0d1,stroke:#33aa66,stroke-width:1px
    class COLLECTOR collectorStyle

```

こちらのシステムに対する**技術的なシステム概要とアピールポイント**を整理しました。Gitや技術資料に載せる形でまとめています。

---

### システム概要

本システムは、GCP上で稼働する**統合監視・アラート通知プラットフォーム**であり、複数のGCPサービスからメトリクスやログ、IAMロール情報を収集・分析し、異常検知や履歴分析、通知・自動対応を行います。

主な構成要素：

1. **データ収集対象サービス (GCP Services)**

   * IAM / Roles & Permissions
   * BigQuery
   * GCS
   * Compute Engine
   * Cloud Composer
   * Vertex AI

2. **Cloud Function Collector**

   * サービス上で稼働するサーバレス収集・分析エンジン
   * 2つの主要スクリプトで構成：

     * **Metrics & Logs Analysis Script**

       * メトリクス、ログからリアルタイムで異常アクセスやエラーを検知
       * Handler DB定義に基づき、自動スクリプト実行やTeams通知
       * 異常情報はError DBに格納
     * **Historical & Performance Script**

       * 過去のメトリクス・ログ・ユーザーアクション履歴を収集
       * IAMロール設定の集計
       * Schema DBに格納し、可視化やプラットフォーム保守者向け分析に利用

3. **データ保存 (Databases)**

   * **Error DB:** 異常アクセスやエラーを格納
   * **Schema DB:** サービスパフォーマンス履歴やユーザー行動・ロール情報を格納

4. **通知・アクション (Notification & Action)**

   * 異常検知時は**Teams通知**
   * プラットフォーム保守者がエラーDBやSchema DBを確認
   * 可視化ダッシュボードやレポートによる分析サポート

---

### 技術的アピールポイント

* **サーバレス設計によるスケーラビリティ**

  * Cloud Functionを利用したサーバレスアーキテクチャにより、収集対象増加時にも自動スケーリング可能。

* **リアルタイム異常検知と自動対応**

  * メトリクス/ログ分析スクリプトで異常アクセスやエラーを即時検知
  * Handler DBを参照し、自動スクリプト実行や通知による迅速な対応が可能

* **履歴データの集約と分析**

  * 過去メトリクスやユーザー行動履歴をSchema DBに集約
  * パフォーマンス分析、アクセス履歴監査、ロール管理可視化に対応

* **統合通知・可視化**

  * Teams通知に加え、プラットフォーム保守者向けダッシュボードに情報集約
  * エラーDBとSchema DBにデータを格納することで、分析と対応フローを分離

* **GCPサービス連携の柔軟性**

  * IAM、BigQuery、GCS、Compute Engine、Composer、Vertex AI など複数サービスの統合監視
  * 将来的な追加サービスにも容易に対応可能

* **運用効率向上**

  * 異常検知・自動スクリプト実行・通知を統合することで、人的対応工数を削減
  * システム全体の可視化と分析により、迅速な意思決定をサポート

---

