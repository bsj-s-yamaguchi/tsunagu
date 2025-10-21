# Tsunagu Architecture Overview

## 1. System Context
- [ ] 高レベルアーキテクチャ図（ユーザー、外部SaaS、KYC/Stripe連携）
- [ ] 境界と責務（Core Platform / Integrations / Analytics）
- [ ] データ分類とフロー概要（PII/決済/ログ）

## 2. Application Architecture
- [ ] サービス構成（API、Worker、AI Pipeline）
- [ ] 技術スタックと主要ライブラリ
- [ ] メッセージング／ジョブ管理（例：キュー、イベント）

## 3. Data & AI Architecture
- [ ] データレイク／DWH構成
- [ ] モデル学習・推論パイプライン
- [ ] データ品質管理とFeature Store

## 4. Integration Architecture
- [ ] Stripe Connect、KYCベンダー、媒体APIの接続方式
- [ ] Webhook処理とエラーハンドリング
- [ ] 将来のエコシステム連携ポイント

## 5. Infrastructure & DevOps
- [ ] 環境構成（Dev/Staging/Production）
- [ ] IaC・構成管理方針
- [ ] セキュリティ対策（WAF、VPC設計、Secrets管理）

## 6. Dependencies & Open Questions
- [ ] 外部サービスのSLAと制約
- [ ] 技術的課題と検討中のアーキ案
- [ ] 今後の評価タスクと担当

> 図表は別ファイルでも可。完成したアセットは `docs/06_appendices.md` からリンクしてください。
