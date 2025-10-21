# Tsunagu Project Overview

## 1. Executive Summary
- **課題**: 中小〜中堅企業は採用専任不在のため求人作成・媒体投稿・候補者管理・成果報酬決済が分断され、時間/コスト/ガバナンス負荷が高い。既存ATSは決済・KYCまでカバーできず、AI活用も説明責任の面で信頼が得づらい。
- **解決策**: Tsunaguは「AIスコアリング＋Explainability＋成果報酬決済＋マルチチャネル通知」を一つのSaaSで提供し、求人生成から採用成立・決済・監査まで自動化する。
- **ターゲットセグメント**: 従業員1〜200名のSMB、採用を兼務するスタートアップ、紹介型ビジネス（SES/人材紹介）で成果報酬管理ニーズが高い顧客。
- **差別化ポイント**:
  1. **Trust-first AI** – Explainability付きスコア、重複検知、導線設計でAIの透明性を担保。
  2. **Outcome-based Monetization** – Stripe Connectを活用した成果報酬・紹介料・API課金をワンストップで管理。
  3. **日本市場最適化** – LINE/Chatwork連携、労働法準拠のKYC/AML、PWA対応で現場運用を支援。

## 2. Product Vision & Goals
- **ビジョン**: 「採用と成果報酬のすべてを1つのAIドリブン・プラットフォームで完結させ、中小企業が最短で“雇いたい”に辿り着ける世界をつくる」。
- **2026-03-31までの主要目標**
  - MVPリリースおよびベータ顧客10社での本番運用。
  - AIマッチング精度80%以上（人事評価との一致率）。
  - 成果報酬決済自動化率95%以上、重複検知Recall 0.92以上。
  - LINE/Chatwork/SMS通知の送達成功率99%を維持。
- **成功指標 (KPI)**
  - ARR ¥120M（初年度）、月次解約率≦5%、NPS＋30（Proプラン）。
  - Onboarding期間（Day0→Day7完了率）85%以上。
  - AI提案採用率30%、重複検知MTTR＜24h、オフライン同期成功率95%以上。

## 3. Customer & Market Context
- **主要ペルソナ（`docs/08_requirements.md` セクション1）**
  - *兼務採用担当（SMB）*: 採用業務に割ける時間が少なく、テンプレ・自動化・Slack通知を求める。
  - *スタートアップCTO/採用責任者*: データドリブンな意思決定とスピードを重視。Explainability、カレンダー統合が必須。
  - *紹介エージェント*: 進捗共有と成果報酬の透明性を要求。エージェントポータル＋決済連携が価値。
- **市場トレンド（`docs/09_competitor_analysis.md`）**
  - 国内ATSは成果報酬決済やAI説明責任が未成熟。ジョブカン/採用一括かんりくんはLINEや多チャネル連携を強化中だが、決済・KYC連動は不十分。
  - マルチチャネルで候補者とやり取りする文化が定着し、LINE/Chatworkとの直結が差別化要素になる。
- **規制・コンプライアンス観点**
  - 個人情報保護法、職業安定法、割賦販売法への準拠。成果報酬の前受・返金制御にはKYC/AMLが必須。
  - Stripe Connect審査、LINE/Chatwork利用規約、AI倫理指針（公平性・説明責任）を満たすガバナンスが必要。

## 4. Solution Scope
- **コア機能モジュール（詳細は `docs/07_feature_catalog.md`）**
  | モジュール | 概要 | 代表機能 |
  | --- | --- | --- |
  | Recruiter Workspace | 求人生成・媒体配信・ダッシュボード | AI求人ドラフト、自社ページ、媒体一括配信 |
  | Candidate Intelligence | AIスコア／重複検知／タレントプール | 履歴書解析、Explainability、タグ管理 |
  | Pipeline & Communication | 選考管理とマルチチャネル通知 | カレンダー同期、LINE/SMS/Chatwork通知、PWAチェックイン |
  | Payments & Compliance | 成果報酬決済・KYC/AML | Stripeエスクロー、紹介者分配、監査証跡 |
  | Analytics & Governance | KPI/ROI/監査 | リアルタイムダッシュボード、AI監査エクスポート |
- **フェーズ別リリース計画（`docs/14_detailed_design.md §9.1`）**
  - *MVP (〜2026-03-31)*: 求人生成、AIスコア、LINE/SMS/Chatwork通知、成果報酬決済、KYC、重複検知初版、PWA。
  - *Growth (〜2026-09-30)*: エージェント発注ポータル、AMLルールエンジン、候補者ポータル高度化、Explainabilityダッシュボード。
  - *Network (〜2027-03-31)*: 離職予測、Payroll/EOR連携、マーケットプレイス、成果報酬スマートコントラクト。
- **PoC / MVP / Growth の境界**
  - *PoC重点*：AI説明責任＋KYC統合（`docs/07_feature_catalog.md` セクション7）、LINE/Twilio/Chatwork連携の実運用検証。
  - *MVPスコープ*：セクション2.2のQAマトリクスで定義されたP0機能（求人生成、AIスコア、決済、マルチチャネル通知、重複検知、PWA）。
  - *Growth以降*：紹介ネットワーク拡張、AML高度化、外部SaaS（給与/会計）とのAPIマッシュアップ。
- **非機能要件サマリー（`docs/11_nonfunctional_requirements.md`）**
  - 応答時間P95 < 1.5s、AI推論 < 5s、決済API < 800ms。
  - 稼働率99.5%、RTO 4h / RPO 1h、WCAG 2.1 AA準拠、PWAオフラインキャッシュ対応。
  - Zero Trust＋MFA＋監査ログによるセキュリティ統制。
- **アーキテクチャ依存関係**
  - 詳細は `docs/14_detailed_design.md`（アプリ構成・外部連携）と `docs/15_db_design.md`（テーブル定義）を参照。
  - Stripe Connect、KYCベンダー（Prove/Liquid）、LINE/Twilio/Chatwork API、OpenAI GPT-5、BigQuery。

## 5. Strategic Positioning
- **メッセージング**
  1. *「AIに任せても説明責任が取れる採用」*: Explainability＋監査証跡＋重複検知で人事が安心してAIを活用できる。
  2. *「成果報酬まで一気通貫」*: KYC/AML・Stripeエスクロー・紹介者ポータルを組み合わせ、支払い漏れと手続コストを削減。
  3. *「日本の現場にフィット」*: LINE/Chatwork連携、PWAオフライン、法令準拠テンプレートで導入初日から運用できる。
- **勝ち筋レビュー**
  | 勝ち筋 | 補足 |
  | --- | --- |
  | 信頼できるAI採用 | Explainabilityと人手承認フローでアカウンタビリティを担保。 |
  | 決済・報酬自動化 | Stripe Connect＋KYCで成果報酬の未払/返金リスクを低減。 |
  | マルチチャネル即応 | Slack/Chatwork/LINE/SMSの統合で候補者体験を向上し、競合より迅速。 |
- **課題・克服策**
  | 課題 | 対策 |
  | --- | --- |
  | Stripe / チャネル審査リードタイム | 事前ドキュメント整備、代替手段（メール/銀行振込）、審査状況をOpsで監視。 |
  | AIバイアス懸念 | バイアス監視ダッシュボード、Human-in-the-loop運用、透明性の高いレポート。 |
  | データ移行負荷 | CSVクレンジングツール、Runbook、CS伴走支援、移行後ダッシュボード検証。 |
- **リスクと対策（`docs/13_risk_register.md`連動）**
  - LINE/Twilio/Chatwork API制限 → フォールバックチャネルと再送制御。
  - Stripe審査遅延 → 事前書類、銀行振込バックアップ。
  - モデル精度低下 → 月次再学習、Explainabilityレビュー、CSフィードバック。

## 6. Dependencies & Next Actions
- **内部依存関係**
  | 項目 | 詳細 | Owner |
  | --- | --- | --- |
  | AIモデル開発 | スコアリング／重複検知モデルの学習・監視（`docs/05_operational_plan.md §2.3`） | Lead Data Scientist |
  | インフラ整備 | PWA・Zero Trust・監視体制（`docs/14_detailed_design.md §7`） | Engineering Manager / SRE Lead |
  | サポートRunbook | 多言語SLA、通知テンプレ、移行トレーニング（`docs/05_operational_plan.md §4`） | CX Manager |
- **外部パートナー依存**
  | パートナー | 役割 | 現状 |
  | --- | --- | --- |
  | Stripe Connect | 決済・エスクロー・分配 | 事前審査中、月次チェックイン設定済 |
  | KYCベンダー（Prove / Liquid） | eKYC・反社チェック | Sandbox接続PoC進行中 |
  | LINE/Twilio/Chatwork | コミュニケーションAPI | 開発者アカウント審査待ち、利用規約レビュー中 |
  | データ匿名化BPO | データクレンジング支援 | 候補リスト選定（`docs/10_partner_prioritization.md`） |
- **直近アクション（`docs/05_operational_plan.md §6`参照）**
  - KPIダッシュボード要件書（10/30までにData Lead）。
  - LINE/SMS/Chatworkテンプレ整備とPWA検証（11/15までにCX Ops Lead）。
  - 候補者重複検知Runbook＋CSトレーニング（11/12までにData Lead/CX Ops）。
  - 非機能測定計画策定（11/10までにSRE Lead）。

> 本概要は2025-10-20時点のWorking Draftです。更新時は `docs/00_cover.md` のステータスとリビジョンログを併せて更新してください。
