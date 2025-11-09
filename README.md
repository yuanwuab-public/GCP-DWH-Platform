# GCP Data Warehouse & ML Pipeline Architecture

## アーキテクチャ概要
複数のオンプレミスデータソースを統合し、BigQueryを基盤としたクラウドDWHを設計・構築。
ETL／データクオリティパイプライン、機械学習パイプライン、アクセス制御、監視・自動化を統合した高度なデータプラットフォームアーキテクチャ。

## Mermaid 図
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
